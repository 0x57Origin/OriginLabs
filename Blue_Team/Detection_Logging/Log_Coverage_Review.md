# Log Coverage Review: Let's Find the Blind Spots (Paper Lab Review)

Well, in the last few labs I already turned on the logging: Sysmon, 4688, and auditd. Now we will line it up against a handful of attacker techniques and mark what would catch and what would miss. For each miss, we will see what we can do to close it and what it costs in log volume.

We will use the standard attacks, which is MITRE ATT&CK. This is a paper review, no attacks run. For each technique I check whether my current process-creation logging would have caught it.

What is MITRE ATT&CK? It is a big public list of things attackers actually do in real life, sorted by stage. Each move, like stealing a password, making a scheduled task, or dumping credentials, is a technique with an ID. Example: T1059. It exists so defenders like us have one shared language. Instead of guessing what to look out for, we just check the logging against a known list of real attack behavior. Why you are using it here: because it gives us the techniques we should care about.

Also, what is a scheduled task? It is a Windows job set to run automatically at a certain time or trigger, like at login, and attackers abuse it to launch or relaunch their code and stay on the box.

Here are the 6 attacks and their IDs:

```
Command execution (T1059) - attacker runs programs or shell commands on the host
Encoded PowerShell (T1059.001) - PowerShell run with -enc to hide what it is doing
Scheduled task persistence (T1053.005) - attacker makes a scheduled task to relaunch their code
New local account (T1136.001) - attacker creates an account to keep access
Failed / suspicious logons (T1110) - password guessing or brute force showing up in logon events
Outbound connection (T1071) - malware calling out to its server
```

## Catch or Miss: Does It Show Up as a Process?

1. Command Execution: CATCH. It is a process running and it will log it.
2. Encoded PowerShell: CATCH. It runs powershell.exe -enc, a process with the command line. I even caught it in my Sysmon lab.
3. Scheduled task persistence: PARTIAL. You will see schtasks.exe run, but not the task that got created or when it fires later.
4. New local account: PARTIAL. You will see net.exe user... run, but not the actual account-creation event. If they decide to make the account another way, you will not catch anything.
5. Failed / suspicious logons: MISS. A failed login is not a process running.
6. Outbound connection: MISS. A network connection is not a process. Process-creation logging does not record network traffic.

## The Fixes for Each Gap

3. Scheduled task persistence: Turn on Event ID 4698 = a scheduled task was created. Volume cost: Low. Tasks are not made often, so this barely will add any noise.
4. New local account: Turn on Event ID 4720 = a user account was created. Volume cost: Low. Account creation is rare, so almost pure signal something malicious happened.
5. Failed / suspicious logons: Turn on Event ID 4625, which is the failed logon, and 4624, which is the successful logon. Volume cost: medium to high. Logons happen constantly, so 4624 especially is noisy. 4625 alone is much cheaper and catches the brute force.
6. Outbound connection: Turn on Sysmon Event ID 3 = which is the network connection. Volume cost: High. Every connection every process makes gets logged. This is the loudest one so far. We gotta tame it. We will filter the Sysmon config to only the process or ports we care about.

Conclusion: The cheap wins are persistence events, which are 4698 and 4720, while the logon and network events cost the most in volume because normal use floods them. So real logging is always a trade between catching more and deciphering the noise.
