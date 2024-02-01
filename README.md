# 纯手动安装Trojan-go的配置文档
- 前置：需要CentOs7操作系统.
- 需要更新对应安装包.
- 需要手动安装BBR plus
- 安装htop
- 自己域名为 `smart.baiduge.store` ，可以根据自己域名将其替换！

## 安装nginx服务器与配置

```bash
# 更新安装源库
yum update

# 安装htop
yum install epel-release

# 安装Nginx服务器
yum install nginx

# 配置Nginx
cd /etc/nginx/conf.d

# 备份nginx配置
cd /etc/nginx/conf.d && mv default.conf default.confbak

echo "" > /etc/nginx/conf.d/trojan-go.conf && vi /etc/nginx/conf.d/trojan-go.conf


# 写入如下内容

server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;
    location / {
        proxy_pass https://demo.cloudreve.org;
        proxy_ssl_server_name on;
    }
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}

server {
    listen 443 ssl; 
    server_name localhost;
    ssl_certificate   /usr/local/etc/smart.baiduge.store/fullchain.crt;
    ssl_certificate_key  /usr/local/etc/smart.baiduge.store/privkey.key;
    ssl_session_timeout 12d;
}

# 确定nginx服务状态

systemctl stop nginx && systemctl status nginx

```

## 安装服务器证书

- acme.sh申请证书，会自动续证书有效期.

```bash
# 安装acme.sh
yum update &&  yum -y install socat

curl https://get.acme.sh | sh
source ~/.bashrc

# 使用acme.sh获取签发证书.
acme.sh --issue -d smart.baiduge.store --standalone -k ec-256 --force

# 创建证书目录，安装证书.
mkdir -p /usr/local/etc/smart.baiduge.store

# 安装证书.
acme.sh --installcert -d smart.baiduge.store --fullchainpath /usr/local/etc/smart.baiduge.store/fullchain.crt --keypath /usr/local/etc/smart.baiduge.store/privkey.key --ecc --force
```

## 安装Trojan-go服务与配置

- 下载安装Trojan-go服务
```bash
mkdir -p /etc/trojan-go/bin

wget --no-check-certificate -O /etc/trojan-go/bin/trojan-go-linux-amd64.zip "https://github.com/p4gefau1t/trojan-go/releases/download/v0.10.6/trojan-go-linux-amd64.zip"

# 解压安装.
unzip -o -d /etc/trojan-go/bin /etc/trojan-go/bin/trojan-go-linux-amd64.zip

# 创建server配置.
mkdir -p /etc/trojan-go/conf
vi /etc/trojan-go/conf/server.json

# 创建启动服务.
echo "" > /etc/systemd/system/trojan-go.service && vi /etc/systemd/system/trojan-go.service

# 写入如下内容.
[Unit]
Description=trojan-go
Documentation=https://github.com/p4gefau1t/trojan-go
After=network.target nginx.service

[Service]
Type=simple
StandardError=journal
PIDFile=/usr/src/trojan/trojan/trojan.pid
ExecStart=/etc/trojan-go/bin/trojan-go -config /etc/trojan-go/conf/server.json
ExecReload=
ExecStop=/etc/trojan-go/bin/trojan-go
LimitNOFILE=51200
Restart=no

[Install]
WantedBy=multi-user.target


# 加载服务文件.
systemctl daemon-reload

# 编辑Trojan-go服务器设置.
echo "" > /etc/trojan-go/conf/server.json && vi /etc/trojan-go/conf/server.json

# 写入如下内容，并且退出，这个是simple版本，但是足够安全，来自于官网设置.
{
    "run_type": "server",
    "local_addr": "0.0.0.0",
    "local_port": 24052,
    "remote_addr": "127.0.0.1",
    "remote_port": 80,
    "password": [
        "9ae23b11-4dba-450d-a92e-0c3f23686754"
    ],
    "ssl": {
        "cert": "/usr/local/etc/smart.baiduge.store/fullchain.crt",
        "key": "/usr/local/etc/smart.baiduge.store/privkey.key"
    }
}

# 手动测试运行

/etc/trojan-go/bin/trojan-go -config /etc/trojan-go/conf/server.json
```


## 检测设置是否正确.

```bash
# 设置开机自动nginx，Trojan-go
systemctl enable nginx
systemctl enable trojan-go

# 启动
systemctl start nginx
systemctl start trojan-go

# 查看状态.
systemctl status nginx
systemctl status trojan-go
```

- 默认服务器配置不启用CDN，但是开启多线程.

```bash
# 完整Trojan-go服务器端配置.

echo "" > /etc/trojan-go/conf/server.json && vi /etc/trojan-go/conf/server.json

```json
{
    "run_type": "server",
    "local_addr": "0.0.0.0",
    "local_port": 24052,
    "remote_addr": "127.0.0.1",
    "remote_port": 80,
    "log_level": 3,
    "log_file": "",
    "password": [
        "9ae23b11-4dba-450d-a92e-0c3f23686754"
    ],
    "buffer_size": 32,
    "dns": [],
    "ssl": {
        "verify": true,
        "verify_hostname": true,
        "cert": "/usr/local/etc/smart.baiduge.store/fullchain.crt",
        "key": "/usr/local/etc/smart.baiduge.store/privkey.key",
        "key_password": "",
        "cipher": "",
        "cipher_tls13": "",
        "curves": "",
        "prefer_server_cipher": false,
        "sni": "smart.baiduge.store",
        "alpn": [
            "http/1.1"
        ],
        "session_ticket": true,
        "reuse_session": true,
        "plain_http_response": "",
        "fallback_port": 443,
        "fingerprint": "firefox",
        "serve_plain_text": false
    },
    "tcp": {
        "no_delay": true,
        "keep_alive": true,
        "reuse_port": false,
        "prefer_ipv4": false,
        "fast_open": false,
        "fast_open_qlen": 20
    },
    "mux": {
        "enabled": true,
        "concurrency": 8,
        "idle_timeout": 60
    },
    "router": {
        "enabled": false,
        "bypass": [],
        "proxy": [],
        "block": [],
        "default_policy": "proxy",
        "domain_strategy": "as_is",
        "geoip": "./geoip.dat",
        "geosite": "./geoip.dat"
    },
    "websocket": {
        "enabled": false,
        "path": "",
        "hostname": "127.0.0.1",
        "obfuscation_password": "",
        "double_tls": false,
        "ssl": {
            "verify": true,
            "verify_hostname": true,
            "cert": "/usr/local/etc/smart.baiduge.store/fullchain.crt",
            "key": "/usr/local/etc/smart.baiduge.store/privkey.key",
            "key_password": "",
            "prefer_server_cipher": false,
            "sni": "",
            "session_ticket": true,
            "reuse_session": true,
            "plain_http_response": ""
        }
    },
    "forward_proxy": {
        "enabled": false,
        "proxy_addr": "",
        "proxy_port": 0,
        "username": "",
        "password": ""
    },
    "mysql": {
        "enabled": false,
        "server_addr": "localhost",
        "server_port": 3306,
        "database": "",
        "username": "",
        "password": "",
        "check_rate": 60
    },
    "redis": {
        "enabled": false,
        "server_addr": "localhost",
        "server_port": 6379,
        "password": ""
    },
    "api": {
        "enabled": false,
        "api_addr": "",
        "api_port": 0
    }
}
```
