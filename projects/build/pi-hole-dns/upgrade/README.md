# Pi-hole DNS Ad-Blocking Server - Upgrade and Troubleshooting
(PoE & USB Hub hat and VLAN Segmentation)

## Overview

After successfully deploying the Pi-hole on the network, I decided to make some changes to enhance the reliability and security of the Pi-hole DNS server deployment.
Despite the Pi-hole not skipping a beat over wifi, I decided to install a PoE hat to improve cable management and introduce a USB port on the device for back ups, configs, logs, and scripts.

Additionally the server was reloacted to an isolated VLAN, and access was restricted through firewall rules to improve security and reduce the attack surface.


## Hardware Used

| Hardware | Purpose |
|----------|---------|
| Raspberry Pi | Hosts the Pi-hole DNS service. |
| Raspberry Pi PoE / USB HAT | Provides power, Ethernet connectivity, and USB expansion. |
| PoE-Compatible Switch | Supplies power and network connectivity via Power over Ethernet (PoE). |
| Cat6 Ethernet Cable | Connects the Raspberry Pi to the network. |
| USB Flash Drive | Stores backups and configuration archives. |
| Administration Workstation | Used for SSH administration and ongoing management. |


## Software Used

| Software             | Purpose                                             |
| -------------------- | --------------------------------------------------- |
| Raspberry Pi OS Lite | Lightweight operating system for the Raspberry Pi   |
| Pi-hole              | Network-wide DNS filtering and ad blocking          |
| OpenSSH              | Enables secure remote administration over SSH       |
| UFW                  | Host-based firewall used to restrict network access |


## Documentation IP Addressing

| Device                     | IP Address      |
| -------------------------- | --------------- |
| Previous Pi-hole Address   | `192.0.2.10`    |
| New Pi-hole Address        | `198.51.100.10` |
| Administration Workstation | `192.0.2.11`    |

> [!NOTE]
> The IP addresses used throughout this documentation are from the RFC 5737 documentation ranges and are provided for illustrative purposes only. 


---


## Process

### 1. PoE upgrade

#### Set alternative DNS server
Before installing the HAT, consider configuring an alternative DNS server on the network to minimise disruption during the upgrade.


#### Create a back up of Pi hole

It's also a good idea to create a backup of the Pi-hole configuration from the web interface before proceeding.

```text
http://192.0.2.10/admin
```
(Settings > Teleporter > Backup) and download the archive to a safe location.


#### Gracefully power off the Pi
SSH into the Pi
```bash
ssh pi@192.0.2.10
```

Update the Pi
```bash
sudo apt update
sudo apt upgrade -y
```

Power off the Pi
```bash
sudo poweroff
```


#### Install the PoE hat

Install the PoE HAT according to the manufacturer's instructions,ensuring the board is securely seated.

Ensure the port on the switch has PoE enabled, then connect an Ethernet cable from the PoE-enabled switch port to the HAT. 
This provides both network connectivity and power to the Raspberry Pi over a single cable.

Once installed, power on the Raspberry Pi and verify that the device boots successfully and obtains network connectivity.


---


### 2. Move to another VLAN

It is a good idea to place the Raspberry Pi in a dedicated VLAN, as DNS is a critical network service. This improves network segmentation, enhances security, and enables more granular firewall policies.

#### Create VLAN

Example VLAN configuration

| Setting                    | Value             |
| -------------------------- | ----------------- |
| VLAN Name                  | Infrastructure    |
| VLAN ID                    | 100               |
| Subnet                     | `198.51.100.0/24` |
| Gateway                    | `198.51.100.1`    |
| Pi-hole IP                 | `198.51.100.10`   |
| DHCP Mode                  | DHCP Reservation  |
| Native VLAN (Switch Port)  | 100               |
| Tagged VLANs (Switch Port) | None              |


#### Configure Switch Port
Configure the switch port connected to the Raspberry Pi to use the Infrastructure VLAN as the native (untagged) VLAN.


#### Assign a Static IP address to the Pi
| Device  | Previous IP  | New IP          |
| ------- | ------------ | --------------- |
| Pi-hole | `192.0.2.10` | `198.51.100.10` |


#### Create Firewall Rules
Create firewall rules to permit only the required traffic to the Pi-hole server.

Allow:

- DNS (TCP/UDP 53) from required VLANs.
- SSH (TCP 22) from trusted administration devices.
- HTTP (TCP 80) from trusted administration devices.
- HTTPS (TCP 443) from trusted administration devices.

Example Firewall Configuration

| Rule # | Source | Destination | Protocol | Port | Action | Description |
|---------|--------|-------------|----------|------|--------|-------------|
| 1 | `192.0.2.11` | `198.51.100.10` | TCP | 22 | Allow | Permit SSH administration |
| 2 | `192.0.2.11` | `198.51.100.10` | TCP | 80/443 | Allow | Permit access to the Pi-hole web interface |
| 4 | `192.0.2.0/24` | `198.51.100.10` | TCP/UDP | 53 | Allow | Permit DNS queries from client devices |
| 5 | `Any` | `198.51.100.10` | Any | Any | Deny | Block all other inbound traffic |


#### Verify connectivity
From the administration workstation, attempt to ping the device
```powershell
ping 198.51.100.10
```

Verify SSH connectivity from a trusted administration workstation:
```powershell
ssh pi@198.51.100.10
```
> [!NOTE]
> Because the Raspberry Pi address has changed, don't forget to update any SSH clients, saved sessions, scripts, or configuration files that reference the previous IP address.
> This may include `.ssh/config`, PuTTY saved sessions, automation tooling, or the local `.ssh/known_hosts` file.
> Failure to update these settings may prevent SSH connections or generate host key verification warnings.


Example:
```text
Host pihole
    HostName 198.51.100.10
    User pi
    IdentityFile ~/.ssh/id_ed25519
```



Then verify that the Pi-hole web interface is accessible by navigating to:
```text
http://198.51.100.10/admin
```

Confirm that Pi-hole is healthy and operational before configuring it as the DNS server for the subnet.
Check the status of the Pi-hole services on the Pi:

```bash
pihole status
```

Expected output:

```text
  [✓] FTL is listening on port 53
      [✓] UDP (IPv4)
      [✓] TCP (IPv4)

  [✓] Pi-hole blocking is enabled
```


---


### 3. Configure Pi-hole as the DNS Server

Once the Raspberry Pi has been successfully migrated to the Infrastructure VLAN and its health has been verified, configure client devices to use Pi-hole as their DNS server.
It is recommended to distribute the DNS server via DHCP rather than configuring each client manually.

Example configuration:

| Setting              | Value           |
| -------------------- | --------------- |
| Subnet               | `192.0.2.0/24`  |
| Primary DNS Server   | `198.51.100.10` |
| Secondary DNS Server | `None`          |

> [!NOTE]
> Avoid configuring public DNS servers (for example, `1.1.1.1` or `8.8.8.8`) as secondary DNS servers, as clients may bypass Pi-hole filtering.



#### Renew Client DHCP Leases and test DNS

After updating the DHCP scope, renew the DHCP lease before receiving the new DNS server settings.

```powershell
ipconfig /release
ipconfig /renew
```


Verify that the client has received the correct DNS server:

```powershell
ipconfig /all
```

Expected output:

```text
DNS Servers . . . . . . . . . . . : 198.51.100.10
```


#### Validate DNS Resolution

From a client device, confirm that DNS queries are being resolved through Pi-hole:
```powershell
nslookup google.com
```
The output should indicate that the DNS server being used is `198.51.100.10`.

Alternatively, manually navigate to several websites and confirm that they load successfully.
If clients continue to use the previous DNS server, clear the local DNS cache and renew the DHCP lease.
```powershell
ipconfig /flushdns
ipconfig /release
ipconfig /renew
```


---


## Troubleshooting

If following the migration to the Infrastructure VLAN, client devices are unable to resolve DNS queries through Pi-hole.

### Symptoms

* Client devices report **"Connected without Internet"**.
* DNS queries from devices fail


### Firewall and Network Checklist


Use the following checklist to validate the network configuration:

* [ ] Verify network connectivity from the Raspberry Pi:
```bash
ping google.com
```
* [ ] Verify that Pi-hole is listening on port 53:
```bash
sudo ss -tulpn | grep :53
```
* [ ] Confirm no additional hardening or firewall rules are blocking traffic.
* [ ] Confirm the switch port is assigned to the correct native VLAN.
* [ ] Confirm the Raspberry Pi has obtained the expected IP address.
* [ ] Confirm clients have received the correct DNS server via DHCP.
* [ ] Confirm inter-VLAN firewall rules permit DNS traffic (TCP/UDP 53).
* [ ] Confirm clients can successfully resolve DNS queries using `nslookup`.
* [ ] Confirm DNS queries appear in the Pi-hole Query Log.


If the Raspberry Pi has network connectivity, Pi-hole services are operational, and firewall rules permit DNS traffic, investigate how Pi-hole is handling requests originating from other VLANs. Review the Pi-hole DNS listening mode:

```bash
sudo pihole-FTL --config dns.listeningMode
```

Example output:

```text
dns.listeningMode = LOCAL
```

If the Raspberry Pi has been migrated to a dedicated VLAN, a listening mode of `LOCAL` will prevent DNS queries originating from other VLANs from being processed.
Modify the listening mode as required and restart the Pi-hole FTL service before retesting DNS resolution from client devices.

Modify the listening mode as required for the environment:
```bash
sudo pihole-FTL --config dns.listeningMode 'ALL'
```

Restart the Pi-hole FTL service and retest DNS resolution from client devices:
```bash
sudo systemctl restart pihole-FTL
```

The following listening modes are available:

| Mode     | Description                                                                                                 |
| -------- | ----------------------------------------------------------------------------------------------------------- |
| `LOCAL`  | Accept DNS queries only from devices on directly connected local networks.                                  |
| `SINGLE` | Accept queries on a single interface and its associated subnets. Suitable for many multi-VLAN environments. |
| `BIND`   | Bind to all interfaces while still restricting responses to local networks.                                 |
| `ALL`    | Accept DNS queries from all interfaces and all origins. Use with caution and appropriate firewall rules.    |


---


## 4. Configure USB Storage for back ups and recovery

Using external USB storage for backups helps protect important Pi-hole configuration data from SD card failure and simplifies recovery.

### Connect the USB Drive

Insert the USB drive into the USB port on the PoE HAT.
Verify that the Raspberry Pi detects the device:

```bash
lsblk
```


Example output:

```text
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda           8:0    1  28.9G  0 disk
└─sda1        8:1    1  28.9G  0 part
mmcblk0     179:0    0  29.7G  0 disk
├─mmcblk0p1 179:1    0   512M  0 part /boot/firmware
└─mmcblk0p2 179:2    0  29.2G  0 part /
```


### Identify the USB Device

Confirm the USB drive identifier:
```bash
sudo fdisk -l
```

In this example, the USB device is:
```text
/dev/sda1
```


### Mount the USB Drive
Create a directory that will be used to mount the USB drive:

```bash
sudo mkdir -p /mnt/storage
```

Mount the USB drive:
```bash
sudo mount /dev/sda1 /mnt/storage
```

Verify that the drive has been mounted successfully:
```bash
df -h
```

Expected output:
```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        29G   32M   27G   1% /mnt/storage
```

#### Configure Persistent Mounting

Retrieve the UUID of the USB drive:
```bash
sudo blkid
```

Example output:
```text
/dev/sda1: UUID="1234-ABCD" TYPE="ext4"
```

Edit the filesystem table:
```bash
sudo nano /etc/fstab
```

Add the following entry, replacing the UUID with the value returned by `blkid`:
```text
UUID=1234-ABCD /mnt/storage ext4 defaults,nofail 0 2
```

Test the configuration before rebooting:
```bash
sudo mount -a
```

If no errors are displayed, reboot the Raspberry Pi and verify that the drive is automatically mounted.
```bash
sudo reboot
```


---


## Summary

Although the Raspberry Pi operated reliably over Wi-Fi with very few issues, migrating to Ethernet provided the benefits of a dedicated wired connection and improved cable management with PoE. In practice, the most noticeable improvement was that the Pi-hole service became available slightly faster following gateway or network device reboots.


### Benefits

* Improved reliability by replacing Wi-Fi connectivity with wired Ethernet.
* Simplified power and networking through the use of PoE.
* Improved network security through VLAN segmentation and firewall policies.
* Reduced attack surface by restricting management access to trusted devices.
* Improved resilience by storing backups on external USB storage.
* Simplified recovery in the event of SD card failure.


### Future Enhancements

Future improvements could include automated backups, a documented recovery procedure, and a secondary Pi-hole instance for DNS redundancy. Centralised logging and monitoring through Wazuh or another SIEM service could provide greater visibility into system activity, security events, and service health, while network boot could be investigated to reduce reliance on local storage.
These improvements would increase reliability and security while providing hands-on experience with enterprise concepts such as redundancy, monitoring, logging, backup, and disaster recovery.





