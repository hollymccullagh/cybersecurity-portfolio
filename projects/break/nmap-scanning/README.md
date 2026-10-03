# Reconnaissance & Enumeration with Nmap

## Objective

This project documents the use of Nmap and Hydra within an isolated lab environment to perform network reconnaissance, service enumeration, and controlled authentication testing against a deliberately vulnerable virtual machine. 
The objective was to develop practical understanding of exposed services, weak credentials, attack surface identification, and how offensive security activity may appear from both attacker and defensive perspectives.

For documentation purposes, the attacker system is referenced as `198.51.100.10` and the target environment as `192.0.2.0/24`, with the primary vulnerable host referenced as `192.0.2.10`. This is in accordance with RFC 5737 reserved documentation address ranges.

| System | Role | Documentation IP |
|---|---|---|
| Kali Linux VM | Attacking / Scanning Host | 198.51.100.10 |
| Vulnerable VM | Vulnerable Target Host | 192.0.2.10 |


---
## Tools & Environment

• Nmap – Network discovery, port scanning, service enumeration, NSE scripting, and operating system fingerprinting  
• Hydra – Credential brute forcing and authentication testing against exposed services  
• Kali Linux – Attacking host and testing platform  
• Metasploitable 2 – Intentionally vulnerable target system used for security testing and enumeration practice  
• VMware Workstation Pro – Virtualisation platform used to isolate the lab environment  
• Host-only virtual network – Isolated lab network preventing internet or LAN exposure  
• Wireshark – traffic inspection, authentication analysis, and protocol inspection  


----
## Scans Used

### Host Discovery Scan

Used to identify active hosts within the isolated lab subnet prior to deeper enumeration activities.

```bash
sudo nmap -sn 192.0.2.0/24
```

This scan performs host discovery without conducting a port scan. Nmap primarily uses ICMP echo requests, ARP requests, and other lightweight discovery techniques to determine whether systems are online and responding on the network.

This stage demonstrated how attackers and administrators can quickly map reachable devices within a subnet before focusing on individual hosts. It also reinforced the importance of network segmentation and limiting unnecessary host visibility within internal environments.

Key Learning:
- Difference between host discovery and service enumeration
- How systems respond to ICMP and ARP traffic
- Initial reconnaissance methodology prior to deeper scanning
- How subnet scanning quickly identifies potential targets

---

### TCP SYN Scan

Performed a stealth-oriented TCP SYN scan to identify open TCP ports without fully completing the TCP three-way handshake.

```bash
sudo nmap -sS 192.0.2.10
```

The SYN scan works by sending SYN packets to target ports and analysing returned responses. If a SYN-ACK response is received, the port is considered open. Rather than completing the handshake with a final ACK packet, Nmap resets the connection instead.

This scan demonstrated how attackers can identify exposed services while reducing the amount of fully established connections appearing in service logs. It also reinforced understanding of the TCP three-way handshake and how different port states behave at the network level.

Key Learning:
- TCP handshake behaviour and connection establishment
- Difference between open, closed, and filtered ports
- How SYN scans differ from full TCP connect scans
- Why SYN scanning is often considered more stealth-oriented

---

### Full TCP Port Scan

Scanned all 65,535 TCP ports to identify additional services not exposed on common ports.

```bash
sudo nmap -sS -p- 192.0.2.10
```

By default, Nmap scans only a subset of commonly used ports. Using `-p-` forces Nmap to scan the entire TCP port range, potentially identifying hidden, forgotten, or non-standard services.

This demonstrated how relying solely on common-port scanning may overlook accessible services and how non-standard ports can still expose vulnerable applications.

Key Learning:
- Difference between common-port scans and full-port enumeration
- How services can operate on non-standard ports
- Importance of complete attack surface visibility
- Risks associated with legacy or forgotten services

---

### Service & Version Detection

Performed service fingerprinting to identify running applications and software versions.

```bash
sudo nmap -sV 192.0.2.10
```

This scan sends carefully crafted probes to detected services and analyses returned responses to identify application names, versions, and protocol details.

The scan demonstrated how exposed version information can assist vulnerability research by revealing outdated or insecure software versions that may be publicly associated with known CVEs or exploits.

Key Learning:
- Service fingerprinting and banner analysis
- How software versions assist vulnerability identification
- Risks associated with version disclosure
- Importance of patch management and software maintenance

---

### Default Script Scan

Executed default Nmap NSE (Nmap Scripting Engine) scripts against identified services.

```bash
sudo nmap -sC 192.0.2.10
```

The Nmap Scripting Engine allows automated interaction with detected services to gather additional information and identify common misconfigurations. Default scripts may check for anonymous access, insecure configurations, weak encryption support, exposed shares, or unsafe default behaviour.

This stage demonstrated how enumeration extends beyond identifying open ports and into analysing how services are configured and exposed.

Key Learning:
- Importance of service enumeration beyond simple port discovery
- How automated scripts gather deeper configuration information
- Risks associated with insecure default settings
- Exposure created by weak or anonymous access controls

---

### Operating System Fingerprinting

Used TCP/IP stack fingerprinting techniques to estimate the target operating system.

```bash
sudo nmap -O 192.0.2.10
```

Nmap analyses subtle differences in how operating systems implement TCP/IP networking behaviour, including packet responses, TTL values, window sizes, and protocol handling.

This demonstrated how systems unintentionally expose identifiable characteristics through network behaviour alone, even without authenticated access.

Key Learning:
- TCP/IP stack fingerprinting concepts
- How operating systems differ in network behaviour
- Passive information leakage through protocol implementation
- Importance of minimising exposed information where possible

---

### Aggressive Scan

Combined multiple enumeration stages into a single comprehensive scan.

```bash
sudo nmap -A 192.0.2.10
```

The aggressive scan enables operating system detection, version detection, default scripting, and traceroute functionality simultaneously. While highly informative, it also produces significantly more traffic and is generally much more visible from a defensive monitoring perspective.

This demonstrated the trade-off between scan depth and operational stealth, reinforcing how broad automated enumeration may trigger alerts within monitored environments.

Key Learning:
- Consolidated reconnaissance methodology
- Trade-offs between information gathering and stealth
- Increased visibility of aggressive scanning behaviour
- How noisy scans may appear within IDS, firewall, or honeypot logs

---

### UDP Scan

Performed UDP enumeration against common UDP services.

```bash
sudo nmap -sU 192.0.2.10
```

Unlike TCP, UDP is connectionless and does not use a handshake mechanism. This makes UDP scanning slower and less reliable, as open ports often do not respond unless specific application data is sent.

This scan demonstrated the challenges associated with UDP enumeration and reinforced understanding of protocol differences between TCP and UDP communication.

Key Learning:
- Differences between TCP and UDP communication
- Why UDP scanning is slower and more difficult
- How connectionless protocols behave during enumeration
- Common UDP services such as DNS, SNMP, DHCP, and NTP

---

### Timing & Stealth Testing

Experimented with different timing templates and scan behaviour.

```bash
sudo nmap -T2 -sS 192.0.2.10
```

```bash
sudo nmap -T4 -sS 192.0.2.10
```

Nmap timing templates adjust the speed and aggressiveness of scanning activity. Slower scans may reduce detection likelihood but take significantly longer, whereas faster scans increase speed at the cost of visibility and reliability.

This demonstrated how scan behaviour itself can influence detection opportunities within monitored environments.

Key Learning:
- Relationship between scan speed and detection likelihood
- Trade-offs between stealth, speed, and reliability
- How timing changes network behaviour and visibility
- Importance of scan tuning within different environments

---

## Authentication Testing with Hydra

### SSH Credential Brute Force Testing

Performed controlled authentication testing against the intentionally vulnerable SSH service exposed by the target machine.

```bash
hydra -l msfadmin -P /usr/share/wordlists/rockyou.txt ssh://192.0.2.10
```

Hydra repeatedly attempts authentication using passwords supplied from the `rockyou.txt` wordlist. This demonstrated how weak or reused credentials may be compromised when internet-facing or internally exposed authentication services lack sufficient protection controls.

The test also reinforced how repeated failed login attempts may appear within authentication logs, IDS alerts, or defensive monitoring systems.

Key Learning:
- Risks associated with weak passwords
- Credential brute force methodologies
- Importance of MFA and account lockout controls
- Visibility of authentication attacks within logs and monitoring systems

---

### FTP Credential Testing

Tested FTP authentication security using common passwords against exposed FTP services.

```bash
hydra -l msfadmin -P /usr/share/wordlists/rockyou.txt ftp://192.0.2.10
```
The `rockyou.txt` password wordlist commonly included with Kali Linux was used for authentication testing within the isolated lab environment. On minimal Kali installations where the wordlist package may not be preinstalled, it can typically be obtained through the standard Kali `wordlists` package repository with the commands below.

```bash
sudo apt update
sudo apt install wordlists
```

Extract/decompress the wordlist into a usable `.txt` file:

```bash
sudo gzip -d /usr/share/wordlists/rockyou.txt.gz
```

This demonstrated how legacy protocols combined with weak authentication controls can significantly increase organisational risk. FTP additionally transmits credentials insecurely unless specifically protected, further reinforcing why insecure legacy services should be minimised or replaced where possible.

Key Learning:
- Risks associated with legacy authentication protocols
- Weak password exposure through brute force attacks
- Security limitations of FTP
- Importance of reducing unnecessary exposed services

  
