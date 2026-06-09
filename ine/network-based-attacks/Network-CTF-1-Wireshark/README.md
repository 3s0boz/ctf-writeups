# Network-Based Attacks CTF 1: Wireshark Forensics - INE eJPT

INE network forensics CTF. The objective is to analyze a captured network traffic file (`test.pcap`) to identify a compromised Windows client, extract attacker infrastructure details, and recover artifacts from PowerShell execution and browser extension activity. No exploitation is performed - the entire challenge is traffic analysis.

---

## Analysis Setup

```bash
wireshark test.pcap
```

---

## Flag 1 - Domain Returning HTTP 200

Filter for successful HTTP responses:

```
http.response.code == 200
```

Select a packet, expand "Hypertext Transfer Protocol", locate the "Request URI" field. Extract the root domain from the URI.

```
623start.site
```

---

## Flag 2 - Infected Client IP and MAC Address

Filter for HTTP traffic to isolate client activity:

```
http
```

Suspicious requests originate from a single source IP. Expand "Ethernet II" to obtain the MAC address.

```
10.7.10.47, 80:86:5b:ab:1e:c4
```

---

## Flag 3 - Victim Hostname via NetBIOS

Apply NetBIOS Name Service filter:

```
nbns
```

Expand "NetBIOS Name Service" and the "Queries" section. The source hostname in registration traffic identifies the infected machine.

```
Filter: nbns
Hostname: DESKTOP-9PEA63H
```

---

## Flag 4 - User Who Executed the PowerShell Script

Remove all filters. Press `Ctrl+F`, search for the string `mystery_file.ps1` with search type set to "Packet Bytes". Right-click the matching packet, select "Copy as Printable Text". Examine the output in a text editor.

```
rwalters
```

---

## Flag 5 - PowerShell User-Agent

Press `Ctrl+F`, search for `PowerShell` with search type "Packet Details". Expand "Hypertext Transfer Protocol" on the matching request, locate the "User-Agent" field.

```
WindowsPowerShell
```

---

## Flag 6 - Coinbase Extension ID

Press `Ctrl+F`, search for `Coinbase` with search type "Packet Bytes". Right-click the relevant request, select "Follow - TCP Stream". Locate the extension ID in the stream output.

```
hnfanknocfeofbddgcijnmhnfnkdnaad
```

---

## Key Takeaways

- Wireshark is an incident response tool as much as a network troubleshooting tool. Traffic captures from a compromised host expose the full attack chain.
- `http.response.code == 200` filters to successful outbound connections - useful for identifying C2 domains or malicious download sources.
- `nbns` filter reveals the Windows hostname from NetBIOS registration broadcasts. IP and MAC correlation builds a complete victim identity.
- PowerShell execution artifacts appear in HTTP User-Agent headers when PowerShell makes web requests. The string `WindowsPowerShell` or `Microsoft BITS` identifies automated activity.
- `Ctrl+F` with "Packet Bytes" search mode retrieves script content, usernames, and other strings embedded in raw packet data.
- Browser extension IDs in traffic correlate to installed extensions - relevant for detecting crypto wallet stealers or credential-harvesting extensions.

---

## Disclaimer

This lab was completed in a controlled environment provided by INE as part of the eJPT preparation path. All actions were performed strictly for educational purposes.
