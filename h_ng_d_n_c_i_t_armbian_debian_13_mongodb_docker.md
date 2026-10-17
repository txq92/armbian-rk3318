# Hướng Dẫn Cấu Hình Armbian (Debian 13) & Cài Đặt MongoDB Qua Docker

> **Thông tin hệ thống mục tiêu:**
> - **Thiết bị:** Rockchip RK3318 Box (Kiến trúc ARMv8.0-A / Cortex-A53 `aarch64`)
> - **Hệ điều hành:** Armbian Linux (Debian 13 - Trixie)
> - **RAM:** 4 GB | **Bộ nhớ eMMC:** 64 GB (`mmcblk2`)
> - **Địa chỉ IP tĩnh:** `192.168.1.234`
> - **Thư mục dữ liệu MongoDB:** `/root/mongo`

---

## 1. Thiết lập địa chỉ IP tĩnh (`192.168.1.234`) bằng Netplan

### Bước 1.1: Sao lưu cấu hình mạng mặc định
Di chuyển các file cấu hình DHCP mặc định sang thư mục `/root/` để tránh xung đột:
```bash
ls -la /etc/netplan/
mv /etc/netplan/*.yaml /root/
```

### Bước 1.2: Tạo file cấu hình Netplan mới
Tạo file `/etc/netplan/01-netcfg.yaml` cho cổng mạng `eth0`:
```bash
cat << 'EOF' > /etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    eth0:
      dhcp4: no
      dhcp6: no
      addresses:
        - 192.168.1.234/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
EOF
```

### Bước 1.3: Cấp quyền bảo mật và áp dụng cấu hình
```bash
chmod 600 /etc/netplan/01-netcfg.yaml
netplan apply
```

### Bước 1.4: Kiểm tra kết nối
```bash
ip -4 a show eth0
ping -c 4 google.com
```

---

## 2. Các lệnh kiểm tra phần cứng trên Armbian / Debian

- **Kiểm tra thông tin CPU:**
  ```bash
  lscpu
  ```
- **Kiểm tra dung lượng RAM và Swap:**
  ```bash
  free -h
  ```
- **Kiểm tra phân vùng ổ cứng eMMC / Thẻ nhớ:**
  ```bash
  lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,MODEL
  df -hT
  ```
- **Kiểm tra tên bo mạch và nhiệt độ CPU:**
  ```bash
  cat /proc/device-tree/model; echo ""
  cat /sys/class/thermal/thermal_zone0/temp
  ```

---

## 3. Cài đặt Docker & Docker Compose trên Debian 13 (Trixie)

### Bước 3.1: Cài đặt các gói phụ thuộc và GPG Key
```bash
apt update
apt install -y ca-certificates curl gnupg
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
```

### Bước 3.2: Thêm kho lưu trữ Docker chính thức
```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null
apt update
```

### Bước 3.3: Cài đặt Docker Engine và Docker Compose Plugin
```bash
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Bước 3.4: Giới hạn dung lượng Log để bảo vệ bộ nhớ eMMC
```bash
mkdir -p /etc/docker
cat << 'EOF' > /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF
systemctl restart docker
```

---

## 4. Cài đặt MongoDB qua Docker (Tối ưu cho chip RK3318 ARMv8.0)

> **Lưu ý quan trọng:** Chip Rockchip RK3318 sử dụng tập lệnh `ARMv8.0-A` (Cortex-A53). Do đó cần sử dụng bản **MongoDB 7.0 biên dịch cho ARMv8.0** (`dburszty/mongodb-raspberrypi:7.0.14`), **MongoDB 4.4.18 chính thức** (`mongo:4.4.18`), hoặc **FerretDB**.

### Phương án A: Cài đặt MongoDB 7.0.14 (hoặc 4.4.18) tại `/root/mongo`

#### 1. Tạo thư mục lưu trữ dữ liệu
```bash
mkdir -p /root/mongo/data
cd /root/mongo
```

#### 2. Tạo file `docker-compose.yml`
```bash
cat << 'EOF' > /root/mongo/docker-compose.yml
services:
  mongodb:
    image: dburszty/mongodb-raspberrypi:7.0.14
    # Nếu muốn dùng bản 4.4 chính thức, đổi dòng trên thành: image: mongo:4.4.18
    container_name: mongodb
    restart: unless-stopped
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: PasswordCuaBan123
    volumes:
      - /root/mongo/data:/data/db
    command: ["mongod", "--wiredTigerCacheSizeGB", "0.5", "--bind_ip_all"]
    deploy:
      resources:
        limits:
          memory: 800M
EOF
```

#### 3. Khởi chạy và kiểm tra
```bash
cd /root/mongo
docker compose up -d
docker ps
ls -la /root/mongo/data
```

- **Chuỗi kết nối (Connection String) từ máy tính trong mạng LAN (MongoDB Compass / App):**
  ```text
  mongodb://admin:PasswordCuaBan123@192.168.1.234:27017/
  ```

---

### Phương án B: Cài đặt FerretDB (Giải pháp thay thế siêu nhẹ ~30MB RAM)

Nếu chỉ cần hệ quản trị tương thích giao thức MongoDB nhưng tiêu tốn cực ít RAM trên TV Box:

```bash
mkdir -p /root/mongo/state
cat << 'EOF' > /root/mongo/docker-compose.yml
services:
  ferretdb:
    image: ghcr.io/ferretdb/ferretdb:latest
    container_name: ferretdb
    restart: unless-stopped
    ports:
      - "27017:27017"
    environment:
      - FERRETDB_HANDLER=sqlite
      - FERRETDB_TELEMETRY=disable
    volumes:
      - /root/mongo/state:/state
EOF
cd /root/mongo && docker compose up -d
```

---

## 5. Các lệnh quản lý Container MongoDB thường dùng

| Thao tác | Lệnh thực thi |
| :--- | :--- |
| Xem trạng thái hoạt động & RAM tiêu thụ | `docker stats --no-stream mongodb` |
| Xem nhật ký (Log) của MongoDB | `docker logs -f mongodb` |
| Khởi động lại MongoDB | `cd /root/mongo && docker compose restart` |
| Tắt dịch vụ MongoDB | `cd /root/mongo && docker compose down` |
| Sao lưu nhanh thư mục dữ liệu | `tar -czvf /root/mongo_backup_$(date +%F).tar.gz /root/mongo/data` |