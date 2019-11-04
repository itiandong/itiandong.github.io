---
title: V2Ray 安装配置
categories:
  - - 技术
    - 安装配置
tags:
  - V2Ray
  - 配置
  - 科学上网
date: 2019-11-02 14:21:31
---

简单记录一下有点繁琐的 `V2Ray` 安装过程，供日后复制粘贴。方案采用的是据说很安全的 `WebSocket+TLS+Nginx`。网站正常访问的情况下是一个无害 https 静态站，真实面目却是一个罪恶（滑稽）的偷渡网站。
<!--more-->

服务器选用的 `Vultr` 的洛杉矶区，系统使用的 `Ubuntu 18.04`，客户端是 `Win10`，配合火狐插件 `Proxy SwichyOmega`食用。

## 时间校准

对于 V2Ray，它的验证方式包含时间，就算是配置没有任何问题，如果时间不正确，也无法连接 V2Ray 服务器的。需要保证时间误差在**90秒**之内就没问题。

查看系统时间

```bash
date -R
```

```
Sat, 02 Nov 2019 04:37:43 +0000
```

如此处换算成东八区与标准时间相同，否则，通过 `date --set="xxxx-xx-xx xx:xx:xx"` 设置。

## 配置服务端

### 生成 UUID

在后续配置当中，有一个 `id` ，作用类似于 `ss` 的密码, `VMess` 的 `id` 的格式必须与 `UUID` 格式相同。 

两种方式生成：

1.  `cat /proc/sys/kernel/random/uuid` 
2.  [UUID Generator](https://www.uuidgenerator.net/) 

### 脚本安装 V2Ray

```bash
apt update
apt upgrade
bash <(curl -L -s https://install.direct/go.sh)
```

此脚本会自动安装以下文件：

- `/usr/bin/v2ray/v2ray`：V2Ray 程序；
- `/usr/bin/v2ray/v2ctl`：V2Ray 工具；
- `/etc/v2ray/config.json`：配置文件；
- `/usr/bin/v2ray/geoip.dat`：IP 数据文件
- `/usr/bin/v2ray/geosite.dat`：域名数据文件

如果是更新，执行：

```bash
sudo bash go.sh
```

### 配置 TLS

#### 添加域名解析

在域名`DNS 面板`添加 `A记录`，指向服务器的 IP 地址。后文使用 `free.itiandong.com` 域名，已指向了 Vultr 服务器。

#### 生成证书

因为我是傻瓜，这里使用傻瓜脚本。执行以下命令，

```bash
apt install socat
curl  https://get.acme.sh | sh
```

输入下列命令生成证书，证书有两种，`ECC` 和 `RSA`。这里选用兼容性更好的 `RSA` 方案，

```bash
sudo ~/.acme.sh/acme.sh \
--issue -d free.itiandong.com \
--standalone -k 4096
```

值得一提的是，acme.sh 脚本会每 60 天自动更新证书。也可以选择手动更新，命令如下，

 ```bash
sudo ~/.acme.sh/acme.sh --renew \
-d free.itiandong.com --force
 ```

#### 安装证书

```bash
sudo ~/.acme.sh/acme.sh \
--installcert -d free.itiandong.com \
--fullchainpath /etc/v2ray/v2ray.crt \
--keypath /etc/v2ray/v2ray.key
```

### 配置服务端V2Ray

**`config.json`**

```json
{
  "inbounds": [
    {
      "port": 10000,
      "listen":"127.0.0.1",//只监听 127.0.0.1，避免除本机外的机器探测到开放了 10000 端口
      "protocol": "vmess",
      "settings": {
        "clients": [
          {
            "id": "<填入生成的 UUID >",
            "alterId": 64
          }
        ]
      },
      "streamSettings": {
        "network": "ws",
        "wsSettings": {
        "path": "/free"
        }
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "settings": {}
    }
  ]
}
```

### 配置 Nginx

目的是伪装成一个无害的静态站，同时然后转发客户端请求给服务端。

首先安装 `nginx`,

```bash
apt install nginx
```

**`nginx.conf`** 添加以下内容：

强制转发 `http` 到 `https`：

```properties
server {  
    listen  80;
    server_name free.itiandong.com;
	rewrite ^(.*)$  https://$host$1 permanent;
}
```

配置静态服务器以及转发请求：

```properties
server {
    listen  443 ssl;
    ssl on;
    ssl_certificate       /etc/v2ray/v2ray.crt;
    ssl_certificate_key   /etc/v2ray/v2ray.key;
    ssl_protocols         TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers           HIGH:!aNULL:!MD5;
    server_name           free.itiandong.com;

	location /free { # 与 V2Ray 配置中的 path 保持一致
        proxy_redirect off;
        proxy_pass http://127.0.0.1:10000;#假设WebSocket监听在环回地址的10000端口上
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $http_host;

        # Show realip in v2ray access.log
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    
    location / {
		root /home/www;
		index index.html index.htm;
	}
}
```

静态站放点啥呢？直接用 `Web编程课` 的某次课堂作业吧，2333：

```bash
git clone https://github.com/itiandong/project0-itiandong.git /home/www
```

想想还是重启下两个服务比较好？

```bash
systemctl restart v2ray
systemctl restart nginx
```



## 配置客户端

### CLI

点[这里](https://github.com/v2ray/v2ray-core/releases)下载。

配置**`config.json`**，监听`1081`端口，避免和 `ssr` 冲突。

```json
{
  "inbounds": [
    {
      "port": 1081,
      "listen": "127.0.0.1",
      "protocol": "socks",
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls"]
      },
      "settings": {
        "auth": "noauth",
        "udp": false
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "vmess",
      "settings": {
        "vnext": [
          {
            "address": "free.itiandong.com",
            "port": 443,
            "users": [
              {
                "id": "<填入生成的 UUID >",
                "alterId": 64
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "ws",
        "security": "tls",
        "wsSettings": {
          "path": "/free"
        }
      }
    }
  ]
}
```

运行 `v2ray.exe` 或者  `wv2ray.exe`  均可。前者有个 `cmd窗口`，后者没有。

配合 `Proxy SwichyOmega` 即可食用。

### GUI

GUI软件有很多，比如我所使用的：

1. 【Windows】 [v2rayN](https://github.com/2dust/v2rayN)
2. 【Android】 [v2rayNG](https://github.com/2dust/v2rayNG)

添加服务器配置方法大同小异：

1. 添加 `VMess` 服务器
2. 地址填入域名，端口填入 `443`，填入用户id、额外id，加密方式选择 `auto` ，传输协议选择 `ws`。路径填入 `/free`，底层传输协议选择 `tls`。 
3. 配合 `Proxy SwichyOmega` 即可食用。
