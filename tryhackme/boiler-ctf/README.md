# Boiler CTF - TryHackMe

Ubuntu 16.04 machine (hostname `Vulnerable`) exposing FTP, a default Apache page with a
Joomla install, Webmin, and SSH on a non-standard port. The path runs through anonymous
FTP and web enumeration to a log file with plaintext SSH credentials, a pivot to a second
user via a backup file, and privilege escalation through a SUID binary that shows up in
the very first enumeration command.

Target: `10.67.131.212`

---

## Reconnaissance

```bash
nmap -sV -sC -p- 10.67.131.212 -T5
```

Four services: FTP (vsftpd 3.0.3, port 21), HTTP (Apache 2.4.18, port 80), a Webmin
instance (MiniServ 1.930, port 10000, no title), and SSH (OpenSSH 7.2p2) on port 55007
instead of the default 22.

---

## Enumeration

### FTP

Anonymous login is allowed:

```bash
ftp 10.67.131.212
```

The only file inside is `.info.txt`, encoded in ROT13:

```bash
cat .info.txt
```

Decoding it with CyberChef gives:

```
Just wanted to see if you find it. Lol. Remember: Enumeration is the key!
```

No credentials, just a message pointing back to enumeration.

### Web

```bash
gobuster dir -u http://10.67.131.212 -w /usr/share/dirb/wordlists/common.txt
```

Notable paths: `/joomla` (301), `/manual` (301), `robots.txt` (200, listing decoy
directories that lead nowhere), `/server-status` (403). The real find is deeper in the
Joomla install and its exposed paths: a log file containing an SSH authentication entry
in plaintext.

```
Accepted password for basterd ... #pass: superduperp@$$
```

---

## Initial Access

```bash
ssh basterd@10.67.131.212 -p 55007
```

The credentials from the log file work directly on the non-standard SSH port.

---

## Local Enumeration and Pivot

`basterd`'s home directory holds a backup file with credentials for a second user,
`stoner`. Switching to that account gives access to the user flag:

```bash
cat /home/stoner/.secret
```

---

## Privilege Escalation

`sudo -l` for `stoner` shows a NOPASSWD entry on `/NotThisTime/MessinWithYa`, but no
binary exists at that path, so sudo still prompts for a password and the entry is a dead
end.

The actual vector was already visible in the first privesc command run:

```bash
find / -perm -4000 2>/dev/null
```

`/usr/bin/find` is in the output, with the SUID bit set. GTFOBins documents the
exploitation directly:

```bash
find . -exec /bin/sh -p \; -quit
```

Root shell, with the root flag under `/root/root.txt`.

---

## Key Takeaways

- SUID bits on common system binaries like `find` are worth checking first in Linux
  privilege escalation - the output of `find / -perm -4000` is easy to run and easy to
  skim past without spotting the one entry that matters.
- Credentials leaked in log files and backup copies are a common lateral movement path
  between users on the same box.
- A NOPASSWD sudo rule doesn't help if the target binary doesn't exist at that path.

---

## Disclaimer

This lab was completed in a controlled environment provided by TryHackMe. All actions
were performed strictly for educational purposes.
