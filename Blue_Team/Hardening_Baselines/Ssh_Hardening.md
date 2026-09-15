# SSH Hardening: Disabling Root Login and Enforcing Account Lockout (NIST 800-171: 3.1.1, 3.1.8)
---

## I am going to use Kali for this, but I'm not really going to mess with my Kali settings, so Docker it is.

---

## Docker Installation

```
sudo apt update && sudo apt install -y docker.io
sudo systemctl enable --now docker
```

Then in the Kali terminal type **systemctl status docker** and it will give you the systemd service logs. You will see **Active: active (running)** in green.

systemd is the core system and service manager for the Linux OS. systemd -> System daemon.

---

## Alpine OS - just a few MB download.

```
sudo docker run -dit --name VulBuild alpine
sudo docker exec -it VulBuild sh
```

First command -> it will download the alpine image and create a container called VulBuild, start it, and leave it running. -d is the operator that lets you run it in the background.

Second command -> basically go into the VulBuild container that is already running and open a shell.

---

# What it will look like

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

# Alpine Commands

```
apk update
apk add openssh openrc
```

Now Alpine handles server services a tad bit differently than Debian, so we will need openssh, which is the SERVER and CLIENT. openrc is Alpine's service manager.

---

# I'm going to add the host keys and accounts so I can make it act like a real server

```
ssh-keygen -A
adduser -D randomUserName
echo 'randomUserName:RandomPassword123!' | chpasswd
echo 'root:Root123!' | chpasswd
```

Host keys? They are unique cryptographic keys that the SSH server uses to identify itself and secure the initial connection to clients.

ssh-keygen -A will create the missing host keys (one for each encryption type, which on modern OpenSSH is RSA, ECDSA, and ED25519). It will not break your working keys or mess with keys that already exist on the system. If your server is missing an RSA key but already has an ED25519 key, it will only create the missing RSA key and leave the ED25519 key completely alone.

adduser -D randomUserName -> -D stands for do not assign a password. It creates the account with no password set, so no one can log in to it until a password is set (which is the very next step).

echo 'randomUserName:randomUserPassword' | chpasswd -> chpasswd stands for Change Password.

echo 'root:123' | chpasswd -> changing the root user's password.

<img width="590" height="162" alt="image" src="https://github.com/user-attachments/assets/ca8e8421-4e86-4745-81f3-34e7fefe530e" />

---

## Now we get to break the server!!

```
Turn root login on and start the SSH server:
sed -i 's/^#*PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
/usr/sbin/sshd

Explanation:
sed - stream editor. Alpine uses sed to find and replace something in a file.
-i - in place. Edit and save it right away without printing it to the screen.
s - substitute.
^#*PermitRootLogin.* - what to find.
PermitRootLogin yes - what to replace it with.

Breakdown of regex:
The / is just a divider. It separates the three parts of the substitute command:
s / find / replace /

^ = start of line
#* = zero or more # characters (the line might be commented out as #PermitRootLogin, or not).
PermitRootLogin = the actual text
.* = anything else at the end of the line (like no, or prohibit-password)
```

# Start the server now!

```
/usr/sbin/sshd
```

Silence means it started. If it complains that something is broken, run ssh-keygen -A again.

# Now let's check with a command if our server is good or not!

```
netstat -tlnp | grep 22
```

The command:
netstat = show network connections
-t = TCP only
-l = only things listening (waiting for connections)
-n = show numbers, not names (port 22, not "ssh")
-p = show which program owns it
| grep 22 = filter to lines with 22 -> port 22 because it's the secure shell default port.

Output ->

<img width="867" height="85" alt="image" src="https://github.com/user-attachments/assets/2bda259a-fd14-4f63-9b60-4c1a91a0d9c6" />

0.0.0.0:22 ... LISTEN 32/sshd = sshd waiting on connection port 22 via IPv4. 0.0.0.0 means any address on this box!
LISTEN 32 -> 32 is the process ID for sshd.

**Bottom line: yes, the server is up, and it's reachable from the network, not just locally. That last part is why root login being on is a real finding.**

---

## Let's see our finding

```
ssh root@localhost
```

Once that happens, type in the password. After the connection is established, type exit and exit out.

The vulnerability here is that root can be easily accessed, meaning we set up a Linux box and turned ON the ability to log in as root from the network. Root is the master key to the computer. Every hacker on the planet knows that, so the hacker knows half the login information. They just have to guess the password.

So the door is open to the main login, and it can be accessed from anywhere on the network. A hacker can guess the password as many times as they want. No limit. Guess forever.

---

# HOW TO FIX IT

TURN OFF ROOT LOGIN OVER SSH

```
sed -i 's/^#*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
```

Then check with sshd -t. -t is test mode, and it will test the config files for errors. If nothing is wrong, it will stay silent.

But remember, sshd is still running in the background. You have to restart sshd for the config to take effect.

```
pkill sshd -> process kill
/usr/sbin/sshd -> start fresh, now reading the newest config we made
```

Now let's prove it works -> ssh root@localhost. If you put in the right password and it still says wrong password, the fix worked! Then we can see if we can let normal users in now -> ssh randomUserName@localhost.

<img width="982" height="520" alt="image" src="https://github.com/user-attachments/assets/76d1b057-1a27-46c7-8b35-dfdaa75b8c33" />

After that, if you type whoami it will say the user's name!

---

# Bruteforce Fix

Alpine is kind of a pain. It deleted **pam_faillock** so it can stay tiny. The best option is to use fail2ban, which will monitor the SSH log and ban an IP after X amount of failed tries. This is what real servers run.

Let's add it -> apk add fail2ban. After that, fail2ban ships with a default config file, but you should not edit it because updates will overwrite it, so we just make our own config file. We can call it -> jail.local.

```
cat > /etc/fail2ban/jail.local << 'EOF'
[sshd]
enabled = true
maxretry = 3
bantime = 900
findtime = 600
EOF
```

What each line means:
[sshd] = the jail for SSH
enabled = true = turn it on
maxretry = 3 = 3 strikes
bantime = 900 = banned for 900 seconds (15 min)
findtime = 600 = the 3 strikes have to happen within 600 seconds to count

That cat > file << 'EOF' thing writes everything between the two EOF markers into the file. EOF = END OF FILE.

<img width="575" height="145" alt="image" src="https://github.com/user-attachments/assets/a9d5fd0d-3d31-40af-98c4-06734f82f266" />

After that, check if it worked or not -> cat /etc/fail2ban/jail.local. It should return all the new config we made.

# Now start fail2ban so it actually watches: fail2ban-server -b

<img width="1122" height="75" alt="image" src="https://github.com/user-attachments/assets/3461f9c9-f61b-4bc5-bcb6-6af36ba04504" />

See, right away we ran into issues. fail2ban works by reading the SSH log, but in Alpine, since it's so small, it's not writing SSH logs. We have to make it write logs. On a real server that log exists by default. In this minimal container it doesn't, so we have to point fail2ban at where the logs will be and make sshd actually write there.

### Tell sshd to write logs -> Alpine's sshd logs to the system logger, but nothing is there to catch it, so we can start a logger and sshd fresh so entries land in -> /var/log/messages

```
apk add busybox-openrc
syslogd
```

syslogd is the little service that catches log messages and writes them to /var/log/messages.

### Then point the jail at that file. Add a logpath line so fail2ban knows where to look:

```
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

backend = polling = fail2ban re-checks the log file every second instead of waiting to be notified, because the notify method often doesn't work in containers.

Now we can run fail2ban-server -b again to see what it says, and it should spit out Server ready.

We can also run fail2ban-client status to really see if it loaded, and it should say Jail list: sshd, sshd-ddos. We can see the sshd, so we're good.

Then to truly see the sshd, type -> fail2ban-client status sshd. It should show you everything.

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

### We can test and see if it works or not!

I put -> ssh randomUserName@localhost and typed in a garbage password. Look at the image!

<img width="1012" height="142" alt="image" src="https://github.com/user-attachments/assets/87e34165-f61e-4298-b2fd-7ccd64631804" />

But when I put fail2ban-client status sshd, it is not counting the failures. Now we have to figure out why.

### Let's check the SSHD logs!

```
cat /var/log/messages | grep -i sshd

Sep 11 23:38:16 e7a56772d35a auth.info sshd-session[91]: Failed password for randomUserName from ::1 port 44056 ssh2
Sep 11 23:38:16 e7a56772d35a auth.info sshd-session[91]: Failed password for randomUserName from ::1 port 44056 ssh2
Sep 11 23:38:17 e7a56772d35a auth.info sshd-session[91]: Failed password for randomUserName from ::1 port 44056 ssh2
Sep 11 23:38:17 e7a56772d35a auth.info sshd-session[91]: Connection closed by authenticating user randomUserName ::1 port 44056 [preauth]
```

The log is good, but fail2ban isn't working. Now if you look at the log, you will see that instead of sshd it says sshd-session, and that is the problem. It's not looking at the right name. So now tell fail2ban to also recognize sshd-session.

```
cat > /etc/fail2ban/filter.d/sshd.local << 'EOF'
[Init]
_daemon = (?:sshd|sshd-session|sshd-auth)
EOF
```

Then reload it -> fail2ban-client reload.

---

# Well, it still did not fix it.

---

## Fail2ban has a test command -> fail2ban-regex /var/log/messages /etc/fail2ban/filter.d/sshd.conf

So here it says -> take my real log file, run the SSH filter against it, and tell me how many lines it caught. It's a test tool. It changes nothing, it just checks if the filter works.

First path = the log to read
2nd path = the filter to test with

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

THE OUTPUT LOOKS SCARY, BUT:

-> Failregex: 0 total = the filter has zero patterns for catching failures, and that is the whole problem. The part of the filter that was supposed to catch the FAILED PASSWORDS is empty. Our override file erased it.

# Override FIX

First we remove it -> rm /etc/fail2ban/filter.d/sshd.local ->>> REMEMBER LOCAL not CONF. sshd.conf is the real filter. If you happen to delete the conf, just type -> apk fix fail2ban and it will auto-fix it. Then we can do ls /etc/fail2ban/filter.d/sshd.conf and see if it lists the file, and it does.

---

# The honest ending

The ban never actually triggered in this lab. My OpenSSH logs as sshd-session, and the stock fail2ban filter on this Alpine box only matches sshd, so my failed logins were never counted. That mismatch is a niche version quirk, not a real skill, and Alpine is a bad pick for this job anyway since it strips out the normal lockout tools. On a real Debian or RHEL server, fail2ban works out of the box.

What I actually learned, and what carries to any server:

1. Finding to fix to evidence. Anyone can change a config. The real skill is proving the running system enforces it.
2. Config file vs running state. sshd -t and fail2ban-regex check what is actually loaded, not just what I typed.
3. .conf vs .local. .conf is the shipped default (never delete it), .local is your override. Delete the wrong one and you break the tool.
4. Reading a log to debug. The sshd-session clue was sitting in /var/log/messages the whole time.
