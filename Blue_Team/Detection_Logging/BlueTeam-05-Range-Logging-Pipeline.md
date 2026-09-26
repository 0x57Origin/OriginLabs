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

---
Result:
```
NAME                            IMAGE                          COMMAND                  SERVICE           CREATED          STATUS         PORTS
single-node-wazuh.dashboard-1   wazuh/wazuh-dashboard:4.12.0   "/entrypoint.sh"         wazuh.dashboard   9 seconds ago    Up 3 seconds   443/tcp, 0.0.0.0:443->5601/tcp, [::]:443->5601/tcp
single-node-wazuh.indexer-1     wazuh/wazuh-indexer:4.12.0     "/entrypoint.sh open…"   wazuh.indexer     16 seconds ago   Up 4 seconds   0.0.0.0:9200->9200/tcp, [::]:9200->9200/tcp
single-node-wazuh.manager-1     wazuh/wazuh-manager:4.12.0     "/init"                  wazuh.manager     16 seconds ago   Up 4 seconds   0.0.0.0:1514-1515->1514-1515/tcp, [::]:1514-1515->1514-1515/tcp, 0.0.0.0:514->514/udp, [::]:514->514/udp, 0.0.0.0:55000->55000/tcp, [::]:55000->55000/tcp, 1516/tcp
```
All three part of wazuh is up. 
manager — takes agent logs (1514 / 1515)
indexer — stores them (9200)
dashboard — the web UI (443 → 5601)

Now what are those numbers? Those are ports. Doors on Kali. Other machines talk to Wazuh through them.

Port,Service,Meaning
1514,manager,Agent sends logs here (the main pipe)
1515,manager,Agent enrollment / first handshake
55000,manager,Wazuh API
514/udp,manager,Extra syslog door (we are not using it yet)
9200,indexer,Where events get stored (like a database)
443,dashboard,HTTPS in your browser. Inside the container the app is on 5601; Docker maps 443 → 5601

Right now I only care about 2 things:
Browser: https://127.0.0.1 (443)
Windows agent later: Kali lab IP 192.168.56.103 ports 1514 and 1515

Well now I did go to https://127.0.0.1 and it states that the site can't be reached. So I searched online and found that dashboard might be the issue, it might be still booting or it crashed. Fixes:
```
sudo docker compose ps
sudo docker compose logs --tail=50 wazuh.dashboard
curl -k -I https://127.0.0.1
```
Ps = Is the dashboard still up?
logs = Why is my Wazuh unhappy for?
curl -k -I https://127.0.0.1 = -k ignore bad cert, -I headers only. I want either HTTP/2 200 or 302 or a login page, not connection refused.

Okay I kind of figured out the problem here: I had bad internet issues so while I was trying to compose I did some dumb stuff and caused the compose issues so now, the dashboard is crashing, it is trying to read an SSL(Secure Sockets Layer) file , but Docker created a folder with that name. So apparently, it happens when your run compose run before making the certs.
Fix in this order:
```
cd ~/Desktop/wazuh-docker/single-node
sudo docker compose down
```
Then take a look at the broken certs:
```
ls -l config/wazuh_indexer_ssl_certs
```
Result:
```
total 40
drwxr-xr-x 2 root root 4096 Sep 25 21:09 admin-key.pem
drwxr-xr-x 2 root root 4096 Sep 25 21:09 admin.pem
drwxr-xr-x 2 root root 4096 Sep 25 21:09 root-ca-manager.pem
drwxr-xr-x 2 root root 4096 Sep 25 21:09 root-ca.pem
drwxr-xr-x 2 root root 4096 Sep 25 21:09 wazuh.dashboard-key.pem
drwxr-xr-x 2 root root 4096 Sep 25 21:09 wazuh.dashboard.pem
drwxr-xr-x 2 root root 4096 Sep 25 21:09 wazuh.indexer-key.pem
drwxr-xr-x 2 root root 4096 Sep 25 21:09 wazuh.indexer.pem
drwxr-xr-x 2 root root 4096 Sep 25 21:09 wazuh.manager-key.pem
drwxr-xr-x 2 root root 4096 Sep 25 21:09 wazuh.manager.pem
```
If you see names like wazuh.dashboard.pem with a d at the start of the line, those are folders. Delete that whole cert folder:
```
sudo rm -rf config/wazuh_indexer_ssl_certs
mkdir -p config/wazuh_indexer_ssl_certs
```
Then generate real certs:
```
sudo docker compose -f generate-indexer-certs.yml run --rm generator
```
Check again after that:
```
ls -l config/wazuh_indexer_ssl_certs
```
We want files which is (-rw-), not directories (drwx)...
Start again
```
sudo docker compose up -d
```
Wait a few minutes, then:
```
curl -k -I https://127.0.0.1
```
Okay after that I got HTTP/1.1 302 Found → /app/login from curl command. That is the login page. We will just accept the cert warning. 
Login:
User: admin
Password: SecretPassword
And that is it for Wazhue for right now.
---
## Back to Windows VM now












