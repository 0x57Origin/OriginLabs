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

```
