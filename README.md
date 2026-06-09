# CTF & Lab Writeups

Methodology-first writeups from TryHackMe, INE (eJPT preparation), and HackTheBox. Each writeup documents the full attack chain - reconnaissance through post-exploitation - with focus on technique reasoning rather than just flag capture.

**42 writeups** across 2 platforms (HackTheBox coming).

---

## TryHackMe

| Room | Key Technique |
|---|---|
| [Anonymous](tryhackme/anonymous/) | FTP anonymous login, SUID cron misconfiguration |
| [Blog](tryhackme/blog/) | WordPress credential brute force, SUID binary abuse |
| [Brute It](tryhackme/brute-it/) | HTTP admin panel brute force, SSH key cracking, sudo cat shadow |
| [Smag Grotto](tryhackme/smag-grotto/) | PCAP credential extraction, cron job injection, sudo apt-get GTFOBins |
| [Tomghost](tryhackme/tomghost/) | Ghostcat CVE-2020-1938, GPG passphrase cracking, sudo zip GTFOBins |
| [Wgel](tryhackme/wgel/) | HTML comment user leak, exposed SSH key, sudo wget exfiltration |
| [Year of the Rabbit](tryhackme/year-of-the-rabbit/) | FTP steganography, base64 decode, Hydra SSH, sudo vi GTFOBins |
| [GamingServer](tryhackme/gaming-server/) | gobuster secret key, ssh2john cracking, LXD container escape |
| [COLDDBOX:EASY](tryhackme/colddbox-easy/) | Non-standard SSH port, wpscan brute force, wp-config SSH reuse, sudo vim |
| [Chocolate Factory](tryhackme/chocolate-factory/) | FTP steganography chain, SHA-512 cracking, PATH hijack, sudo vi |
| [Lookup](tryhackme/lookup/) | Username enumeration, elFinder CVE, PATH hijacking, sudo look file read |

---

## INE

### Assessment Methodologies

| Lab | Key Technique |
|---|---|
| [Footprinting and Scanning](ine/assessment-methodologies/Footprinting-and-Scanning/) | Host discovery, port scanning, service detection |
| [SMB Enumeration with Nmap](ine/assessment-methodologies/SMB-Enumeration-Nmap/) | 10 SMB NSE scripts, guest session detection |
| [NetBIOS Hacking](ine/assessment-methodologies/NetBIOS-Hacking/) | Null session, Hydra SMB, PsExec SYSTEM, autoroute pivot |
| [ProFTPd Recon](ine/assessment-methodologies/ProFTPd-Recon/) | Hydra FTP brute force, 7 credential pairs, 7 flags |
| [SSH Enumeration](ine/assessment-methodologies/SSH-Enumeration/) | Metasploit ssh_version + ssh_login, STOP_ON_SUCCESS |
| [Postfix SMTP Recon](ine/assessment-methodologies/Postfix-SMTP-Recon/) | VRFY user enum, EHLO capability disclosure, smtp-user-enum |
| [Samba Dictionary Attack](ine/assessment-methodologies/Samba-Dictionary-Attack/) | smb_login + Hydra, smbmap permissions, non-browsable shares, RID cycling |
| [Password Cracker Linux](ine/assessment-methodologies/Password-Cracker-Linux/) | ProFTPD 1.3.3c backdoor, hashdump, crack_linux auxiliary |
| [SNMP Enumeration](ine/assessment-methodologies/SNMP-Enumeration/) | UDP 161, community string brute force, user list to SMB pivot, PsExec |

### System/Host Penetration Testing

| Lab | Key Technique |
|---|---|
| [Shellshock](ine/system-host-penetration-testing/Shellshock/) | CVE-2014-6271, Burp Suite User-Agent injection, bash env exec |
| [Exploitation CTF 1](ine/system-host-penetration-testing/Exploitation-CTF-1/) | FlatCore CMS RCE, WordPress Duplicator file read |
| [Exploitation CTF 2](ine/system-host-penetration-testing/Exploitation-CTF-2/) | SMB brute force, pass-the-hash, msfvenom ASPX shell |
| [Exploitation CTF 3](ine/system-host-penetration-testing/Exploitation-CTF-3/) | ProFTPD mod_copy, SMB anonymous write PHP shell, SUID find |
| [System-Host CTF 2 Linux](ine/system-host-penetration-testing/System-Host-CTF-2-Linux/) | Shellshock CGI, libssh auth bypass, binary PATH hijacking |
| [Linux PrivEsc via Cron Job](ine/system-host-penetration-testing/Linux-PrivEsc-Cron-Job/) | Writable cron script, printf sudoers injection |
| [Windows HnNPT](ine/system-host-penetration-testing/Windows-HnNPT/) | WebDAV davtest + cadaver ASP shell, SMB Hydra brute force |
| [MSF CTF 1 Windows](ine/system-host-penetration-testing/MSF-CTF-1-Windows/) | MSSQL RCE, SeImpersonatePrivilege, getsystem |
| [Windows PrivEsc PowerUp](ine/system-host-penetration-testing/Windows-PrivEsc-PowerUp/) | Unattend.xml base64 password, runas, HTA reverse shell |
| [Windows Token Impersonation](ine/system-host-penetration-testing/Windows-Token-Impersonation/) | HFS RCE, Incognito load_tokens, impersonate_token |
| [Windows Credential Dumping Kiwi](ine/system-host-penetration-testing/Windows-Credential-Dumping-Kiwi/) | BadBlue RCE, migrate lsass, kiwi creds_all + lsa_dump |
| [Rootkit Scanner PrivEsc](ine/system-host-penetration-testing/Rootkit-Scanner-PrivEsc/) | chkrootkit 0.49 cron abuse, MSF local exploit |
| [Windows UAC Bypass](ine/system-host-penetration-testing/Windows-UAC-Bypass/) | HFS RCE, UACMe Akagi64 method 23, hashdump |
| [Windows NTLM Hash Cracking](ine/system-host-penetration-testing/Windows-NTLM-Hash-Cracking/) | BadBlue RCE, migrate lsass, hashdump, John + Hashcat + MSF crack_windows |

### Network-Based Attacks

| Lab | Key Technique |
|---|---|
| [Network CTF 1 - Wireshark Forensics](ine/network-based-attacks/Network-CTF-1-Wireshark/) | HTTP filter, NBNS hostname, PowerShell UA detection, TCP stream extraction |

### Post-Exploitation

| Lab | Key Technique |
|---|---|
| [Meterpreter Basics](ine/post-exploitation/Meterpreter-Basics/) | Xdebug RCE, Meterpreter filesystem ops, upload/download, webshell staging |
| [Windows Persistence](ine/post-exploitation/Windows-Persistence/) | persistence_service auto-start, getgui RDP backdoor user |
| [Post-Exploitation CTF 2](ine/post-exploitation/Post-Exploitation-CTF-2/) | SSH brute force, NTLM hash crack (John), PrintSpoofer SYSTEM, icacls ACL bypass |
| [Clearing Tracks Linux](ine/post-exploitation/Clearing-Tracks-Linux/) | Samba is_known_pipename, history -c, /dev/null bash_history |
| [Clearing Tracks Windows](ine/post-exploitation/Clearing-Tracks-Windows/) | BadBlue RCE, Meterpreter clearev Windows Event Log |

### Pivoting

| Lab | Key Technique |
|---|---|
| [Pivoting](ine/pivoting/Pivoting-INE/) | HFS RCE, autoroute route injection, TCP scan via pivot, portfwd, bind_tcp |
| [Pivoting with Metasploit](ine/pivoting/Pivoting-Metasploit/) | HFS RCE, autoroute, portscan/tcp pivoted, portfwd fingerprint, BadBlue bind shell |

---

## HackTheBox

Coming soon.

---

All labs were completed in controlled environments for educational purposes. See the disclaimer section in each writeup.
