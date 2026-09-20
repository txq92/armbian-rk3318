# Armbian (Debian) Post-Installation Guideline: Static IP, Docker & Docker Compose

This guideline provides step-by-step instructions to configure a **Static IP address**, install **Docker**, and install **Docker Compose** on Armbian (Debian-based) running on your TV Box (Rockchip RK3318 / RK3228 / Amlogic).

---

## 📑 Table of Contents
1. [Prerequisites](#1-prerequisites)
2. [Configuring a Static IP Address](#2-configuring-a-static-ip-address)
   - [Method 1: Using `nmtui` (Recommended / GUI in Terminal)](#method-1-using-nmtui-recommended)
   - [Method 2: Using `armbian-config`](#method-2-using-armbian-config)
   - [Method 3: Using `nmcli` (Command Line)](#method-3-using-nmcli-command-line)
3. [Installing Docker](#3-installing-docker)
4. [Installing Docker Compose](#4-installing-docker-compose)
5. [Managing Docker as a Non-Root User](#5-managing-docker-as-a-non-root-user)
6. [Testing the Installation](#6-testing-the-installation)

---

## 1. Prerequisites

Before starting, update your package lists and system packages:

```bash
sudo apt update && sudo apt upgrade -y
```

Make sure required utility packages are installed:

```bash
sudo apt install -y curl wget git nano ca-certificates gnupg lsb-release
```

---

## 2. Configuring a Static IP Address

Setting a static IP ensures your TV Box always retains the same IP address on your local network (essential for servers, Home Assistant, Pi-hole, Docker containers, etc.).

### Method 1: Using `nmtui` (Recommended)

`nmtui` is the NetworkManager Text User Interface built into Armbian.

1. Open `nmtui`:
   ```bash
   sudo nmtui
   ```

2. Select **Edit a connection** and press **Enter**:
   ```
   ┌─ NetworkManager TUI ─┐
   │                      │
   │ Edit a connection    │ <--- Select this
   │ Activate a connection│
   │ Set system hostname  │
   │                      │
   │ Quit                 │
   └──────────────────────┘
   ```

3. Choose your network interface:
   - **Ethernet (LAN):** e.g., `Wired connection 1` or `eth0` / `end0`.
   - **Wi-Fi:** e.g., your Wi-Fi SSID or `wlan0`.

4. Navigate to **IPv4 CONFIGURATION**:
   - Change from `<Automatic>` to `<Manual>`.
   - Select **<Show>**.

5. Enter your network details:
   - **Addresses:** e.g., `192.168.1.100/24` *(Replace with your desired IP and subnet)*
   - **Gateway:** e.g., `192.168.1.1` *(Your router IP)*
   - **DNS servers:** `8.8.8.8` (or `1.1.1.1`)

6. Scroll down to select **<OK>**, then select **<Back>** and **<Quit>**.

7. Restart NetworkManager to apply changes:
   ```bash
   sudo systemctl restart NetworkManager
   ```

8. Verify your new static IP:
   ```bash
   ip a
   ```

---

### Method 2: Using `armbian-config`

1. Launch Armbian config:
   ```bash
   sudo armbian-config
   ```
2. Navigate to **Network** > **IP** > **Static**.
3. Select the desired adapter (`eth0` or `wlan0`).
4. Enter your IP, Subnet Mask, Gateway, and DNS server.
5. Save and exit.

---

### Method 3: Using `nmcli` (Command Line)

If you prefer configuring via commands without interactive menus:

1. List available network connections:
   ```bash
   nmcli connection show
   ```

2. Set static IP (replace `"Wired connection 1"` with your connection name):
   ```bash
   sudo nmcli connection modify "Wired connection 1" \
     ipv4.method manual \
     ipv4.addresses 192.168.1.100/24 \
     ipv4.gateway 192.168.1.1 \
     ipv4.dns "8.8.8.8 1.1.1.1"
   ```

3. Reactivate the connection:
   ```bash
   sudo nmcli connection up "Wired connection 1"
   ```

---

## 3. Installing Docker

### Option A: Official Docker Automated Script (Fastest & Recommended)

1. Download and run the official installation script:
   ```bash
   curl -fsSL https://get.docker.com -o get-docker.sh
   sudo sh ./get-docker.sh
   ```

2. Remove the temporary installer script:
   ```bash
   rm -f get-docker.sh
   ```

---

### Option B: Official Debian Repository Method

If you prefer manual repository configuration:

1. Add Docker’s official GPG key:
   ```bash
   sudo install -m 0755 -d /etc/apt/keyrings
   sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
   sudo chmod a+r /etc/apt/keyrings/docker.asc
   ```

2. Set up the repository:
   ```bash
   echo \
     "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
     $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
     sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   ```

3. Update package index and install Docker:
   ```bash
   sudo apt update
   sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   ```

---

## 4. Installing Docker Compose

With modern Docker (Docker V2), **Docker Compose** is installed as a plugin (`docker compose`):

1. Verify or install the Compose plugin:
   ```bash
   sudo apt update
   sudo apt install -y docker-compose-plugin
   ```

2. *(Optional)* If you need legacy `docker-compose` command compatibility, install the standalone wrapper:
   ```bash
   sudo apt install -y docker-compose
   ```
   *Or create an alias in `~/.bashrc`:*
   ```bash
   echo 'alias docker-compose="docker compose"' >> ~/.bashrc
   source ~/.bashrc
   ```

---

## 5. Managing Docker as a Non-Root User

By default, running `docker` commands requires `sudo`. To run Docker commands as a normal user:

1. Add your user to the `docker` group:
   ```bash
   sudo usermod -aG docker $USER
   ```

2. Apply group changes:
   ```bash
   newgrp docker
   ```
   *(Or log out and log back in via SSH)*

3. Enable Docker to start automatically on system boot:
   ```bash
   sudo systemctl enable docker
   sudo systemctl start docker
   ```

---

## 6. Testing the Installation

### 1. Check Docker Version
```bash
docker --version
```
*Example output:*
`Docker version 27.x.x, build ...`

### 2. Check Docker Compose Version
```bash
docker compose version
```
*Example output:*
`Docker Compose version v2.x.x`

### 3. Run a Hello-World Test Container
```bash
docker run --rm hello-world
```
If successful, you will see the message:
> **Hello from Docker! This message shows that your installation appears to be working correctly.**

---

## 🚀 Next Steps: Running Your First Compose Stack

Create a `docker-compose.yml` file:

```bash
mkdir -p ~/portainer && cd ~/portainer
nano docker-compose.yml
```

Paste the following example to deploy **Portainer CE** (Docker Web UI):

```yaml
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: always
    ports:
      - "9000:9000"
      - "9443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

volumes:
  portainer_data:
```

Start the container stack:
```bash
docker compose up -d
```

Access Portainer web interface in your browser:
```
http://<YOUR_STATIC_IP>:9000
```
*(e.g., `http://192.168.1.100:9000`)*
