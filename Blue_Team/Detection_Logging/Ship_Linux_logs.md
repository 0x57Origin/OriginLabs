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
