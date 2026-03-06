# SSH 反向隧道方案

在本地机器上保存AWS的指纹信息防止MITM。

## AWS: 安装工具

```bash
sudo apt update && sudo apt -y upgrade
sudo apt -y install nginx certbot python3-certbot-nginx ufw fail2ban
```

## AWS: 下载AWS 登录密钥(私钥)并修改权限为600[ssh的强制要求]

```bash
chmod 600 ~/.ssh/LightsailDefaultKey-ap-northeast-1.pem
```

## AWS: 创建单独的反向代理用户‘tunnel’

```bash
sudo adduser --disabled-password --gecos "" tunnel
sudo usermod -s /usr/sbin/nologin tunnel

sudo mkdir -p /home/tunnel/.ssh
sudo touch /home/tunnel/.ssh/authorized_keys
sudo chown -R tunnel:tunnel /home/tunnel/.ssh
sudo chmod 700 /home/tunnel/.ssh
sudo chmod 600 /home/tunnel/.ssh/authorized_keys
```

## Local: 在本地，创建 tunnel 用户用的登录密钥

```bash
ssh-keygen -t ed25519 -f ~/.ssh/aws_tunnel_ed25519 -C "aws-tunnel"
# 显示刚刚生成的公钥
cat ~/.ssh/aws_tunnel_ed25519.pub
```

## AWS: 回到AWS，在sshd_config 末尾加入用户‘tunnel’的ssh配置

```shell
# nano /etc/ssh/sshd_config
Match User tunnel
    PermitTTY no
    X11Forwarding no
    AllowAgentForwarding no
    PermitTunnel no
    GatewayPorts no
    AllowTcpForwarding remote
```

```bash
# 重启ssh服务
sudo sshd -t && sudo systemctl restart ssh
```

## AWS: 在 AWS 上编辑 `/home/tunnel/.ssh/authorized_keys`

```bash
#现在tunnel只监听11451端口
command="/bin/false",no-pty,no-agent-forwarding,no-X11-forwarding,permitlisten="127.0.0.1:11451" ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... aws-tunnel
```

## AWS: 保证权限

```bash
sudo chown tunnel:tunnel /home/tunnel/.ssh/authorized_keys
sudo chmod 600 /home/tunnel/.ssh/authorized_keys
```

## Local: 去本地编辑ssh配置文件

```bash
Host aws-tokyo-tunnel
    HostName 14.14.514.114
    User tunnel
    IdentityFile ~/.ssh/aws_tunnel_ed25519
    IdentitiesOnly yes
    ServerAliveInterval 30
    ServerAliveCountMax 3
    StrictHostKeyChecking yes
    HostKeyAlgorithms ssh-ed25519
```

## Local: 创建方向代理的服务文件

```bash
nano ~/.config/systemd/user/immich-reverse-tunnel.service
```

```bash
[Unit]
Description=Immich reverse SSH tunnel to AWS (autossh)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
Environment="AUTOSSH_GATETIME=0"
ExecStart=/usr/bin/autossh -M 0 -N \
  -o ExitOnForwardFailure=yes \
  -o ServerAliveInterval=30 \
  -o ServerAliveCountMax=3 \
  -R 127.0.0.1:11451:127.0.0.1:2283 \
  aws-tokyo-tunnel
Restart=always
RestartSec=5

# 安全/稳定性增强（可保留）
KillMode=process
TimeoutStopSec=10

[Install]
WantedBy=default.target

```

## 按当前用户做 **user service**

先创建目录：

```bash
mkdir -p ~/.config/systemd/user
```

创建服务文件：

```bash
nano ~/.config/systemd/user/immich-reverse-tunnel.service
```

查看ssh文件

```bash
nano ~/.ssh/config 
```

## 启动并设置开机自启（user service）

先让 systemd user 模式重新加载配置：

```bash
systemctl --user daemon-reload
```

启动服务：

```bash
systemctl --user start immich-reverse-tunnel.service
```

设置开机自启：

```bash
systemctl --user enable immich-reverse-tunnel.service
```

查看状态：

```bash
systemctl --user status immich-reverse-tunnel.service
```

看日志（实时）：

```bash
journalctl --user -u immich-reverse-tunnel.service -f
journalctl --user -u immich-reverse-tunnel.service -n 50
```

让“用户服务”在你没登录时也能开机运行（非常关键）

默认情况下，`systemctl --user` 服务有时需要用户登录后才运行。
 你要让它开机就跑，需要启用 **linger**（在家里 Ubuntu 上执行）：

```
sudo loginctl enable-linger $USER
```

停止服务

```bash
systemctl --user stop immich-reverse-tunnel.service
# 禁止用户登录时自动启动
systemctl --user disable immich-reverse-tunnel.service
```

如何启用 linger

在终端中执行（将 `$USER` 替换为你的实际用户名，或者直接使用）：

```bash
sudo loginctl enable-linger $USER
```

或者指定具体用户名：

```bash
sudo loginctl enable-linger lukas
```

验证 linger 状态

```bash
loginctl show-user $USER | grep Linger
```

输出应为 `Linger=yes`。

```bash
sudo ln -s /etc/nginx/sites-available/ssh-immich /etc/nginx/sites-enabled/
```

在 AWS 上验证SSH 反向隧道是否在线：

```bash
curl -I http://127.0.0.1:12283
```

##  AWS 上 `curl 127.0.0.1:12283` 偶尔不通

看家里 Ubuntu 上日志：

```bash
journalctl --user -u immich-reverse-tunnel.service -f
```

## Nginx反向代理

## 方案：先上 80-only，再签证书，再启用 443

### 1）先把 443 这段临时注释掉（或改成只监听 80）

编辑你的站点文件：

```bash
sudo nano /etc/nginx/sites-available/o1145141414514o.asia
```

把 `server { listen 443 ... }` 这整个块先注释/删除，暂时只保留这段 **80**：

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name o1145141414514o.asia;

    location ^~ /.well-known/acme-challenge/ {
        root /var/www/html;
        default_type "text/plain";
        try_files $uri =404;
    }

    # 可选：先不跳转，等证书签完再跳
    location / {
        proxy_pass http://127.0.0.1:12283;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

> 我这里先把 80 直接反代到 12283，方便你马上用 http 验证站点通不通；也不会影响 certbot 的 `/.well-known/acme-challenge/`。

然后：

```bash
sudo ln -s /etc/nginx/sites-available/o1145141414514o.asia /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

------

### 2）签发证书（用 nginx 插件最省事）

```bash
sudo apt-get update
sudo apt-get install -y certbot python3-certbot-nginx
sudo certbot --nginx -d o1145141414514o.asia
```

如果你更想用纯 webroot（也很稳）：

```bash
sudo certbot certonly --webroot -w /var/www/html -d o1145141414514o.asia
```

------

### 3）证书出来后，再加回 443 配置

证书签发成功后，文件就会存在：

- `/etc/letsencrypt/live/o1145141414514o.asia/fullchain.pem`
- `/etc/letsencrypt/live/o1145141414514o.asia/privkey.pem`

这时你再把 443 server block 加回去（我给你一份可直接粘贴的完整版：80 跳 443 + 443 反代）。

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name o1145141414514o.asia;

    location ^~ /.well-known/acme-challenge/ {
        root /var/www/html;
        default_type "text/plain";
        try_files $uri =404;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name o1145141414514o.asia;

    ssl_certificate     /etc/letsencrypt/live/o1145141414514o.asia/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/o1145141414514o.asia/privkey.pem;
    
	# --- Security Headers (safe set) ---
    add_header Strict-Transport-Security "max-age=86400" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;

    # 如果你确认不需要被任何站点 iframe 嵌入（一般 app 都不需要）
    add_header X-Frame-Options "SAMEORIGIN" always;
    
    client_max_body_size 0;
    proxy_request_buffering off;

    proxy_read_timeout  3600s;
    proxy_send_timeout  3600s;
    proxy_connect_timeout 60s;

    location / {
        proxy_pass http://127.0.0.1:11451;
        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

最后：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

## Nginx 安全增强

## 1.地域限制

### A. 安装 MaxMind DB 库与工具

```bash
sudo apt update
sudo apt install -y libmaxminddb0 libmaxminddb-dev mmdb-bin geoipupdate
```

**B. 安装 Nginx GeoIP2 模块（优先“系统包”）**

```bash
sudo apt install -y libnginx-mod-http-geoip2
```

然后检查模块是否存在（常见路径之一）：

```bash
ls -al /usr/lib/nginx/modules | grep geoip2
# 有以下两个文件
ngx_http_geoip2_module.so
ngx_stream_geoip2_module.so
```

###  GeoLite2-Country.mmdb（MaxMind 免费库）

```bash
# 注册账户生成licence 文件, 进入配置文件
sudo nano /etc/GeoIP.conf
#进行配置 核心是 AccountID / LicenseKey / EditionIDs
```

执行更新下载

```bash
sudo geoipupdate
ls -a /var/lib/GeoIP
# GeoLite2-ASN.mmdb  GeoLite2-City.mmdb  GeoLite2-Country.mmdb
```

### Nginx 配置 - /etc/nginx/nginx.conf

```nginx
# 读取国家 ISO code（CN/HK/MO/TW/US...）
geoip2 /var/lib/GeoIP/GeoLite2-Country.mmdb {
        $geoip2_country_code country iso_code;
}

map $geoip2_country_code $allow_country {
        default 0;
        CN 1;
}

# 读取城市
geoip2 /var/lib/GeoIP/GeoLite2-City.mmdb {
        $geoip2_city_name   city names en;
        $geoip2_region_name subdivisions 0 names en;
}

map $http_x_forwarded_for $xff_safe { default $http_x_forwarded_for; "" "-"; }
map $http_cf_connecting_ip $cf_safe { default $http_cf_connecting_ip; "" "-"; }

# 很多 IP 没城市，日志会出现 city=""。可以加 map 兜底成 -：
map $geoip2_city_name $city_safe { default $geoip2_city_name; "" "-"; }
map $geoip2_region_name $prov_safe { default $geoip2_region_name; "" "-"; }

log_format sec_f2b '$remote_addr - - [$time_local] '
                   '"$request" $status $body_bytes_sent '
                   '"$http_referer" "$http_user_agent" '
                   'host=$host scheme=$scheme '
                   'cc=$geoip2_country_code prov="$prov_safe" city="$city_safe" '
                   'xff="$xff_safe" cf="$cf_safe" '
                   'rt=$request_time urt=$upstream_response_time uct=$upstream_connect_time uht=$upstream_header_time';   

# WebSocket 的 Connection 头最好用变量（更兼容）
map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
}

# 默认访问日志（可留）
access_log /var/log/nginx/access.log;

# 安全日志（包含城市/国家）
access_log /var/log/nginx/access_security.log sec_f2b;

# 错误日志
error_log /var/log/nginx/error.log;
```

###  server 里

```nginx
# 在 /etc/nginx/sites-enabled/o1145141414514o.asia
# 非允许国家直接断开（省资源）
if ($allow_country = 0) { return 444; }
```

### 重启服务

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### 验证是否生效

```bash
mmdblookup --file /var/lib/GeoIP/GeoLite2-Country.mmdb --ip 8.8.8.8 country iso_code
```

## 2. Nginx 限流

Nginx 配置 - /etc/nginx/nginx.conf

```nginx
# 入口限流与连接数限制
limit_req_zone $binary_remote_addr zone=req_per_ip:10m rate=10r/s;
limit_conn_zone $binary_remote_addr zone=conn_per_ip:10m;
```

新建通用反代头文件： `/etc/nginx/snippets/proxy_immich_common.conf` 文件

```nginx
proxy_pass http://127.0.0.1:11451;
proxy_http_version 1.1;

proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection $connection_upgrade;

proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

新建“上传友好”文件（只放差异项）

```nginx
# 上传大文件建议关闭请求缓冲（你已有）
proxy_request_buffering off;

# 上传/转码/播放可能很久
proxy_read_timeout  3600s;
proxy_send_timeout  3600s;
proxy_connect_timeout 60s;

include /etc/nginx/snippets/proxy_immich_common.conf;
```

编辑 /etc/nginx/nginx.config

```nginx
location / {
    if ($allow_country = 0) { return 444; }

    limit_conn conn_per_ip 30;
    limit_req zone=req_per_ip burst=20 nodelay;
    limit_req_status 429;

    include /etc/nginx/snippets/proxy_immich_common.conf;
}

location ^~ /api/assets {
    if ($allow_country = 0) { return 444; }

    limit_conn conn_per_ip 10;

    include /etc/nginx/snippets/proxy_immich_upload.conf;
}
```

### 重启服务

```bash
sudo nginx -t
sudo systemctl reload nginx
```

## 3.Nginx 非服务域名拒绝

```bash
# 创建 /etc/nginx/sites-available/00-default-deny
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    return 444;
}

server {
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;

    # 随便给个证书（可以用你现有的 o1145141414514o.asia 证书先顶着）
    ssl_certificate     /etc/letsencrypt/live/o1145141414514o.asia/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/o1145141414514o.asia/privkey.pem;

    return 444;
}

# 启动
sudo ln -s /etc/nginx/sites-available/00-default-deny /etc/nginx/sites-enabled/00-default-deny
sudo nginx -t && sudo systemctl reload nginx
```



### 模拟测试

```bash
# 安装 ab（ApacheBench）或 wrk
sudo apt update
sudo apt install -y apache2-utils
# 并发50条请求，共发请求200条
ab -n 200 -c 50 https://o1145141414514o.asia/api/server/about
```

### 查询Nginx 安全日志（之前配置的）

```bash
# 查询携带’status=429‘ 信息的日志
sudo grep -n ' status=429 ' /var/log/nginx/access_security.log | tail -n 20

# 查询 Nginx 安全日志最近50条
tail -n 50 /var/log/nginx/access_security.log
```

用 `websocat` 或 `curl` 模拟多个 websocket 连接：

```bash
sudo apt update
sudo apt install -y websocat

# 开 40 个 websocket 连接（超过你设置的 30）
for i in {1..40}; do
  websocat -n --ping-interval 20 "wss://o1145141414514o.asia/api/socket.io/?EIO=4&transport=websocket" >/dev/null 2>&1 &
done
sleep 2
```

## 4. HTTP-01认证 切换为 DNS-01 认证

```bash
# 安装 certbot-dns-cloudflare 插件
sudo apt install python3-certbot-dns-cloudflare
```

### 获取 Cloudflare API Token

> 登录 Cloudflare 控制台 → **My Profile** → **API Tokens** → **Create Token**
>
> 使用模板：**Zone:DNS:Edit**
>
> 权限设置：
>
> - **Zone:DNS:Edit**（必需）
> - **Zone:Zone:Read**（必需）
> - **Zone Resources**: Include → Specific zone → 你的域名
>
> 创建后复制 Token（只显示一次）

### 配置 Cloudflare 凭证

```bash
# 创建凭证文件
sudo mkdir -p /etc/letsencrypt
sudo nano /etc/letsencrypt/cloudflare.ini

# 添加内容：
dns_cloudflare_api_token = 你的Cloudflare_API_Token

# 设置权限：
sudo chmod 600 /etc/letsencrypt/cloudflare.ini
```

### 申请/续期证书

```bash
# 申请新证书（自动 DNS 验证）强制更新
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
  -d o1145141414514o.asia \
  --preferred-challenges dns-01 \
  --force-renewal

# 查看cert更新脚本
sudo cat /etc/letsencrypt/renewal/o1145141414514o.asia.conf

[renewalparams]
account = 632ed04c54102e604d197a5984729ff7
authenticator = dns-cloudflare
server = https://acme-v02.api.letsencrypt.org/directory
pref_challs = dns-01,
dns_cloudflare_credentials = /etc/letsencrypt/cloudflare.ini
# authenticator 修改为 dns-cloudflare

# 测试续期
sudo certbot renew --dry-run
```

### 设置自动续期定时任务

```bash
# 查看自动更新服务
sudo nano /etc/letsencrypt/renewal/o1145141414514o.asia.conf
```

## Fail2Ban

### 0）Nginx 自定义安全日志格式

```nginx
log_format sec_f2b '$remote_addr - - [$time_local] '
                   '"$request" $status $body_bytes_sent '
                   '"$http_referer" "$http_user_agent" '
                   'host=$host scheme=$scheme '
                   'cc=$geoip2_country_code prov="$prov_safe" city="$city_safe" '
                   'xff="$xff_safe" cf="$cf_safe" '
                   'rt=$request_time urt=$upstream_response_time uct=$upstream_connect_time uht=$upstream_header_time';

# 例子
114.514.141.451 - - [01/Mar/2026:06:19:28 +0000] "GET /api/server/about HTTP/1.0" 429 162 "-" "ApacheBench/2.3" host=o1145141414514o.asia scheme=https cc=CN prov="-" city="-" xff="-" cf="-" rt=0.000 urt=- uct=- uht=-
```

### 1) 安装并启用

```bash
sudo apt update
sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban
sudo fail2ban-client status
```

### 2）配置 444 - 地域限制IP封禁

```bash
sudo nano /etc/fail2ban/filter.d/nginx-app-444.conf

[Definition]
failregex = ^<HOST> - - \[.*\] ".*" (?:401|429) \d+ ".*" ".*" host=app\.yunzhaopian\.cc\b
ignoreregex =
```

### 3）配置 - 扫描敏感资源路径IP封禁

```bash
sudo nano /etc/fail2ban/filter.d/nginx-app-scan.conf

[Definition]
failregex = ^<HOST> - - \[.*\] "(?:GET|POST|HEAD) (?:/wp-admin|/wp-login\.php|/xmlrpc\.php|/phpmyadmin|/pma|/\.env|/\.git|/\.svn|/cgi-bin|/actuator|/manager|/admin|/login)\b.*" \d+ \d+ ".*" ".*" host=app\.yunzhaopian\.cc\b
ignoreregex =
```

### 5) 配置 401 : 密码错/撞库(401)

```bash
sudo nano /etc/fail2ban/filter.d/nginx-app-401.conf

[Definition]
failregex = ^<HOST> - - \[.*\] ".*" (?:401) \d+ ".*" ".*" host=app\.yunzhaopian\.cc\b
ignoreregex =
```

### 启用 fail2ban - jail 并配置

```bash
sudo nano /etc/fail2ban/jail.d/custom.local

[DEFAULT]
ignoreip = 127.0.0.1/8 ::1
bantime  = 24h
findtime = 10m

[sshd]
enabled = true
maxretry = 5
findtime = 10m
bantime = 6h

[nginx-app-444]
enabled  = true
port     = http,https
logpath  = /var/log/nginx/access_security.log
maxretry = 10
findtime = 2m
bantime  = 24h

[nginx-app-scan]
enabled  = true
port     = http,https
logpath  = /var/log/nginx/access_security.log
maxretry = 3
findtime = 10m
bantime  = 7d

[nginx-app-401]
enabled  = true
port     = http,https
logpath  = /var/log/nginx/access_security.log
# 5 分钟内 15 次 401/403 才封（先保守，防误封）
findtime = 5m
maxretry = 10
bantime  = 12h
```

### 重启 & 验证（用 fail2ban-regex 先测正则）

```bash
sudo systemctl restart fail2ban

sudo fail2ban-regex /var/log/nginx/access_security.log /etc/fail2ban/filter.d/nginx-app-444.conf
sudo fail2ban-client status nginx-app-444
```

### 常用运维命令

```bash
# 查看已封 IP：
sudo fail2ban-client get nginx-app-401 banip
sudo fail2ban-client get nginx-app-444 banip
sudo fail2ban-client get sshd banip

# 查 iptables 里“是哪个链封的 看所有 fail2ban 链
sudo iptables -L -n --line-numbers | grep -E "f2b-|Chain f2b" -n

# 看 fail2ban 自己的日志
sudo tail -n 200 /var/log/fail2ban.log
# or
sudo grep -E "Ban|Unban" /var/log/fail2ban.log | tail -n 50
# 更精准（查某 IP）
sudo grep -F "222.87.162.22" /var/log/fail2ban.log | tail -n 50
# Nginx 安全日志里反查这个 IP 做了什么
sudo grep -F "155.254.122.8" /var/log/nginx/access_security.log | tail -n 50

# 解封某个 IP：
sudo fail2ban-client set nginx-app-401 unbanip 222.87.162.22
```

### 测试

```bash
# 并发访问50 共200个 触发 429
ab -n 200 -c 50 https://o1145141414514o.asia/ai/server/about
```

## SSHD安全增强

### 1) UFW 限速 SSH

```bash
sudo ufw limit 22/tcp
sudo ufw status verbose
```

### 2)开启防火墙

```bash
sudo ufw enable
# 注意别把自己关外面了

# sudo ufw allow from 192.168.0.0/24 to any port 2283 proto tcp
# sudo ufw deny 2283/tcp
# sudo ufw status numbered
```

### 3)关闭ufw ipv6

```bash
sudo nano /etc/default/ufw
# 修改 IPV6=yes -> no
IPV6=no
sudo ufw disable
sudo ufw enable
```

### 4)shhd 配置

```bash
sudo nano /etc/ssh/sshd_config

AllowUsers ubuntu tunnel

MaxStartups 10:30:60
LoginGraceTime 20

PermitRootLogin no
MaxAuthTries 3

PubkeyAcceptedAlgorithms ssh-ed25519,rsa-sha2-512,rsa-sha2-256,sk-ssh-ed25519@openssh.com
PermitRootLogin no
PasswordAuthentication no

AllowTcpForwarding no
AllowAgentForwarding no
X11Forwarding no

ClientAliveInterval 300
ClientAliveCountMax 2

PasswordAuthentication no
PermitEmptyPasswords no
KbdInteractiveAuthentication no

LogLevel VERBOSE

sudo sshd -t
sudo systemctl reload ssh

# 查看日志
sudo tail -n 60 /var/log/auth.log
```

### 5) 编辑用户连接的密钥设置

```bash
# sshd(服务端)配置链: sshd_config > Match User tunnel > authorized_keys
sudo cat /home/tunnel/.ssh/authorized_keys
restrict,port-forwarding,permitlisten="127.0.0.1:11451" ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIMp8TPHHV1sFlFKGtdm+9ZNWTFp+EArylo9wF8Mgm1A3 aws-tunnel

#监听
journalctl --user -u immich-reverse-tunnel.service -n 50

```

## 架构

<img width="1033" height="672" alt="immich_v3" src="https://github.com/user-attachments/assets/961f984c-bed5-411c-9102-5f792b7d1fba" />


# SSH重要文件配置信息

```bash
# 本地配置
# 1. ssh连接配置文件: ~/.ssh/config
Host aws-tokyo-ubuntu
    HostName 14.14.514.114
    User ubuntu
    IdentityFile ~/.ssh/LightsailDefaultKey-ap-northeast-3.pem
    IdentitiesOnly yes
    ServerAliveInterval 30
    ServerAliveCountMax 3
    StrictHostKeyChecking yes
    HostKeyAlgorithms ssh-ed25519

Host aws-tokyo-tunnel
    HostName 14.14.514.114
    User tunnel
    IdentityFile ~/.ssh/aws_tunnel_ed25519
    IdentitiesOnly yes
    ServerAliveInterval 30
    ServerAliveCountMax 3
    StrictHostKeyChecking yes
    HostKeyAlgorithms ssh-ed25519
# 2. 服务配置文件:  ~/.config/systemd/user/immich-reverse-tunnel.service
[Unit]
Description=Immich reverse SSH tunnel to AWS (autossh)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
Environment="AUTOSSH_GATETIME=0"
ExecStart=/usr/bin/autossh -M 0 -N \
  -o ExitOnForwardFailure=yes \
  -o ServerAliveInterval=30 \
  -o ServerAliveCountMax=3 \
  -R 127.0.0.1:11451:127.0.0.1:2283 \
  aws-tokyo-tunnel
Restart=always
RestartSec=5

# 安全/稳定性增强（可保留）
KillMode=process
TimeoutStopSec=10

[Install]
WantedBy=default.target

# AWS 配置
# 1. 修改 sshd_config 文件 末尾添加: /etc/ssh/sshd_config

Include /etc/ssh/sshd_config.d/*.conf
AllowUsers ubuntu tunnel

LogLevel VERBOSE

# Authentication:
LoginGraceTime 20
PermitRootLogin no
MaxAuthTries 3
PermitRootLogin no
PasswordAuthentication no
PubkeyAcceptedAlgorithms ssh-ed25519,rsa-sha2-512,rsa-sha2-256,sk-ssh-ed25519@openssh.com

PasswordAuthentication no
PermitEmptyPasswords no
KbdInteractiveAuthentication no

UsePAM yes

AllowAgentForwarding no
AllowTcpForwarding no
X11Forwarding no
PrintMotd no

ClientAliveInterval 300
ClientAliveCountMax 2
MaxStartups 10:30:60

AcceptEnv LANG LC_*
Subsystem       sftp    /usr/lib/openssh/sftp-server
TrustedUserCAKeys /etc/ssh/lightsail_instance_ca.pub

Match User tunnel
    PermitTTY no
    X11Forwarding no
    AllowAgentForwarding no
    PermitTunnel no
    GatewayPorts no
    AllowTcpForwarding remote
    PermitListen 127.0.0.1:11451 

# 2. 在创建用户‘tunnel’的ssh中配置: /home/tunnel/.ssh/authorized_keys
restrict,port-forwarding,permitlisten="127.0.0.1:11451" ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIMp8TPHHV1sFlFKGtdm+9ZNWTFp+EArylo9wF8Mgm1A3 aws-tunnel

# 3.1 配置nginx反向代理: 修改全局配置 ‘/etc/nginx/nginx.conf’
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
        worker_connections 768;
}

http {
        # 入口限流与连接数限制
        # # 每 IP 请求速率：适合普通接口
        limit_req_zone $binary_remote_addr zone=req_per_ip:10m rate=10r/s;
        # # 每 IP 并发连接：适合上传/长连接
        limit_conn_zone $binary_remote_addr zone=conn_per_ip:10m;
        # 登录更严格（防爆破）
        limit_req_zone $binary_remote_addr zone=login_per_ip:10m rate=5r/s;

        limit_req_log_level notice;
        limit_conn_log_level notice;

        # 读取国家 ISO code（CN/HK/MO/TW/US...）
        geoip2 /var/lib/GeoIP/GeoLite2-Country.mmdb {
                $geoip2_country_code country iso_code;
        }

        map $geoip2_country_code $allow_country {
                default 0;
                CN 1;
        }

        # 读取城市
        geoip2 /var/lib/GeoIP/GeoLite2-City.mmdb {
                $geoip2_city_name   city names en;
                $geoip2_region_name subdivisions 0 names en;
        }

        map $http_x_forwarded_for $xff_safe { default $http_x_forwarded_for; "" "-"; }
        map $http_cf_connecting_ip $cf_safe { default $http_cf_connecting_ip; "" "-"; }

        # 很多 IP 没城市，日志会出现 city=""。可以加 map 兜底成 -：
        map $geoip2_city_name $city_safe { default $geoip2_city_name; "" "-"; }
        map $geoip2_region_name $prov_safe { default $geoip2_region_name; "" "-"; }

        log_format sec_f2b '$remote_addr - - [$time_local] '
                   '"$request" $status $body_bytes_sent '
                   '"$http_referer" "$http_user_agent" '
                   'host=$host scheme=$scheme '
                   'cc=$geoip2_country_code prov="$prov_safe" city="$city_safe" '
                   'xff="$xff_safe" cf="$cf_safe" '
                   'rt=$request_time urt=$upstream_response_time uct=$upstream_connect_time uht=$upstream_header_time';

        # WebSocket 的 Connection 头最好用变量（更兼容）
        map $http_upgrade $connection_upgrade {
                default upgrade;
                ''      close;
        }

        ##
        # Basic Settings
        ##

        sendfile on;
        tcp_nopush on;
        types_hash_max_size 2048;
        server_tokens off;
        include /etc/nginx/mime.types;
        default_type application/octet-stream;

        ##
        # SSL Settings
        ##

        ssl_protocols TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
        ssl_prefer_server_ciphers on;

        ##
        # Logging Settings
        ##

        # 默认访问日志（可留）
        access_log /var/log/nginx/access.log;

        # 安全日志（包含城市/国家）
        access_log /var/log/nginx/access_security.log sec_f2b;

        # 错误日志
        error_log /var/log/nginx/error.log;

        ##
        # Gzip Settings
        ##

        gzip on;
        
        ##
        # Virtual Host Configs
        ##

        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;
}

 
 # 3.2 配置nginx反向代理: /etc/nginx/sites-available/o1145141414514o.asia
server {
    listen 80;
    listen [::]:80;
    server_name o1145141414514o.asia;

    location / {
        limit_except GET HEAD { deny all; }
        if ($allow_country = 0) { return 444; }
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name o1145141414514o.asia;

    ssl_certificate     /etc/letsencrypt/live/o1145141414514o.asia/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/o1145141414514o.asia/privkey.pem;

    # --- Security Headers (safe set) ---
    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
    add_header X-Frame-Options "SAMEORIGIN" always;

    client_max_body_size 0;
    proxy_request_buffering off;

    proxy_read_timeout  3600s;
    proxy_send_timeout  3600s;
    proxy_connect_timeout 60s;

    location ^~ /_app/ {
        if ($allow_country = 0) { return 444; }
        # 静态资源不要限连接/限速
        # 长缓存（可选）
        expires 30d;
        add_header Cache-Control "public, max-age=2592000, immutable" always;
        include /etc/nginx/snippets/proxy_immich_common.conf;
    }

    location / {
        # 非允许国家直接断开（放在 location，避免影响其它 location）
        if ($allow_country = 0) { return 444; }
        # limit_conn conn_per_ip 30;
        include /etc/nginx/snippets/proxy_immich_common.conf;
    }

    location ^~ /api/assets {
        if ($allow_country = 0) { return 444; }
        limit_conn conn_per_ip 10;
        # expires 7d;
        include /etc/nginx/snippets/proxy_immich_upload.conf;
    }

    #登录：严格限流（只卡爆破）
    location = /api/auth/login {
        if ($allow_country = 0) { return 444; }
        limit_req zone=login_per_ip burst=10 nodelay;
        limit_req_status 429;
        include /etc/nginx/snippets/proxy_immich_common.conf;
    }
 }
 
# 4. fail2ban - jail
sudo nano /etc/fail2ban/jail.d/custom.local
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1
bantime  = 24h
findtime = 10m

[sshd]
enabled = true
maxretry = 5
findtime = 10m
bantime = 6h

[nginx-app-444]
enabled  = true
port     = http,https
logpath  = /var/log/nginx/access_security.log
maxretry = 10
findtime = 2m
bantime  = 24h

[nginx-app-scan]
enabled  = true
port     = http,https
logpath  = /var/log/nginx/access_security.log
maxretry = 3
findtime = 10m
bantime  = 7d

[nginx-app-401]
enabled  = true
port     = http,https
logpath  = /var/log/nginx/access_security.log
# 5 分钟内 15 次 401/403 才封（先保守，防误封）
findtime = 5m
maxretry = 10
bantime  = 12h

# 4.1 444
sudo nano /etc/fail2ban/filter.d/nginx-app-444.conf

[Definition]
failregex = ^<HOST> - - \[.*\] ".*" (?:401|429) \d+ ".*" ".*" host=app\.yunzhaopian\.cc\b
ignoreregex =

# 4.2 ~app
sudo nano /etc/fail2ban/filter.d/nginx-app-scan.conf

[Definition]
failregex = ^<HOST> - - \[.*\] "(?:GET|POST|HEAD) (?:/wp-admin|/wp-login\.php|/xmlrpc\.php|/phpmyadmin|/pma|/\.env|/\.git|/\.svn|/cgi-bin|/actuator|/manager|/admin|/login)\b.*" \d+ \d+ ".*" ".*" host=app\.yunzhaopian\.cc\b
ignoreregex =

# 4.3 401
sudo nano /etc/fail2ban/filter.d/nginx-app-401.conf

[Definition]
failregex = ^<HOST> - - \[.*\] ".*" (?:401) \d+ ".*" ".*" host=app\.yunzhaopian\.cc\b
ignoreregex =
```

# 查询

```bash
journalctl --user -u immich-reverse-tunnel.service -n 50

sudo fail2ban-client get nginx-app-444 banip
sudo fail2ban-client get sshd banip
sudo fail2ban-client status nginx-app-444
sudo fail2ban-client status sshd
sudo grep -F "155.254.122.8" /var/log/nginx/access_security.log | tail -n 50

sudo tail -n 60 /var/log/nginx/access_security.log
sudo tail -n 60 /var/log/auth.log

sudo journalctl -u ssh  
sudo journalctl -u ssh   -n 100
```

