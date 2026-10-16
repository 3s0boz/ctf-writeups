# Mr Robot CTF - TryHackMe

Linux machine running a WordPress site alongside SSH, with three keys to collect. The path
runs through a login form that leaks valid usernames, a brute-forced WordPress password, a
theme editor that doubles as a code execution primitive, a crackable MD5 hash found in a
user's home directory, and privilege escalation through a decades-old SUID `nmap` binary.

The target's IP changes across machine reboots - each command below uses the real IP from
the session it was run in.

---

## Reconnaissance

```bash
nmap -sV -sC 10.67.139.86
nmap -sV -sC -p- 10.67.139.86
```

A first scan on the top 1000 ports shows everything closed or filtered. A full port scan
run a minute later finds the same three ports open with full service detection - the
services simply weren't up yet on a freshly booted machine, not a difference between the
two commands.

Three services: SSH (22), HTTP (80, Apache with a WordPress install), and HTTPS (443, a
certificate for `www.example.com` expired a decade ago - not relevant to the chain).

---

## Enumeration

### robots.txt

Two entries: `key-1-of-3.txt`, read directly for the first key, and `fsocity.dic`, a large
wordlist full of duplicates.

```bash
sort -u fsocity.dic > dic_sort.txt
```

Deduplicating first avoids repeating the same guesses during brute forcing later.

### WordPress user enumeration

`wp-login.php` responds differently depending on whether the submitted username exists.
An invalid username returns `Invalid username`; a valid one with the wrong password returns
`The password you entered for the username elliot is incorrect`. Testing `elliot` confirms
it's a valid account. The same form would let a tool like Hydra enumerate valid usernames by
using `Invalid username` as the failure string.

### Password brute force

The machine was rebooted at this point, picking up a new IP:

```bash
hydra -l elliot -P dic_sort.txt 10.65.131.254 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:is incorrect" -f
```

This recovers a working password for `elliot` from the deduplicated wordlist.

---

## Initial Access

Logging into `wp-admin` with the recovered credentials, the account has access to the theme
editor. The default theme's `404.php` template is writable, so pasting the Pentest Monkey
PHP reverse shell over its contents and requesting a non-existent page triggers it:

```bash
nc -lnvp <port>
```

Shell as the user `daemon`.

---

## Local Enumeration and Privilege Pivot

The home directory holds two files: `key-2-of-3.txt` and `password.raw-md5`.

```
$ cat key-2-of-3.txt
cat: key-2-of-3.txt: Permission denied
$ cat password.raw-md5
robot:<md5 hash>
```

The key isn't readable as `daemon`, but the file next to it holds an MD5 hash for the user
`robot`. Cracking it (CrackStation) recovers the password:

```
robot:abcdefghijklmnopqrstuvwxyz
```

```bash
ssh robot@10.65.131.254
```

SSH as `robot` gives access to the second key.

---

## Privilege Escalation

Enumerating SUID binaries as `robot`:

```bash
find / -perm -4000 -type f 2>/dev/null
```

`/usr/local/bin/nmap` stands out: a dated build (3.81) with the SUID bit set. Modern GTFOBins
syntax for escaping nmap's interactive mode (`!sh`) doesn't apply here - this old version
runs typed commands directly, with the binary's own privileges:

```
nmap --interactive
nmap> whoami
root
nmap> cat /root/key-3-of-3.txt
```

Root, and the third key.

---

## Key Takeaways

- A login form whose error message differs for an invalid versus a valid username is a
  direct user enumeration channel, independent of any password attempt.
- A theme editor reachable from `wp-admin` is a direct code execution path: overwriting a
  template file like `404.php` needs no plugin, no upload feature, nothing beyond edit
  access to an existing theme.
- Older SUID binaries don't necessarily follow current GTFOBins escape syntax. This nmap
  3.81 interactive mode ran commands typed without any prefix, directly as root - worth
  testing a binary's actual behavior before assuming the modern exploitation syntax applies.

---

## Disclaimer

This lab was completed in a controlled environment provided by TryHackMe. All actions were
performed strictly for educational purposes.
