# Python Log Parser Lab

---

**`execve`:** This is the Linux kernel call that starts a program; `auditd` records the binary, arguments, user, and PID every time it happens.

---

### Real Output from `auditd`

```log
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

---

### Code Breakdown

In the Python code within the `fields` helper function, we have the line:

```python
type_match = re.search(r"type=(\w+)", line)
```

What that captures is:

- `type=PROCTITLE` → `\w+` grabs `PROCTITLE`
- `type=PATH` → `\w+` grabs `PATH`
- `type=CWD` → `\w+` grabs `CWD`
- `type=EXECVE` → `\w+` grabs `EXECVE`
- `type=SYSCALL` → `\w+` grabs `SYSCALL`

#### Note 1: `type_match`

- The `type_match` variable matches exactly what kind of type we have in `auditd`.
- If you just write `w`, regex searches for the literal letter `"w"`. But if you write `\w`, the backslash changes its meaning to match any word character.
- **Why the `+`?** Because if you put just `\w`, it will grab only the first character (like `P` from `PROCTITLE`). Adding `+` grabs the entire word. `\w+` tells the engine to keep grabbing word characters until it hits something that is not one, such as a space, symbol, or newline.

---

#### Note 2: `audit_id_match`

```python
audit_id_match = re.search(r"msg=audit\(([^)]+)\)", line)
```

This pattern searches for fragmented pieces like `msg=audit(1789847344.787:2727)`.

Breaking it down:

1. **`msg=audit\(`**  
   Searches for the literal text `msg=audit`. The backslash `\` is required because a regular parenthesis `(` is a special syntax character in regex.

2. **`([^)]+)`**  
   This is the capture group inside parentheses `(...)`, which is what we want to extract:
   - `[^)]`: Matches any character that is **not** a closing parenthesis `)`. The caret `^` inside square brackets negates the set (acts as the word "NOT").
   - `+`: Takes everything one by one until it hits that closing parenthesis.

3. **`\)`**  
   Stops exactly on the literal closing parenthesis.

> **In simple terms:** Match `msg=audit(`, take everything from the middle, and stop when the bracket closes `)`. When given `msg=audit(1789847344.787:2727)`, it cleanly extracts `1789847344.787:2727`.

---

#### Note 3: Key-Value Field Parser

```python
r'(\w+)=(?:"([^"]*)"|([^\s]+))'
```

| Part | Regex | Captured As | Example Match | Group Value |
| --- | --- | --- | --- | --- |
| **Group 1** | `(\w+)` | `field_name` | `exe="..."` | `exe` |
| **Group 2** | `"([^"]*)"` | `quoted_value` | `"/usr/bin/whoami"` | `/usr/bin/whoami` |
| **Group 3** | `([^\s]+)` | `bare_value` | `pid=6940` | `6940` |

**Regex structure:**

- **`r'...'`:** A Python raw string so backslashes `\` are not treated as escape characters.
- **`(\w+)` (Group 1 / `field_name`):** Matches one or more word characters (`a-z`, `A-Z`, `0-9`, and `_`). Parentheses make it a capture group. Matches keys like `item`, `name`, `pid`, `exe`, etc.
- **`=`:** Matches the literal equals sign.
- **`(?:"([^"]*)"|([^\s]+))`:** A non-capturing group `(?:...)` with an `|` (OR) branch:
  - **Branch 1 (`"([^"]*)"`):** Captures values inside quotes without including the quotes themselves.
  - **Branch 2 (`([^\s]+)`):** Captures non-whitespace characters for bare values, stopping at the next space.

So what output is it capturing:

```python
re.findall(r'(\w+)=(?:"([^"]*)"|([^\s]+))', line)
```

That line is one search pattern. It walks the log string and tries to pick up `name=value` pairs over and over again.

Now we will do it piece by piece:

- **`(\w+)`** = A field name. Letters / numbers / `_`.  
  Example from our own log: `argc`, `a0`, `pid`, `uid`, `exe`, `cwd`, `comm`.

- **`=`** = The equal sign just sits between the name and the value.

- **`(?:"([^"]*)"|([^\s]+))`** = The value will be either in quotes or not in quotes.

Two options:

**`"([^"]*)"`**  
A `"` then anything that is not a `"`, then `"`. Captures the text inside the quotes.

From our own log we can see:

- `a0="whoami"` → inside is `whoami`
- `cwd="/home/kali/Desktop"` → `/home/kali/Desktop`
- `exe="/usr/bin/whoami"` → `/usr/bin/whoami`

`[^"]` means any one character that is not a double-quote.  
`*` means repeat that zero or more times.  
`[^"]*` means everything inside the quotes, but stop at the closing quote.

**`([^\s]+)`**  
If there are no quotes, grab the next blob until a space.

From our log:

- `pid=6940` → `6940`
- `uid=0` → `0`
- `syscall=59` → `59`
- `argc=1` → `1`

`\s` is any whitespace (space, tab, newline).  
`[^\s]` means not whitespace, so a normal character.  
`+` means one or more of those.  
So `[^\s]+` grabs a value like `6940` or `59` and stops at the next space.

**`findall`** = don't stop at the first match; keep finding and return every `name=value` you find.

Do that for the whole line, not once. Every match becomes a triple:

- `field_name`
- `quoted_value`
- `bare_value`

Then this bit I wrote down:

```python
line_dictionary[field_name] = quoted_value or bare_value
```

Means: use the quoted text if it's there, or use the bare text.

---

### Worked example: the EXECVE line

```text
type=EXECVE msg=audit(1789847344.787:2727): argc=1 a0="whoami"
```

What this regex picks up (we later skip `type` and `msg` on purpose):

| field_name | quoted_value | bare_value | stored |
| --- | --- | --- | --- |
| argc | (empty) | 1 | 1 |
| a0 | whoami | (empty) | whoami |

`type=EXECVE` also matches as name=`type` value=`EXECVE`, but your `if field_name not in ("type", "msg")` throws that away because you already stored type and the audit id with the two `search` calls.

---

### Now look at the SYSCALL line

```text
pid=6940 uid=0 comm="whoami" exe="/usr/bin/whoami"
```

| field_name | how it looks in the log | stored |
| --- | --- | --- |
| pid | `pid=6940` | `6940` |
| uid | `uid=0` | `0` |
| comm | `comm="whoami"` | `whoami` |
| exe | `exe="/usr/bin/whoami"` | `/usr/bin/whoami` |

So: find `name=`, then take either `"this"` or `this`.
