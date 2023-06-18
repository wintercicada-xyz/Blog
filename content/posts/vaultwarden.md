---
title: "Vaultwarden（Bitwarden_rs）搭建"
date: 2022-03-04T09:41:36+08:00
tags: ["Vaultwarden"]
enableOutdatedInfoWarning: true
draft: false 
---
这周在服务器上搭建了 Vaultwarden 服务，从今以后就用 Bitwarden 替代 Keepass 作为密码管理软件了，记录一下过程和踩的坑。

<!--more-->

## Bitwarden 与 Vaultwarden
Bitwarden 是一个开源的跨平台密码管理器，采用 C# 与 Typescript 开发，同时支持自建服务器。它的服务器端为 C# 开发并用 Docker 部署。我的服务器性能太弱，运行原版的 Bitwarden 服务器太吃力。但是网上还有一个用 Rust 写的 Bitwarden 叫 Vaultwarden（原名为 bitwarden_rs），相对来说对配置的要求没有这么高。

## 对比 Bitwarden 与 Keepass
Bitwarden 与 Keepass 可以说是唯有的两个开源且跨平台的密码管理服务了。开源保障密码管理服务的安全性，安全漏洞能被及时发现和处理，也不用担心有后门，而跨平台又保障了便利性，密码可以同步，不需要每个设备单独导入密码。

Bitwarden 所有客户端拥有一致的功能与界面，而且支持诸如通过桌面端获取指纹来认证浏览器插件的密码访问操作这种联动功能。同时 Bitwarden 把所有密码都同步到服务器端，用官方的服务器可能信不过，那就自己搭一个，这样也兼顾了便利性与安全性。

Keepass 没有一个统一的组织来开发所有平台的客户端，各个客户端之间界面风格与使用方式都有一定的区别。且 Keepass 设计之初就是没有密码同步机制的，所有文件都放在本地，要同步只能自己配置 Webdav 来实现，完全没有体现出跨平台的便利性优势。


## Vaultwarden 搭建

### 软件准备 
Vaultwarden 与 Bitwarden 一样都要用 Docker 部署，所以首先是要安装 Docker，这里就以 Debian 为例。

卸载旧版 Docker：
```Bash
sudo apt-get remove docker docker-engine docker.io containerd runc
```

安装添加 Docker GPG key 和下载 Docker 所需的软件：
```Bash
sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

添加 Docker GPG key：
```Bash
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

添加 Docker 的源：
```Bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

安装 Docker：
```Bash
 sudo apt-get update
 sudo apt-get install docker-ce docker-ce-cli containerd.io
```

安装 Docker Compose（便于部署 Vaultwarden）：
```Bash
# 下载 Docker Compose
 sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose 
 # 赋予运行权限：
sudo chmod +x /usr/local/bin/docker-compose
```

安装完 Docker 还要安装 Nginx 用于反代 Vaultwarden 服务并提供 HTTPS 加解密：
```Bash
sudo apt install nginx
```

最后是要在域名服务商处的 DNS 中配置域名指向你当前的服务器公网 ip，并准备你对应域名的证书和私钥到你的服务器上。我这里是使用 acme.sh 签发 Zerossl 证书，这里的方法有许多种，就不详述了。


### 安装配置
先在服务器创建一个叫 docker-compose.yml 的文件，里面内容如下：
```yaml
version: '3'
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: always
    environment:
      - WEBSOCKET_ENABLED=true  # Enable WebSocket notifications.
      - DOMAIN=<YOUR DOMAIN>
    volumes:
      - ./vw-data:/data
    ports:
     - "127.0.0.1:80:80"
     - "127.0.0.1:3012:3012"
```
记得把 `<YOUR DOMAIN>` 替换为你的证书对应的域名。


然后配置一下 Nginx， 在 `/etc/nginx/conf.d/` 目录下创建一个名为 `vaultwarden.conf` 的文件，内容如下：
```Nginx
server {
  listen 443 ssl http2;
  server_name <YOUR DOMAIN>;

  ssl_certificate   <YOUR CERTIFICATE>;
  ssl_certificate_key    <YOUR PRIVATE KEY>;
  ssl_protocols TLSv1.2 TLSv1.3;
  
  # Allow large attachments
  client_max_body_size 128M;

  location / {
    proxy_pass http://127.0.0.1:80;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
  
  location /notifications/hub {
    proxy_pass http://127.0.0.1:3012;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
  }
  
  location /notifications/hub/negotiate {
    proxy_pass http://127.0.0.1:80;
  }
}
```
同样是要把 `<YOUR DOMAIN>` 替换为你的域名，还要把 `<YOUR CERTIFICATE>` 和 `<YOUR PRIVATE KEY>` 分别替换为服务器上证书和私钥的路径，后面都是如此替换，就不再提了。


基本配置完了，开启 Docker 和 Nginx：
```Bash
# 为 Docker 启用开机自启并开始运行
sudo systemctl enable docker
sudo systemctl start docker

# 为 Nginx 启用开机自启并开始运行
sudo systemctl enable nginx
sudo systemctl start nginx
```


最后就是运行 Vaultwarden 了，在之前放有 docker-compose.yml 的目录中执行以下代码：
```Bash
sudo docker-compose up -d
```
等到执行完之后，你的 Vaultwarden 服务器就搭建好了，浏览器访问对应域名就可以使用了。


### 使用及踩坑
浏览器访问对应网站的界面如下：
![登录界面](/static/images/login.png)
首先点击右下角按钮注册账号，注意主密码要足够长足够复杂，这一个密码是用来保护你其他所有密码的。


然后登录进去：
![主界面](/static/images/vault.png)


为了保障安全性，避免被暴力破解主密码，强烈建议在两步验证与 [Fail2Ban](https://github.com/dani-garcia/vaultwarden/wiki/Fail2Ban-Setup) 中二选一。
这里只讲如何开启两步认证，想要开启 Fail2Ban 看上面链接自行配置。在登录进入网页后点击左上的设置然后在右边竖栏中选择两步认证，按你意愿选择合适的两步认证开启即可。
![两步验证](/static/images/2FA.png)


还可以进行其他的配置来提高服务的隐蔽性并防止他人滥用，设置禁止其他用户注册，并关闭网页版的 Bitwarden，只需在 docker-compose.yml 中 `- DOMAIN=<YOUR DOMAIN>` 后追加以下内容：
```yaml
      - SIGNUPS_ALLOWED=false   # 禁止注册
      - WEB_VAULT_ENABLED=false　# 关闭网页版 Bitwarden
```
这里关闭网页版 Bitwarden 后会并不会失去分享加密消息（Send）的功能，只是没有了网页登录的界面，分享的加密消息页面还是可以打开的。更改完后重启 Vaultwarden：
```Bash
sudo docker-compose down
sudo docker-compose up -d
```


之后就要配置客户端了，客户端统一都是在右上角齿轮图标打开后设置自建服务器，就输入 `https://<YOUR DOMAIN>` 即可，之后用原先注册的账号登录就行。

但是这在移动端好像行不通，一直都是连接出错，无法登录。这其实是一个坑，移动端无法连接没有配置 `OSCP Stapling` 的服务器，需要在之前的 `/etc/nginx/conf.d/vaultwarden.conf` Nginx 配置文件中 `ssl_protocols TLSv1.2 TLSv1.3;` 后加入以下内容：
```nginx
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate <YOUR CERTFICATE>;

    # 以下两行只在国内服务器使用，如果服务器在国外就不需要加入这两行了
    resolver 8.8.8.8 8.8.4.4 valid=60s ipv6=off;
    resolver_timeout 5s;
```
之后通过 `sudo systemctl restart nginx` 重启 Nginx，移动端就可以登录了。


## 参考
* [安装 Docker](https://docs.docker.com/engine/install/debian/)
* [安装 Docker Compose](https://docs.docker.com/compose/install/)
* [Docker Compose 部署 Vaultwarden](https://github.com/dani-garcia/vaultwarden/wiki/Using-Docker-Compose)
* [Vaultwarden Nginx 配置](https://github.com/dani-garcia/vaultwarden/wiki/Proxy-examples)
* [OSCP Stapling 配置](https://wangejiba.com/4709.html)





