# Pi-hole DNS Ad-Blocking Server  
(Local DNS Filtering + Network-Wide Ad Blocking)

---

## Overview

This project documents the deployment of **Pi-hole** on a Raspberry Pi to provide network-wide DNS filtering for improved privacy and security.

The implementation includes secure headless configuration, static IP assignment, SSH hardening, and integration with the network gateway to enforce DNS filtering across the entire environment.



### Why Pi-hole?

Pi-hole is a free, open-source network-wide DNS filtering solution that blocks advertisements, trackers, and known malicious domains before they reach client devices.
This was one of my first projects, and it remains one of the most rewarding. Not only did it introduce me to foundational networking and Linux concepts, it also improved usability of the network.

To learn more about Pi-hole, visit the official resources:

* **Official Website:** https://pi-hole.net/
* **Documentation:** https://docs.pi-hole.net/
* **GitHub Repository:** https://github.com/pi-hole/pi-hole


---

## Envirnment Configuration

> [!NOTE]
> The concepts and deployment process are platform-independent and can be adapted to alternative hardware, operating systems, or equivalent software where appropriate.


- ### Hardware Used

  - Raspberry Pi 
  - microSD card
  - USB power supply  
  - Separate workstation (for flashing OS and headless administration)


- ### Software Used

  - Raspberry Pi OS Lite  
  - Pi-hole (official installation script)  
  - OpenSSH  


- ### Network Configuration

  | Interface | IPv4 Address    | Default Gateway | DNS Server  | Description               |
  | --------- | --------------- | --------------- | ----------- | ------------------------- |
  | `eth0`    | `192.0.2.10/24` | `192.0.2.1`     | `127.0.0.1` | Wired Ethernet interface. |
  | `wlan0`   | `192.0.2.10/24` | `192.0.2.1`     | `127.0.0.1` | Wireless Wi-Fi interface. |

> [!NOTE]
> The values shown are RFC 5737 documentation examples, and have been documented for consistency and demonstration purposes only.


---

## Process

### 1. Flash and Configure Operating System

Using Raspberry Pi Imager, flash **Raspberry Pi OS Lite** to the microSD card.

During the advanced configuration step, preconfigure:

- Enable SSH  
- Configure Wi-Fi SSID and passphrase (if using Wi-Fi)  
- Set hostname (e.g. `pihole`)  
- Configure locale, keyboard, and timezone as required 

Insert the microSD card into the Raspberry Pi and power it on.

---

### 2. Connect via SSH

Access the device remotely:

```bash
ssh pi@192.0.2.10
```

---

### 3. Update System Packages

Bring the system up to date:

```bash
sudo apt update && sudo apt upgrade -y
```

---

### 4. Secure Administrative Access

Change the default user password:

```bash
passwd
```

Set up [SSH key-based authentication](../..//secure/ssh-key-authentication/README.md)

(Optional) Install and configure fail2ban or ufw for additional SSH hardening, especially if your network is not already segmented.

---

### 5. Install Pi-hole

Install Pi-hole using the official installer:

```bash
curl -sSL https://install.pi-hole.net | bash
```

Follow the guided installation prompts and select the desired upstream DNS provider.

---

### 6. Assign an IP Address

Configure a static IP address to ensure consistent DNS availability.

Edit the DHCP client configuration:

```bash
sudo nano /etc/dhcpcd.conf
```

For Ethernet:

```ini
interface eth0
static ip_address=192.0.2.10/24
static routers=192.0.2.1
static domain_name_servers=127.0.0.1
```

For Wi-Fi:

```ini
interface wlan0
static ip_address=192.0.2.10/24
static routers=192.0.2.1
static domain_name_servers=127.0.0.1
```

Restart networking:

```bash
sudo systemctl restart dhcpcd
```

Alternatively, configure a DHCP reservation on the router to assign a fixed IP address to the Pi-hole device.

---

### 7. Access the Pi-hole Dashboard

Access the administrative interface:

```text
http://192.0.2.10/admin
```

The default blocklist used:

```text
https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts
```

This community-maintained list consolidates advertising, tracking, and known malicious domains to provide effective baseline DNS filtering.

---

### 8. Configure Network DNS

Configure the router to use Pi-hole as the primary DNS server:

Primary DNS: `192.0.2.10`

A secondary DNS server is intentionally not configured, as clients may bypass Pi-hole if an alternative resolver is available.

---

### 9. Firewall Considerations

To keep Pi-hole secure and prevent DNS bypass, basic firewall controls should be applied.

Recommended settings:

- Allow DNS traffic (TCP/UDP port 53) to `192.0.2.10` from internal LAN devices only  
- Do not expose port 53 to the internet (WAN)  
- Restrict SSH (port 22) access to trusted administrative devices (and set up [SSH key-based authentication](../..//secure/ssh-key-authentication/README.md))
- Optionally block outbound DNS traffic from clients to external resolvers (e.g. 8.8.8.8) to ensure all DNS queries pass through Pi-hole. Although try to remember these rules exist if your Pi-hole goes down...

These measures help prevent DNS bypass and reduce the risk of the resolver being misused.


---

## Outcome

Pi-hole is now deployed as a secure, network-wide DNS filtering solution.

By routing DNS traffic through the Raspberry Pi:

- Advertising domains are blocked at the DNS layer  
- Tracking and telemetry are reduced  
- Unwanted or malicious domains are prevented from resolving  
- Centralised DNS visibility and logging are achieved  

Additional enhancements such as custom blocklists and log analysis can be implemented as required.

Enjoy fewer ads!


## Pi-hole upgrade 
(PoE migration, VLAN segmentation, USB storage enhancement)

For details on the PoE migration, VLAN segmentation, and USB storage enhancements, see:
[Pi-hole Upgrade](./upgrade/README.md)



