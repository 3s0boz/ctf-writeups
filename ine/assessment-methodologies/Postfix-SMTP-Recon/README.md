# Postfix SMTP Recon - INE eJPT

INE lab covering SMTP service reconnaissance against a Postfix mail server. The objective is pure enumeration: identify the server and banner, grab the hostname, verify user existence via VRFY, enumerate SMTP capabilities with EHLO, and run automated user discovery. No exploitation or initial access is performed.

Target: `demo.ine.local` - Postfix SMTP on port 25
Server hostname (from banner): `openmailbox.xyz`

---

## Enumeration

### Service and Banner Detection

```bash
nmap -sV --script banner demo.ine.local
```

Result: Postfix SMTP on port 25, banner discloses `openmailbox.xyz ESMTP Postfix`.

### Manual Connection and Hostname Confirmation

```bash
nc demo.ine.local 25
```

The banner discloses the actual mail server hostname. This enables building valid email addresses for subsequent VRFY checks.

### SMTP Capability Enumeration (EHLO)

```bash
telnet demo.ine.local 25
EHLO attacker.xyz
```

`EHLO` returns the full list of SMTP extensions supported. Always use `EHLO` instead of `HELO` - `HELO` returns a minimal response and misses capability disclosure.

### User Enumeration - Manual (VRFY)

```bash
VRFY admin@openmailbox.xyz
VRFY commander@openmailbox.xyz
```

- `admin` - exists (server returns 252 or 250)
- `commander` - does not exist (server returns 550)

`VRFY` enabled is a misconfiguration that allows user enumeration without authentication.

### User Enumeration - smtp-user-enum

```bash
smtp-user-enum -U /usr/share/commix/src/txt/usernames.txt -t demo.ine.local
```

Result: 8 valid users identified.

### User Enumeration - Metasploit

```bash
msfconsole -q
use auxiliary/scanner/smtp/smtp_enum
set RHOSTS demo.ine.local
run
```

Result: 22 valid users identified.

### Manual Mail Submission Test

```bash
telnet demo.ine.local 25
HELO attacker.xyz
mail from: admin@attacker.xyz
rcpt to: root@openmailbox.xyz
data
Subject: Test
Test message body.
.
```

Or via `sendemail`:

```bash
sendemail -f admin@attacker.xyz \
  -t root@openmailbox.xyz \
  -s demo.ine.local \
  -u "Test" \
  -m "Hello" \
  -o tls=no
```

---

## Key Takeaways

- Banner disclosure from SMTP provides the server hostname, which enables building valid email addresses for VRFY checks and targeted phishing in real assessments.
- `VRFY` enabled is a direct user enumeration channel. Any server that responds differently to valid versus invalid addresses is vulnerable.
- `EHLO` is always preferred over `HELO` - it requests the full capability list and exposes supported authentication mechanisms and extensions.
- Manual SMTP interaction before running automated tools builds understanding of the protocol and prevents misinterpreting tool output.
- The user list produced by SMTP enumeration feeds directly into password spraying against any other service on the same infrastructure.

---

## Disclaimer

This lab was completed in a controlled environment provided by INE as part of the eJPT preparation path. All actions were performed strictly for educational purposes.
