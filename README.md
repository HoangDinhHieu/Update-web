# Update-web
hieu@DESKTOP-16V39I2:~$
mk 112005
# Update-web

Lab: Ubuntu (WSL) + Docker + Nginx + Node-RED + Cloudflare Tunnel

Ghi lại toàn bộ quá trình cài đặt, cấu hình và các lỗi thực tế gặp phải (kèm cách sửa) để sau này xem lại dễ dàng.

---

## Phần 0. Chuẩn bị

- Domain: `ngaymaiconang.id.vn` đã Active trên Cloudflare
- Tài khoản Cloudflare Zero Trust (gói Free)
- Có tunnel `chichi` cài trực tiếp trên Windows từ trước — không dùng nữa, cần gỡ để tránh 2 tunnel cùng chạy:

```powershell
cloudflared.exe service uninstall
```

> Lưu ý: lệnh này phải chạy trong **PowerShell với quyền Administrator**. Nếu chạy trong shell Ubuntu/WSL sẽ báo `Access is denied` vì không đủ quyền thao tác Service Control Manager của Windows.

Nếu máy có cài IIS/Apache ở cổng 80 trên Windows, nên tắt để tránh xung đột.

---

## Phần 1. Cài Ubuntu trên WSL

### 1.1. Cài WSL + Ubuntu

PowerShell (Admin):

```powershell
wsl --install -d Ubuntu
```

Khởi động lại máy nếu được yêu cầu.

### 1.2. Thiết lập lần đầu

Mở Ubuntu từ Start Menu, lần đầu chạy sẽ hỏi:

- `Enter new UNIX username:` → `hieu`
- `New password:` → (không hiện ký tự khi gõ, đó là bình thường)

Kiểm tra:

```bash
lsb_release -a
```

### 1.3. Cập nhật hệ thống

```bash
sudo apt update && sudo apt upgrade -y
```

---

## Phần 2. Cài Docker + Docker Compose

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo usermod -aG docker $USER
```

Thoát và mở lại Ubuntu (hoặc `newgrp docker`) để nhóm quyền có hiệu lực.

> **Lưu ý quan trọng:** trên WSL, Docker daemon **không tự khởi động** cùng WSL. Mỗi lần mở Ubuntu mới phải chạy lại:
>
> ```bash
> sudo service docker start
> ```

Kiểm tra:

```bash
docker --version
docker compose version
docker run hello-world
```

---

## Phần 3. Tạo thư mục làm việc

```bash
mkdir -p ~/lab/html
mkdir -p ~/lab/nginx/conf.d
mkdir -p ~/lab/nodered-data
cd ~/lab
sudo chown -R 1000:1000 ~/lab/nodered-data
```

Cấu trúc:

```
~/lab/
├── docker-compose.yml
├── .env
├── html/
│   └── index.html
├── nginx/
│   └── conf.d/
│       └── default.conf
└── nodered-data/
```

---

## Phần 4–5. Tạo tunnel & lấy token

Dashboard Cloudflare → **Networks → Tunnels → Create a tunnel** → chọn **Cloudflared** → đặt tên → **Save tunnel** → tab **Docker** → copy phần token (chuỗi sau `--token`).

⚠️ Token là khóa bí mật — không public lên đâu cả (bài học thực tế bên dưới sẽ giải thích tại sao).

```bash
cd ~/lab
nano .env
```

Nội dung:

```
CF_TOKEN=<token_thật_của_bạn>
```

---

## Phần 6. docker-compose.yml

```yaml
services:
  nginx:
    image: nginx:alpine
    container_name: nginx
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
    networks:
      - webnet

  nodered:
    image: nodered/node-red:latest
    container_name: nodered
    restart: unless-stopped
    environment:
      - TZ=Asia/Ho_Chi_Minh
    ports:
      - "1880:1880"
    volumes:
      - ./nodered-data:/data
    networks:
      - webnet

  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: unless-stopped
    command: tunnel --no-autoupdate run --token ${CF_TOKEN}
    depends_on:
      - nginx
      - nodered
    networks:
      - webnet

networks:
  webnet:
    driver: bridge
```

---

## Phần 7. Cấu hình Nginx

`~/lab/nginx/conf.d/default.conf`:

```nginx
server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location /api/ {
        proxy_pass http://nodered:1880/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /nodered/ {
        proxy_pass http://nodered:1880/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
```

Điểm quan trọng: dấu `/` cuối trong `proxy_pass http://nodered:1880/;` khiến nginx cắt bỏ `/api` trước khi chuyển tiếp — trình duyệt gọi `/api/hello`, Node-RED nhận `/hello`.

---

## Phần 8–9. index.html + chạy hệ thống

`index.html` đặt ở `~/lab/html/index.html` (giao diện web tĩnh có nút "Gọi API" gọi `/api/hello`).

```bash
cd ~/lab
docker compose up -d
docker compose ps
```

Test:

```bash
curl http://localhost:8080
```

---

## Phần 10. Backend Node-RED

Mở `http://localhost:1880`, kéo 3 node: **http in → function → http response**.

- `http in`: Method `GET`, URL `/hello`
- `function`:

```javascript
msg.payload = {
    message: "Xin chào từ Node-RED!",
    time: new Date().toLocaleString('vi-VN', { timeZone: 'Asia/Ho_Chi_Minh' }),
    random: Math.floor(Math.random() * 100)
};
return msg;
```

- `http response`: mặc định (status 200)

Bấm **Deploy**. Test:

```bash
curl http://localhost:1880/hello
curl http://localhost:8080/api/hello
```

---

## Phần 11. Reload Nginx

```bash
docker compose exec nginx nginx -t        # kiểm tra cú pháp
docker compose exec nginx nginx -s reload # reload không ngắt kết nối
```

---

## Phần 12. Trỏ domain qua Cloudflare Tunnel

Dashboard → tunnel → **Public application routes** (không phải "Hostname routes" — mục đó dành cho private hostname qua Cloudflare Gateway, dễ nhầm) → **Add a public hostname**:

| Trường | Giá trị |
|---|---|
| Subdomain | `web` |
| Domain | `ngaymaiconang.id.vn` |
| Type | `HTTP` |
| URL | `nginx:80` |

Vì sao `nginx:80` chứ không phải `localhost:8080`: container `cloudflared` nằm cùng mạng Docker `webnet` với `nginx`, nên gọi nhau qua tên service nội bộ, không qua `localhost` của máy host.

---

## 🔧 Sự cố thực tế gặp phải: "Unauthorized: Invalid tunnel secret"

Đây là phần quan trọng nhất để nhớ lại nếu gặp lại lỗi tương tự.

### Triệu chứng

- `docker compose ps` báo cả 3 container đều `Up`, `cloudflared` không bị crash/restart.
- Nhưng `docker compose logs cloudflared` lặp lại liên tục:

```
ERR Register tunnel error from server side error="Unauthorized: Invalid tunnel secret"
```

- Trên dashboard Cloudflare, tunnel luôn ở trạng thái **Inactive**, mục **Connectors** trống (`No results... yet!`).
- Phần **precheck** (DNS, UDP, TCP, Cloudflare API) đều **PASS** — nghĩa là mạng hoàn toàn thông, vấn đề không nằm ở kết nối internet/firewall mà ở xác thực (auth).

### Đã loại trừ các nguyên nhân sau (theo thứ tự đã kiểm tra)

1. **Docker daemon chưa chạy** → đã `sudo service docker start`, không phải nguyên nhân.
2. **Xung đột 2 tunnel** (một chạy trên Windows qua `cloudflared.exe service install`, một chạy trong Docker) → kiểm tra bằng `cloudflared.exe service uninstall` trên PowerShell Admin, kết quả `agent service Cloudflared is not installed` → xác nhận không có service nào trên Windows, không phải nguyên nhân.
3. **Token dán sai/thiếu ký tự khi copy qua nhiều nơi (chat, terminal...)** → decode thử bằng:

```bash
echo "<token>" | base64 -d
```

Token hợp lệ phải decode ra đúng dạng:

```json
{"a":"<account_tag>","t":"<tunnel_id>","s":"<secret_base64>"}
```

Một lần phát hiện được token bị lỗi thật: trường `"s"` decode ra bị **lai tạp** — nửa đầu là base64 hợp lệ, nửa sau đột ngột là ký tự UUID thô (do bị dán/gõ lại qua trung gian nhiều lần làm hỏng chuỗi). Đây chính là nguyên nhân của một lần lỗi.

4. Sau khi dán lại token **sạch, đúng, không qua trung gian** (copy bằng chuột trực tiếp từ dashboard, `echo "CF_TOKEN=<token>" > ~/lab/.env` để ghi đè sạch) — decode kiểm tra token hợp lệ 100% — **vẫn còn lỗi `Invalid tunnel secret`**.

### Kết luận & cách xử lý cuối cùng

Khi token đã xác nhận đúng cấu trúc nhưng vẫn bị từ chối, nguyên nhân còn lại là **credential phía server của chính tunnel đó đã bị hỏng/thu hồi** (không phải lỗi thao tác của người dùng). Cách xử lý triệt để:

1. **Xóa tunnel cũ** trên dashboard Cloudflare (tunnel `lab-docker` gốc, ID `bd0ab918-...`).
2. **Tạo tunnel mới hoàn toàn** (tên mới, ID mới, ví dụ `acdee893-...`).
3. Lấy token mới từ tab Docker của tunnel mới, copy trực tiếp bằng chuột (không gõ tay, không dán qua ứng dụng trung gian).
4. Ghi đè `.env`:

```bash
echo "CF_TOKEN=<token_mới>" > ~/lab/.env
```

5. Khởi động lại hoàn toàn (không chỉ `restart`, vì token chỉ được nạp lúc container khởi tạo):

```bash
docker compose down
docker compose up -d
docker compose logs cloudflared --tail 20
```

6. Kiểm tra trạng thái **Healthy** trên dashboard (mục Connectors xuất hiện 1 connector, Status của tunnel chuyển từ *Inactive* → *Healthy*).
7. **Vì tunnel đã đổi ID mới, phải làm lại bước Phần 12** (Add a public hostname) trên tunnel mới — route hostname gắn liền với từng tunnel cụ thể, không tự chuyển theo.

Sau khi làm đúng các bước trên, `https://web.ngaymaiconang.id.vn` hoạt động bình thường, gọi API trả JSON thành công.

### Bài học rút ra

- Lỗi `Invalid tunnel secret` là lỗi **xác thực (auth)**, không phải lỗi mạng — precheck pass hết không có nghĩa là token đúng.
- Luôn có thể tự kiểm tra token hợp lệ hay không bằng `base64 -d` trước khi tốn thời gian debug các thứ khác.
- Copy/dán token qua nhiều lớp trung gian (chat, gõ lại thủ công) rất dễ làm hỏng chuỗi — nên copy trực tiếp từ dashboard vào terminal/`nano`.
- Nếu đã chắc chắn token đúng cấu trúc mà vẫn lỗi, đừng cố sửa mãi — xóa tunnel và tạo tunnel mới thường nhanh hơn debug tiếp.
- **Không commit file `.env` (chứa token thật) lên GitHub.** Nên thêm `.env` vào `.gitignore`.

---

## Phần 13. Các lệnh thường dùng

```bash
# Khởi động Docker (mỗi lần mở WSL mới)
sudo service docker start

# Bật toàn bộ hệ thống
cd ~/lab && docker compose up -d

# Tắt toàn bộ
docker compose down

# Xem trạng thái
docker compose ps

# Xem log realtime của 1 service
docker compose logs -f cloudflared

# Kiểm tra cú pháp nginx
docker compose exec nginx nginx -t

# Reload nginx sau khi sửa config
docker compose exec nginx nginx -s reload

# Khởi động lại 1 service
docker compose restart nodered

# Kiểm tra token trong .env có đúng cấu trúc không
cat ~/lab/.env | cut -d= -f2 | base64 -d
```

---

## Phần 14. Xử lý lỗi thường gặp

| Triệu chứng | Nguyên nhân & cách xử lý |
|---|---|
| `Cannot connect to the Docker daemon` | Chưa chạy `sudo service docker start` |
| `permission denied` khi chạy docker | Chưa thoát/mở lại WSL sau `usermod -aG docker` |
| Node-RED container bị restart liên tục | Sai quyền thư mục `nodered-data` → `sudo chown -R 1000:1000 ~/lab/nodered-data` |
| `Unauthorized: Invalid tunnel secret` | Xem mục "Sự cố thực tế" ở trên — kiểm tra token bằng `base64 -d`, nếu vẫn lỗi thì tạo tunnel mới |
| Web hiện 404 | `index.html` đặt sai chỗ, phải ở `~/lab/html/index.html` |
| Gọi API ra lỗi 502 | Node-RED chưa Deploy flow, hoặc endpoint `http in` khai sai (phải là `/hello`) |
| Gọi API ra 404 | Sai đường dẫn — kiểm tra dấu `/` cuối trong `proxy_pass` và endpoint trong `http in` có khớp không |
| Domain không mở được | Tunnel chưa Healthy, hoặc chưa thêm đúng Public Hostname cho tunnel hiện tại |
| Sửa config nginx mà không thấy đổi | Chưa reload → `docker compose exec nginx nginx -s reload` |
| `cloudflared.exe service uninstall` báo `Access is denied` | Phải chạy trong PowerShell **Admin**, không phải trong Ubuntu/WSL |

---

## Tóm tắt luồng hoạt động

```
Trình duyệt người dùng
   │  https://web.ngaymaiconang.id.vn
   ▼
Cloudflare (SSL, CDN)
   │  qua đường hầm mã hóa
   ▼
cloudflared (container)
   │  nginx:80
   ▼
nginx (container)
   ├── "/"        → trả index.html từ /usr/share/nginx/html
   └── "/api/..." → proxy sang nodered:1880
                         │
                         ▼
                   Node-RED (container)
                   http in → function → http response
                         │
                         ▼
                      trả JSON
```

Máy không cần mở cổng nào ra internet, không cần IP tĩnh, không cần cấu hình router — `cloudflared` tự tạo kết nối outbound tới Cloudflare, mọi request từ internet đi ngược qua đường hầm đó vào máy.
# Một lưu ý nhỏ cần nhớ: mỗi lần bạn tắt/mở lại WSL, phải chạy lại sudo service docker start (Docker không tự khởi động cùng WSL). Việc set iptables-legacy thì chỉ cần làm 1 lần, không cần lặp lại.
