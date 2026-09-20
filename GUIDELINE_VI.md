# Hướng Dẫn Cấu Hình Armbian (Debian): Cài Đặt IP Tĩnh, Docker & Docker Compose

Tài liệu này hướng dẫn chi tiết từng bước cách cấu hình **IP tĩnh**, cài đặt **Docker** và **Docker Compose** trên hệ điều hành Armbian (nền Debian) chạy trên TV Box (Rockchip RK3318 / RK3228 / Amlogic).

---

## 📑 Mục Lục
1. [Chuẩn Bị](#1-chuẩn-bị)
2. [Cấu Hình Địa Chỉ IP Tĩnh (Static IP)](#2-cấu-hình-địa-chỉ-ip-tĩnh-static-ip)
   - [Cách 1: Sử dụng `nmtui` (Khuyên dùng - Giao diện trực quan)](#cách-1-sử-dụng-nmtui-khuyên-dùng)
   - [Cách 2: Sử dụng `armbian-config`](#cách-2-sử-dụng-armbian-config)
   - [Cách 3: Sử dụng lệnh `nmcli`](#cách-3-sử-dụng-lệnh-nmcli)
3. [Cài Đặt Docker](#3-cài-đặt-docker)
4. [Cài Đặt Docker Compose](#4-cài-đặt-docker-compose)
5. [Cấp Quyền Cho User Thường (Non-Root User)](#5-cấp-quyền-cho-user-thường-non-root-user)
6. [Kiểm Tra Cài Đặt & Triển Khai Thử Nghiệm](#6-kiểm-tra-cài-đặt--triển-khai-thử-nghiệm)

---

## 1. Chuẩn Bị

Trước khi bắt đầu, hãy cập nhật toàn bộ danh sách gói và hệ thống:

```bash
sudo apt update && sudo apt upgrade -y
```

Cài đặt các gói công cụ cần thiết:

```bash
sudo apt install -y curl wget git nano ca-certificates gnupg lsb-release
```

---

## 2. Cấu Hình Địa Chỉ IP Tĩnh (Static IP)

Việc gán IP tĩnh giúp TV Box luôn giữ nguyên địa chỉ IP trong mạng nội bộ, rất cần thiết khi chạy server, Home Assistant, Pi-hole hoặc các dịch vụ Docker.

### Cách 1: Sử dụng `nmtui` (Khuyên dùng)

`nmtui` là giao diện đồ họa trực quan trên terminal của NetworkManager tích hợp sẵn trong Armbian.

1. Mở công cụ `nmtui`:
   ```bash
   sudo nmtui
   ```

2. Chọn mục **Edit a connection** và nhấn **Enter**:
   ```
   ┌─ NetworkManager TUI ─┐
   │                      │
   │ Edit a connection    │ <--- Chọn dòng này
   │ Activate a connection│
   │ Set system hostname  │
   │                      │
   │ Quit                 │
   └──────────────────────┘
   ```

3. Chọn cổng mạng bạn muốn đặt IP tĩnh:
   - **Mạng dây (Ethernet):** thường là `Wired connection 1` hoặc `eth0` / `end0`.
   - **Mạng không dây (Wi-Fi):** tên mạng Wi-Fi đã kết nối hoặc `wlan0`.

4. Di chuyển xuống phần **IPv4 CONFIGURATION**:
   - Chuyển từ `<Automatic>` sang `<Manual>`.
   - Chọn **<Show>** để hiển thị các ô nhập liệu.

5. Nhập các thông số mạng của bạn:
   - **Addresses:** ví dụ `192.168.1.100/24` *(Thay đổi IP phù hợp với dải mạng nhà bạn, `/24` tương ứng subnet mask `255.255.255.0`)*
   - **Gateway:** ví dụ `192.168.1.1` *(Địa chỉ IP của Router nhà bạn)*
   - **DNS servers:** `8.8.8.8` hoặc `1.1.1.1`

6. Di chuyển xuống chọn **<OK>**, sau đó chọn **<Back>** và chọn **<Quit>**.

7. Khởi động lại NetworkManager để áp dụng cấu hình:
   ```bash
   sudo systemctl restart NetworkManager
   ```

8. Kiểm tra lại địa chỉ IP mới:
   ```bash
   ip a
   ```

---

### Cách 2: Sử dụng `armbian-config`

1. Chạy trình cấu hình của Armbian:
   ```bash
   sudo armbian-config
   ```
2. Đi đến: **Network** > **IP** > **Static**.
3. Chọn cổng mạng muốn cấu hình (`eth0` hoặc `wlan0`).
4. Nhập lần lượt IP, Subnet Mask, Gateway, DNS server.
5. Lưu lại và thoát.

---

### Cách 3: Sử dụng lệnh `nmcli` (Dòng lệnh nhanh)

Nếu muốn cấu hình nhanh qua một dòng lệnh mà không cần qua menu:

1. Xem danh sách kết nối hiện tại:
   ```bash
   nmcli connection show
   ```

2. Gán IP tĩnh (thay `"Wired connection 1"` bằng tên kết nối mạng của bạn):
   ```bash
   sudo nmcli connection modify "Wired connection 1" \
     ipv4.method manual \
     ipv4.addresses 192.168.1.100/24 \
     ipv4.gateway 192.168.1.1 \
     ipv4.dns "8.8.8.8 1.1.1.1"
   ```

3. Áp dụng ngay kết nối:
   ```bash
   sudo nmcli connection up "Wired connection 1"
   ```

---

## 3. Cài Đặt Docker

### Lựa chọn A: Dùng Script Tự Động Chính Thức Của Docker (Nhanh & Khuyên Dùng)

1. Tải và chạy script cài đặt chính thức:
   ```bash
   curl -fsSL https://get.docker.com -o get-docker.sh
   sudo sh ./get-docker.sh
   ```

2. Xóa file cài đặt tạm:
   ```bash
   rm -f get-docker.sh
   ```

---

### Lựa chọn B: Cài Đặt Qua Repository Debian Chính Thức

Nếu bạn muốn quản lý nguồn kho lưu trữ (repository) thủ công:

1. Thêm GPG key chính thức của Docker:
   ```bash
   sudo install -m 0755 -d /etc/apt/keyrings
   sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
   sudo chmod a+r /etc/apt/keyrings/docker.asc
   ```

2. Thiết lập kho lưu trữ:
   ```bash
   echo \
     "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
     $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
     sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   ```

3. Cập nhật và cài đặt Docker Engine:
   ```bash
   sudo apt update
   sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   ```

---

## 4. Cài Đặt Docker Compose

Trên phiên bản Docker mới (Docker V2), **Docker Compose** đã được tích hợp sẵn dưới dạng plugin lệnh `docker compose`:

1. Cài đặt hoặc kiểm tra plugin Docker Compose:
   ```bash
   sudo apt update
   sudo apt install -y docker-compose-plugin
   ```

2. *(Tùy chọn)* Nếu bạn muốn dùng lệnh cũ `docker-compose` thay vì `docker compose`, tạo alias bằng cách chạy:
   ```bash
   echo 'alias docker-compose="docker compose"' >> ~/.bashrc
   source ~/.bashrc
   ```

---

## 5. Cấp Quyền Cho User Thường (Non-Root User)

Mặc định, lệnh `docker` yêu cầu quyền `sudo`. Để chạy trực tiếp mà không cần gõ `sudo`:

1. Thêm tài khoản hiện tại vào nhóm `docker`:
   ```bash
   sudo usermod -aG docker $USER
   ```

2. Kích hoạt nhóm mới:
   ```bash
   newgrp docker
   ```
   *(Hoặc đăng xuất SSH và đăng nhập lại)*

3. Thiết lập Docker tự động bật khi khởi động thiết bị:
   ```bash
   sudo systemctl enable docker
   sudo systemctl start docker
   ```

---

## 6. Kiểm Tra Cài Đặt & Triển Khai Thử Nghiệm

### 1. Kiểm tra phiên bản Docker
```bash
docker --version
```
*Kết quả mẫu:* `Docker version 27.x.x, build ...`

### 2. Kiểm tra phiên bản Docker Compose
```bash
docker compose version
```
*Kết quả mẫu:* `Docker Compose version v2.x.x`

### 3. Chạy thử container Hello-World
```bash
docker run --rm hello-world
```
Nếu màn hình hiện thông báo:
> **Hello from Docker! This message shows that your installation appears to be working correctly.**  
thì Docker đã hoạt động hoàn toàn chính xác!

---

## 🚀 Bước Tiếp Theo: Chạy Thử Stack Docker Compose Đầu Tiên

Tạo file `docker-compose.yml` để cài đặt **Portainer CE** (giao diện Web quản lý Docker):

```bash
mkdir -p ~/portainer && cd ~/portainer
nano docker-compose.yml
```

Dán nội dung sau vào file:

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

Khởi chạy container:
```bash
docker compose up -d
```

Truy cập giao diện Web Portainer từ trình duyệt máy tính của bạn:
```
http://<IP_TINH_CUA_BOX>:9000
```
*(Ví dụ: `http://192.168.1.100:9000`)*
