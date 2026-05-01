# Hướng dẫn build server stk-telebot trên Linux Mint 22 Cinnamon

> **Mục tiêu**: cài đặt từ đầu server tự build (Xeon E5-2686v4 + 48GB DDR3 ECC + Hiksemi 512GB NVMe + GT730) chạy stack stk-telebot, sẵn sàng đón stkapps về sau, scale lên 5-10 service Docker mà không cần đổi OS.
>
> **Đối tượng đọc**: dev tự cài lần đầu, có theo dõi từng bước. Một số bước có giải thích "vì sao" để admin non-tech sau này hiểu tại sao bấm.
>
> **Thời gian dự kiến**: 1 buổi (4-6 giờ) cho người chưa từng cài Linux. 2-3 giờ cho người quen.

---

## Mục lục

- [Phần 0 — Chuẩn bị trước khi đến server](#phần-0--chuẩn-bị-trước-khi-đến-server)
- [Phần 1 — BIOS setup](#phần-1--bios-setup)
- [Phần 2 — Cài Linux Mint 22 Cinnamon](#phần-2--cài-linux-mint-22-cinnamon)
- [Phần 3 — Tinh chỉnh OS cho server 24/7](#phần-3--tinh-chỉnh-os-cho-server-247)
- [Phần 4 — Cài driver + tool nền](#phần-4--cài-driver--tool-nền)
- [Phần 5 — Cài Docker + Tailscale + xrdp](#phần-5--cài-docker--tailscale--xrdp)
- [Phần 6 — Setup 5 instance Chrome cho host-agent](#phần-6--setup-5-instance-chrome-cho-host-agent)
- [Phần 7 — Port host-agent từ PowerShell sang bash](#phần-7--port-host-agent-từ-powershell-sang-bash)
- [Phần 8 — Deploy stk-telebot stack](#phần-8--deploy-stk-telebot-stack)
- [Phần 9 — Chuẩn bị migration stkapps](#phần-9--chuẩn-bị-migration-stkapps)
- [Phần 10 — Reverse proxy + Cloudflare Tunnel](#phần-10--reverse-proxy--cloudflare-tunnel)
- [Phần 11 — Backup + UPS + monitoring](#phần-11--backup--ups--monitoring)
- [Phần 12 — Verification checklist cuối](#phần-12--verification-checklist-cuối)
- [Phụ lục — cheat sheet lệnh hay dùng](#phụ-lục--cheat-sheet-lệnh-hay-dùng)

---

## Phần 0 — Chuẩn bị trước khi đến server

### 0.1 Tải ISO Linux Mint 22 Cinnamon

- Vào <https://linuxmint.com/edition.php?id=313>
- Chọn mirror gần Việt Nam (FPT/SiteGround) tải file `linuxmint-22-cinnamon-64bit.iso` (~2.7 GB)
- Verify checksum SHA256:
  ```
  sha256sum linuxmint-22-cinnamon-64bit.iso
  ```
  So với giá trị trên trang download, phải khớp 100%.

### 0.2 Tạo USB boot

**Trên máy Windows** (laptop dev của anh):

- Cài **Rufus** từ <https://rufus.ie> (portable, không cần install).
- Cắm USB ≥ 8 GB (USB 3.0 nhanh hơn USB 2.0).
- Mở Rufus → Device chọn USB → "Boot selection" trỏ tới ISO Mint vừa tải → Partition scheme: **GPT** + Target system: **UEFI (non-CSM)** → Start.
- Khi hỏi format: **Yes**. Khi hỏi mode: **ISO mode** (mặc định).
- Đợi ~5-10 phút.

> **Vì sao GPT + UEFI**: bo X99H-D3 hỗ trợ UEFI, dùng GPT giúp boot nhanh hơn + hỗ trợ ổ >2TB sau này.

### 0.3 Chuẩn bị USB phụ chứa file ngoài lề

Tạo USB thứ 2 (hoặc thư mục riêng trên USB Mint) chứa:

- **Driver Realtek** (nếu mạng LAN không nhận sau cài): tải `r8125` hoặc `r8168` từ Realtek tùy chip bo
- **Backup pg_dump cũ** từ VPS (nếu có sẵn, để khi migrate stkapps nhanh)
- **File này (`server-build-linux-mint.md`)** để mở đọc trên server khi cài

### 0.4 Liệt kê thông tin cần sẵn

| Thông tin | Giá trị anh chuẩn bị |
|---|---|
| Hostname server | (ví dụ `stkbot-server` hoặc `home-server`) |
| Username admin | `admin` (theo plan) |
| Password admin | đặt trước, ghi ra giấy, **mạnh**: 16+ ký tự |
| WiFi/LAN IP plan | dùng LAN cắm dây + DHCP cố định ở router |
| Tailscale account | đăng ký sẵn ở <https://tailscale.com> (Google login miễn phí) |
| Domain (cho stkapps về sau) | nếu có sẵn (ví dụ `stkapp.com`) |
| Cloudflare account | nếu domain quản lý qua Cloudflare |

---

## Phần 1 — BIOS setup

Cắm USB Mint vào server → bấm nút Power → spam phím **Del** (hoặc **F2**) để vào BIOS.

> **Bo X99H-D3** thường vào BIOS bằng `Del`. Nếu không vào được thử `F2`, `F11`, `F12`.

### 1.1 Cấu hình bắt buộc

| Mục | Giá trị | Vì sao |
|---|---|---|
| **Boot Mode** | UEFI Only (không CSM/Legacy) | match GPT của USB Mint |
| **Secure Boot** | **Disabled** | Mint không ký Microsoft, secure boot chặn |
| **SATA Mode** | AHCI (không Raid/IDE) | NVMe hoạt động đúng tốc độ |
| **Intel Virtualization VT-x** | **Enabled** | Docker container yêu cầu (qua KVM optional) |
| **Intel VT-d** | Enabled (nếu có) | dự phòng, không hại |
| **Above 4G Decoding** | Enabled (nếu có) | tránh memory mapping issue |
| **CPU Hyperthreading** | Enabled | có 36T thay vì 18T |
| **Memory Frequency** | 1866 MHz (đúng theo RAM mua) | tránh chạy 1333 mặc định |
| **Memory ECC** | Enabled (nếu có option) | RAM ECC mới phát huy |
| **Wake on LAN** | Disabled | tránh server tự bật |
| **Restore on Power Loss** | **Power On** | mất điện xong có điện lại tự bật |

### 1.2 Boot order

- Đặt **USB UEFI** lên đầu (chỉ tạm để cài, lát đổi lại thành SSD).
- Save BIOS (`F10` → Yes) → server reboot.

---

## Phần 2 — Cài Linux Mint 22 Cinnamon

### 2.1 Vào Live USB

Server boot từ USB → màn hình GRUB hiện ra:

- Chọn **"Start Linux Mint 22 Cinnamon 64-bit"** → Enter.
- Đợi 1-2 phút → desktop Mint hiện ra với icon **"Install Linux Mint"** ở góc trái.
- Test thử: cắm tai nghe phát thử nhạc, kết nối WiFi/LAN, đảm bảo network OK trước khi cài.

### 2.2 Bắt đầu cài

Double-click **"Install Linux Mint"**:

1. **Welcome → Language**: chọn **English** (khuyến nghị, vì lệnh terminal toàn English).
2. **Keyboard**: English (US).
3. **Wireless**: nếu đang dùng WiFi thì kết nối ngay; nếu LAN thì skip.
4. **Multimedia codecs**: ✅ tick "Install multimedia codecs" (cần để Chrome xem video Canva/YouTube).
5. **Installation type**: chọn **"Something else"** (advanced manual partitioning).

### 2.3 Phân vùng SSD 512 GB

Trên giao diện partitioner, chọn ổ Hiksemi NVMe (`/dev/nvme0n1`):

- Click **"New Partition Table"** → GPT.
- Tạo lần lượt 4 phân vùng theo bảng dưới:

| Partition | Size | Type | Mount point | Format | Vì sao |
|---|---|---|---|---|---|
| `/dev/nvme0n1p1` | **512 MB** | Primary, Beginning | `/boot/efi` | EFI System Partition | UEFI boot |
| `/dev/nvme0n1p2` | **16 GB** | Primary | (swap) | swap area | đệm RAM khi peak |
| `/dev/nvme0n1p3` | **80 GB** | Primary | `/` | ext4 | system + program |
| `/dev/nvme0n1p4` | (còn lại ~415 GB) | Primary | `/home` | ext4 | data, Docker, Chrome profile |

> **Vì sao tách `/home`**: sau này anh muốn cài lại OS chỉ cần format `/`, dữ liệu Chrome profile + Docker volume + stkapps trong `/home` được bảo toàn.

> **Vì sao swap 16 GB**: với 48 GB RAM, swap 16 GB là vừa (một số tool như Postgres autovacuum đôi khi cần swap khi BI peak). Không hibernate nên không cần swap = RAM.

- "Device for boot loader installation": chọn `/dev/nvme0n1` (toàn ổ).
- Click **"Install Now"** → Continue.

### 2.4 Timezone + User

- Timezone: chọn **Ho_Chi_Minh** trên bản đồ.
- User info:
  - Your name: `Admin`
  - Computer name: `stkbot-server` (hoặc gì anh thích)
  - Username: `admin`
  - Password: nhập password đã chuẩn bị
  - **Tick** "Log in automatically" (cần thiết để Chrome session restore khi reboot mất điện)
  - **Không** tick "Encrypt my home folder" (sẽ làm chậm I/O Postgres)

### 2.5 Đợi cài + reboot

- Đợi 10-15 phút.
- Khi báo "Installation Complete" → click **Restart Now**.
- Khi server prompt "Please remove the installation medium" → rút USB → Enter.
- Server boot vào Mint, **tự đăng nhập** vào user `admin` (vì đã tick auto-login).

---

## Phần 3 — Tinh chỉnh OS cho server 24/7

Mở **Terminal** (nút ☰ trên panel hoặc `Ctrl+Alt+T`).

### 3.1 Update toàn bộ hệ thống

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y curl wget git build-essential vim htop iotop nethogs net-tools tree unzip zip
```

Reboot 1 lần để áp kernel mới (nếu update kernel):

```bash
sudo reboot
```

### 3.2 Tắt sleep + screen lock + screensaver

> **Vì sao**: server chạy 24/7, không bao giờ sleep. Cinnamon mặc định khoá màn hình sau 15 phút → admin phải nhập password để mở Chrome.

Vào **Menu → Preferences → Screensaver**:
- "Delay before starting the screensaver" → **Never**
- Tab Settings → uncheck "Lock the computer when put to sleep"

Vào **Menu → Preferences → Power Management**:
- "Turn off the screen when inactive for" → **Never**
- "Suspend when inactive for" → **Never**

Hoặc lệnh terminal:

```bash
gsettings set org.cinnamon.desktop.screensaver lock-enabled false
gsettings set org.cinnamon.desktop.screensaver idle-activation-enabled false
gsettings set org.cinnamon.desktop.session idle-delay 0
gsettings set org.cinnamon.settings-daemon.plugins.power sleep-display-ac 0
gsettings set org.cinnamon.settings-daemon.plugins.power sleep-inactive-ac-timeout 0
```

### 3.3 Tăng giới hạn file descriptor + vm settings

Postgres/Docker/Chrome ăn nhiều file descriptor.

```bash
# /etc/security/limits.conf — thêm cuối file
sudo tee -a /etc/security/limits.conf > /dev/null <<EOF
admin    soft    nofile    65535
admin    hard    nofile    65535
admin    soft    nproc     32768
admin    hard    nproc     32768
EOF

# /etc/sysctl.conf — thêm cuối file
sudo tee -a /etc/sysctl.conf > /dev/null <<EOF
# Server tuning for stk-telebot
vm.swappiness=10
vm.overcommit_memory=1
fs.inotify.max_user_watches=524288
net.core.somaxconn=4096
net.ipv4.tcp_max_syn_backlog=4096
EOF

sudo sysctl -p
```

> **`vm.swappiness=10`**: ưu tiên RAM hơn swap (mặc định 60). BI query không bị đẩy vào swap.

### 3.4 Tắt unattended-upgrade tự reboot

> **Vì sao**: admin không muốn server tự reboot lúc 6 giờ sáng giữa lúc khách đặt đơn.

```bash
sudo dpkg-reconfigure -plow unattended-upgrades
# Chọn No
```

Hoặc edit thẳng:
```bash
sudo nano /etc/apt/apt.conf.d/20auto-upgrades
# Đổi:
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "0";
```

### 3.5 Đặt hostname + /etc/hosts

```bash
sudo hostnamectl set-hostname stkbot-server
echo "127.0.1.1  stkbot-server" | sudo tee -a /etc/hosts
```

---

## Phần 4 — Cài driver + tool nền

### 4.1 Driver NVIDIA GT730 (Kepler)

GT730 dùng chip Kepler GK208 → cần driver legacy 470.

```bash
# Mở Driver Manager GUI
mintdrivers
```

Chọn driver **`nvidia-driver-470`** (ghi "tested, recommended") → Apply Changes → đợi cài → reboot.

Sau reboot test:
```bash
nvidia-smi
```
Phải hiển thị tên GPU "GeForce GT 730".

> Nếu Mint không tự detect, gõ:
> ```bash
> sudo apt install -y nvidia-driver-470
> sudo reboot
> ```

### 4.2 Python 3.12 + venv + pip

Mint 22 đã có Python 3.12 sẵn. Verify:
```bash
python3 --version  # phải là 3.12.x
sudo apt install -y python3-pip python3-venv
```

### 4.3 Node.js LTS (cho admin-ui dev nếu cần build local)

```bash
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install -y nodejs
node --version  # >= v20
```

### 4.4 Git + cấu hình

```bash
git config --global user.name "Trieu Phan"
git config --global user.email "phantrieu3701@gmail.com"
git config --global init.defaultBranch main
git config --global pull.rebase false
```

### 4.5 VS Code (optional, dùng remote-ssh từ laptop nhưng có VS Code trên server cũng ok)

```bash
sudo apt install -y software-properties-common apt-transport-https wget
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/microsoft.gpg > /dev/null
sudo add-apt-repository "deb [arch=amd64] https://packages.microsoft.com/repos/code stable main"
sudo apt update && sudo apt install -y code
```

---

## Phần 5 — Cài Docker + Tailscale + xrdp

### 5.1 Docker Engine + Compose plugin (KHÔNG dùng Docker Desktop)

> **Vì sao không dùng Docker Desktop**: trên Linux, Desktop là phiên bản đóng gói thừa của Engine. Engine native hoạt động trực tiếp trên kernel = nhanh + nhẹ + free.

```bash
# Gỡ phiên bản cũ nếu có
sudo apt remove docker docker-engine docker.io containerd runc 2>/dev/null

# Cài prereq
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

# Add Docker GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add repo (Mint 22 dựa trên Ubuntu noble)
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu noble stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Cho admin chạy docker không cần sudo
sudo usermod -aG docker admin
newgrp docker  # apply ngay không cần logout

# Test
docker run hello-world
docker compose version
```

### 5.2 Cấu hình Docker daemon (log rotate + data root)

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "5"
  },
  "data-root": "/home/admin/docker-data",
  "storage-driver": "overlay2"
}
EOF

sudo mkdir -p /home/admin/docker-data
sudo systemctl restart docker
docker info | grep "Docker Root Dir"  # phải là /home/admin/docker-data
```

> **Vì sao move data-root về `/home`**: `/` chỉ 80 GB, Docker images + volumes có thể lên 100+ GB sau này. `/home` ~415 GB rộng rãi.

### 5.3 Tailscale (VPN private cho remote dev)

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Lệnh `up` sẽ in 1 URL → mở trên laptop dev → đăng nhập tailnet → server join. Sau đó server có IP Tailscale dạng `100.x.x.x`.

Trên laptop dev cài Tailscale → đăng nhập cùng tailnet → ping server bằng `tailscale ping stkbot-server`.

### 5.4 xrdp (cho Microsoft Remote Desktop từ Windows laptop)

```bash
sudo apt install -y xrdp
sudo systemctl enable --now xrdp
sudo ufw allow 3389/tcp comment 'xrdp'

# Add user xrdp vào group ssl-cert
sudo adduser xrdp ssl-cert
sudo systemctl restart xrdp
```

Test từ laptop Windows: mở **Remote Desktop Connection** (mstsc) → nhập `100.x.x.x` (Tailscale IP) → username `admin` + password.

> **Lưu ý**: khi RDP vào, Cinnamon session phải khác session đang login local trên server (xrdp tạo session mới qua Xorg). Nếu lỗi "There is already a session" → log out user local trước khi RDP.

### 5.5 OpenSSH Server (đã có sẵn trên Mint, verify)

```bash
sudo systemctl status ssh
# Nếu chưa enable:
sudo systemctl enable --now ssh

# Tạo SSH key trên laptop dev → copy về server
# (Trên LAPTOP DEV, không phải server):
ssh-keygen -t ed25519 -C "trieu-laptop"
ssh-copy-id admin@<tailscale-ip>

# Test login bằng key
ssh admin@<tailscale-ip>
```

Sau khi key OK, disable password SSH:
```bash
# Trên server
sudo nano /etc/ssh/sshd_config
# Sửa các dòng:
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no

sudo systemctl restart ssh
```

### 5.6 UFW firewall

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Chỉ cho Tailscale subnet
sudo ufw allow in on tailscale0
# Cho LAN nội bộ (giả sử LAN 192.168.1.0/24)
sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp
sudo ufw allow from 192.168.1.0/24 to any port 3389 proto tcp
sudo ufw allow from 192.168.1.0/24 to any port 8880 proto tcp comment 'stk-telebot api'
sudo ufw allow from 192.168.1.0/24 to any port 3000 proto tcp comment 'admin-ui'
sudo ufw allow from 192.168.1.0/24 to any port 9222:9230 proto tcp comment 'chrome cdp + host-agent'

sudo ufw enable
sudo ufw status verbose
```

---

## Phần 6 — Setup 5 instance Chrome cho host-agent

### 6.1 Cài Google Chrome stable

```bash
wget -O /tmp/chrome.deb https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo apt install -y /tmp/chrome.deb
google-chrome --version  # >= 120
```

### 6.2 Tạo 5 thư mục user-data-dir

```bash
mkdir -p /home/admin/chrome-profiles/{google,canva,autodesk,quizlet,datacamp}
```

### 6.3 Tạo 5 launcher .desktop trong `~/.local/share/applications/`

Mỗi launcher mở Chrome với port + user-data-dir riêng + URL gốc của site.

```bash
mkdir -p /home/admin/.local/share/applications

# === Google (port 9225 — theo memory project) ===
cat > /home/admin/.local/share/applications/chrome-google.desktop <<'EOF'
[Desktop Entry]
Version=1.0
Type=Application
Name=Chrome — Google Family
GenericName=Chrome CDP for Google Family invite
Exec=/usr/bin/google-chrome \
    --remote-debugging-port=9225 \
    --remote-debugging-address=0.0.0.0 \
    --remote-allow-origins=* \
    --remote-allow-hosts=host.docker.internal,localhost,127.0.0.1 \
    --user-data-dir=/home/admin/chrome-profiles/google \
    --disable-features=IsolateOrigins,site-per-process \
    --no-default-browser-check \
    --no-first-run \
    https://myaccount.google.com/family
Terminal=false
Icon=google-chrome
Categories=Network;WebBrowser;
StartupWMClass=Chrome-Google
EOF

# === Canva (port 9226) ===
cat > /home/admin/.local/share/applications/chrome-canva.desktop <<'EOF'
[Desktop Entry]
Version=1.0
Type=Application
Name=Chrome — Canva
Exec=/usr/bin/google-chrome \
    --remote-debugging-port=9226 \
    --remote-debugging-address=0.0.0.0 \
    --remote-allow-origins=* \
    --remote-allow-hosts=host.docker.internal,localhost,127.0.0.1 \
    --user-data-dir=/home/admin/chrome-profiles/canva \
    --no-default-browser-check \
    --no-first-run \
    https://www.canva.com/team/
Terminal=false
Icon=google-chrome
Categories=Network;WebBrowser;
StartupWMClass=Chrome-Canva
EOF

# === Autodesk (port 9223) ===
cat > /home/admin/.local/share/applications/chrome-autodesk.desktop <<'EOF'
[Desktop Entry]
Version=1.0
Type=Application
Name=Chrome — Autodesk
Exec=/usr/bin/google-chrome \
    --remote-debugging-port=9223 \
    --remote-debugging-address=0.0.0.0 \
    --remote-allow-origins=* \
    --remote-allow-hosts=host.docker.internal,localhost,127.0.0.1 \
    --user-data-dir=/home/admin/chrome-profiles/autodesk \
    --no-default-browser-check \
    --no-first-run \
    https://manage.autodesk.com/
Terminal=false
Icon=google-chrome
Categories=Network;WebBrowser;
StartupWMClass=Chrome-Autodesk
EOF

# === Quizlet (port 9224) ===
cat > /home/admin/.local/share/applications/chrome-quizlet.desktop <<'EOF'
[Desktop Entry]
Version=1.0
Type=Application
Name=Chrome — Quizlet
Exec=/usr/bin/google-chrome \
    --remote-debugging-port=9224 \
    --remote-debugging-address=0.0.0.0 \
    --remote-allow-origins=* \
    --remote-allow-hosts=host.docker.internal,localhost,127.0.0.1 \
    --user-data-dir=/home/admin/chrome-profiles/quizlet \
    --no-default-browser-check \
    --no-first-run \
    https://quizlet.com/
Terminal=false
Icon=google-chrome
Categories=Network;WebBrowser;
StartupWMClass=Chrome-Quizlet
EOF

# === DataCamp (port 9227) ===
cat > /home/admin/.local/share/applications/chrome-datacamp.desktop <<'EOF'
[Desktop Entry]
Version=1.0
Type=Application
Name=Chrome — DataCamp
Exec=/usr/bin/google-chrome \
    --remote-debugging-port=9227 \
    --remote-debugging-address=0.0.0.0 \
    --remote-allow-origins=* \
    --remote-allow-hosts=host.docker.internal,localhost,127.0.0.1 \
    --user-data-dir=/home/admin/chrome-profiles/datacamp \
    --no-default-browser-check \
    --no-first-run \
    https://app.datacamp.com/
Terminal=false
Icon=google-chrome
Categories=Network;WebBrowser;
StartupWMClass=Chrome-DataCamp
EOF

chmod +x /home/admin/.local/share/applications/chrome-*.desktop
update-desktop-database /home/admin/.local/share/applications/
```

### 6.4 Pin 5 launcher lên panel Cinnamon

- Mở Menu → tìm "Chrome — Google Family" → chuột phải → **Add to panel**.
- Lặp cho 4 launcher còn lại.

Hoặc terminal:
```bash
gsettings get org.cinnamon panel-launchers  # xem list hiện tại
# Thêm bằng GUI sẽ dễ hơn vì cần index chính xác
```

Sau bước này, panel có 5 icon Chrome, mỗi icon tên khác nhau, click 1 phát mở đúng instance + đúng port + đúng URL.

### 6.5 First login mỗi instance

Click từng launcher, làm 1 lượt:

| Instance | Login với | Ghi chú |
|---|---|---|
| Chrome — Google Family | Gmail admin (vd `family-admin@gmail.com`) | bật 2FA app TOTP, **tắt sync** Chrome |
| Chrome — Canva | Email Canva team-admin | login Canva for Teams |
| Chrome — Autodesk | Autodesk SSO admin | accept cookie |
| Chrome — Quizlet | Quizlet admin | login Quizlet |
| Chrome — DataCamp | DataCamp admin | login DataCamp |

Sau login xong: **đóng từng cửa sổ Chrome** (vẫn giữ session vì `--user-data-dir` lưu cookie trên đĩa). Lần sau click launcher mở lại → cookie vẫn còn.

### 6.6 Tắt Chrome auto-update (tùy chọn, tránh tự update khi đang chạy CDP)

```bash
sudo apt-mark hold google-chrome-stable
```

Khi muốn update thủ công: `sudo apt-mark unhold google-chrome-stable && sudo apt update && sudo apt install --only-upgrade google-chrome-stable`.

---

## Phần 7 — Port host-agent từ PowerShell sang bash

### 7.1 Hiểu cấu trúc hiện tại

Repo có 4 file Windows-specific:

- `scripts/family_host_agent.py` — Python core, **đã cross-platform**, không cần đổi
- `scripts/run_host_agent.ps1` — wrapper PowerShell ⚡ port sang bash
- `scripts/install_host_agent.ps1` — installer PowerShell ⚡ port sang bash
- `scripts/install_host_agent.bat` — bootstrap batch ⚡ port sang shell script

Auto-start cũ: Windows Startup folder (`shell:startup`).
Auto-start mới: **systemd user service** + **Cinnamon autostart .desktop**.

### 7.2 Tạo `scripts/run_host_agent.sh`

```bash
cat > /home/admin/source_code/stk-telebot/scripts/run_host_agent.sh <<'BASH_EOF'
#!/usr/bin/env bash
# =======================================================================
# scripts/run_host_agent.sh
# Wrapper for python scripts/family_host_agent.py on Linux Mint
# =======================================================================
set -euo pipefail

SCRIPT_DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"
REPO_ROOT="$( cd "$SCRIPT_DIR/.." && pwd )"
ENV_FILE="${ENV_FILE:-$REPO_ROOT/.env.host-agent}"
PORT="${PORT:-9230}"
BIND_HOST="${BIND_HOST:-127.0.0.1}"

echo "[family-host-agent] repo root: $REPO_ROOT"

# 1. Load or generate HOST_AGENT_KEY
if [[ -f "$ENV_FILE" ]]; then
    set -a
    # shellcheck disable=SC1090
    source "$ENV_FILE"
    set +a
fi

if [[ -z "${HOST_AGENT_KEY:-}" ]]; then
    echo "[family-host-agent] No HOST_AGENT_KEY found, generating..."
    HOST_AGENT_KEY="$(python3 -c 'import secrets; print(secrets.token_urlsafe(32))')"
    echo "HOST_AGENT_KEY=$HOST_AGENT_KEY" >> "$ENV_FILE"
    chmod 600 "$ENV_FILE"
    export HOST_AGENT_KEY
fi

# 2. Activate venv if exists
VENV_DIR="$REPO_ROOT/.venv"
if [[ -d "$VENV_DIR" ]]; then
    # shellcheck disable=SC1091
    source "$VENV_DIR/bin/activate"
fi

# 3. Ensure deps
python3 -m pip install --quiet fastapi uvicorn httpx pydantic >/dev/null

# 4. Run
echo "[family-host-agent] starting on $BIND_HOST:$PORT"
exec python3 "$REPO_ROOT/scripts/family_host_agent.py" --host "$BIND_HOST" --port "$PORT"
BASH_EOF

chmod +x /home/admin/source_code/stk-telebot/scripts/run_host_agent.sh
```

### 7.3 Tạo systemd user service cho auto-start

```bash
mkdir -p /home/admin/.config/systemd/user

cat > /home/admin/.config/systemd/user/family-host-agent.service <<'EOF'
[Unit]
Description=Family Host Agent (stk-telebot)
After=graphical-session.target
Wants=graphical-session.target

[Service]
Type=simple
ExecStart=/home/admin/source_code/stk-telebot/scripts/run_host_agent.sh
Restart=on-failure
RestartSec=10
StandardOutput=append:/home/admin/.local/state/family-host-agent.log
StandardError=append:/home/admin/.local/state/family-host-agent.log

[Install]
WantedBy=default.target
EOF

mkdir -p /home/admin/.local/state

# Enable + start
systemctl --user daemon-reload
systemctl --user enable family-host-agent.service
systemctl --user start family-host-agent.service

# Cho service tiếp tục chạy kể cả khi user logout (vì chế độ auto-login + RDP có thể logout)
sudo loginctl enable-linger admin

# Verify
systemctl --user status family-host-agent
curl -s http://127.0.0.1:9230/health
```

### 7.4 (Tuỳ chọn) Thêm autostart .desktop để hiển thị trong "Startup Applications"

```bash
mkdir -p /home/admin/.config/autostart
cat > /home/admin/.config/autostart/family-host-agent.desktop <<'EOF'
[Desktop Entry]
Type=Application
Name=Family Host Agent
Comment=stk-telebot host agent (CDP launcher)
Exec=systemctl --user restart family-host-agent.service
Terminal=false
Hidden=false
NoDisplay=false
X-GNOME-Autostart-enabled=true
EOF
```

> Phần này có ý nghĩa "biểu tượng visible" trong **Menu → Startup Applications** để admin non-tech thấy được service đang auto-start. Logic chạy thật vẫn là systemd user service ở 7.3.

### 7.5 Cập nhật installer cho người không clone repo (optional, để build/deploy admin-UI gửi installer)

Tạo `scripts/install_host_agent.sh` — port từ `install_host_agent.ps1`. Endpoint `/api/v1/family/host-agent/installer.sh` của bot phải serve file này. Em sẽ làm trong PR riêng vì đụng tới backend route.

> **Tasks-out-of-scope**: tạo PR `feat(host-agent): linux installer` chỉnh `bot/api/routes/host_agent_installer.py` để serve `.sh` thay vì `.bat`.

---

## Phần 8 — Deploy stk-telebot stack

### 8.1 Clone repo

```bash
mkdir -p /home/admin/source_code
cd /home/admin/source_code
git clone https://github.com/phanhaitrieu37/stk-telebot.git
cd stk-telebot
```

### 8.2 Tạo `.env.dev`

```bash
cp .env.example .env.dev 2>/dev/null || touch .env.dev
nano .env.dev
```

Điền các biến (xem từ `.env.dev` của repo cũ trên Win nếu có):

```
TELEGRAM_BOT_TOKEN=...
DATABASE_URL=postgresql+asyncpg://stkbot:password@stkapps_postgres_dev:5432/stkbot
REDIS_URL=redis://stkapps_redis_dev:6379/0
API_SECRET_KEY=...      # 32 ký tự random
JWT_SECRET=...          # 32 ký tự random
FERNET_KEY=...          # 44 ký tự base64 (giữ nguyên từ Win, không đổi tránh corrupt encrypt)
HOST_AGENT_HOST=127.0.0.1
HOST_AGENT_PORT=9230
HOST_AGENT_KEY=...      # match với .env.host-agent
# Family CDP ports
FAMILY_CHROME_GOOGLE_CDP=http://host.docker.internal:9225
FAMILY_CHROME_CANVA_CDP=http://host.docker.internal:9226
FAMILY_CHROME_AUTODESK_CDP=http://host.docker.internal:9223
FAMILY_CHROME_QUIZLET_CDP=http://host.docker.internal:9224
FAMILY_CHROME_DATACAMP_CDP=http://host.docker.internal:9227
```

> **Quan trọng**: nếu anh đang chạy bot ở Win với data Postgres encrypt qua Fernet, **phải giữ nguyên `FERNET_KEY`** lúc migrate sang Linux. Đổi key = data cũ không decrypt được.

### 8.3 Cài STKApps Postgres + Redis trước (vì bot phụ thuộc)

> **Tạm thời**: chạy 2 container Postgres + Redis riêng cho bot, đợi migrate stkapps đầy đủ ở Phần 9.

```bash
mkdir -p /home/admin/services/stk-shared-db
cat > /home/admin/services/stk-shared-db/docker-compose.yml <<'EOF'
services:
  postgres:
    image: postgres:16-alpine
    container_name: stkapps_postgres_dev
    restart: unless-stopped
    environment:
      POSTGRES_DB: stkbot
      POSTGRES_USER: stkbot
      POSTGRES_PASSWORD: change-me-strong-password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U stkbot"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - stkapp_network_dev

  redis:
    image: redis:7-alpine
    container_name: stkapps_redis_dev
    restart: unless-stopped
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    networks:
      - stkapp_network_dev

networks:
  stkapp_network_dev:
    name: stkapps_stkapp_network_dev
    driver: bridge

volumes:
  postgres_data:
  redis_data:
EOF

cd /home/admin/services/stk-shared-db
docker compose up -d
docker compose ps  # cả 2 healthy
```

### 8.4 Up stk-telebot stack

```bash
cd /home/admin/source_code/stk-telebot
docker compose --env-file .env.dev -f docker-compose.dev.yml up --build -d
docker compose logs -f stkbot  # Ctrl-C để thoát
```

Verify:
```bash
curl http://localhost:8880/healthz   # 200 OK
curl http://localhost:3000           # admin-UI HTML
curl http://localhost:9230/health    # host-agent
```

### 8.5 Restore data từ Postgres cũ (nếu migrate từ Win)

```bash
# Trên máy Win cũ:
# pg_dump -U stkbot stkbot | gzip > stkbot.sql.gz
# scp stkbot.sql.gz admin@<tailscale-ip>:/home/admin/

# Trên server mới
gunzip -c /home/admin/stkbot.sql.gz | docker exec -i stkapps_postgres_dev psql -U stkbot stkbot
```

Sau restore: chạy migration alembic để chắc schema khớp:
```bash
docker compose exec stkbot alembic upgrade head
```

---

## Phần 9 — Chuẩn bị migration stkapps

> Phần này **không cài ngay**. Chỉ pre-create thư mục + plan để khi anh sẵn sàng migrate (sau 1-2 tháng stk-telebot ổn) thì có sẵn pattern.

### 9.1 Pre-create directory

```bash
mkdir -p /home/admin/services/stkapps
mkdir -p /home/admin/services/stkapps/credentials
mkdir -p /home/admin/services/stkapps/data/{postgres,redis,media,static,logs}
```

### 9.2 Plan migration steps (ghi để 1-2 tháng nữa làm)

```bash
cat > /home/admin/services/stkapps/MIGRATION-PLAN.md <<'EOF'
# Migration plan stkapps from VPS → home server

## Khi nào làm
Sau khi stk-telebot trên server nhà chạy ổn 4-6 tuần (không restart bất thường,
admin login Chrome lưu loát, BI dashboard không lag).

## Bước

1. Trên VPS:
   - `docker exec stkapp_postgres_prod pg_dumpall -U stkapp > stkapp.sql`
   - `tar czf stkapp_media.tar.gz -C /var/lib/docker/volumes/stkapp_media_prod/_data .`
   - `tar czf stkapp_credentials.tar.gz -C /home/.../stkapps/credentials .`
   - `scp *.{sql,tar.gz} admin@<tailscale>:/home/admin/services/stkapps/`

2. Trên server nhà:
   - `cd /home/admin/services/stkapps`
   - Clone stkapps source: `git clone <repo> source/`
   - Copy `docker-compose.production.yml` về root, edit:
     - Đổi `image: postgres:16-alpine` → port mapping `5433:5432` (tránh đụng 5432 của bot)
     - Đổi `redis:7-alpine` → port `6380:6379`
     - Mount `./data/postgres:/var/lib/postgresql/data`
     - Mount `./credentials:/app/credentials:ro`
   - `docker compose up -d postgres redis`
   - Restore: `docker exec -i stkapps_postgres_v2 psql -U stkapp < stkapp.sql`
   - Restore media: `tar xzf stkapp_media.tar.gz -C ./data/media/`
   - `docker compose up -d`
   - Verify: `curl http://localhost:8000/health/`

3. DNS switch (sau khi smoke test pass):
   - Cloudflare DNS: A record `stkapp.com` → IP server nhà (qua Tailscale Funnel hoặc Cloudflare Tunnel)
   - HOẶC tốt hơn: bật Cloudflare Tunnel (xem Phần 10), DNS trỏ về tunnel hostname

4. Tắt VPS khi đã verify 7 ngày liên tục không issue.
EOF
```

### 9.3 Pre-cài Cloudflare Tunnel daemon (chuẩn bị)

```bash
# Đăng ký account Cloudflare (free), add domain của anh vào CF
# Cài cloudflared
curl -L --output /tmp/cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo apt install -y /tmp/cloudflared.deb
cloudflared --version

# Login (mở URL, chọn domain)
cloudflared tunnel login

# Tạo tunnel
cloudflared tunnel create stkbot-home

# Note tunnel ID + credentials path, save lại để Phần 10 dùng
```

---

## Phần 10 — Reverse proxy + Cloudflare Tunnel

### 10.1 Traefik làm reverse proxy nội bộ

```bash
mkdir -p /home/admin/services/traefik/{config,letsencrypt}
cat > /home/admin/services/traefik/docker-compose.yml <<'EOF'
services:
  traefik:
    image: traefik:v3.1
    container_name: traefik
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "8081:8080"  # dashboard, chỉ bind LAN
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./config/traefik.yml:/etc/traefik/traefik.yml:ro
      - ./letsencrypt:/letsencrypt
    networks:
      - traefik_public

networks:
  traefik_public:
    name: traefik_public
    driver: bridge
EOF

cat > /home/admin/services/traefik/config/traefik.yml <<'EOF'
api:
  dashboard: true
  insecure: true   # chỉ bind LAN qua firewall, không expose
log:
  level: INFO
entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"
providers:
  docker:
    exposedByDefault: false
    network: traefik_public
EOF

cd /home/admin/services/traefik
docker compose up -d
```

Sau này service muốn expose chỉ cần thêm label vào compose:
```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.myservice.rule=Host(`my.stkapp.com`)"
  - "traefik.http.routers.myservice.entrypoints=websecure"
  - "traefik.http.services.myservice.loadbalancer.server.port=8000"
```

### 10.2 Cloudflare Tunnel route

```bash
# Edit ingress config
mkdir -p /home/admin/.cloudflared
cat > /home/admin/.cloudflared/config.yml <<'EOF'
tunnel: <TUNNEL-ID-từ-Phần-9.3>
credentials-file: /home/admin/.cloudflared/<TUNNEL-ID>.json

ingress:
  - hostname: stkapp.example.com
    service: http://localhost:8000
  - hostname: bot.stkapp.example.com
    service: http://localhost:8880
  - hostname: admin.stkapp.example.com
    service: http://localhost:3000
  - service: http_status:404
EOF

# DNS trên CF dashboard: tạo CNAME stkapp.example.com → <TUNNEL-ID>.cfargotunnel.com
cloudflared tunnel route dns stkbot-home stkapp.example.com
cloudflared tunnel route dns stkbot-home bot.stkapp.example.com
cloudflared tunnel route dns stkbot-home admin.stkapp.example.com

# Service file
sudo cloudflared service install
sudo systemctl enable --now cloudflared
```

Test: từ máy ngoài mạng, mở `https://admin.stkapp.example.com` → admin-UI hiển thị qua HTTPS.

---

## Phần 11 — Backup + UPS + monitoring

### 11.1 UPS daemon (NUT)

Cài app NUT để khi UPS sắp hết pin → server tự shutdown sạch (tránh corrupt Postgres):

```bash
sudo apt install -y nut nut-client nut-server
# Cấu hình UPS theo model của anh (APC/Eaton/Santak)
sudo nano /etc/nut/ups.conf
# Thêm:
# [myups]
#     driver = usbhid-ups
#     port = auto
#     desc = "Server UPS"

sudo systemctl enable --now nut-server nut-monitor
upsc myups
```

> Chi tiết tuỳ model UPS — em sẽ làm script riêng khi anh chọn UPS.

### 11.2 Daily Postgres backup

```bash
mkdir -p /home/admin/backup/postgres
cat > /home/admin/scripts/backup-postgres.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
DATE=$(date +%Y%m%d-%H%M)
BACKUP_DIR=/home/admin/backup/postgres
mkdir -p "$BACKUP_DIR"

docker exec stkapps_postgres_dev pg_dumpall -U stkbot | gzip > "$BACKUP_DIR/stkbot-$DATE.sql.gz"

# Giữ 14 ngày gần nhất
find "$BACKUP_DIR" -name "stkbot-*.sql.gz" -mtime +14 -delete

# Sync lên Google Drive (qua rclone, đã setup)
rclone copy "$BACKUP_DIR/stkbot-$DATE.sql.gz" gdrive:stkbot-backup/
EOF
chmod +x /home/admin/scripts/backup-postgres.sh

# Cron daily 03:00
(crontab -l 2>/dev/null; echo "0 3 * * * /home/admin/scripts/backup-postgres.sh >> /home/admin/.local/state/backup.log 2>&1") | crontab -
```

Setup rclone với Google Drive:
```bash
sudo apt install -y rclone
rclone config  # New remote → Google Drive → follow prompts
```

### 11.3 Uptime Kuma monitoring (Telegram alert)

```bash
mkdir -p /home/admin/services/uptime-kuma
cat > /home/admin/services/uptime-kuma/docker-compose.yml <<'EOF'
services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - "3001:3001"
    volumes:
      - data:/app/data
volumes:
  data:
EOF

cd /home/admin/services/uptime-kuma
docker compose up -d
```

Mở `http://localhost:3001` → tạo admin → add monitors:
- `http://localhost:8880/healthz` (bot api)
- `http://localhost:9230/health` (host-agent)
- `http://localhost:9222/json` qua mỗi port 9223-9227 (5 Chrome CDP)
- Telegram bot notification (tạo bot riêng qua @BotFather)

### 11.4 (Tùy chọn) Telegram alert tự tạo

```bash
cat > /home/admin/scripts/check-services.sh <<'EOF'
#!/usr/bin/env bash
TELEGRAM_BOT_TOKEN="..."
CHAT_ID="..."

check() {
    local name=$1 url=$2
    if ! curl -sf "$url" >/dev/null 2>&1; then
        curl -s "https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/sendMessage" \
            -d chat_id="$CHAT_ID" \
            -d text="🚨 stk-telebot DOWN: $name ($url)"
    fi
}

check "bot-api" http://localhost:8880/healthz
check "host-agent" http://localhost:9230/health
for port in 9223 9224 9225 9226 9227; do
    check "chrome-$port" http://localhost:$port/json
done
EOF
chmod +x /home/admin/scripts/check-services.sh

# Cron mỗi 5 phút
(crontab -l; echo "*/5 * * * * /home/admin/scripts/check-services.sh") | crontab -
```

---

## Phần 12 — Verification checklist cuối

Sau khi xong tất cả các phần, chạy checklist:

### 12.1 Reboot test
- [ ] `sudo reboot` → server bật lại sau ~30s
- [ ] Auto-login admin OK
- [ ] `systemctl --user status family-host-agent` → active
- [ ] `curl http://localhost:9230/health` → 200
- [ ] `docker ps` → tất cả container Up

### 12.2 5 Chrome instance
- [ ] Click 5 launcher trên panel → 5 Chrome window mở đúng port + đúng URL
- [ ] `curl http://localhost:9225/json` → list tab Google
- [ ] `curl http://localhost:9226/json` → list tab Canva
- [ ] (3 site còn lại tương tự)
- [ ] Login từng site → cookie persist sau đóng + mở lại

### 12.3 Bot E2E
- [ ] Tạo 1 đơn test trên Telegram bot
- [ ] Đơn được record vào Postgres
- [ ] Family invite Google flow chạy được (verify trên `family_invite_records` table)
- [ ] Admin-UI tại `http://localhost:3000` login + xem dashboard 6 tab BI

### 12.4 Remote dev
- [ ] Từ laptop dev, `ssh admin@<tailscale-ip>` không hỏi password (key auth)
- [ ] `mstsc` (Windows) RDP vào `<tailscale-ip>:3389` → desktop Cinnamon hiện
- [ ] VS Code Remote-SSH connect OK, edit code remote, save tự sync

### 12.5 Stress test
- [ ] Mở 10 tab Google + 2 Canva cùng lúc
- [ ] `htop` → RAM peak < 70% (under 33 GB / 48 GB)
- [ ] `htop` → CPU không quá 60% sustained
- [ ] BI dashboard load 6 tab Financial/Operations/... → query Postgres < 5s mỗi cái
- [ ] `iotop` → NVMe write < 50 MB/s sustained

### 12.6 Power loss
- [ ] Rút điện server đột ngột (qua công tắc, không qua UPS)
- [ ] UPS giữ → đợi 60s → graceful shutdown qua NUT
- [ ] Cắm lại điện → server tự bật (BIOS "Restore on Power Loss = On")
- [ ] Postgres healthy, không cần manual recovery

### 12.7 Backup
- [ ] Chạy `bash /home/admin/scripts/backup-postgres.sh` thủ công → file `.sql.gz` được tạo
- [ ] `rclone ls gdrive:stkbot-backup/` → thấy file vừa upload
- [ ] Đợi đến 03:00 sáng hôm sau → cron tự chạy → verify backup mới

---

## Phụ lục — cheat sheet lệnh hay dùng

### Service management

```bash
# Restart bot
cd /home/admin/source_code/stk-telebot
docker compose restart stkbot

# Xem log realtime
docker compose logs -f stkbot

# Restart host-agent
systemctl --user restart family-host-agent

# Status host-agent
systemctl --user status family-host-agent

# Xem log host-agent
tail -f /home/admin/.local/state/family-host-agent.log

# Restart 1 Chrome instance
pkill -f 'chrome-profiles/google'
# Click lại launcher để mở
```

### Database

```bash
# Vào psql
docker exec -it stkapps_postgres_dev psql -U stkbot stkbot

# Backup tay
docker exec stkapps_postgres_dev pg_dump -U stkbot stkbot | gzip > /tmp/manual-backup.sql.gz

# Xem connection count
docker exec stkapps_postgres_dev psql -U stkbot stkbot -c "SELECT count(*) FROM pg_stat_activity;"
```

### Disk + RAM

```bash
df -h                       # disk usage tổng quát
du -sh /home/admin/*        # ai chiếm dung lượng
docker system df            # docker chiếm bao nhiêu
docker system prune -a      # xóa image không dùng (cẩn thận)

free -h                     # RAM
htop                        # CPU + RAM live
nethogs                     # network per-process
iotop                       # disk I/O per-process
```

### Network

```bash
# Tailscale
tailscale status
tailscale ip -4

# UFW
sudo ufw status verbose
sudo ufw allow from 192.168.1.0/24 to any port XXXX

# Test port
nc -zv localhost 8880
ss -tulpn | grep LISTEN     # all listening ports
```

### Update

```bash
# Update OS
sudo apt update && sudo apt full-upgrade -y

# Update Docker images
cd /home/admin/source_code/stk-telebot
docker compose pull
docker compose up -d

# Update bot code
git pull
docker compose --env-file .env.dev -f docker-compose.dev.yml up --build -d
```

### Khi gặp issue

```bash
# Bot không vào được DB
docker compose logs stkbot | grep -i error
docker exec stkapps_postgres_dev pg_isready -U stkbot

# Chrome không nhận CDP
ps -ef | grep chrome | grep remote-debugging-port
curl http://localhost:9225/json   # phải trả JSON list tab

# Disk đầy
docker system prune -a --volumes
sudo journalctl --vacuum-time=7d
```

---

## Khi nào nâng cấp gì

| Triệu chứng | Action |
|---|---|
| `df -h` `/home` > 80% | Mua Samsung 980 1TB → mount `/mnt/data` → di Docker volume + Postgres data về |
| `htop` RAM peak > 90% | Mua thêm 1 thanh DDR3 ECC 16GB → 64GB (kiểm tra bo có khe trống) |
| BI query > 10s | Tăng Postgres `shared_buffers` lên 8GB, `work_mem` lên 256MB |
| Internet upload nghẽn | Liên hệ FPT/Viettel nâng gói doanh nghiệp / fiber static IP |
| 5 service không đủ | Vẫn 1 host được, scale lên 10-15 service Docker không cần Proxmox |
| > 10 service hoặc cần HA | Mua server thứ 2, cài Proxmox cluster (giai đoạn 4) |

---

**Build xong → bắt đầu chạy production. Anh cần em hỗ trợ phần nào (port host-agent, deploy, troubleshoot) thì chỉ cần chạy lại script ở phần đó hoặc nhắn em.**
