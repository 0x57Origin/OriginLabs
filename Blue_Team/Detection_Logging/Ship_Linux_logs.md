# Linux Audit Logging: Watching Linux Commands with auditd

I am going to turn on the Linux audit system, run a command, and then pull the exact record that proves it was logged. This is the Linux version of the Windows lab I made before: [Windows Audit Logging](https://0x57origin.github.io/OriginLabs/Blue_Team/Detection_Logging/Windows_Audit_Logging.html).

`auditd` = the Linux Audit Daemon.

## Setup

**Host:** Kali box

**1. Install auditd**

```bash
sudo apt install auditd
```

**2. Start it and enable it on boot**

```bash
sudo systemctl enable --now auditd
```

**3. Confirm it is running**

```bash
sudo systemctl status auditd
```

## Results

```
● auditd.service - Security Audit Logging Service
     Loaded: loaded (/usr/lib/systemd/system/auditd.service; enabled; preset: disabled)
     Active: active (running) since Sat 2026-09-19 05:55:28 EDT; 8min ago
 Invocation: 27716a94743f4dc881b1d9201dfaaa97
       Docs: man:auditd(8)
             https://github.com/linux-audit/audit-documentation
    Process: 534143 ExecStart=/usr/sbin/auditd (code=exited, status=0/SUCCESS)
   Main PID: 534152 (auditd)
      Tasks: 2 (limit: 4560)
     Memory: 640K (peak: 2.2M)
        CPU: 23ms
     CGroup: /system.slice/auditd.service
             └─534152 /usr/sbin/auditd

Sep 19 05:55:28 kali systemd[1]: Starting auditd.service - Security Audit Logging Service...
Sep 19 05:55:28 kali auditd[534152]: No plugins found, not dispatching events
Sep 19 05:55:28 kali auditd[534152]: Init complete, auditd 4.1.2 listening for events (startup state enable)
Sep 19 05:55:28 kali systemd[1]: Started auditd.service - Security Audit Logging Service.
```

---

Now the audit daemon is running, but it is not watching anything yet. We must give it a rule to work with. Same idea as the Windows lab. For this lab I want a `syscall` (system call) rule that will catch command execution in the terminal.

Syscall = when a program wants the kernel to do something it just cannot do on its own, it has to ask the kernel. That request itself is the syscall.

execve = execute. It is the system call that launches a new program, replacing the current process with the one we are running. Basically, if I have to simplify it really fast, it is this: every time you run a command in the terminal, execve is the syscall that actually starts it. That is why we will be watching execve, it will catch all the command execution.

---

## The rule

```bash
sudo auditctl -a always,exit -F arch=b64 -S execve -k command_exec
```

Break it down:

```
auditctl       = audit control. The tool that loads rules into the running auditd.
-a always,exit = add a rule; log always, at the exit of the syscall.
-F arch=b64    = filter for 64-bit programs (b64 = binary 64-bit).
-S execve      = the syscall to watch (-S = syscall).
-k command_exec = a key, your own label to search for later (-k = key).
```

Run it, then confirm the rule is loaded in there:

```bash
sudo auditctl -l
```

`-l` = list.

Results:

```
└─# sudo auditctl -l
-a always,exit -F arch=b64 -S execve -F key=command_exec
```

Auditd is now logging every program that runs. Now we can trigger it and pull the record. We will run a command:

```bash
id
```

After that we will search the audit log for the key:

```bash
sudo ausearch -k command_exec
```

`ausearch` = audit search. Searches the audit log.
`-k command_exec` = find records tagged with our key.

Now yes, that will return the event, but it will return like a million other ones too. So for this we will just look at the most recent ones then:

```bash
sudo ausearch -k command_exec | tail -20
```

`tail -20` = show only the last 20 lines. That gives you one clean record to read.

Now yes, that might help, but it already scrolled off because Kali has so many other things and syscalls going on. So the best way to search for the log will be:

```bash
whoami
sudo ausearch -k command_exec | grep -A3 whoami
```

`-A3` = show the matching line plus 3 lines after it, so you get the record, not just one line.

Results:

```
└─# sudo ausearch -k command_exec | grep -A3 whoami
type=PROCTITLE msg=audit(1789847344.787:2727): proctitle="whoami"
type=PATH msg=audit(1789847344.787:2727): item=2 name="/lib64/ld-linux-x86-64.so.2" inode=131099 dev=08:01 mode=0100755 ouid=0 ogid=0 rdev=00:00 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=PATH msg=audit(1789847344.787:2727): item=1 name="/usr/bin/whoami" inode=132348 dev=08:01 mode=0100755 ouid=0 ogid=0 rdev=00:00 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=PATH msg=audit(1789847344.787:2727): item=0 name="/usr/bin/whoami" inode=132348 dev=08:01 mode=0100755 ouid=0 ogid=0 rdev=00:00 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=CWD msg=audit(1789847344.787:2727): cwd="/home/kali/Desktop"
type=EXECVE msg=audit(1789847344.787:2727): argc=1 a0="whoami"
type=SYSCALL msg=audit(1789847344.787:2727): arch=c000003e syscall=59 success=yes exit=0 a0=7ffd970b8bf0 a1=7f0c91569f70 a2=564fc7b0ab00 a3=8 items=3 ppid=2766 pid=6940 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts1 ses=2 comm="whoami" exe="/usr/bin/whoami" subj=unconfined key="command_exec"
----
time->Sat Sep 19 15:49:05 2026
type=PROCTITLE msg=audit(1789847345.119:2728): proctitle=2F62696E2F7368002F7573722F73686172652F6B616C692D7468656D65732F78666365342D70616E656C2D67656E6D6F6E2D76706E69702E7368
--
type=EXECVE msg=audit(1789847360.163:2819): argc=4 a0="grep" a1="--color=auto" a2="-A3" a3="whoami"
type=SYSCALL msg=audit(1789847360.163:2819): arch=c000003e syscall=59 success=yes exit=0 a0=7ffd970b8b40 a1=7f0c9156a3a0 a2=564fc7b0ab00 a3=8 items=3 ppid=2766 pid=7091 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts1 ses=2 comm="grep" exe="/usr/bin/grep" subj=unconfined key="command_exec"
----
time->Sat Sep 19 15:49:20 2026
```

We will read the records from top to bottom:

- `PROCTITLE` = the command we actually typed in.
- `type=CWD ... cwd="/home/kali/Desktop"` = where you were when you ran the command.
- `type=EXECVE msg=audit(1789847344.787:2727): argc=1 a0="whoami"` = the execve call. `argc=1`, which means one argument. `a0="whoami"` = the argument was whoami.
- `exe="/usr/bin/whoami"` = the program on disk.
- `auid=1000` = who is accountable, even if it ran as root.

## So what does this prove

Auditd was off in the Kali VM, but once I wrote the execve rule it was catching every command on the box, and the record shows exactly what ran, from where, and who ran it.
