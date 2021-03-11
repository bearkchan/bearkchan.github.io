---
title: "Nginx自动跳转到www域名设置"
date: 2020-03-02T00:29:44+08:00
draft: false
Categories: ["nginx"]
Tags: ["nginx", "www"]
---



## 步骤一

在域名解析中将定义的xxx.com和www.xxx.com都指向你的主机ip地址。

![image-20210309234859912](http://cdn.bearkchan.top/image-20210309234859912.png)

配置完成之后可在终端使用`nslookup`命令查看域名执行ip的A记录。

```bash
$ nslookup xxx.com
Server:		192.168.99.1
Address:	192.168.99.1#53

Non-authoritative answer:
Name:	xxx.com
Address: xx.xx.xx.xx
```

## 步骤二

找到网站的nginx配置文件，配置以下内容：

```nginx
server{
    listen 80;
	server_name www.xxx.com xxx.com;
	if ($host != 'www.xxx.com') {
	rewrite ^/(.*)$ http://www.xxx.com/$1 permanent;
}
```

这样就是用户直接访问 xxx.com 直接跳转的www.xxx.com。即让不带 www 的域名跳转到带 www 的域名。

