# Send VM, Docker, and AWS Logs to One Place

---

## What Is This Lab

Get logs from 3 places into Wazuh on Kali:

- Windows 11 VM (Sysmon)
- Juice Shop (Docker)
- AWS CloudTrail (one event)

No domain needed.

---

## Network Setup

We will need to make sure the Kali and Windows VMs are talking to each other. The Kali VM will get internet, and the Windows VM will not need it. It only needs the lab network so it can reach Kali (Wazuh agent → Kali IP).

What we will do:

- **Kali:** internet + lab network. I already have NAT + Host-only on Kali.
- Pull images and start Wazuh + Juice Shop on Kali.
- **Win11 VM:** lab network only, ping Kali.
- Use AWS from Kali (internet), not from the Win11 VM.

### Ping Test

Let's ping really fast. From the Windows 11 VM, ping Kali's VM:

```
ping 192.168.56.103
```

Also ping the Windows VM from Kali. This is the command from Kali's side:

```
ping 192.168.56.101
```

But here is the thing: my Windows VM is already configured, but by default Windows will not let you ping its IP from other devices. Usually it will drop or block it.

So, on the Win11 VM, open Admin PowerShell and enable the rule **File and Printer Sharing (Echo Request - ICMPv4-In)**:

```
Enable-NetFirewallRule -Name FPS-ICMP4-ERQ-In
```

But if that rule is missing from Windows:

```
New-NetFirewallRule -DisplayName "Allow ICMPv4-In" -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow
```

Then you can ping again! But since I already have that rule installed, let me check:

```
Get-NetFirewallRule -Name FPS-ICMP4-ERQ-In | Format-List Name,DisplayName,Enabled,Direction,Action
```

And look:

```
Name        : FPS-ICMP4-ERQ-In
DisplayName : File and Printer Sharing (Echo Request - ICMPv4-In)
Enabled     : True
Direction   : Inbound
Action      : Allow
```

So, we are set.

---

## Wazuh on Kali, via Docker

```
docker --version
sudo docker ps
```

`ps` = Process Status. We use that command to list all containers on our system that are currently running.

Docker seems to be just fine, so let's start Wazuh:

```
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf

cd ~
git clone https://github.com/wazuh/wazuh-docker.git -b v4.12.0 --depth=1
cd wazuh-docker/single-node
sudo docker compose up -d
sudo docker compose ps
```

### Breakdown

**`sudo sysctl -w vm.max_map_count=262144`**

- `sysctl` = change a kernel setting
- `-w` = write the value
- `vm.max_map_count=262144` = required because the Wazuh Indexer will refuse to start and crash your Docker containers if it is left at the standard Linux default. The Wazuh Indexer is there to search all our security alerts.

**`echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf`**

- `echo` = print that text
- `|` = pipe: send that text into the next command
- `sudo tee` = write to a file as admin
- `-a` = append: add to the end, do not wipe the file
- `/etc/sysctl.conf` = the file Linux reads at boot for sysctl settings

So: print the line → append it to the boot file.

**`sudo docker compose up -d`**

- `docker compose` = read the `docker-compose.yml` in this folder and manage that stack
- `up` = create and start the containers
- `-d` = detached: start in the background and give my terminal back

**`sudo docker compose ps`**

- `ps` = Process Status: list the containers in this compose project










