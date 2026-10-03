# SSH Key-Based Authentication  
*(SSH Key-based authentication to replace passwords)*



## Overview

Using Key-based authentication for SSH is best practice. This process replaces password-based SSH authentication with public key authentication. Instead of typing a password when connecting, the client proves identity using a private key, while the server verifies it against a stored public key. This improves security by eliminating password exposure and brute-force risk associated with SSH.



## Hardware used

- Windows workstation (`192.0.2.10`)
- Linux server (`192.0.2.11`)



## Software used

- OpenSSH



## Process

### 1. Generate SSH Key Pair (Windows Client)

On the Windows workstation (PowerShell), type:

```powershell
ssh-keygen -t ed25519 -a 100 -C "user@client" -f ~/.ssh/id_ed25519
```


**Parameters:**

-t ed25519 Modern, secure elliptic curve algorithm  
-a 100 Increases key derivation rounds (protects passphrase against brute-force)  
-C Comment identifying the originating device  
-f Where to save and what to name the key
 
If you manage multiple keys on one device in the same location, you may want to differentiate by naming each for their purpose. Specifying algorithm type in the name is good practice.

When prompted press Enter to accept default location:

```
C:\Users\%USERNAME%\.ssh\id_ed25519
```

Enter a secure passphrase (recommended).

This creates:

- `id_ed25519` (Private key)  
- `id_ed25519.pub` (Public key)  

---

### 2. Copy Public Key to Linux Server

From PowerShell:

```powershell
scp $env:USERPROFILE\.ssh\id_ed25519.pub linux@192.0.2.11:/home/linux/
```

---

### 3. Configure Server Authorized Keys (Linux Server)

SSH into the Linux server:

```powershell
ssh linux@192.0.2.11
```

Create `.ssh` directory (safe even if it already exists):

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

Append Public Key to `authorized_keys`:

```bash
cat ~/id_ed25519.pub >> ~/.ssh/authorized_keys
```

Set correct permissions:

```bash
chmod 600 ~/.ssh/authorized_keys
```

Ensure correct ownership (recommended safeguard):

```bash
chown -R "$USER:$USER" ~/.ssh
```
This recursively sets your user as the owner and group of your .ssh directory and all its contents, then silently returns you to the prompt.

Remove temporary public key file:

```bash
rm ~/id_ed25519.pub
```

Please note SSH will refuse key authentication if:

- `~/.ssh` is more permissive than 700  
- `authorized_keys` is more permissive than 600  
- The files are owned by the wrong user  

---

### 4. Test Key Authentication

From Windows client:

```powershell
ssh linux@192.0.2.11
```

You should now:

- Be prompted for your key passphrase (if set)  
- Not be prompted for a server password  

If successful, key authentication is working.

---

### 5. Disable Password Authentication (After Testing)

On the Linux server:

```bash
sudo nano /etc/ssh/sshd_config
```

Find and modify:

```
PasswordAuthentication no
ChallengeResponseAuthentication no
```

Ensure:

```
PubkeyAuthentication yes
```

Save and exit.

Restart SSH:

```bash
sudo systemctl restart ssh
```

---

### 6. Confirm Password Login Is Disabled

Open a new PowerShell window and try:

```powershell
ssh linux@192.0.2.11
```

It should:

- Only allow key authentication  
- Reject password attempts  

---

### 7. SSH Client Configuration File

#### Overview

The SSH client configuration file (`~/.ssh/config`) allows predefined connection profiles to be created for remote systems. This simplifies SSH usage and ensures the correct private key, username, and connection settings are used automatically.

Instead of specifying parameters manually each time, the SSH client reads this file and applies the appropriate configuration based on the host alias used.

This improves efficiency, consistency, and security, especially when managing multiple systems.

---

#### File Location

Linux:

```bash
~/.ssh/config
```

Windows:

```powershell
C:\Users\%USERNAME%\.ssh\config
```

If the file does not exist, create it:

Linux:

```bash
nano ~/.ssh/config
```

Set secure permissions:

```bash
chmod 600 ~/.ssh/config
```

Windows:

```powershell
New-Item -Path $env:USERPROFILE\.ssh\config -ItemType File -Force
```
Then open it in Notepad to edit
```powershell
notepad $env:USERPROFILE\.ssh\config
```

Here's an example config:
```ssh
Host linux-server # SSH connection alias; used in place of the full hostname or IP address.
    HostName 192.0.2.11
    User linuxusername
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
```

When accessing multiple servers, it is best practice to use separate SSH key pairs for each system instead of reusing the same key everywhere. This reduces risk, as access to individual systems can be controlled or revoked without affecting others if a key is compromised. 

The SSH client configuration file (`~/.ssh/config`) allows you to define multiple host blocks, each with its own hostname, username, and private key. This improves security through proper key separation while also making connections more convenient, since you can use simple aliases like ssh linux-server or raspberrypi instead of remembering full connection details. 
This method is commonly used in both homelab and enterprise environments to securely and efficiently manage access to multiple systems.

---

### 8. Back up your private keys

Private keys should be treated as sensitive credentials and securely backed up to prevent permanent loss of access to systems that rely on key-based authentication. 
If a private key is lost and no alternative authentication method exists, access to the server may be unrecoverable.

---

## Outcome

The Linux server now accepts SSH connections using cryptographic key authentication instead of passwords. This significantly reduces exposure to brute-force attacks and credential theft while maintaining secure remote administrative access. Given how simple and convenient it is to set up, and the substantial security improvement it provides, implementing SSH key-based authentication is a no brainer.
