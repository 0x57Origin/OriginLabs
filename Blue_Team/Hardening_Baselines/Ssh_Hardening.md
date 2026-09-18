# SSH Hardening: Disabling Root Login and Enforcing Account Lockout

**Control mapping:** NIST SP 800-171, 3.1.1 (limit system access to authorized users) and 3.1.8 (limit unsuccessful logon attempts)

**Environment:** Kali host, Alpine Linux container via Docker

I am using Kali for this, but I am not going to mess with my actual Kali settings, so Docker it is. Build the box, break it on purpose, then fix it.

---

## 1. Docker Setup

```bash
sudo apt update && sudo apt install -y docker.io
sudo systemctl enable --now docker
```

Check that it came up:

```bash
systemctl status docker
```

That returns the systemd service logs. You want to see `Active: active (running)` in green.

> systemd is the core system and service manager for Linux. The name is short for "system daemon."

---

## 2. Build the Container

Alpine is only a few MB, so it downloads fast.

```bash
sudo docker run -dit --name VulBuild alpine
sudo docker exec -it VulBuild sh
```

**What each one does:**

- `docker run -dit --name VulBuild alpine` pulls the Alpine image, creates a container named VulBuild, starts it, and leaves it running. `-d` is the flag that runs it in the background.
- `docker exec -it VulBuild sh` drops you into a shell inside the container that is already running.

**What it looks like:**

```
┌──(kali㉿kali)-[~/Desktop/Docker]
└─$ sudo docker run -dit --name VulBuild alpine
[sudo] password for kali:
Unable to find image 'alpine:latest' locally
latest: Pulling from library/alpine
55afa1ecc21d: Pull complete
56dceff11b33: Download complete
f5124fb579e2: Download complete
Digest: sha256:28bd5fe8b56d1bd048e5babf5b10710ebe0bae67db86916198a6eec434943f8b
Status: Downloaded newer image for alpine:latest
e7a56772d35ab964551832a7d3fc6efaf486e6674b9857d27b4e6fb188e08860

┌──(kali㉿kali)-[~/Desktop/Docker]
└─$ sudo docker exec -it VulBuild sh
/ # whoami
root
/ #
```

---

## 3. Install SSH

```sh
apk update
apk add openssh openrc
```

Alpine handles services a little differently than Debian. `openssh` gives you both the server and the client. `openrc` is Alpine's service manager.

---

## 4. Add Host Keys and Accounts

This is what makes the container behave like a real server instead of an empty box.

```sh
ssh-keygen -A
adduser -D randomUserName
echo 'randomUserName:RandomPassword123!' | chpasswd
echo 'root:Root123!' | chpasswd
```

**Host keys** are unique cryptographic keys the SSH server uses to identify itself and secure the initial connection to clients.

- `ssh-keygen -A` creates any missing host keys, one per encryption type (on modern OpenSSH that is RSA, ECDSA, and ED25519). It will not break or overwrite keys that already exist. If the server is missing an RSA key but already has an ED25519 key, it creates only the RSA key and leaves the ED25519 key completely alone.
- `adduser -D randomUserName` creates the account with no password assigned, so nobody can log in to it until a password is set, which is the very next step.
- `chpasswd` is short for "change password." It reads `user:password` from standard input, which is why the `echo ... | chpasswd` pattern works.

<img width="590" height="162" alt="Creating host keys and user accounts in the Alpine container" src="https://github.com/user-attachments/assets/ca8e8421-4e86-4745-81f3-34e7fefe530e" />

---

## 5. Break the Server on Purpose

Turn root login on and start the SSH daemon:

```sh
sed -i 's/^#*PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
/usr/sbin/sshd
```

**Breaking down the sed command:**

| Piece | Meaning |
| --- | --- |
| `sed` | stream editor, used to find and replace inside a file |
| `-i` | in place, edit and save immediately instead of printing to the screen |
| `s` | substitute |
| `^#*PermitRootLogin.*` | what to find |
| `PermitRootLogin yes` | what to replace it with |

**Breaking down the regex.** The `/` characters are just dividers separating the three parts of the substitute command: `s / find / replace /`.

| Token | Meaning |
| --- | --- |
| `^` | start of line |
| `#*` | zero or more `#` characters, since the line might be commented out as `#PermitRootLogin` or not |
| `PermitRootLogin` | the literal text |
| `.*` | anything else at the end of the line, such as `no` or `prohibit-password` |

**Starting the server:** silence means it started. If it complains that something is broken, run `ssh-keygen -A` again.

---

## 6. Confirm the Server Is Listening

```sh
netstat -tlnp | grep 22
```

| Flag | Meaning |
| --- | --- |
| `netstat` | show network connections |
| `-t` | TCP only |
| `-l` | only things listening, meaning waiting for connections |
| `-n` | show numbers, not names (port 22, not "ssh") |
| `-p` | show which program owns it |
| `\| grep 22` | filter to lines containing 22, which is the SSH default port |

<img width="867" height="85" alt="netstat output showing sshd listening on port 22" src="https://github.com/user-attachments/assets/2bda259a-fd14-4f63-9b60-4c1a91a0d9c6" />

Reading the output: `0.0.0.0:22 ... LISTEN 32/sshd` means sshd is waiting for connections on port 22 over IPv4. `0.0.0.0` means any address on this box, not just loopback. The `32` is the process ID for sshd.

**Bottom line:** the server is up and reachable from the network, not just locally. That last part is exactly why root login being enabled is a real finding and not a theoretical one.

---

## 7. The Finding

```sh
ssh root@localhost
```

Enter the password, confirm the session opens, then `exit` back out.

**The vulnerability:** we set up a Linux box and turned on the ability to log in as root from the network. Root is the master key to the computer. Every attacker on the planet knows that account exists, so they already have half the credential pair. All that is left is guessing the password.

So the door to the main account is open, it can be reached from anywhere on the network, and there is no limit on how many times someone can guess. They can guess forever.

---

## 8. Fix 1: Turn Off Root Login Over SSH

```sh
sed -i 's/^#*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sshd -t
```

`sshd -t` is test mode. It checks the config files for errors and stays silent if nothing is wrong.

Remember that sshd is still running with the old config loaded in memory. You have to restart it for the change to take effect:

```sh
pkill sshd        # process kill
/usr/sbin/sshd    # start fresh, reading the new config
```

**Proving it works:** run `ssh root@localhost` again. If you type the correct root password and it still rejects you, the fix worked. Then confirm normal users can still get in with `ssh randomUserName@localhost`, and run `whoami` once you are inside to see the account name.

<img width="982" height="520" alt="Root login rejected while the standard user account still authenticates" src="https://github.com/user-attachments/assets/76d1b057-1a27-46c7-8b35-dfdaa75b8c33" />

---

## 9. Fix 2: Brute Force Protection

Alpine is kind of a pain here. It drops `pam_faillock` to stay tiny, so the usual lockout module is not available. The alternative is **fail2ban**, which watches the SSH log and bans an IP after a set number of failed attempts. This is what real servers run anyway.

```sh
apk add fail2ban
```

fail2ban ships with a default config, but you should not edit that file directly because updates will overwrite it. Write your own override instead, called `jail.local`:

```sh
cat > /etc/fail2ban/jail.local << 'EOF'
[sshd]
enabled = true
maxretry = 3
bantime = 900
findtime = 600
EOF
```

| Line | Meaning |
| --- | --- |
| `[sshd]` | the jail for SSH |
| `enabled = true` | turn it on |
| `maxretry = 3` | three strikes |
| `bantime = 900` | banned for 900 seconds (15 minutes) |
| `findtime = 600` | the three strikes have to happen within 600 seconds to count |

The `cat > file << 'EOF'` pattern writes everything between the two EOF markers into the file. EOF stands for end of file.

<img width="575" height="145" alt="Writing the jail.local configuration file" src="https://github.com/user-attachments/assets/a9d5fd0d-3d31-40af-98c4-06734f82f266" />

Verify it landed with `cat /etc/fail2ban/jail.local`. It should return the config you just wrote.

---

## 10. Troubleshooting: No Logs to Watch

Start the service so it actually monitors:

```sh
fail2ban-server -b
```

<img width="1122" height="75" alt="fail2ban failing to start because the log file does not exist" src="https://github.com/user-attachments/assets/3461f9c9-f61b-4bc5-bcb6-6af36ba04504" />

Right away we ran into issues. fail2ban works by reading the SSH log, but Alpine is so minimal that nothing is writing SSH logs in the first place. On a real server that log exists by default. In this container it does not, so we have to point fail2ban at a log path and make sshd actually write there.

**Give the system a logger.** Alpine's sshd sends messages to the system logger, but nothing is running to catch them:

```sh
apk add busybox-openrc
syslogd
```

`syslogd` is the small service that catches log messages and writes them to `/var/log/messages`.

**Point the jail at that file:**

```sh
cat > /etc/fail2ban/jail.local << 'EOF'
[sshd]
enabled = true
maxretry = 3
bantime = 900
findtime = 600
logpath = /var/log/messages
backend = polling
EOF
```

`backend = polling` makes fail2ban re-check the log file every second instead of waiting for a filesystem notification, because the notify method often does not work inside containers.

**Restart and confirm it loaded:**

```sh
fail2ban-server -b
fail2ban-client status
fail2ban-client status sshd
```

`fail2ban-server -b` should print `Server ready`. `fail2ban-client status` should show `Jail list: sshd, sshd-ddos`. Then the detailed view:

```
/ # fail2ban-client status sshd
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     0
|  `- File list:        /var/log/messages
`- Actions
   |- Currently banned: 0
   |- Total banned:     0
   `- Banned IP list:
/ #
```

---

## 11. Testing the Lockout

I ran `ssh randomUserName@localhost` and fed it a garbage password several times.

<img width="1012" height="142" alt="Repeated failed SSH password attempts" src="https://github.com/user-attachments/assets/87e34165-f61e-4298-b2fd-7ccd64631804" />

But `fail2ban-client status sshd` was still showing zero failures. So something in the chain is broken, and now we figure out where.

**Check the log first:**

```sh
cat /var/log/messages | grep -i sshd
```

```
Sep 11 23:38:16 e7a56772d35a auth.info sshd-session[91]: Failed password for randomUserName from ::1 port 44056 ssh2
Sep 11 23:38:16 e7a56772d35a auth.info sshd-session[91]: Failed password for randomUserName from ::1 port 44056 ssh2
Sep 11 23:38:17 e7a56772d35a auth.info sshd-session[91]: Failed password for randomUserName from ::1 port 44056 ssh2
Sep 11 23:38:17 e7a56772d35a auth.info sshd-session[91]: Connection closed by authenticating user randomUserName ::1 port 44056 [preauth]
```

The logging side is working fine. But look at the process name: the entries say **sshd-session**, not **sshd**. fail2ban is not matching because it is looking for the wrong daemon name.

**Attempted fix:** tell fail2ban to recognize the other names too.

```sh
cat > /etc/fail2ban/filter.d/sshd.local << 'EOF'
[Init]
_daemon = (?:sshd|sshd-session|sshd-auth)
EOF
```

Then `fail2ban-client reload`.

**It still did not fix it.**

---

## 12. Finding the Real Cause

fail2ban has a test command:

```sh
fail2ban-regex /var/log/messages /etc/fail2ban/filter.d/sshd.conf
```

This says: take my real log file, run the SSH filter against it, and tell me how many lines it caught. It is purely a test tool. It changes nothing.

- First path = the log to read
- Second path = the filter to test with

```
/ # fail2ban-regex /var/log/messages /etc/fail2ban/filter.d/sshd.conf

Running tests
=============

Use      filter file : sshd, basedir: /etc/fail2ban
Use         maxlines : 1
Use      datepattern : {^LN-BEG} : Default Detectors
Use         log file : /var/log/messages
Use         encoding : UTF-8


Results
=======

Prefregex: 10 total
|  ^(?P<mlfid>(?:\[\])?\s*(?:<[^.]+\.[^.]+>\s+)?(?:\S+\s+)?(?:kernel:\s?\[ *\d+\.\d+\]:?\s+)?(?:@vserver_\S+\s+)?(?:(?:(?:\[\d+\])?:\s+[\[\(]?(?:sshd|sshd-session|sshd-auth)(?:\(\S+\))?[\]\)]?:?|[\[\(]?(?:sshd|sshd-session|sshd-auth)(?:\(\S+\))?[\]\)]?:?(?:\[\d+\])?:?)\s+)?(?:\[ID \d+ \S+\]\s+)?)(?:(?:error|fatal): (?:PAM: )?)?(?P<content>.+)$
`-

Failregex: 0 total

Ignoreregex: 0 total

Date template hits:
|- [# of hits] date format
|  [10] {^LN-BEG}(?:DAY )?MON Day %k:Minute:Second(?:\.Microseconds)?(?: ExYear)?
`-

Lines: 10 lines, 0 ignored, 0 matched, 10 missed
[processed in 0.00 sec]

|- Missed line(s):
|  Sep 11 23:33:16 e7a56772d35a syslog.info syslogd started: BusyBox v1.37.0
|  Sep 11 23:33:22 e7a56772d35a syslog.info syslogd started: BusyBox v1.37.0
|  Sep 11 23:38:16 e7a56772d35a auth.info sshd-session[91]: Failed password for randomUserName from ::1 port 44056 ssh2
|  Sep 11 23:38:16 e7a56772d35a auth.info sshd-session[91]: Failed password for randomUserName from ::1 port 44056 ssh2
|  Sep 11 23:38:17 e7a56772d35a auth.info sshd-session[91]: Failed password for randomUserName from ::1 port 44056 ssh2
|  Sep 11 23:38:17 e7a56772d35a auth.info sshd-session[91]: Connection closed by authenticating user randomUserName ::1 port 44056 [preauth]
|  Sep 11 23:43:18 e7a56772d35a auth.info sshd-session[101]: Failed password for randomUserName from ::1 port 51184 ssh2
|  Sep 11 23:43:19 e7a56772d35a auth.info sshd-session[101]: Failed password for randomUserName from ::1 port 51184 ssh2
|  Sep 11 23:43:19 e7a56772d35a auth.info sshd-session[101]: Failed password for randomUserName from ::1 port 51184 ssh2
|  Sep 11 23:43:19 e7a56772d35a auth.info sshd-session[101]: Connection closed by authenticating user randomUserName ::1 port 51184 [preauth]
`-
/ #
```

The output looks scary, but the line that matters is this one:

**`Failregex: 0 total`**

The filter has zero patterns for catching failures, and that is the whole problem. The part of the filter that was supposed to match failed password lines is empty. My `sshd.local` override wiped it out instead of extending it.

**Rolling it back:**

```sh
rm /etc/fail2ban/filter.d/sshd.local
```

Note the extension: **`.local`, not `.conf`**. `sshd.conf` is the real filter. If you delete that one by accident, run `apk fix fail2ban` and it will restore it. Confirm the real filter is still there with `ls /etc/fail2ban/filter.d/sshd.conf`.

---

## The Honest Ending

The ban never actually triggered in this lab.

My OpenSSH build logs as `sshd-session`, and the stock fail2ban filter on this Alpine box only matches `sshd`, so the failed logins were never counted. My attempt to override the daemon name blanked out the failure patterns entirely, which made it worse.

That mismatch is a niche version quirk, not a real skill, and Alpine is a bad pick for this job in the first place since it strips out the normal lockout tooling. On a real Debian or RHEL server, fail2ban works out of the box.

**What did work:**

- Root login over SSH was confirmed enabled, confirmed reachable from the network, then disabled and verified as disabled (3.1.1)
- Normal user authentication was confirmed still working after the change
- SSH logging was stood up from nothing so failed attempts are recorded and reviewable

**What did not:**

- Automated lockout after failed attempts (3.1.8) was configured but never fired, for the reason above

**Next time:** run this on Debian, where `pam_faillock` and the stock fail2ban filter both work without fighting the distro.
