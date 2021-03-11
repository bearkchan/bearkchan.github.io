---
title: "phpcms的使用"
date: 2020-03-02T00:29:44+08:00
draft: false
Categories: ["php"]
Tags: ["php", "phpcms"]
---



## 模板文件PC标签的使用

### 静态文件的调用

- WEB_PATH：网站根目录
- JS_PATH：JS路径
- CSS_PATH：CSS路径
- IMG_PATH：图片路径

> 默认是添加斜杠的。

### 首页链接的调用

```php
{siteurl($siteid)}/
```

### 某个单独栏目url/名称的调用

```php
{$CATEGORYS[id]['url']}
{$CATEGORYS[id]['catname']}
```

### 文章列表的调用

```php
{pc:content action="lists" catid="2" order="id DESC" num="4"}
	<ul>
  {loop $data $key $val}
		<li><a href="{$val['url']}"{$val['title']}</a></li>
	{/loop}
  </ul>
{/pc}
```

### 分类列表

![image-20210228091648040](http://cdn.bearkchan.top/image-20210228091648040.png)