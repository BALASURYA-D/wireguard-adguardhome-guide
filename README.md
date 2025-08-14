# Trang chủ WireGuard & AdGuardHome của tôi

> Tài liệu thiết lập **WireGuard** và **AdGuard Home** trên Ubuntu với Docker + Nginx reverse proxy, kèm các lệnh nhanh, mẹo bảo mật và cách tự động hoá.  
> **Cảnh báo:** Thay **tất cả** giá trị ví dụ (IP, domain, đường dẫn file, mật khẩu, key) bằng của **bạn**. **Không commit khoá riêng (private key) hay file chứng chỉ** lên GitHub.

<p align="left">
  <img alt="Ubuntu" src="https://img.shields.io/badge/Ubuntu-22.04%7C24.04-E95420?logo=ubuntu&logoColor=white">
  <img alt="Docker" src="https://img.shields.io/badge/Docker-Engine-2496ED?logo=docker&logoColor=white">
  <img alt="Nginx" src="https://img.shields.io/badge/Nginx-Proxy-009639?logo=nginx&logoColor=white">
  <img alt="WireGuard" src="https://img.shields.io/badge/WireGuard-wg--easy-88171A?logo=wireguard&logoColor=white">
  <img alt="AdGuard" src="https://img.shields.io/badge/AdGuard-Home-68BC71?logo=adguard&logoColor=white">
  <img alt="License" src="https://img.shields.io/badge/License-MIT-green">
</p>

---

## Mục lục
- [1) Chuẩn bị & lưu ý bảo mật](#1-chuẩn-bị--lưu-ý-bảo-mật)
- [2) WireGuard (wg-easy)](#2-wireguard-wg-easy)
  - [2.1 Xoá lịch sử lệnh (không bắt buộc)](#21-xoá-lịch-sử-lệnh-không-bắt-buộc)
  - [2.2 SSH key & SSH vào server](#22-ssh-key--ssh-vào-server)
  - [2.3 Cập nhật hệ thống](#23-cập-nhật-hệ-thống)
  - [2.4 Cài Docker Engine](#24-cài-docker-engine)
  - [2.5 Chạy wg-easy (Docker)](#25-chạy-wg-easy-docker)
  - [2.6 Nâng cấp wg-easy](#26-nâng-cấp-wg-easy)
  - [2.7 Reverse proxy qua Nginx](#27-reverse-proxy-qua-nginx)
  - [2.8 Sử dụng WireGuard trên client](#28-sử-dụng-wireguard-trên-client)
  - [2.9 Tự động chạy wg-quick với systemd](#29-tự-động-chạy-wg-quick-với-systemd)
- [3) AdGuard Home](#3-adguard-home)
  - [3.1 SSH vào server & cập nhật](#31-ssh-vào-server--cập-nhật)
  - [3.2 Cài Docker & docker-compose plugin](#32-cài-docker--docker-compose-plugin)
  - [3.3 Tạo & chạy container AdGuard Home](#33-tạo--chạy-container-adguard-home)
  - [3.4 Giải phóng cổng 53 & đặt DNS upstream](#34-giải-phóng-cổng-53--đặt-dns-upstream)
  - [3.5 UFW firewall](#35-ufw-firewall)
  - [3.6 Tắt IPv6 (tùy chọn)](#36-tắt-ipv6-tùy-chọn)
  - [3.7 SSL/TLS với Let’s Encrypt](#37-ssltls-với-lets-encrypt)
  - [3.8 Reverse proxy cho AdGuard UI](#38-reverse-proxy-cho-adguard-ui)
  - [3.9 Gia hạn chứng chỉ](#39-gia-hạn-chứng-chỉ)
- [4) Swap (RAM ảo)](#4-swap-ram-ảo)
- [5) Ghi chú bảo mật SSH & geo-whitelist (tùy chọn)](#5-ghi-chú-bảo-mật-ssh--geo-whitelist-tùy-chọn)
- [6) License](#6-license)

---

## 1) Chuẩn bị & lưu ý bảo mật

> [!IMPORTANT]
> - **Thay thế** mọi giá trị ví dụ: `YOUR_SERVER_IP`, `your.domain`, đường dẫn `~/.ssh/your_key`, v.v.
> - **Không commit**: private key, file `*.pem`, `*.conf` chứa key/secret. Lưu trong **Secret Manager** hoặc máy chủ.
> - Sao lưu trước khi thay đổi dịch vụ mạng (DNS, systemd-resolved, UFW…).

---

## 2) WireGuard (wg-easy)

### 2.1 Xoá lịch sử lệnh (không bắt buộc)
> [!WARNING]  
> Xóa lịch sử có thể gây khó debug. Dùng khi thực sự cần.
```bash
history -c && rm -f ~/.bash_history && gnome-terminal && exit
```

### 2.2 SSH key & SSH vào server
Tạo SSH key (RSA 4096) & phân quyền key:
```bash
ssh-keygen -t rsa -b 4096 -C "ubuntu" -f ~/.ssh/new_guard
chmod 400 ~/.ssh/new_guard
ssh -i ~/.ssh/new_guard ubuntu@YOUR_SERVER_IP
```

### 2.3 Cập nhật hệ thống
```bash
sudo apt update && sudo apt upgrade -y
```

### 2.4 Cài Docker Engine
**Cách 1 (khuyên dùng – chính chủ Docker):**
```bash
curl -sSL https://get.docker.com | sh && sudo usermod -aG docker $(whoami) && sudo apt update && sudo apt install -y docker-compose-plugin nginx && sudo systemctl enable --now docker
```

**Cách 2 (APT Ubuntu):**
```bash
sudo apt install -y docker.io docker-compose nginx
sudo systemctl enable --now docker
```

### 2.5 Chạy wg-easy (Docker)
> [!TIP]
> `PASSWORD_HASH` là **bcrypt**. Tạo bằng:
> ```bash
> sudo apt install -y apache2-utils

> ```bash
> htpasswd -nbBC 12 "" "YourStrongPassword" | tr -d ':\n'

```bash
docker run --detach   --name wg-easy   --env LANG=en   --env WG_HOST=YOUR_SERVER_IP_OR_DOMAIN   --env PASSWORD_HASH='$2a$12$YOUR_BCRYPT_HASH_HERE'   --env PORT=51821   --env WG_PORT=51820   --env WG_MTU=1420   --env WG_PERSISTENT_KEEPALIVE=25   --volume ~/.wg-easy:/etc/wireguard   --publish 51820:51820/udp   --publish 51821:51821/tcp   --cap-add NET_ADMIN   --cap-add SYS_MODULE   --sysctl 'net.ipv4.conf.all.src_valid_mark=1'   --sysctl 'net.ipv4.ip_forward=1'   --restart unless-stopped   ghcr.io/wg-easy/wg-easy
```

### 2.6 Nâng cấp wg-easy
```bash
docker stop wg-easy && docker rm wg-easy
docker pull ghcr.io/wg-easy/wg-easy
# chạy lại với lệnh ở 2.5
```

### 2.7 Reverse proxy qua Nginx
**`/etc/nginx/sites-available/vpn.your.domain`**
```nginx
server {
    listen 80;
    server_name vpn.your.domain;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name vpn.your.domain;

    ssl_certificate /etc/letsencrypt/live/your.domain/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your.domain/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://127.0.0.1:51821;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```
Kích hoạt site & reload:
```bash
sudo ln -s /etc/nginx/sites-available/vpn.your.domain /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

### 2.8 Sử dụng WireGuard trên client
```bash
sudo apt install -y wireguard curl
sudo wg-quick up ~/Downloads/your-client.conf
curl ifconfig.me
sudo wg-quick down ~/Downloads/your-client.conf
```

### 2.9 Tự động chạy wg-quick với systemd
```bash
# Sao chép file client/peer (đã tạo từ wg-easy) làm wg0.conf
sudo cp ~/Downloads/your-client.conf /etc/wireguard/wg0.conf
sudo chmod 600 /etc/wireguard/wg0.conf
sudo chown root:root /etc/wireguard/wg0.conf

sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
sudo systemctl status wg-quick@wg0
```
> **Không khuyến nghị** dùng `crontab @reboot` thay cho systemd vì kém tin cậy & cần quyền sudo không mật khẩu.

---

## 3) AdGuard Home

### 3.1 SSH vào server & cập nhật
```bash
ssh -i ~/.ssh/new_guard ubuntu@YOUR_SERVER_IP
sudo apt update && sudo apt upgrade -y
```

### 3.2 Cài Docker & docker-compose plugin
**Cách 1 (khuyên dùng – chính chủ Docker):**
```bash
curl -sSL https://get.docker.com | sh && sudo usermod -aG docker $(whoami) && sudo apt update && sudo apt install -y docker-compose-plugin nginx && sudo systemctl enable --now docker
```

**Cách 2 (APT Ubuntu):**
```bash
sudo apt install -y docker.io docker-compose nginx
sudo systemctl enable --now docker
```

### 3.3 Tạo & chạy container AdGuard Home
```bash
sudo mkdir -p /opt/adguard/work /opt/adguard/conf
sudo chown -R $(whoami):$(whoami) /opt/adguard

docker pull adguard/adguardhome

docker run --name adguardhome   --restart unless-stopped   -v /opt/adguard/work:/opt/adguardhome/work   -v /opt/adguard/conf:/opt/adguardhome/conf   -p 53:53/tcp -p 53:53/udp   -p 3000:3000/tcp   -p 853:853/tcp -p 853:853/udp   -p 784:784/udp -p 8853:8853/udp   -p 5443:5443/tcp -p 5443:5443/udp   -d adguard/adguardhome
```
> Nếu dùng DHCP trong container, mở thêm `-p 67:67/udp -p 68:68/udp`  
> Nếu dùng HTTPS trực tiếp của AdGuard UI: thêm `-p 443:443/tcp -p 443:443/udp`

### 3.4 Giải phóng cổng 53 & đặt DNS upstream
```bash
sudo lsof -i :53

# (Tuỳ chọn) Tắt systemd-resolved nếu nó chiếm cổng 53
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved

# Đặt DNS tạm thời để máy còn phân giải tên miền
sudo rm /etc/resolv.conf
echo -e "nameserver 127.0.0.1
nameserver 1.1.1.1" | sudo tee /etc/resolv.conf

sudo reboot
```

### 3.5 UFW firewall
```bash
sudo apt install -y ufw
sudo ufw --force reset
sudo ufw default deny incoming
sudo ufw default allow outgoing

# DNS
sudo ufw allow 53/tcp
sudo ufw allow 53/udp

# HTTP/HTTPS + AdGuard UI
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 443/udp
sudo ufw allow 3000/tcp

# SSH
sudo ufw allow 22/tcp

# DoT/DoQ/DNSCrypt (nếu dùng)
sudo ufw allow 853/tcp
sudo ufw allow 784/udp
sudo ufw allow 853/udp
sudo ufw allow 8853/udp
sudo ufw allow 5443/tcp
sudo ufw allow 5443/udp

sudo ufw enable
sudo ufw status verbose
```

### 3.6 Tắt IPv6 (tùy chọn)
```bash
sudo bash -c 'cat >>/etc/sysctl.conf <<EOF
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
EOF'
sudo sysctl -p
```

### 3.7 SSL/TLS với Let’s Encrypt
```bash
sudo apt install -y certbot
sudo docker stop adguardhome
sudo certbot certonly --standalone   -d your.domain -d www.your.domain -d dns.your.domain -d vpn.your.domain
sudo docker start adguardhome
```
**Thay thế (DNS challenge):**
```bash
sudo certbot certonly --manual --preferred-challenges dns   -d your.domain -d www.your.domain -d dns.your.domain -d vpn.your.domain
# Làm theo hướng dẫn để thêm DNS TXT record _acme-challenge
```

> [!WARNING]  
> **Không** commit nội dung `-----BEGIN PRIVATE KEY-----` hay `fullchain.pem` vào repo.

### 3.8 Reverse proxy cho AdGuard UI
**`/etc/nginx/sites-available/dns.your.domain`**
```nginx
server {
    listen 80;
    server_name dns.your.domain;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name dns.your.domain;

    ssl_certificate /etc/letsencrypt/live/your.domain/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your.domain/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # Nếu AdGuard UI chạy HTTP trên 3000:
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```
Kích hoạt site:
```bash
sudo ln -s /etc/nginx/sites-available/dns.your.domain /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
sudo systemctl enable --now nginx
```

### 3.9 Gia hạn chứng chỉ
**Thủ công:**
```bash
sudo certbot renew --pre-hook "sudo docker stop adguardhome"   --post-hook "sudo docker start adguardhome"
```
**Tự động (cron):**
```bash
sudo crontab -e
# ví dụ chạy 02:30 mỗi ngày
30 2 * * * /usr/bin/certbot renew --pre-hook "sudo docker stop adguardhome" --post-hook "sudo docker start adguardhome" >> /var/log/le-renew.log 2>&1
```

---

## 4) Swap (RAM ảo)
```bash
# Tạo 4G swap
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Xoá swap
sudo swapoff /swapfile
sudo rm /swapfile
sudo sed -i '/\/swapfile none swap sw 0 0/d' /etc/fstab
df -h
```

---

## 5) Ghi chú bảo mật SSH & geo-whitelist (tùy chọn)

**SSH hardening cơ bản:**
```bash
# Giới hạn phiên/khởi tạo
echo "MaxSessions 2" | sudo tee -a /etc/ssh/sshd_config
echo "MaxStartups 2:50:5" | sudo tee -a /etc/ssh/sshd_config
# Tự đăng xuất sau 1h không hoạt động
echo -e "\nTMOUT=3600\nexport TMOUT" | sudo tee -a /etc/profile
sudo systemctl restart ssh
```

**Chỉ cho phép IP Việt Nam (UFW whitelist theo quốc gia) — ví dụ:**
```bash
sudo ufw --force reset
sudo ufw default deny incoming
sudo ufw default allow outgoing

wget https://www.ipdeny.com/ipblocks/data/countries/vn.zone
while read ip; do sudo ufw allow from "$ip"; done < vn.zone

sudo ufw enable
sudo ufw status numbered
```
> [!NOTE]  
> Cách này **khó bảo trì** (dải IP thay đổi). Cân nhắc giải pháp WAF/GeoIP upstream.

---

## 6) License

Phát hành theo **MIT License** — xem [LICENSE](./LICENSE).  
Gợi ý SPDX cho file script/tài liệu do bạn tạo:
```text
SPDX-License-Identifier: MIT
```
> Nội dung & tên thương hiệu của bên thứ ba giữ nguyên giấy phép và quyền sở hữu tương ứng.

---

### Tác giả
**THONG NGUYEN HOANG** — đóng góp/issue/PR luôn được chào mừng.

<br>

<p align="center">
  <a href="mailto:thongnguyenslife@gmail.com" aria-label="Email">
    <img alt="Email" src="https://img.shields.io/badge/Email-thongnguyenslife%40gmail.com-1a73e8?logo=gmail&logoColor=white&style=flat"/>
  </a>
  <a href="https://github.com/thongnguyenslife" aria-label="GitHub Profile">
    <img alt="GitHub" src="https://img.shields.io/badge/GitHub-@thongnguyenslife-1a73e8?logo=github&logoColor=white&style=flat"/>
  </a>
  <!-- Optional: LinkedIn (uncomment and set your handle)
  <a href="https://www.linkedin.com/in/your-id" aria-label="LinkedIn">
    <img alt="LinkedIn" src="https://img.shields.io/badge/LinkedIn-View_Profile-0A66C2?logo=linkedin&logoColor=white&style=flat" />
  </a> -->
</p>

<!-- End of README -->
