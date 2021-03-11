---
title: "让phpcms支持https"
date: 2020-03-02T00:29:44+08:00
draft: false
Categories: ["php"]
Tags: ["php", "https", "phpcms"]
---



本文整理自PHPCMS官方论坛的一篇文章，感谢作者的奉献。

假设已经配置好ssl证书，不知如何申请ssl证书者请自行百度。

**1、如果已经安装好phpcms，则需要对caches/configs/system.php中的配置选项做替换，**将"http://"全部替换为"https://"。如有必要，数据库中已存在的链接也要完全替换为https开头。

**2、程序修改部分：**
**（1）**修改phpcms/modules/admin/site.php 大约45行和128行的正则

```
('/http:\/\/(.+)\/$/i', $domain))
```

  修改为

```
('/(http|https):\/\/(.+)\/$/i', $domain))
```

**（2）**修改phpcms/modules/admin/templates/setting.tpl.php 大约18行中的正则

```
http:\/\/(.+)[^/]$
```

  修改为

```
http[s]?:\/\/(.+)[^/]$
```

**（3）**修改phpcms/modules/admin/templates/site_add.tpl.php 大约13行中的正则

```
http:\/\/(.+)\/$
```

  修改为

```
http[s]?:\/\/(.+)\/$
```

**（4）**修改phpcms/modules/admin/templates/site_edit.tpl.php 大约11行中的正则

```
http:\/\/(.+)\/$
```

  修改为

```
http[s]?:\/\/(.+)\/$
```

**（5）**修改phpcms/modules/link/index.php 大约41行和51行中的正则

```
/^http:\/\/(.*)/i
```

  修改为

```
/^http[s]?:\/\/(.*)/i
```

**（6）**修改phpcms/modules/link/templates/link_add.tpl.php 大约10行中的正则

```
^http:\/\/[A-Za-z0-9]+\.[A-Za-z0-9]+[\/=\?%\-&]*([^<>])*$
```

  修改为

```
^http[s]?:\/\/[A-Za-z0-9]+\.[A-Za-z0-9]+[\/=\?%\-&]*([^<>])*$
```

**（7）**修改phpcms/modules/link/templates/link_edit.tpl.php 大约11行中的正则

```
^http:\/\/[A-Za-z0-9]+\.[A-Za-z0-9]+[\/=\?%\-&]*([^<>])*$
```

  修改为

```
^http[s]?:\/\/[A-Za-z0-9]+\.[A-Za-z0-9]+[\/=\?%\-&]*([^<>])*$
```

   严格按照以上步骤修改后，注册用户 帐号登录等操作完全正常 和PHPSSO通信完全正常，后台添加信息和前台链接URL完全正常

   注意：
  a.如注册用户提示‘操作失败’，请在后台会员模块设置中关闭‘注册时可选会员模型’或者保证会员不少于两个会员模型

  b.在PHP5.6或以上的PHP版本中会出现和PHPSSO无法正常通信的情况，因为PHP5.6及以上fsockopen和file_get_contents等函数openssl需要验证目标的SSL证书是否可信，需要安装openssl根证书才可以，否则openssl会报警告信息 证书验证失败！
  如需在php5.6或以上版本中使用HTTPS的请参阅PHP官方有关php5.6和openssl的资料[http://php.net/manual/en/migration56.openssl.php](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fphp.net%2Fmanual%2Fen%2Fmigration56.openssl.php)

**3、经过上面修改后，phpcms中的内容可通过https访问，但分页出现错误。解决方法为：**
   打开文件 phpcms\libs\functions \global.func.php ，找到738行的位置：

```
$url = str_replace(array('http://','//','~'), array('~','/','http://'), $url);
```

  修改为

```
$url = str_replace(array('https://','//','~'), array('~','/','https://'), $url);
```


经过以上三步，phpcms完美支持https，结合页面静态化和url伪静态规则，亲测静态页面也可用https。
**此时的网站实际上通过http和https都能访问**，如果要强制全站跳转到https，请继续看另一篇博客 **[Apache环境下强制跳转到https](https://my.oschina.net/codercpf/blog/3082342)**

 



**补充：**

经过以上所有步骤后，出现新的问题：新闻列表页面点击标题跳转的URL会重复带有网站域名。**解决方法如下：**
打开 phpcms\modules\content\templates\content_list.tpl.php，找到97行的位置：

```
strpos($r['url'],'http://')!==false
```

修改为：

```
strpos($r['url'],'https://')!==false
```