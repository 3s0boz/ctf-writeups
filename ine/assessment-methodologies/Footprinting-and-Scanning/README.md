# Footprinting & Scanning — INE eJPT

## 📌 Context

This write-up documents my work on the **Footprinting & Scanning** module from the **INE eJPT (Junior Penetration Tester)** training path.

The purpose of this lab was to practice the **early stages of a penetration test**, focusing on reconnaissance, host discovery, and service enumeration using industry-standard tools.

This is not a competitive CTF write-up, but a **methodology-driven lab walkthrough**, aligned with real-world penetration testing workflows.

---

## 🎯 Objective

The goal of this lab was to identify live hosts, discover exposed services, and gather useful information about the target systems without performing exploitation.

The focus was on:
- Understanding how different scanning techniques work
- Choosing the correct approach based on the network context
- Correctly interpreting scan results
- Building a clear attack surface map for later phases

---

## 🧭 Methodology

The lab followed a structured reconnaissance workflow:

1. **Host Discovery**
   - Identify which hosts are alive within the target subnet
   - Avoid unnecessary noise and false positives
   - Respect scope restrictions (e.g. excluding gateway addresses)

2. **Port Scanning**
   - Detect open TCP ports on discovered hosts
   - Distinguish between filtered, closed, and open ports
   - Adjust scan techniques for performance and reliability

3. **Service & Version Detection**
   - Identify running services and versions
   - Correlate services with potential attack vectors
   - Prepare the ground for deeper enumeration

4. **Service-Specific Enumeration**
   - Use targeted scripts and techniques (e.g. SMB enumeration)
   - Extract meaningful information such as users, shares, and configurations

---

## 🛠 Tools Used

- **Nmap**
  - Host discovery
  - Port scanning
  - Service and version detection
  - Nmap Scripting Engine (NSE)

- **Zenmap**
  - GUI-based network scanning
  - Visualization of scan results
  - Easier exploration of large subnets

---

## 🔍 Key Commands & Techniques

### Host Discovery

nmap -sn 10.0.0.0/20

### Port Scanning

nmap -sV <target>

### SMB Enumeration (NSE example)

nmap -p445 --script smb-enum-shares,smb-enum-users <target>

## 🧠 Key Takeaways

- Proper reconnaissance is critical before exploitation
- Host discovery techniques behave differently depending on firewall rules
- NSE scripts provide powerful, automated enumeration when used correctly
- Scanning is not about running one command, but about reading and adapting results
- Clean methodology matters more than speed
