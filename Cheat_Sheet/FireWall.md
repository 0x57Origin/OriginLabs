# 1. Check the firewall is ON (if False, rules do nothing)
Get-NetFirewallProfile | Select-Object Name, Enabled

# 2. Turn it on
Set-NetFirewallProfile -Profile Domain,Private,Public -Enabled True

# 3. Test before and after blocking
Test-NetConnection 1.1.1.1 -Port 443

# 4. Block an outbound IP + port
New-NetFirewallRule -DisplayName "Block 1.1.1.1 443" -Direction Outbound -Action Block -RemoteAddress 1.1.1.1 -RemotePort 443 -Protocol TCP

# 5. Block an IP on all ports
New-NetFirewallRule -DisplayName "Block 1.1.1.1" -Direction Outbound -Action Block -RemoteAddress 1.1.1.1

# 6. Block a program from reaching the internet
New-NetFirewallRule -DisplayName "Block App" -Direction Outbound -Action Block -Program "C:\Path\app.exe"

# 7. Block inbound on a port
New-NetFirewallRule -DisplayName "Block In 3389" -Direction Inbound -Action Block -LocalPort 3389 -Protocol TCP

# 8. See your rule
Get-NetFirewallRule -DisplayName "Block 1.1.1.1 443"

# 9. Turn a rule off / on
Disable-NetFirewallRule -DisplayName "Block 1.1.1.1 443"
Enable-NetFirewallRule -DisplayName "Block 1.1.1.1 443"

# 10. Delete a rule
Remove-NetFirewallRule -DisplayName "Block 1.1.1.1 443"

# 11. Log blocked traffic (off by default)
Set-NetFirewallProfile -Profile Domain,Private,Public -LogBlocked True -LogFileName "C:\Windows\System32\LogFiles\Firewall\pfirewall.log"

# 12. Read the log
notepad C:\Windows\System32\LogFiles\Firewall\pfirewall.log
