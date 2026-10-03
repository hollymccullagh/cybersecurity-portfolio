# Remote Linux Desktop Access – VNC Through SSH Tunnel  
*(Local-Only VNC for Secure Remote Access)*


---

## Overview

This project documents secure remote access to a graphical Linux desktop from a Windows workstation on the same local network.

To minimise attack surface, the VNC server is configured to accept **local connections only (`127.0.0.1`)**, preventing direct network exposure.

Remote desktop access is achieved by tunnelling VNC traffic through an SSH connection, ensuring:

- All traffic is encrypted in transit  
- The VNC service is inaccessible without SSH authentication  
- No VNC ports are exposed to the LAN  

This approach eliminates the risk associated with open VNC ports while still providing full graphical desktop control.

### Why VNC?

I needed to be able to access the GUI on a *remote* linux device, **on my LAN**, from another workstation.
Attempts to use **xrdp** resulted in persistent black screen issues. After troubleshooting without much success, I pivoted to using VNC. 

However, exposing VNC directly would have introduced unencrypted traffic on the network. Instead, I configured VNC to bind locally and accessed it via an SSH tunnel to ensure all traffic remained encrypted and authenticated.


---

## Tools

* **VNC Server** – Hosts a remote graphical desktop session on the target system.
* **VNC Client** – Connects to and displays the remote graphical desktop.
* **SSH** – Provides encrypted remote administration and securely tunnels VNC traffic between the client and server.


---

## Process

1. Install Server on Linux Host

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install #<vnc-server-package>
```

2. Start server bound to local host only

```bash
vncserver -localhost :1
```

3. Verify it's not exposed:

```bash
ss -tulpn | grep 5901
```

expected output:

```
LISTEN 127.0.0.1:5901
```

## SSH Port Forwarding

On the Windows side, open an SSH tunnel to forward the local VNC port across the encrypted channel. Because VNC traffic is not encrypted by default, tunnelling it through SSH ensures that data is protected in transit.

4. SSH Port Forwarding (Run in Powershell from Windows PC)

```powershell
ssh -L 5901:localhost:5901 linuxusername@192.0.2.10
```

(Here you’ll be prompted to enter your SSH password, OR if you have set up [SSH key-based authentication](../ssh-key-authentication/README.md#ssh-key-based-authentication), you will be prompted for your passphrase. No output will be displayed after authentication; this is expected. The terminal will appear to hang because it is maintaining the active SSH tunnel rather than returning to the prompt.)

5. With the SSH tunnel active, open the VNC client and connect to:

```
localhost:5901
```

All VNC traffic is now encapsulated within the SSH tunnel and encrypted in transit.

> [!NOTE]
> To ensure the port forwarding is working, and the traffic is encrypted, open Wireshark and ensure packets are flowing only through the SSH tunnel.
> Traffic should appear as encrypted SSH rather than clear-text VNC.


6. Kill server

When complete with the VNC session, use the following command on the Linux machine with the VNC server enabled to turn the server off.

```bash
vncserver -kill :1
```


---

## Outcome

This configuration provides secure remote graphical access to a Linux system without exposing VNC services to the network. By combining local-only VNC binding with SSH port forwarding, the solution reduces attack surface while ensuring encrypted, authenticated remote access.


