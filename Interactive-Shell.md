# Spawning Interactive Shells

> Use these techniques after obtaining command execution or a limited shell on an authorized target.

## 1. What Are We Trying to Achieve?

A limited shell may look like:

```text
$
sh-4.2$
```

but may lack:

* Job control
* Proper terminal handling
* Interactive programs
* `sudo` behavior
* `Ctrl+C`, `Ctrl+Z`
* Tab completion
* A proper TTY/PTY

### Important distinction

```text
Interactive shell
    ≠
Fully functional TTY
```

A shell can be interactive but still report:

```text
no job control in this shell
```

When possible, upgrade the session to a proper TTY/PTY.

---

# 2. `/bin/sh -i`

Start the shell in interactive mode:

```bash
/bin/sh -i
```

or:

```bash
/bin/bash -i
```

Possible result:

```text
sh: no job control in this shell
sh-4.2$
```

### Meaning

```text
-i = interactive
```

This can provide a better shell prompt and interactive behavior, but **does not necessarily give you a real TTY**.

---

# 3. Python → PTY / Interactive Bash

### Python 3

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

### Python

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

### Preferred when available

```text
python3
   ↓
pty.spawn()
   ↓
/bin/bash
   ↓
more interactive shell
```

Check whether Python exists:

```bash
which python3
which python
```

or:

```bash
command -v python3
command -v python
```

> `pty.spawn()` creates/attaches a pseudo-terminal, making it more useful than simply starting `/bin/bash -i`.

---

# 4. Perl → Shell

Check for Perl:

```bash
which perl
```

Spawn `/bin/sh`:

```bash
perl -e 'exec "/bin/sh";'
```

Equivalent with Bash:

```bash
perl -e 'exec "/bin/bash";'
```

### Script form

```perl
exec "/bin/sh";
```

---

# 5. Ruby → Shell

Check:

```bash
which ruby
```

Spawn shell:

```bash
ruby -e 'exec "/bin/sh"'
```

or:

```bash
ruby -e 'exec "/bin/bash"'
```

### Script form

```ruby
exec "/bin/sh"
```

---

# 6. Lua → Shell

Check:

```bash
which lua
```

Spawn shell:

```bash
lua -e 'os.execute("/bin/sh")'
```

or:

```bash
lua -e 'os.execute("/bin/bash")'
```

### Script form

```lua
os.execute("/bin/sh")
```

---

# 7. AWK → Shell

AWK can invoke external commands through `system()`:

```bash
awk 'BEGIN {system("/bin/sh")}'
```

Bash:

```bash
awk 'BEGIN {system("/bin/bash")}'
```

### Why it works

```text
awk
 ↓
system()
 ↓
/bin/sh
```

Useful when AWK is available but Python/Perl/Ruby/Lua aren't.

Check:

```bash
which awk
```

---

# 8. `find` → Shell

`find` supports command execution through `-exec`.

### Direct shell execution

```bash
find . -exec /bin/sh \; -quit
```

or:

```bash
find . -exec /bin/bash \; -quit
```

### Important

`find` must successfully process a path for `-exec` to execute.

A simple reliable form is:

```bash
find . -exec /bin/sh \; -quit
```

### Understand the syntax

```text
find .
  ↓
search current directory

-exec
  ↓
execute a command

/bin/sh
  ↓
command to execute

\;
  ↓
end of -exec expression

-quit
  ↓
stop find after execution
```

---

# 9. `find` + AWK

Another method:

```bash
find / -name <filename> -exec /bin/awk 'BEGIN {system("/bin/sh")}' \;
```

Flow:

```text
find
 ↓
find matching file
 ↓
-exec AWK
 ↓
AWK system()
 ↓
/bin/sh
```

This is mainly useful as a fallback when you have `find` and `awk`.

---

# 10. Vim → Shell

If Vim is available:

```bash
vim -c ':!/bin/sh'
```

This runs `/bin/sh` through Vim's command execution mechanism.

### From inside Vim

Open Vim:

```bash
vim
```

Then:

```vim
:set shell=/bin/sh
:shell
```

You can also use:

```vim
:!id
```

to execute a single command without leaving Vim.

### Why it works

Vim can execute external commands and can launch the configured shell.

---

# 11. Quick Availability Checks

Before choosing a technique, check what exists:

```bash
command -v python3
command -v python
command -v perl
command -v ruby
command -v lua
command -v awk
command -v find
command -v vim
```

Example:

```text
/usr/bin/python3
/usr/bin/awk
/usr/bin/find
```

Then choose an available method.

---

# 12. Priority / Decision Tree

When you have a limited Linux shell:

```text
Do I have Python?
       │
      YES
       ↓
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

If Python isn't available:

```text
Perl?
 ↓
perl -e 'exec "/bin/sh";'
```

If not:

```text
Ruby?
 ↓
ruby -e 'exec "/bin/sh"'
```

If not:

```text
Lua?
 ↓
lua -e 'os.execute("/bin/sh")'
```

If not:

```text
AWK?
 ↓
awk 'BEGIN {system("/bin/sh")}'
```

If not:

```text
find?
 ↓
find . -exec /bin/sh \; -quit
```

If Vim is available:

```text
vim -c ':!/bin/sh'
```

Always prefer the method that gives you the **most functional terminal**, not simply the first command that happens to spawn `/bin/sh`.

---

# 13. Shell Interpreter Can Change

`/bin/sh` and `/bin/bash` are examples.

You can substitute another available shell:

```bash
/bin/zsh
/bin/dash
/bin/fish
```

Check available shells:

```bash
cat /etc/shells
```

Check the current shell:

```bash
echo "$SHELL"
```

Check the actual shell process:

```bash
ps
```

or:

```bash
ps -p $$ -o pid,ppid,cmd
```

---

# 14. Check Permissions

When working from a limited shell, inspect permissions on relevant files/binaries:

```bash
ls -la <path>
```

Example:

```bash
ls -la /usr/bin/python3
```

For directories:

```bash
ls -ld <directory>
```

Useful when determining whether you can execute, read, or modify something.

---

# 15. Check `sudo` Permissions

Run:

```bash
sudo -l
```

This shows commands the current user may execute through `sudo`.

Example:

```text
User apache may run the following commands:
    (ALL : ALL) NOPASSWD: ALL
```

This indicates the user can execute commands as another user, potentially including root, without entering a password.

### Important

`sudo -l` may behave poorly from an unstable shell.

If necessary:

```text
Limited shell
    ↓
spawn/upgrade shell
    ↓
sudo -l
```

---

# 16. Important Post-Shell Checks

After obtaining an interactive shell, establish your situation:

### Current user

```bash
whoami
id
```

### Current directory

```bash
pwd
```

### Operating system

```bash
uname -a
```

### Shell

```bash
echo "$SHELL"
ps -p $$ -o cmd=
```

### Available interpreters

```bash
which python3 python perl ruby lua awk
```

### Sudo privileges

```bash
sudo -l
```

### Environment

```bash
env
```

---

# 17. Interactive Shell vs TTY

### Basic interactive shell

```bash
/bin/sh -i
```

May produce:

```text
sh: no job control in this shell
```

### PTY-backed shell

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Generally gives better terminal behavior.

### Check whether you have a TTY

```bash
tty
```

Possible output:

```text
/dev/pts/0
```

or:

```text
not a tty
```

A real PTY is preferable for things such as:

```text
sudo
su
vim
ssh
job control
Ctrl+C / Ctrl+Z
terminal resizing
interactive applications
```

---

# 18. Minimal "Shell Spawn" Cheatsheet

```bash
# Interactive sh
/bin/sh -i

# Interactive bash
/bin/bash -i

# Python 3 → Bash PTY
python3 -c 'import pty; pty.spawn("/bin/bash")'

# Python → Bash PTY
python -c 'import pty; pty.spawn("/bin/bash")'

# Perl → sh
perl -e 'exec "/bin/sh";'

# Ruby → sh
ruby -e 'exec "/bin/sh"'

# Lua → sh
lua -e 'os.execute("/bin/sh")'

# AWK → sh
awk 'BEGIN {system("/bin/sh")}'

# Find → sh
find . -exec /bin/sh \; -quit

# Vim → sh
vim -c ':!/bin/sh'
```

---

# 19. Remember the Goal

The goal isn't:

```text
"Run a cool shell-spawning command."
```

The goal is:

```text
Limited command execution
        ↓
Interactive shell
        ↓
PTY/TTY where possible
        ↓
Reliable terminal interaction
        ↓
Enumeration / privilege escalation / pivoting
```

## Core Rule

```text
Shell spawn = get an interactive interpreter
PTY upgrade = get a more functional terminal
```

Don't confuse the two.
