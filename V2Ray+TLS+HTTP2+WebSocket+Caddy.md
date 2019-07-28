# V2Ray+TLS+HTTP/2+WebSocket+Caddy

> * I wrote this in Chinese because I think this tutorial is super useful to Chinese people.
>
> * 参考：
>
>   [从零开始在Ubuntu上搭建 V2Ray h2 + tls + web 代理服务](https://canmipai.com/index.php/2018/06/28/v2ray_h2_web_tutorial/)
>
>   [WebSocket+TLS+Web](https://toutyrater.github.io/advanced/wss_and_web.html)
>
>   [caddy官方脚本一键安装与使用]([https://medium.com/@jestem/caddy%E5%AE%98%E6%96%B9%E8%84%9A%E6%9C%AC%E4%B8%80%E9%94%AE%E5%AE%89%E8%A3%85%E4%B8%8E%E4%BD%BF%E7%94%A8-1e6d25154804](https://medium.com/@jestem/caddy官方脚本一键安装与使用-1e6d25154804))

## 前言

本教程实现了：

> * 利用`Caddy`反向代理隐藏`V2Ray`服务
> * `HTTP/2`和`WebSocket`两种协议的支持（支持后者是因为iOS平台的`Shadowrocket`不支持前者）

在开始前，你需要：

> * 运行`Ubuntu 18.04`的服务器（别的系统也行，不过指令可能略有不同）
> * 域名（必须要有，可在`Freenom`免费申请）
> * 提升服务器安全性，尽量榨干性能（参考另一篇教程：`Optimize & Secure a Linux Server`）

## Step I. 安装Caddy

使用官方脚本一键安装`Caddy`：

```
#如需更新，再次执行该命令即可（其他的命令需不要执行第二次）
sudo curl https://getcaddy.com | bash -s personal
```

配置root拥有二进制文件防止其他账户修改：

```
sudo chown root:root /usr/local/bin/caddy
```

修改权限为755，`root`可读写执行，其他账户不可写：

```
sudo chmod 755 /usr/local/bin/caddy
```

使用`setcap`允许`Caddy`作为用户进程绑定低号端口（服务器需要80和443）：

```
sudo setcap 'cap_net_bind_service=+ep' /usr/local/bin/caddy
```

创建文件夹存储`Caddy`的配置文件：

```
sudo mkdir /etc/caddy
```

创建文件夹存储`Caddy`所管理的站点证书：

```
sudo mkdir /etc/ssl/caddy
```

允许`root`及`www-data`组访问相关文件，允许`Caddy`写入站点证书文件夹：

```
sudo chown -R root:www-data /etc/caddy
sudo chown -R root:www-data /etc/ssl/caddy
sudo chmod 0770 /etc/ssl/caddy
```

如果默认站点根目录不存在，创建以下文件夹：

```
sudo mkdir /var/www
```

允许`www-data`组拥有站点文件夹：

```
sudo chown www-data:www-data /var/www
```

创建空的`Caddy`配置文件：

```
sudo touch /etc/caddy/Caddyfile
```

配置`systemd`服务：

```
sudo curl -s  https://raw.githubusercontent.com/mholt/caddy/master/dist/init/linux-systemd/caddy.service  -o /etc/systemd/system/caddy.service
```

调整权限使其只可被`root`修改：

```
sudo chmod 644 /etc/systemd/system/caddy.service
```

重载`systemd`使其检测到新安装的`Caddy`服务：

```
sudo systemctl daemon-reload
```

**验证`Caddy`是否安装成功：**

```
sudo systemctl status caddy
#看到如下输出表示服务注册成功
	caddy.service - Caddy HTTP/2 web server
	Loaded: loaded (/etc/systemd/system/caddy.service; disabled; vendor preset: enabled)
	Active: inactive (dead)
	Docs: https://caddyserver.com/docs
```

## Step II. 配置Caddy和HTTPS

先创建一个站点文件：

```
sudo touch /var/www/index.html
```

编辑该文件：

```
sudo nano /var/www/index.html
```

假如你没有预先写好的站点文件，可以粘贴以下内容保存：

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Hello from Caddy!</title>
  </head>
  <body>
    <h1 style="font-family: sans-serif">This page is being served via Caddy</h1>
  </body>
</html>
```

**注：这个页面将成为直接访问域名后看到的页面，保持这个状态是可行的，但是一个只有简单页面的网站每个月流量几十GB感觉还是很可疑的，建议后期自行写一个好看点的网站覆盖。**

编辑`Caddy`配置文件：

```
sudo nano /etc/caddy/Caddyfile
```

粘贴如下内容保存（`<example.com>`替换成指向服务器地址的域名，`<user@example.com>`替换成邮箱，以防万一还是要多嘴一句尖括号也是要被替换掉内容的一部分）：

```
http://<example.com> {
    redir https://<example.com>{url}
}
 
https://<example.com> {
    root /var/www/
    gzip
 
    tls <user@example.com> {
        ciphers ECDHE-ECDSA-WITH-CHACHA20-POLY1305 ECDHE-ECDSA-AES256-GCM-SHA384 ECDHE-ECDSA-AES256-CBC-SHA
        curves p384
        key_type p384
    }
 
    header / {
        Strict-Transport-Security "max-age=31536000;"
        X-XSS-Protection "1; mode=block"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "DENY"
    }
}
```

手动切换到配置文件所在文件夹：

```
cd /etc/caddy
```

以命令行方式启动`Caddy`：

```
#假如你很心急已经启动过Caddy，需先结束Caddy进程
#sudo systemctl stop caddy
caddy
```

第一次启动采用命令行启动的方式时因为需要接受`Let’s Encrypt`的协议，一路同意过去就行了，当看到`done`表示证书获取成功，此时进程并不会退出，可以`Ctrl + C`终止然后通过`Systemd`重启`Caddy`：

```
sudo systemctl start caddy
#如需开机自启，将start换成enable执行一次即可，如需重启，将start换成restart
```

如需验证`HTTPS`配置可到[SSL Labs](https://www.ssllabs.com/)检测一下评分，正常情况下可以获得满分`A+`的评价。

## Step III. 安装与配置V2Ray

获取并执行一键安装脚本：

```
wget https://install.direct/go.sh
#如需更新，再次执行go.sh即可
sudo bash go.sh
```

编辑配置文件：

```
sudo nano /etc/v2ray/config.json
```

删光里面的内容替换成下面这些（按照要求替换尖括号内容，建议先在本地编辑好，到[这里](https://www.json.cn/)检验`JSON`文件正确性，然后再复制回去）：

```

```

注意到`tlsSettings`配置项中有两个目前不存在的文件路径，这里原本需要填写`Caddy`生成的证书文件的路径，但是这个路径太长了，而且假如后期更换域名还要再修改配置文件，因此需要新建两个软链：

```

```

**注意：软链生成完毕后可在`/etc/v2ray`目录下执行`ls`命令验证，如果创建成功应有蓝色字样的`.crt`文件和`.key`文件，若为红色则表示软链指向一个不存在的文件，此时运行`v2ray`会报错，造成此问题的原因可能是之前配置`Caddy`的过程中出错导致证书文件被保存到了别的位置，可使用`linux`的搜索文件命令查找证书文件被放在了哪里：**

```

```

**施工中，还没写完**