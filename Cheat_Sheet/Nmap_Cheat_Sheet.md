# Nmap Cheat Sheet

Scan only authorized network please!

---

## Host Discovery (who is alive)

```bash
nmap -sn 192.168.1.0/24        # ping sweep, no port scan, just find live hosts
nmap -sn 192.168.1.1-50        # sweep a range
nmap -Pn <ip>                  # skip discovery, treat host as up (use when ping is blocked)
nmap -sL 192.168.1.0/24        # list targets only, no packets sent
```

## Basic Port Scans

```bash
nmap <ip>                      # top 1000 TCP ports, default
nmap -p 22,80,443 <ip>         # specific ports
nmap -p- <ip>                  # ALL 65535 TCP ports (slow but complete)
nmap -F <ip>                   # fast, top 100 ports only
nmap --top-ports 20 <ip>       # top N most common ports
```

## Scan Types

```bash
nmap -sS <ip>                  # SYN scan (default as root, fast, stealthy)
nmap -sT <ip>                  # TCP connect scan (used when not root)
nmap -sU <ip>                  # UDP scan (slow, but DNS/SNMP/DHCP live here)
nmap -sU -sS <ip>              # TCP and UDP together
```

## Service and Version Detection

```bash
nmap -sV <ip>                  # identify service + version on each open port
nmap -sV --version-intensity 9 <ip>   # try harder to fingerprint versions
nmap -O <ip>                   # OS detection (needs root)
nmap -A <ip>                   # aggressive: -sV, -O, scripts, traceroute all at once
```

## The One You'll Run Most (full baseline)

```bash
nmap -sV -sC -p- <ip> -oN baseline.txt
# -sV version detection
# -sC default safe scripts
# -p- every port
# -oN save readable output to a file
```

## NSE Scripts (the power feature)

```bash
nmap -sC <ip>                          # run the default script set
nmap --script vuln <ip>                # check for known vulns
nmap --script smb-enum-shares <ip>     # list SMB shares
nmap --script smb-os-discovery <ip>    # OS/hostname/domain via SMB
nmap --script http-title <ip>          # grab web page titles
nmap --script "http-*" <ip>            # run all http scripts
nmap --script-help <script-name>       # read what a script does before running it
```

## Timing (speed vs stealth)

```bash
nmap -T0 <ip>    # paranoid, very slow, evades IDS
nmap -T3 <ip>    # normal (default)
nmap -T4 <ip>    # faster, fine on a lab or LAN you own
nmap -T5 <ip>    # insane, fast, noisy, can miss results
```

## Output (always save your scans)

```bash
nmap -sV <ip> -oN out.txt       # normal readable text
nmap -sV <ip> -oG out.gvp       # greppable, easy to filter with grep
nmap -sV <ip> -oX out.xml       # XML, for importing into other tools
nmap -sV <ip> -oA scan_name     # all three formats at once
```

## Useful Extras

```bash
nmap -v <ip>                    # verbose, show progress
nmap -vv <ip>                   # very verbose
nmap --reason <ip>              # why nmap marked a port open/closed
nmap --open <ip>               # only show open ports, hide the rest
nmap -6 <ipv6>                  # scan an IPv6 target
nmap -iL targets.txt            # read targets from a file, one per line
nmap --resume out.txt           # resume an interrupted scan
```

## Reading the Results

Port states you'll see:
- **open** something is listening and accepting connections
- **closed** host is up, nothing listening on that port
- **filtered** a firewall is dropping the probe, nmap can't tell
- **open|filtered** nmap couldn't decide (common on UDP)

## Typical Workflow

```bash
# 1. Find live hosts
nmap -sn 192.168.1.0/24

# 2. Full scan each live host, save it
nmap -sV -sC -p- 192.168.1.10 -oN host10_baseline.txt

# 3. If something looks weak, dig into that service
nmap --script vuln -p 445 192.168.1.10
```
