# Python Log Parser Lab 
---
Execve = This is the Linux kernel call that starts a program; auditd records the binary, args, user, and pid every time it happens.
---

Real outPut from auditd:
```
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

I will be breaking down code lines here too in the Python code in the field helper function we have a line: type_match = re.search(r"type=(\w+)", line) 
So what that will capture is:
```
type=PROCTITLE-> \w+ grabs PROCTITLE

type=PATH-> \w+ grabs PATH

type=CWD-> \w+ grabs CWD

type=EXECVE-> \w+ grabs EXECVE

type=SYSCALL-> \w+ grabs SYSCALL
```
---
NOTE: 1
So that ```type_match``` variable matches exactly what kind of type we have in the auditd.
if you just write w, regex searches for the letter "w" but if you write \w the backslash changes it's meaning to = match any word character.
what is the reason we put + = because if you put just \w it will lets say grab only the first character like P from PROCTITLE but if we do + it will grab the whole word. \w+ just tells it to keep grabbing word characters until you hit something that is not one like a space, symbol or newline. 
---
NOTE: 2 
Now the line is ```audit_id_match = re.search(r"msg=audit\(([^)]+)\)", line)``` : I know it looks like I accidentally punched my keyboard. We are looking for fragmented pieces like this: msg=audit(1789847344.787:2727)
Let us break it down:
1. msg=audit\( = It simply searches for the literal text msg=audit and the slash \ must be there because a regular bracket is a special character in regex.
2. ([^)]+) = This is the group in parentheses (...). Which is exactly what we want to extract: [^)] - means any character that is NOT a closing bracket which is ) - the caret ^ in the square brackets acts as the word NOT.
3. + means take everything one by one until you hit that closing bracket.
4. \) - Stop exactly on the closing parenthesis.
In dummy terms msg=audit(, take everything from the middle of the bracket and stop when the bracket closes.) so when we get the information like msg=audit(1789847344.787:2727) we can strip out the ID clean and get 1789847344.787:2727.
---
NOTE: 3
r'(\w+)=(?:"([^"]*)"|([^\s]+))', line 
Table:
| Part | Regex | Captured As | Example Match | Group Value |
| --- | --- | --- | --- | --- |
| **Group 1** | `(\w+)` | `field_name` | `exe="..."` | `exe` |
| **Group 2** | `"([^"]*)"` | `quoted_value` | `"/usr/bin/whoami"` | `/usr/bin/whoami` |
| **Group 3** | `([^\s]+)` | `bare_value` | `pid=6940` | `6940` |
Breakdown:
The outer structure: r'...' = This is the raw string and do not treat \ as escape characters.
(\w+) = Group 1 (field_name) \w+ matches one or more word characters like letters a-z, A-Z , digits 0-9 and underscores _ .... Then it is closed in parentheses (...) to make it a capture group. Matches with item, name , pid and exe etc. 

CODE:




