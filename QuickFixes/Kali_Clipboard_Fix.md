# Kali Clipboard Not Working in VirtualBox

Copy/paste between Kali and the Windows host stopped working, even with Shared Clipboard set to Bidirectional and Guest Additions installed. The cause was the `VBoxClient` clipboard service not running in a valid desktop session.

## Checks that ruled things out

Guest Additions was loaded:

```bash
lsmod | grep vboxguest
# vboxguest    53248  5 vboxsf
```

- `lsmod` = **l**i**s**t **mod**ules. Shows the kernel modules currently loaded.
- `grep vboxguest` = filter that list down to lines containing "vboxguest".
- Together: is the VirtualBox guest module loaded? A line came back, so yes.

VirtualBox setting was already correct: Devices > Shared Clipboard > Bidirectional. Toggling it did nothing.

## The real error

```bash
pkill VBoxClient
VBoxClient --clipboard
# error: XDG_RUNTIME_DIR is invalid or not set in the environment.
```

- `pkill` = **p**rocess **kill**. Kills a process by its name (here, every `VBoxClient`), instead of needing its number.
- `VBoxClient --clipboard` = start the clipboard-sharing service by hand so I can see its output.

I was running it as root. The clipboard belongs to the graphical session, and root has none, so the service had nothing to attach to.

## Fix

```bash
pkill VBoxClient
sudo -u kali DISPLAY=:0 XDG_RUNTIME_DIR=/run/user/1000 VBoxClient --clipboard
```

- `sudo -u kali` = run the command as the **u**ser `kali`, not root.
- `DISPLAY=:0` = point it at the active graphical display (screen 0).
- `XDG_RUNTIME_DIR=/run/user/1000` = give it the per-user session folder it was missing. `1000` is the kali user's ID (confirm with `id kali`).
- `VBoxClient --clipboard` = start the clipboard service, now with a real session behind it.

Clipboard worked right after.

**Permanent fix:** log out and back in, or restart the VM. That starts `VBoxClient` cleanly in a proper session, so it works from boot.

## Takeaway

The setting was right but the service was not working. The error message named the real problem, which pointed at a session issue, not a VirtualBox config issue.
