# Blog - TryHackMe Write-Up

## Context

This write-up documents my work on the **Blog** room from **TryHackMe**.

The lab simulates a realistic enumeration and exploitation scenario involving a personal **WordPress blog** hosted on a Linux machine.

The objective was to identify exposed services, enumerate the application correctly, obtain an initial foothold, and escalate privileges to retrieve the flags.

This is not a speedrun or puzzle-based CTF write-up. 

The focus is on **methodology, decision-making, and enumeration discipline**, aligned with real-world penetration testing workflows.

---

## Objectives

- Identify exposed network services
- Properly enumerate a WordPress application
- Discover valid credentials through enumeration, not guessing
- Achieve initial access
- Perform local enumeration and privilege escalation
- Retrieve user and root flags

---

## Methodology Overview

The lab followed a structured penetration testing approach:

1. Network Scanning
2. Service Enumeration
3. Web Application Enumeration
4. Credential Discovery
5. Initial Access
6. Local Enumeration
7. Privilege Escalation

---

## Enumeration

Enumeration was the most critical phase of this lab.  

Multiple services were exposed, but only **one path was actually viable**.

### Network & Service Discovery

An initial Nmap scan revealed the following services:

- **SSH (22)**
- **HTTP (80)**
- **SMB (139, 445)**

While SSH and SMB appeared interesting, neither provided immediate access without credentials.  

This reinforced the importance of **prioritizing application-layer enumeration**.

---

### Web Enumeration (HTTP)

The web service initially appeared broken. 

Adding the following entry to `/etc/hosts` was required:

10.64.164.200 blog.thm

Once resolved, the application was identified as WordPress.

## Key findings:

- WordPress version **5.0** (outdated)
- XML-RPC enabled
- Directory listing enabled on `/wp-content/uploads/`
- Administrative paths exposed via `robots.txt`

## WordPress Enumeration (WPScan)

WPScan was used to enumerate users and configuration issues.

Identified users:

- `bjoel`
- `kwheel`

No plugins or direct vulnerabilities were immediately exposed.

Rather than forcing exploits, the focus shifted to **credential discovery** via XML-RPC.

## SMB Enumeration

SMB enumeration revealed a share with **anonymous read/write access**.

Files within the share appeared benign but contained **indirect hints**:

- Media files
- Encoded references
- Contextual clues related to user credentials

This reinforced an important lesson:

Enumeration is not about file type, but about information value.

## Credential Discovery

Using WPScan against the XML-RPC endpoint with the discovered usernames led to valid credentials:

- **Username**: `kwheel`
- **Password**: `cutiepie1`

This step was achieved through targeted enumeration, not blind brute forcing.

## Initial Access

With valid credentials, a WordPress Remote Code Execution vulnerability was leveraged.

Using Metasploit:

```bash
use exploit/multi/http/wp_crop_rce
set RHOSTS blog.thm
set USERNAME kwheel
set PASSWORD cutiepie1
run
```
A shell was obtained as the `www-data` user.

The shell was upgraded to a proper TTY for stability:
```python
python3 -c 'import pty; pty.spawn("/bin/bash")'
```
## Local Enumeration

Post-exploitation enumeration revealed:

- User home directories
- A fake `user.txt` flag
- A suspicious PDF file belonging to user `bjoel`

The PDF contained contextual hints pointing toward **privilege escalation**, rather than a direct exploit.

## Privilege Escalation

Enumerating SUID binaries revealed a custom binary:
```bash
find / -perm -u=s -type f 2>/dev/null
```
The binary performed a check against an environment variable named `admin`.

Reverse engineering revealed:

If `admin` was set, the binary would spawn a root shell.

Exploitation:
```bash
export admin=anyvalue
/usr/sbin/checker
```
This resulted in a **root shell**.

## Flags

- **User Flag**: located in `/media/usb/user.txt`
- **Root Flag**: located in `/root/root.txt`

## Key Takeaways

- Enumeration is more important than exploitation
- Not all exposed services are useful entry points
- WordPress misconfigurations often lead to credential disclosure
- SMB shares can act as **information carriers**, not just file storage
- Custom binaries frequently introduce logic flaws exploitable via environment variables
- Methodology beats guessing every time

## ⚠️ Disclaimer

This lab was completed in a controlled and legal environment provided by **TryHackMe**.
All actions were performed strictly for educational purposes.
