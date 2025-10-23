# Nextcloud All-in-One (AIO) Installation Guide for Ubuntu Server 24.04 LTS

## Complete Setup for Local Network Deployment with Caddy Reverse Proxy

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Architecture](#architecture)
4. [System Requirements](#system-requirements)
5. [Installation Steps](#installation-steps)
   - [Step 1: System Preparation](#step-1-system-preparation)
   - [Step 2: Docker Installation](#step-2-docker-installation)
   - [Step 3: Network Configuration](#step-3-network-configuration)
   - [Step 4: Caddy Reverse Proxy Setup](#step-4-caddy-reverse-proxy-setup)
   - [Step 5: Nextcloud AIO Installation](#step-5-nextcloud-aio-installation)
   - [Step 6: Initial Configuration](#step-6-initial-configuration)
6. [Post-Installation](#post-installation)
7. [Backup Configuration](#backup-configuration)
8. [Troubleshooting](#troubleshooting)
9. [Maintenance](#maintenance)
10. [Security Considerations](#security-considerations)
11. [FAQ](#faq)
12. [References](#references)

---

## Overview

This guide provides a **complete, production-ready installation** of Nextcloud All-in-One (AIO) on Ubuntu Server 24.04.3 LTS for **local network deployment** without requiring a public domain or SSL certificate from a public Certificate Authority.

### What is Nextcloud AIO?

Nextcloud All-in-One is the **official Nextcloud installation method** that provides:

- **Nextcloud** (latest stable version)
- **Collabora Office** (built-in document editing)
- **Nextcloud Talk** with TURN server (video/audio conferencing)
- **High-performance backend** for Files and Talk
- **BorgBackup** integration (automated backups)
- **Imaginary** (preview generation for various file formats)
- **ClamAV** (optional antivirus)
- **Fulltextsearch** (optional)
- **One-command updates** for all components

### Why This Guide?

While Nextcloud AIO officially supports public domain deployments with Let's Encrypt certificates, many users need to deploy Nextcloud **locally** within their home or office network without exposing it to the internet. This guide addresses this specific use case with:

✅ **Local network deployment** (no public domain required)  
✅ **Automatic SSL/TLS certificates** using Caddy's internal CA  
✅ **DNS integration** with AdGuard Home / Pi-hole  
✅ **Complete security** with encrypted connections  
✅ **Production-ready** configuration  

---

## Prerequisites

### Required Knowledge

- Basic Linux command-line skills
- Understanding of networking concepts (IP addresses, DNS)
- Familiarity with Docker concepts (helpful but not required)

### Required Infrastructure

| Component | Requirement |
|-----------|-------------|
| **Server/VM** | Ubuntu Server 24.04.3 LTS (fresh installation recommended) |
| **CPU** | Minimum: 2 cores (4+ recommended) |
| **RAM** | Minimum: 4 GB (8 GB recommended) |
| **Storage** | Minimum: 50 GB (100+ GB recommended for production) |
| **Network** | Static IP address on local network |
| **DNS Server** | AdGuard Home, Pi-hole, or similar local DNS server |

### Software Prerequisites

- Ubuntu Server 24.04.3 LTS installed and updated
- Root or sudo access
- SSH access (recommended for remote administration)

---

## Architecture

### Network Topology

```text
┌──────────────────────────────────────────────────────────────┐
│                     Local Network (192.168.0.0/24)           │
│                                                              │
│  ┌─────────────────┐         ┌──────────────────┐            │
│  │  AdGuard Home   │         │   Client Device  │            │
│  │  192.168.0.118  │◄────────│   (Browser)      │            │
│  │  DNS Server     │  DNS    │                  │            │
│  └────────┬────────┘  Query  └──────────────────┘            │
│           │ A Record:                                        │
│           │ nextcloud.lan → 192.168.0.XXX                    │
│           │                                                  │
│  ┌────────▼──────────────────────────────────────────────┐   │
│  │         Ubuntu Server 24.04.3 (VM)                    │   │
│  │         IP: 192.168.0.XXX (Static)                    │   │
│  │  ┌──────────────────────────────────────────────┐     │   │
│  │  │  Caddy Reverse Proxy (Docker Container)      │     │   │
│  │  │  Ports: 80, 443                              │     │   │
│  │  │  TLS: Internal CA (self-signed)              │     │   │
│  │  └────────┬─────────────────────────────────────┘     │   │
│  │           │                                           │   │
│  │           ▼                                           │   │
│  │  ┌────────────────────────────────────────────────┐   │   │
│  │  │  Nextcloud AIO Mastercontainer                 │   │   │
│  │  │  Port: 8080 (AIO Interface)                    │   │   │
│  │  │  Manages: Apache, Database, Redis, etc.        │   │   │
│  │  └────────┬───────────────────────────────────────┘   │   │
│  │           │                                           │   │
│  │           ├─► Apache (Port 11000)                     │   │
│  │           ├─► PostgreSQL Database                     │   │
│  │           ├─► Redis Cache                             │   │
│  │           ├─► Collabora Office                        │   │
│  │           ├─► Talk (Port 3478 TCP/UDP)                │   │
│  │           └─► BorgBackup                              │   │
│  │                                                       │   │
│  └───────────────────────────────────────────────────────┘   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Component Interaction

1. **Client** requests `https://nextcloud.lan`
2. **DNS Server** (AdGuard) resolves to server's local IP
3. **Caddy** terminates TLS, proxies to AIO Apache container
4. **Apache** serves Nextcloud, communicates with backend services
5. All containers managed by **AIO Mastercontainer**

---

## System Requirements

### Minimum Configuration (Testing/Personal Use)

- **CPU**: 2 cores
- **RAM**: 4 GB
- **Storage**: 50 GB
- **Concurrent Users**: 1-5

### Recommended Configuration (Small Team)

- **CPU**: 4 cores
- **RAM**: 8 GB
- **Storage**: 100-500 GB SSD
- **Concurrent Users**: 5-20

### Production Configuration (Medium Organization)

- **CPU**: 8+ cores
- **RAM**: 16+ GB
- **Storage**: 500+ GB SSD (NVMe preferred)
- **Concurrent Users**: 20-100

**Note**: Nextcloud AIO supports up to 100 users on the free version. For larger deployments, consider [Nextcloud Enterprise](https://nextcloud.com/all-in-one/).

---

## Installation Steps

### Step 1: System Preparation

#### 1.1 Update System Packages

```bash
# Update package lists and upgrade existing packages
sudo apt update && sudo apt upgrade -y

# Install essential tools
sudo apt install -y \
    curl \
    wget \
    git \
    nano \
    htop \
    net-tools \
    ca-certificates \
    gnupg \
    lsb-release
```

#### 1.2 Set Static IP Address

**Using Netplan** (Ubuntu Server 24.04 default):

```bash
# Identify your network interface
ip addr show

# Edit netplan configuration
sudo nano /etc/netplan/00-installer-config.yaml
```

**Example configuration**:

```yaml
network:
  version: 2
  ethernets:
    ens18:  # Replace with your interface name (e.g., eth0, enp0s3)
      dhcp4: false
      addresses:
        - 192.168.0.100/24  # Your desired static IP
      routes:
        - to: default
          via: 192.168.0.1  # Your router/gateway IP
      nameservers:
        addresses:
          - 192.168.0.118  # Your AdGuard Home IP
          - 8.8.8.8        # Fallback DNS
```

Apply the configuration:

```bash
# Test the configuration
sudo netplan try

# If successful, apply permanently
sudo netplan apply

# Verify IP address
ip addr show
```

#### 1.3 Set Hostname

```bash
# Set hostname to something meaningful
sudo hostnamectl set-hostname nextcloud-server

# Update /etc/hosts
sudo nano /etc/hosts
```

Add the following line:

```text
192.168.0.100   nextcloud-server nextcloud.lan
```

#### 1.4 Configure Firewall (UFW)

```bash
# Enable UFW
sudo ufw enable

# Allow SSH (if using remote access)
sudo ufw allow 22/tcp comment 'SSH'

# Allow HTTP/HTTPS for Caddy
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'

# Allow AIO Interface (optional, for local access)
sudo ufw allow 8080/tcp comment 'AIO Interface'

# Allow Talk TURN server
sudo ufw allow 3478/tcp comment 'Talk TURN TCP'
sudo ufw allow 3478/udp comment 'Talk TURN UDP'

# Check firewall status
sudo ufw status verbose
```

---

### Step 2: Docker Installation

#### 2.1 Remove Old Docker Versions (if any)

```bash
# Remove any existing Docker packages
sudo apt remove -y docker docker-engine docker.io containerd runc
```

⚠️ **IMPORTANT**: Do NOT use snap-based Docker installation on Ubuntu. Nextcloud AIO is not compatible with snap Docker.

**Check if Docker is installed via snap**:

```bash
sudo docker info | grep "Docker Root Dir" | grep "/var/snap/docker/"
```

If output contains `/var/snap/docker/`, remove it:

```bash
sudo snap remove docker
```

#### 2.2 Install Docker from Official Repository

```bash
# Add Docker's official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update package index
sudo apt update

# Install Docker Engine, CLI, and Compose plugin
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Verify Docker installation
sudo docker --version
sudo docker compose version
```

#### 2.3 Configure Docker for Local DNS Resolution

This step is **critical** for local network deployment. Docker containers must use your local DNS server to resolve `nextcloud.lan`.

```bash
# Create or edit Docker daemon configuration
sudo nano /etc/docker/daemon.json
```

Add the following configuration:

```json
{
  "dns": ["192.168.0.118", "8.8.8.8"],
  "log-driver": "local",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

**Explanation:**

- `dns`: Configures Docker to use your AdGuard Home (192.168.0.118) as primary DNS
- `log-driver` & `log-opts`: Limits container log sizes to prevent disk space issues

Restart Docker to apply changes:

```bash
# Restart Docker daemon
sudo systemctl restart docker

# Enable Docker to start on boot
sudo systemctl enable docker

# Verify Docker is running
sudo systemctl status docker
```

#### 2.4 Add User to Docker Group (Optional)

Allows running Docker without `sudo`:

```bash
# Add current user to docker group
sudo usermod -aG docker $USER

# Apply group changes (logout/login or use newgrp)
newgrp docker

# Test Docker without sudo
docker ps
```

---

### Step 3: Network Configuration

#### 3.1 Configure DNS Record in AdGuard Home

Access your AdGuard Home admin interface (typically `http://192.168.0.118`).

**Add Custom DNS Record**:

1. Navigate to **Filters** → **DNS rewrites** (or **Custom filtering rules**)
2. Add an **A record**:
   - **Domain**: `nextcloud.lan`
   - **IP Address**: `192.168.0.100` (your Nextcloud server's IP)
3. Save the configuration

**Alternative (Pi-hole)**:

```bash
# SSH into Pi-hole server
ssh pi@192.168.0.118

# Edit custom DNS records
sudo nano /etc/pihole/custom.list
```

Add the line:

```text
192.168.0.100 nextcloud.lan
```

Restart DNS:

```bash
pihole restartdns
```

#### 3.2 Verify DNS Resolution

From your client machine and the Nextcloud server:

```bash
# Test DNS resolution
nslookup nextcloud.lan

# Should return:
# Server: 192.168.0.118
# Address: 192.168.0.118#53
#
# Name: nextcloud.lan
# Address: 192.168.0.100

# Also test with dig
dig nextcloud.lan +short
# Should return: 192.168.0.100
```

---

### Step 4: Caddy Reverse Proxy Setup

Caddy will provide automatic HTTPS with a self-signed certificate for local network access.

#### 4.1 Create Caddy Configuration Directory

```bash
# Create directories for Caddy
sudo mkdir -p /opt/caddy/config
sudo mkdir -p /opt/caddy/data

# Create Caddyfile
sudo nano /opt/caddy/Caddyfile
```

#### 4.2 Configure Caddyfile

Add the following configuration:

```caddyfile
# Nextcloud AIO - Local Network Configuration
# This configuration uses Caddy's internal CA for local HTTPS

{
    # Global options
    admin off
    auto_https disable_redirects
    
    # Optional: Increase timeouts for large file uploads
    servers {
        timeouts {
            read_body 24h
            read_header 24h
            write 24h
            idle 24h
        }
    }
}

# Main Nextcloud domain
https://nextcloud.lan:443 {
    # Use Caddy's internal CA for self-signed certificates
    tls internal
    
    # Reverse proxy to Nextcloud AIO Apache container
    reverse_proxy http://localhost:11000 {
        # Forward original headers
        header_up X-Forwarded-For {remote_host}
        header_up X-Forwarded-Proto {scheme}
        header_up X-Real-IP {remote_host}
        header_up Host {host}
        
        # Enable transparent transport
        transport http {
            # Keep connections alive
            keepalive 24h
            keepalive_idle_conns 10
        }
    }
    
    # Logging (optional, for debugging)
    log {
        output file /var/log/caddy/nextcloud_access.log {
            roll_size 100mb
            roll_keep 5
            roll_keep_for 720h
        }
        format json
        level INFO
    }
}

# HTTP to HTTPS redirect
http://nextcloud.lan:80 {
    redir https://nextcloud.lan{uri} permanent
}
```

**Configuration Breakdown**:

- **`tls internal`**: Tells Caddy to generate a self-signed certificate using its internal CA
- **`reverse_proxy http://localhost:11000`**: Proxies traffic to AIO's Apache container (port 11000)
- **Headers**: Preserves client information for Nextcloud
- **Timeouts**: Extended to 24 hours to support large file uploads
- **Logging**: Optional, helps with troubleshooting

#### 4.3 Create Docker Compose File for Caddy

```bash
sudo nano /opt/caddy/docker-compose.yml
```

Add the following:

```yaml
version: '3.8'

services:
  caddy:
    image: caddy:2.10.2-alpine
    container_name: caddy-reverse-proxy
    restart: always
    
    # Use host network for simplest configuration
    network_mode: host
    
    volumes:
      # Caddyfile configuration
      - /opt/caddy/Caddyfile:/etc/caddy/Caddyfile:ro
      
      # Persistent data (certificates, etc.)
      - /opt/caddy/data:/data
      - /opt/caddy/config:/config
      
      # Log directory
      - /var/log/caddy:/var/log/caddy
    
    environment:
      - TZ=UTC  # Adjust to your timezone, e.g., Europe/London
    
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

**Why `network_mode: host`?**

- Simplifies networking between Caddy and AIO containers
- Allows Caddy to bind directly to ports 80 and 443 on the host
- Recommended by Nextcloud AIO documentation for reverse proxy setups

#### 4.4 Create Log Directory

```bash
sudo mkdir -p /var/log/caddy
sudo chmod 755 /var/log/caddy
```

#### 4.5 Start Caddy Container

```bash
cd /opt/caddy

# Pull Caddy image
sudo docker compose pull

# Start Caddy in detached mode
sudo docker compose up -d

# Check Caddy logs
sudo docker logs caddy-reverse-proxy

# Verify Caddy is running
sudo docker ps | grep caddy
```

**Expected output:**

```text
CONTAINER ID   IMAGE                 COMMAND                  CREATED         STATUS         PORTS     NAMES
abc123def456   caddy:2.10.2-alpine   "caddy run --config …"   10 seconds ago  Up 9 seconds             caddy-reverse-proxy
```

#### 4.6 Verify Caddy Certificate Generation

```bash
# Check Caddy data directory
sudo ls -la /opt/caddy/data/caddy/

# Should see certificates directory
sudo ls -la /opt/caddy/data/caddy/certificates/
```

---

### Step 5: Nextcloud AIO Installation

#### 5.1 Create AIO Mastercontainer

Run the following command to start the Nextcloud AIO mastercontainer:

```bash
sudo docker run \
  --init \
  --sig-proxy=false \
  --name nextcloud-aio-mastercontainer \
  --restart always \
  --publish 8080:8080 \
  --env APACHE_PORT=11000 \
  --env APACHE_IP_BINDING=0.0.0.0 \
  --env SKIP_DOMAIN_VALIDATION=true \
  --volume nextcloud_aio_mastercontainer:/mnt/docker-aio-config \
  --volume /var/run/docker.sock:/var/run/docker.sock:ro \
  ghcr.io/nextcloud-releases/all-in-one:latest
```

**Command Explanation**:

| Flag | Purpose |
|------|---------|
| `--init` | Ensures proper process handling |
| `--sig-proxy=false` | Disables signal proxying |
| `--name nextcloud-aio-mastercontainer` | Names the container for easy management |
| `--restart always` | Auto-restarts container on failure or reboot |
| `--publish 8080:8080` | Exposes AIO web interface on port 8080 |
| `--env APACHE_PORT=11000` | Tells AIO to expose Apache on port 11000 (for Caddy) |
| `--env APACHE_IP_BINDING=0.0.0.0` | Allows connections from any IP (needed for reverse proxy) |
| `--env SKIP_DOMAIN_VALIDATION=true` | **Critical**: Disables public domain validation for local deployment |
| `--volume nextcloud_aio_mastercontainer:/mnt/docker-aio-config` | Persists AIO configuration |
| `--volume /var/run/docker.sock:/var/run/docker.sock:ro` | Allows AIO to manage other containers |

#### 5.2 Verify Mastercontainer is Running

```bash
# Check container status
sudo docker ps -a | grep nextcloud-aio-mastercontainer

# View container logs
sudo docker logs nextcloud-aio-mastercontainer

# Expected log message:
# "Initial startup of Nextcloud All-in-One complete!
#  You should be able to open the Nextcloud AIO Interface now on port 8080"
```

#### 5.3 Optional: Configure Data Directory Location

By default, Nextcloud stores files in a Docker volume. To use a specific directory on your host:

**Example: Store data on separate drive mounted at `/mnt/nextcloud-data`**:

```bash
# Stop the mastercontainer
sudo docker stop nextcloud-aio-mastercontainer
sudo docker rm nextcloud-aio-mastercontainer

# Create data directory with correct permissions
sudo mkdir -p /mnt/nextcloud-data
sudo chown -R 33:0 /mnt/nextcloud-data
sudo chmod -R 750 /mnt/nextcloud-data

# Recreate mastercontainer with custom data directory
sudo docker run \
  --init \
  --sig-proxy=false \
  --name nextcloud-aio-mastercontainer \
  --restart always \
  --publish 8080:8080 \
  --env APACHE_PORT=11000 \
  --env APACHE_IP_BINDING=0.0.0.0 \
  --env SKIP_DOMAIN_VALIDATION=true \
  --env NEXTCLOUD_DATADIR="/mnt/nextcloud-data" \
  --volume nextcloud_aio_mastercontainer:/mnt/docker-aio-config \
  --volume /var/run/docker.sock:/var/run/docker.sock:ro \
  ghcr.io/nextcloud-releases/all-in-one:latest
```

**Important**: This must be done **before** the initial Nextcloud installation. Changing it later requires manual migration.

---

### Step 6: Initial Configuration

#### 6.1 Access AIO Interface

Open your browser and navigate to the AIO interface using the **server's IP address** (not the domain):

```text
https://192.168.0.100:8080
```

⚠️ **IMPORTANT**: Always use an IP address when accessing port 8080, not a domain. This avoids HSTS conflicts.

**Accept the self-signed certificate warning:**

- Browser will show "Your connection is not private" or similar
- Click "Advanced" → "Proceed to 192.168.0.100 (unsafe)"

#### 6.2 Save the Secure Password

On the first access, AIO generates a **one-time password**.

**CRITICAL**: Copy this password and save it securely (password manager recommended). You'll need it to log in to the AIO interface.

![AIO Password Screen](https://github.com/nextcloud/all-in-one/raw/main/screenshots/password.png)

#### 6.3 Enter Your Domain

After logging in, you'll see the domain configuration screen.

1. **Enter your local domain**: `nextcloud.lan`
2. Click **"Submit domain"**

Since `SKIP_DOMAIN_VALIDATION=true` is set, AIO will accept any domain without validation.

![Domain Configuration](https://github.com/nextcloud/all-in-one/raw/main/screenshots/domain.png)

#### 6.4 Select Optional Containers

AIO will show available optional containers. For a complete setup, consider enabling:

**Recommended:**

- ✅ **Nextcloud Office** (Collabora) - Document editing
- ✅ **Nextcloud Talk** - Video/audio conferencing
- ✅ **Nextcloud Talk Recording** - Call recording
- ✅ **Imaginary** - Enhanced preview generation
- ✅ **Fulltextsearch** - Search across file contents

**Optional** (resource-intensive):

- ⬜ **ClamAV** - Antivirus scanning (uses ~1 GB RAM)

Click **"Save changes"** after selecting containers.

#### 6.5 Configure Timezone

Set your server's timezone for proper scheduling:

```bash
# Set timezone
sudo timedatectl set-timezone Europe/London  # Adjust to your timezone

# Verify
timedatectl
```

In the AIO interface, enter the same timezone (e.g., `Europe/London`).

#### 6.6 Start Containers

Click **"Start and update containers"**.

AIO will:

1. Download all required Docker images (~5-10 GB total)
2. Create and configure all containers
3. Initialize the Nextcloud database
4. Configure applications

**This process takes 10-30 minutes** depending on internet speed and server performance.

Monitor progress in the AIO interface or via terminal:

```bash
# Watch all AIO containers
watch -n 2 'sudo docker ps --filter "name=nextcloud-aio" --format "table {{.Names}}\t{{.Status}}"'
```

#### 6.7 Retrieve Initial Admin Credentials

Once installation completes, AIO will display:

- **Nextcloud URL**: `https://nextcloud.lan`
- **Admin Username**: (displayed in AIO interface)
- **Admin Password**: (displayed in AIO interface)

**CRITICAL**: Save these credentials securely before closing the window!

---

## Post-Installation

### Access Nextcloud

Open your browser and navigate to:

```text
https://nextcloud.lan
```

**First-time access:**

1. Browser will warn about the self-signed certificate
2. Click "Advanced" → "Accept the Risk and Continue"
3. Log in with admin credentials from Step 6.7

### Install Caddy Certificate in Nextcloud Container (CRITICAL)

**This step is required** to fix SSL errors in Nextcloud Talk and Collabora Office:

```bash
# Export Caddy's root certificate
sudo docker exec caddy-reverse-proxy cat /data/caddy/pki/authorities/local/root.crt > /tmp/caddy-root-ca.crt

# Copy certificate to Nextcloud container
sudo docker cp /tmp/caddy-root-ca.crt nextcloud-aio-nextcloud:/usr/local/share/ca-certificates/caddy-root-ca.crt

# Update CA certificates in the container
sudo docker exec nextcloud-aio-nextcloud update-ca-certificates

# Restart Nextcloud container to apply changes
sudo docker restart nextcloud-aio-nextcloud
```

**Verify the fix:**

```bash
# Test that Nextcloud can access itself via HTTPS without SSL errors
sudo docker exec nextcloud-aio-nextcloud curl -s https://nextcloud.lan/status.php
```

Expected output (no errors):

```json
{"installed":true,"maintenance":false,"needsDbUpgrade":false,"version":"32.0.0.13"}
```

**Why is this needed?**

- Nextcloud Talk and Collabora Office make internal HTTPS requests to `https://nextcloud.lan`
- Without trusting Caddy's self-signed certificate, these requests fail with "SSL certificate problem: unable to get local issuer certificate"
- Installing the certificate resolves this issue

### Trust the Certificate (Optional but Recommended)

To avoid certificate warnings on every visit, install Caddy's root CA certificate on your devices.

#### On Ubuntu/Debian

```bash
# Export Caddy's root CA certificate
sudo docker exec caddy-reverse-proxy \
  cat /data/caddy/pki/authorities/local/root.crt \
  > /tmp/caddy-root-ca.crt

# Install the certificate
sudo cp /tmp/caddy-root-ca.crt /usr/local/share/ca-certificates/caddy-root-ca.crt
sudo update-ca-certificates

# Restart browsers to apply
```

#### On Windows

1. Download the certificate from the server:

   ```powershell
   scp user@192.168.0.100:/tmp/caddy-root-ca.crt C:\Users\YourUsername\Downloads\
   ```

2. Double-click `caddy-root-ca.crt`
3. Click "Install Certificate"
4. Select "Local Machine" → "Place all certificates in the following store"
5. Browse → Select "Trusted Root Certification Authorities"
6. Click "OK" → "Finish"

#### On macOS

```bash
# Download certificate
scp user@192.168.0.100:/tmp/caddy-root-ca.crt ~/Downloads/

# Open Keychain Access
open ~/Downloads/caddy-root-ca.crt

# In Keychain Access: Set trust to "Always Trust"
```

#### On Android

1. Settings → Security → Encryption & credentials → Install a certificate
2. Choose "CA certificate"
3. Select the downloaded `.crt` file

#### On iOS

1. Email the `.crt` file to yourself or download via browser
2. Open the file → Install Profile
3. Settings → General → About → Certificate Trust Settings
4. Enable full trust for the certificate

#### On OpenSUSE (and other Linux distributions)

```bash
# Copy certificate from server
scp user@192.168.0.100:/tmp/caddy-root-ca.crt /tmp/

# Install certificate
sudo cp /tmp/caddy-root-ca.crt /etc/pki/trust/anchors/caddy-root-ca.crt
sudo update-ca-certificates --fresh

# Restart browsers and applications
```

**⚠️ Important for Nextcloud Desktop Client on Linux**:

If you're using **Flatpak version** of Nextcloud Desktop Client and encounter SSL certificate errors or requests for client certificates (PKCS12), this is a known bug in the Flatpak package.

**Solution**: Use the official **AppImage** version instead:

1. Download from [Nextcloud Desktop Client Downloads](https://nextcloud.com/install/#install-clients)
2. Make executable: `chmod +x Nextcloud-*.AppImage`
3. Run: `./Nextcloud-*.AppImage`
4. Remove Flatpak version: `flatpak uninstall com.nextcloud.desktopclient.nextcloud`

The AppImage version works correctly with self-signed certificates after installing the root CA certificate system-wide.

---

## Backup Configuration

Nextcloud AIO includes a robust backup solution based on **BorgBackup**.

### Configure Backup Location

#### Option 1: Local Backup Drive

**Example: External USB drive mounted at `/mnt/backup`**

```bash
# Create mount point
sudo mkdir -p /mnt/backup

# Add to /etc/fstab for automatic mounting
# First, identify the drive UUID
sudo blkid /dev/sdb1  # Adjust device name

# Add to /etc/fstab:
# UUID=xxxx-xxxx-xxxx /mnt/backup ext4 defaults,nofail 0 2

# Mount the drive
sudo mount -a

# Create backup directory with correct permissions
sudo mkdir -p /mnt/backup/nextcloud-aio-backups
sudo chown -R root:root /mnt/backup/nextcloud-aio-backups
sudo chmod 700 /mnt/backup/nextcloud-aio-backups
```

**In AIO Interface:**

1. Navigate to "Backup" section
2. Enter backup location: `/mnt/backup/nextcloud-aio-backups`
3. Click "Submit backup location"
4. Save the **backup encryption password** (AIO generates this)

#### Option 2: Remote Borg Repository

For remote backups (e.g., to another server):

```bash
# Install Borg on backup server
# On backup server (e.g., 192.168.0.200):
sudo apt install borgbackup

# Create backup user
sudo useradd -m -s /bin/bash borgbackup
sudo mkdir -p /mnt/backups/nextcloud-aio
sudo chown borgbackup:borgbackup /mnt/backups/nextcloud-aio

# Generate SSH key on Nextcloud server
ssh-keygen -t ed25519 -f ~/.ssh/borgbackup_key -N ""

# Copy public key to backup server
ssh-copy-id -i ~/.ssh/borgbackup_key.pub borgbackup@192.168.0.200
```

**In AIO Interface:**

1. Backup location: `ssh://borgbackup@192.168.0.200/mnt/backups/nextcloud-aio`
2. SSH key path: `/root/.ssh/borgbackup_key`

### Create Initial Backup

1. In AIO interface, click **"Create backup"**
2. Wait for backup to complete (10 minutes to several hours depending on data size)
3. Verify backup: Click **"Check backup integrity"**

### Enable Automatic Backups

1. In AIO interface, enable **"Daily backups"**
2. Backups will run at 04:00 AM server time
3. **Optional**: Enable **"Automatic updates"** (updates containers after successful backup)

### Backup Retention Policy

Default retention (customizable):

- **Keep within**: 7 days (all backups from last week)
- **Keep weekly**: 4 weekly backups
- **Keep monthly**: 6 monthly backups

**To customize**, stop the mastercontainer and recreate with:

```bash
--env BORG_RETENTION_POLICY="--keep-within=7d --keep-weekly=4 --keep-monthly=12"
```

---

## Troubleshooting

### Container Fails to Start

**Check logs:**

```bash
sudo docker logs nextcloud-aio-mastercontainer
sudo docker logs nextcloud-aio-apache
sudo docker logs nextcloud-aio-nextcloud
```

**Common issues:**

- Port conflicts: Ensure ports 80, 443, 8080, 11000, 3478 are not in use
- Permissions: Data directory must be owned by UID 33 (www-data)
- Docker socket: Ensure `/var/run/docker.sock` is accessible

### Cannot Access Nextcloud

**Verify DNS resolution:**

```bash
nslookup nextcloud.lan
ping nextcloud.lan
```

**Check Caddy:**

```bash
sudo docker logs caddy-reverse-proxy
curl -I http://localhost:11000  # Should return HTTP 200
```

**Check AIO Apache:**

```bash
sudo docker exec nextcloud-aio-apache curl -I http://localhost:80
```

### Certificate Errors Persist

**Verify Caddy certificate generation:**

```bash
sudo docker exec caddy-reverse-proxy caddy trust list
```

**Manually regenerate:**

```bash
sudo docker exec caddy-reverse-proxy caddy trust
```

### Slow Performance

**Check system resources:**

```bash
# CPU and memory usage
htop

# Disk I/O
sudo iotop

# Docker container resources
sudo docker stats
```

**Optimize:**

- Increase PHP memory limit: Set `NEXTCLOUD_MEMORY_LIMIT=1024M` in mastercontainer
- Enable Redis cache (already enabled in AIO)
- Add more RAM/CPU if consistently high usage

### Database Connection Errors

**Restart database container:**

```bash
sudo docker restart nextcloud-aio-database
```

**Check database logs:**

```bash
sudo docker logs nextcloud-aio-database
```

---

## Maintenance

### Update Containers

AIO provides **one-click updates** for all components:

1. Open AIO interface: `https://192.168.0.100:8080`
2. Log in with saved password
3. If updates available, you'll see: **"Container updates available!"**
4. Click **"Stop containers"**
5. Click **"Start and update containers"**
6. Wait for update to complete (~5-15 minutes)

**Alternatively**, enable **automatic updates:**

- Creates daily backup at 04:00 AM
- Updates containers automatically
- Starts containers after successful update

### Update Mastercontainer

When a mastercontainer update is available:

1. Stop containers from AIO interface
2. Stop mastercontainer:

   ```bash
   sudo docker stop nextcloud-aio-mastercontainer
   sudo docker rm nextcloud-aio-mastercontainer
   ```

3. Pull latest image:

   ```bash
   sudo docker pull ghcr.io/nextcloud-releases/all-in-one:latest
   ```

4. Recreate mastercontainer (use your original docker run command from Step 5.1)
5. Open AIO interface and start containers

### Run OCC Commands

Nextcloud's `occ` command-line tool for administration:

```bash
# General syntax
sudo docker exec --user www-data -it nextcloud-aio-nextcloud php occ [command]

# Examples:

# List all users
sudo docker exec --user www-data -it nextcloud-aio-nextcloud php occ user:list

# Scan files for new/changed files
sudo docker exec --user www-data -it nextcloud-aio-nextcloud php occ files:scan --all

# Maintenance mode
sudo docker exec --user www-data -it nextcloud-aio-nextcloud php occ maintenance:mode --on
sudo docker exec --user www-data -it nextcloud-aio-nextcloud php occ maintenance:mode --off

# Clear cache
sudo docker exec --user www-data -it nextcloud-aio-nextcloud php occ cache:flush

# Update apps
sudo docker exec --user www-data -it nextcloud-aio-nextcloud php occ app:update --all
```

### Monitor System Health

**Nextcloud Admin Overview:**

- Navigate to `https://nextcloud.lan/settings/admin/overview`
- Check for warnings and recommendations
- Verify background jobs are running

**Docker container health:**

```bash
# Check all AIO containers
sudo docker ps -a --filter "name=nextcloud-aio" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Resource usage
sudo docker stats --no-stream
```

### Log Management

**View logs:**

```bash
# Mastercontainer
sudo docker logs -f nextcloud-aio-mastercontainer

# Apache
sudo docker logs -f nextcloud-aio-apache

# Database
sudo docker logs -f nextcloud-aio-database

# Caddy
sudo docker logs -f caddy-reverse-proxy
```

**Nextcloud logs:**

```bash
# View Nextcloud application logs
sudo docker exec nextcloud-aio-nextcloud cat /var/www/html/data/nextcloud.log
```

---

## Security Considerations

### Firewall Configuration

Ensure only necessary ports are open:

```bash
# Review current rules
sudo ufw status numbered

# If you don't need external AIO access, block port 8080 from WAN
sudo ufw deny from any to any port 8080
sudo ufw allow from 192.168.0.0/24 to any port 8080 comment 'AIO Interface - LAN only'

# Reload firewall
sudo ufw reload
```

### Regular Security Updates

**Ubuntu system updates:**

```bash
# Enable automatic security updates
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

**AIO container updates:**

- Enable automatic updates in AIO interface, or
- Manually update weekly via AIO interface

### Strong Passwords

- Use **unique, complex passwords** for:
  - Nextcloud admin account
  - AIO interface password
  - Backup encryption password
  - Database passwords (auto-generated by AIO)

**Change admin password:**

```bash
sudo docker exec --user www-data -it nextcloud-aio-nextcloud php occ user:resetpassword admin
```

### Two-Factor Authentication (2FA)

**Enable for admin account:**

1. Log in to Nextcloud as admin
2. Navigate to Settings → Security
3. Install **"Two-Factor TOTP Provider"** app
4. Settings → Security → Enable TOTP
5. Scan QR code with authenticator app (e.g., Authy, Google Authenticator)

**Enforce 2FA for all users:**

```bash
sudo docker exec --user www-data -it nextcloud-aio-nextcloud php occ app:install twofactor_totp
sudo docker exec --user www-data -it nextcloud-aio-nextcloud php occ config:app:set core enforce_2fa --value=yes
```

### Brute-Force Protection

Nextcloud includes built-in brute-force protection. Monitor attempts:

```bash
# View brute-force attempts
sudo docker exec --user www-data -it nextcloud-aio-nextcloud php occ security:bruteforce:attempts

# Unblock an IP if needed
sudo docker exec --user www-data -it nextcloud-aio-nextcloud php occ security:bruteforce:reset <IP_ADDRESS>
```

**Optional**: Add Fail2Ban for additional protection (see [AIO Fail2Ban documentation](https://github.com/nextcloud/all-in-one/tree/main/community-containers/fail2ban)).

---

## FAQ

### Can I expose this to the public internet later?

Yes, but it requires additional configuration:

1. **Obtain a real domain** (e.g., `example.com`)
2. **Configure port forwarding** on your router (80, 443 to server)
3. **Modify Caddyfile** to use Let's Encrypt:

   ```caddyfile
   https://cloud.example.com {
       tls your-email@example.com  # Automatic Let's Encrypt
       reverse_proxy http://localhost:11000
   }
   ```

4. **Update DNS** to point `cloud.example.com` to your public IP
5. **Reconfigure AIO** with new domain:
   - Stop containers
   - Stop and remove mastercontainer
   - Recreate with `--env SKIP_DOMAIN_VALIDATION=false`
   - Submit new domain in AIO interface

### Can I use a different domain name?

Yes, but consistency is important:

1. **Before installation**: Simply use your preferred domain (e.g., `cloud.local`, `mycloud.home`)
2. **After installation**: Requires manual reconfiguration (see AIO documentation on changing domain)

### How much disk space do I need?

**Formula**: Base installation (10-15 GB) + User data + Backups + Buffer

**Examples:**

- **Personal use** (50 GB files): 100 GB total (50 + 15 base + 25 backup + 10 buffer)
- **Small team** (200 GB files): 500 GB total (200 + 15 base + 200 backup + 85 buffer)
- **Large team** (1 TB files): 2.5 TB total (1000 + 15 base + 1000 backup + 500 buffer)

### Can I use NFS/SMB for data storage?

Yes, but with caveats:

**Mount network share:**

```bash
# Example: SMB share
sudo apt install cifs-utils
sudo mkdir -p /mnt/nextcloud-smb

# Add to /etc/fstab
//192.168.0.200/nextcloud /mnt/nextcloud-smb cifs credentials=/etc/smb-credentials,uid=33,gid=0,file_mode=0770,dir_mode=0770 0 0

# Mount
sudo mount -a
```

**Configure AIO** with `NEXTCLOUD_DATADIR=/mnt/nextcloud-smb` before first start.

**Caveats:**

- Performance may be slower than local storage
- Ensure network stability
- SMB locking issues can occur (use `mfsymlinks,seal` options)

### How do I migrate from an existing Nextcloud installation?

See official [AIO migration guide](https://github.com/nextcloud/all-in-one/blob/main/migration.md).

**Summary:**

1. Backup existing Nextcloud instance
2. Install fresh AIO instance
3. Stop AIO containers
4. Restore database and files to AIO volumes
5. Run `occ maintenance:repair` and `occ files:scan --all`
6. Start AIO containers

### Can I run multiple Nextcloud instances?

Yes. See [AIO multiple instances guide](https://github.com/nextcloud/all-in-one/blob/main/multiple-instances.md).

**Requirements per instance:**

- Unique domain name
- Unique Apache port (e.g., 11000, 11001, 11002)
- Unique AIO mastercontainer port (8080, 8081, 8082)
- Unique Talk port (3478, 3479, 3480)

### What if AdGuard Home / Pi-hole fails?

**Fallback DNS**: Docker is configured with `8.8.8.8` as fallback.

**Resolution:**

- Local access to Nextcloud will fail (can't resolve `nextcloud.lan`)
- Access via IP temporarily: `https://192.168.0.100` (will show certificate error)
- Fix/restart DNS server

**Prevention**: Configure router to use AdGuard Home as primary DNS for entire network.

### How do I disable Collabora Office?

In AIO interface:

1. Uncheck "Nextcloud Office" in container selection
2. Click "Save changes"
3. Stop and restart containers

**Manual removal:**

```bash
sudo docker stop nextcloud-aio-collabora
sudo docker rm nextcloud-aio-collabora
```

---

## References

### Official Documentation

- **Nextcloud AIO GitHub**: [https://github.com/nextcloud/all-in-one](https://github.com/nextcloud/all-in-one)
- **Local Instance Guide**: [https://github.com/nextcloud/all-in-one/blob/main/local-instance.md](https://github.com/nextcloud/all-in-one/blob/main/local-instance.md)
- **Reverse Proxy Documentation**: [https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md](https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md)
- **Nextcloud Admin Manual**: [https://docs.nextcloud.com/server/latest/admin_manual/](https://docs.nextcloud.com/server/latest/admin_manual/)

### Community Resources

- **Nextcloud Forums**: [https://help.nextcloud.com/](https://help.nextcloud.com/)
- **Nextcloud AIO Discussions**: [https://github.com/nextcloud/all-in-one/discussions](https://github.com/nextcloud/all-in-one/discussions)
- **Docker Documentation**: [https://docs.docker.com/](https://docs.docker.com/)
- **Caddy Documentation**: [https://caddyserver.com/docs/](https://caddyserver.com/docs/)

### Related Projects

- **AdGuard Home**: [https://github.com/AdguardTeam/AdGuardHome](https://github.com/AdguardTeam/AdGuardHome)
- **Pi-hole**: [https://pi-hole.net/](https://pi-hole.net/)
- **BorgBackup**: [https://borgbackup.readthedocs.io/](https://borgbackup.readthedocs.io/)

---

## License

This guide is provided under the **MIT License**.

```text
MIT License

Copyright (c) 2025

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

---

## Changelog

### Version 1.0.0 (2025-01-XX)

- Initial release
- Complete installation guide for Ubuntu Server 24.04.3 LTS
- Caddy reverse proxy configuration with internal TLS
- AdGuard Home / Pi-hole DNS integration
- Backup configuration and best practices
- Troubleshooting section
- Security hardening recommendations

---

## Contributing

Contributions to this guide are welcome! If you find errors, have suggestions, or want to add sections:

1. Fork this repository
2. Create a feature branch
3. Submit a pull request with your changes

**Reporting Issues:**

- For guide-related issues: Open an issue on this repository
- For Nextcloud AIO bugs: Report on [official AIO repository](https://github.com/nextcloud/all-in-one/issues)

---

## Acknowledgments

- **Nextcloud GmbH** for developing and maintaining Nextcloud AIO
- **Simon L.** and AIO contributors for excellent documentation
- **Caddy community** for the powerful reverse proxy
- **BorgBackup project** for robust backup solution
- **Docker** for containerization technology

---

**Last Updated**: 2025-01-XX  
**Tested On**: Ubuntu Server 24.04.3 LTS, Nextcloud AIO latest (as of 2025-01)  
**Guide Version**: 1.0.0
