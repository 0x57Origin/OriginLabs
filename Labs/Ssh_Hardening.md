# SSH Hardening Lab: Locking Down a Linux Box

**Goal:** Turn off root login over SSH, lock accounts after repeated bad passwords, then *prove* the fix actually took effect.

**Controls this maps to (NIST SP 800-171):**
- `3.1.1` limit system access
- `3.1.8` limit unsuccessful logon attempts
- `3.1.11` terminate idle sessions

> I ran this on Kali, but I didn't want to touch my Kali settings, so I did the whole thing inside a throwaway Docker container.

---

## 1. Docker Installation

```bash
sudo apt update && sudo apt install -y docker.io
sudo systemctl enable --now docker
```

Then check it's alive:

```bash
systemctl status docker
```

You should see **`Active: active (running)`** in green. That output comes from the systemd service logs.

> **systemd** is the core system and service manager for Linux. The name comes from **system daemon**.

---

## 2. Spin Up Alpine (only a few MB)

```bash
sudo docker run -dit --name VulBuild alpine
sudo docker exec -it VulBuild sh
```

**First command:** downloads the Alpine image, creates a container called `VulBuild`, starts it, and leaves it running. The `-d` flag is what runs it in the background.

**Second command:** go into the `VulBuild` container that's already running and open a shell.

### What it looks like

```
┌──(kali㉿kali)-[~/Desktop/Docker]
└─$ sudo docker run -dit --name VulBuild alpine
[sudo] password for kali:
Unable to find image 'alpine:latest' locally
latest: Pulling from library/alpine
55afa1ecc21d: Pull complete
56dceff11b33: Download complete
f5124fb579e2: Download complete
Status: Downloaded newer image for alpine:latest
e7a56772d35ab964551832a7d3fc6efaf486e6674b9857d27b4e6fb188e08860

┌──(kali㉿kali)-[~/Desktop/Docker]
└─$ sudo docker exec -it VulBuild sh
/ # whoami
root
/ #
```

---

## 3. Install SSH on Alpine

```sh
apk update
apk add openssh openrc
```

Alpine handles services a bit differently than Debian, so:
- `openssh` is the SSH **server and client**.
- `openrc` is Alpine's service manager.

> **Note:** Alpine uses `apk`, not `apt`. Different Linux family, different package manager.

---

## 4. Add Host Keys and a User Account

This makes the container behave like a real server.

```sh
ssh-keygen -A
adduser -D randomUserName
echo 'randomUserName:RandomPassword123!' | chpasswd
echo 'root:Root123!' | chpasswd
```

**Host keys** are the SSH server's own cryptographic keys. They identify the server and secure the first handshake with any client.

`ssh-keygen -A` generates the missing host keys, one per encryption type (on modern OpenSSH that's **RSA, ECDSA, ED25519**). It only creates keys that are missing and never touches keys that already exist. If the box has an ED25519 key but no RSA key, it makes the RSA one and leaves ED25519 alone.

`adduser -D randomUserName` creates the account with **no password set** (`-D`). The account can't be logged into until a password is set, which is the next step.

`chpasswd` = **change password**. The two `echo ... | chpasswd` lines set the password for the user and for root.

<img width="590" height="162" alt="image" src="https://github.com/user-attachments/assets/ca8e8421-4e86-4745-81f3-34e7fefe530e" />

---

## 5. Break It On Purpose

Turn root login **on** so I have a finding to fix later.

```sh
sed -i 's/^#*PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
/usr/sbin/sshd
```

### Reading the `sed` command

- `sed` = **stream editor**. Find and replace text inside a file.
- `-i` = **in place**. Edit and save the file directly instead of just printing the result.
- `s` = **substitute**.
- `^#*PermitRootLogin.*` = what to find.
- `PermitRootLogin yes` = what to replace it with.

### Reading the pattern (regex)

The `/` characters are just dividers. They split the substitute command into three parts:

```
s / find / replace /
```

- `^` = start of the line
- `#*` = zero or more `#` characters (the line might be commented out as `#PermitRootLogin`, or not)
- `PermitRootLogin` = the literal text
- `.*` = anything else to the end of the line (like `no`, or `prohibit-password`)

### Start the server

```sh
/usr/sbin/sshd
```

Silence means it started. If it complains that something is missing, run `ssh-keygen -A` again.

---

## 6. Confirm the Server Is Up

```sh
netstat -tlnp | grep 22
```

- `netstat` = show network connections
- `-t` = TCP only
- `-l` = only things listening (waiting for connections)
- `-n` = show numbers, not names (port 22, not `ssh`)
- `-p` = show which program owns the port
- `| grep 22` = filter to lines with 22 (port 22 is the SSH default)

<img width="867" height="85" alt="image" src="https://github.com/user-attachments/assets/2bda259a-fd14-4f63-9b60-4c1a91a0d9c6" />

Reading the output:

- `0.0.0.0:22 ... LISTEN 32/sshd` = sshd is waiting for connections on port 22 over IPv4. `0.0.0.0` means **any address on this box**.
- `LISTEN` = the server is up and accepting.
- `32/sshd` = process ID 32, named sshd.

> **Bottom line:** the server is up, and it's reachable from the network, not just locally. That last part is exactly why root login being on is a real finding.

---

## 7. See the Finding

```sh
ssh root@localhost
```

Type in the password. Once you're in, type `exit` to leave.

### The vulnerability

I set up a Linux box and turned **on** the ability to log in as **root** from the network.

Root is the master key to the whole computer, and every hacker on the planet knows the username is always "root." So an attacker already has half the login. They only need the password.

The door to the most powerful account is open, reachable from anywhere on the network, and an attacker can guess the password as many times as they want. No limit. Guess forever.

---

## 8. Fix Part 1: Turn Off Root Login

```sh
sed -i 's/^#*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sshd -t
```

`sshd -t` is **test mode**. It checks the config file for errors and stays silent if everything is fine.

> **Always run `sshd -t` before restarting sshd on a real server.** A typo in the config can lock you out of a machine you can't walk over to.

The running sshd is still the old one, so the config change isn't live yet. Restart it:

```sh
pkill sshd          # process kill: stop the running sshd
/usr/sbin/sshd      # start fresh, reading the new "no root" config
```

### Prove it works

```sh
ssh root@localhost
```

If you enter the **correct** root password and it **still** refuses you, the fix worked. Then confirm a normal user can still get in:

```sh
ssh randomUserName@localhost
```

<img width="982" height="520" alt="image" src="https://github.com/user-attachments/assets/76d1b057-1a27-46c7-8b35-dfdaa75b8c33" />

Once logged in, `whoami` will print the user's name.

---

## 9. Fix Part 2: Stop Brute-Force Guessing

Alpine is stripped down and removes `pam_faillock` to stay tiny, so the standard PAM lockout isn't available here. The realistic option is **fail2ban**, which watches the SSH log and bans an IP after X failed tries. This is what real servers actually run.

```sh
apk add fail2ban
```

fail2ban ships with a default **config** file, but you should never edit that one directly because updates overwrite it. Instead you make your own override file, `jail.local`.

```sh
cat > /etc/fail2ban/jail.local << 'EOF'
[sshd]
enabled = true
maxretry = 3
bantime = 900
findtime = 600
EOF
```

What each line means:
- `[sshd]` = the jail for SSH
- `enabled = true` = turn it on
- `maxretry = 3` = 3 strikes
- `bantime = 900` = banned for 900 seconds (15 min)
- `findtime = 600` = the 3 strikes have to happen within 600 seconds to count

> `cat > file << 'EOF'` writes everything between the two `EOF` markers into the file. **EOF = End Of File**, just a marker word that says "the text stops here."

<img width="575" height="145" alt="image" src="https://github.com/user-attachments/assets/a9d5fd0d-3d31-40af-98c4-06734f82f266" />

Check it took:

```sh
cat /etc/fail2ban/jail.local
```

It should print back all the config you just wrote.

---

## 10. Start fail2ban (and hit the first snag)

```sh
fail2ban-server -b
```

<img width="1122" height="75" alt="image" src="https://github.com/user-attachments/assets/3461f9c9-f61b-4bc5-bcb6-6af36ba04504" />

Right away, a problem. fail2ban works by **reading an SSH log file**, but Alpine is so minimal it isn't writing SSH logs anywhere yet. On a real server that log exists by default. Here I have to point fail2ban at where the logs will live and make sshd actually write there.

### Make sshd write logs

Alpine's sshd sends logs to the system logger, but nothing is catching them. Start a logger so entries land in `/var/log/messages`:

```sh
apk add busybox-openrc
syslogd
```

`syslogd` is the small service that catches log messages and writes them to `/var/log/messages`.

### Point the jail at that file

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

> `backend = polling` = fail2ban re-checks the log file every second instead of waiting to be notified, because the notify method often doesn't work inside containers.

Now `fail2ban-server -b` should print **`Server ready`**. Check the jail loaded:

```sh
fail2ban-client status
fail2ban-client status sshd
```

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
```

---

## 11. Test It (and hit the real snag)

```sh
ssh randomUserName@localhost
```

I typed a garbage password a few times.

<img width="1012" height="142" alt="image" src="https://github.com/user-attachments/assets/87e34165-f61e-4298-b2fd-7ccd64631804" />

But `fail2ban-client status sshd` still showed all zeros. The failures weren't being counted. Time to debug.

### Check the SSH log

```sh
cat /var/log/messages | grep -i sshd
```

```
Sep 11 23:38:16 e7a56772d35a auth.info sshd-session[91]: Failed password for randomUserName from ::1 port 44056 ssh2
Sep 11 23:38:16 e7a56772d35a auth.info sshd-session[91]: Failed password for randomUserName from ::1 port 44056 ssh2
Sep 11 23:38:17 e7a56772d35a auth.info sshd-session[91]: Failed password for randomUserName from ::1 port 44056 ssh2
Sep 11 23:38:17 e7a56772d35a auth.info sshd-session[91]: Connection closed by authenticating user randomUserName ::1 port 44056 [preauth]
```

The log **is** being written. Look closely: the process name is **`sshd-session`**, not `sshd`. Newer OpenSSH split login handling into a helper process. The stock fail2ban filter is matching `sshd`, so it slips right past these lines. This is a known headache with recent OpenSSH, not something I did wrong.

---

## 12. The Debug Tool: `fail2ban-regex`

fail2ban has a built-in test command that runs a filter against a real log and tells you how many lines it caught. It changes nothing, it just tests.

```sh
fail2ban-regex /var/log/messages /etc/fail2ban/filter.d/sshd.conf
```

- First path = the log to read
- Second path = the filter to test with

```
Failregex: 0 total
...
Lines: 10 lines, 0 ignored, 0 matched, 10 missed
```

### Reading this output

- **`Failregex: 0 total`** = the filter has **zero** patterns for catching failures. That's the whole problem. The part that's supposed to match "Failed password" is empty.

I caused this earlier by writing a custom `sshd.local` override that **replaced** the filter's brain instead of extending it, wiping out the failure patterns. `10 lines read, 0 matched` confirms the filter sees the log but has no rule to recognize a failed login.

---

## 13. The `.conf` vs `.local` Lesson

To recover, delete the broken override so the stock filter comes back whole:

```sh
rm /etc/fail2ban/filter.d/sshd.local
```

> **REMEMBER: `.local`, not `.conf`.**
> `.conf` = the package's shipped default. Never delete it.
> `.local` = your own override. Safe to delete.
> This `.conf` vs `.local` split is true across tons of Linux tools, not just fail2ban.

If you slip and delete `sshd.conf` by accident, put it back with:

```sh
apk fix fail2ban
ls /etc/fail2ban/filter.d/sshd.conf   # confirm the file is back
```

---

## What I Actually Learned (honest takeaway)

The brute-force **ban never triggered in this lab**. My OpenSSH logs as `sshd-session`, and the stock fail2ban filter on this Alpine box only matches `sshd`, so the failures were never counted. That mismatch is a niche version quirk, not a core skill, and Alpine is a poor choice for this job anyway (it strips out the normal lockout tools). On a real Debian or RHEL server, fail2ban works out of the box.

The real skills this lab drilled, which carry to any server:

1. **Finding to fix to evidence.** Anyone can change a config. The paid skill is *proving* the running system enforces it.
2. **Config file vs running state.** `sshd -t` and `fail2ban-regex` check what's actually loaded, not just what I typed.
3. **`.conf` vs `.local`.** Ship-default vs override. Delete the wrong one and you break the tool.
4. **Reading a log to debug.** The `sshd-session` clue was sitting in `/var/log/messages` the whole time.

**Same control, other platforms:** on RHEL the PAM file is `/etc/pam.d/password-auth`; on Windows the same lockout is enforced with a Group Policy Object (GPO) instead of a config file.
