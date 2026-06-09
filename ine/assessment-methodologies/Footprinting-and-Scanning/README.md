# Footprinting and Scanning - INE eJPT

INE lab from the Assessment Methodologies module. The objective is to practice the reconnaissance phase of a penetration test: identifying live hosts, mapping open ports, detecting services and versions, and running targeted enumeration scripts - without performing any exploitation.

---

## Host Discovery

Identify live hosts within the target subnet before scanning individual machines:

```bash
nmap -sn 10.0.0.0/20
```

The `-sn` flag performs a ping sweep only - no port scanning. This avoids unnecessary noise and respects scope boundaries (e.g. excluding gateway addresses).

---

## Port Scanning

Once live hosts are identified, scan for open TCP ports:

```bash
nmap -sV <target>
```

Key distinctions to read in results:

- `open` - port is accessible and a service is listening
- `closed` - port is reachable but no service is listening
- `filtered` - port is blocked by a firewall or packet filter

For a faster initial scan followed by a version-detected deep scan on found ports:

```bash
nmap -p- --min-rate 5000 <target> -oN ports.txt
nmap -sV -sC -p <found_ports> <target>
```

---

## Service Enumeration

Version detection with NSE default scripts provides immediate context on running services:

```bash
nmap -sC -sV <target>
```

Useful NSE categories for targeted enumeration:

```bash
nmap --script vuln <target>          # known vulnerability checks
nmap --script auth <target>          # default/anonymous authentication
nmap --script discovery <target>     # additional service details
```

---

## SMB Enumeration

SMB is a high-value target during enumeration. NSE scripts allow deep inspection without requiring exploitation:

```bash
nmap -p 445 --script smb-enum-shares,smb-enum-users <target>
```

Additional useful SMB scripts:

```bash
nmap -p 445 --script smb-security-mode <target>    # guest access, signing
nmap -p 445 --script smb-os-discovery <target>     # OS and domain info
```

---

## Key Takeaways

- Host discovery and port scanning are distinct phases. Running a full port scan before confirming which hosts are alive wastes time and generates unnecessary noise.
- Firewall rules change how scan results look. A `filtered` result is not a dead end - it means a packet filter is present.
- NSE scripts provide structured enumeration without exploiting anything. They are the correct tool for the reconnaissance phase.
- SMB enumeration via Nmap exposes misconfigurations (guest access, unsigned packets, exposed shares) that become exploitation vectors in later phases.
- Methodology matters more than speed. Reading and adapting to scan output is the skill being trained here.

---

## Disclaimer

This lab was completed in a controlled environment provided by INE as part of the eJPT preparation path. All actions were performed strictly for educational purposes.
