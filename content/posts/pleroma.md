---
title: "Pleroma的安装、初步设置、数据备份"
date: 2021-03-23
lastmod: 2021-03-23
author: "rimo"

tags: [折腾笔记]
categories: [记录]
hiddenFromHomePage: false
hiddenFromSearch: false

---
<font size=3>
{{< admonition warning >}}
原文写于2021年3月23日 
  
内容可能已经过时,仅作为原文备份
  
图片也丢了
{{< /admonition >}}


本文主要是备忘。关于Pleroma初次安装配置遇到很多的坑,查阅官方文档和issue也有一些过时的信息。

## 1.
丢了没了懒得改了,总之就是提醒不要用国内的域名商

## 2.准备服务器

示例环境:debian 10 64位

### 2.1 为服务器准备SSH key

网上很多教程是使用ssh-keygen生成密钥,这种生成方式是不能直接用于windows的SSH客户端的,如果你的电脑是windows系统,不建议这么做

**以windows为例,下载PuTTY 并安装**

-----
- 进入PuTTY安装目录,打开puttygen.exe

  - 点击按钮“Generate”。程序会检测鼠标,需要将鼠标在窗口内不停移动进度条才会进展。

   - 保存公钥及私钥,上传公钥。私钥用于ssh登录,妥善保管私钥。
密码登录SSH后上传,对于一些服务器提供商如vultr,也可以从网站面板上传,不如说更推荐这么做。

    * vultr上传公钥只需复制图中长长的文本部分到网站面板即可。

-上传完成后设置禁止密码登录SSH
```
nano /etc/ssh/sshd_config
```
去除注释PasswordAuthentication并修改为no

重启sshd
```
systemctl restart sshd.service
```
### 2.2 安全设置

- 安装 fail2ban 以阻止重复登录尝试 
```
apt-get install fail2ban
```
* 以下内容来自[长毛象官方文档](https://docs.joinmastodon.org/zh-cn/admin/prerequisites/),照抄即可


-----


> 编辑/etc/fail2ban/jail.local并添加以下内容:
```
[DEFAULT]
destemail = your@email.here
sendername = Fail2Ban
[sshd]``
enabled = true
port = 22
[sshd-ddos]
enabled = true
port = 22
```


-----


最后重启fail2ban:
```
systemctl restart fail2ban
```

 
* 安装并设置防火墙
>长毛象官方文档 中使用的是 iptables-persistent

参考尽量细致的Pleroma搭建教程(仮),建议使用ufw,简单嘛
```
apt-get install ufw

//默认开启出站关闭入站
sudo ufw default allow outcoming
sudo ufw default deny incoming

//按需开启入站端口
sudo ufw allow 80
sudo ufw allow 443
sudo ufw allow 22
ufw enable
 ```
### 2.3 设置域名解析
参考长毛象社区搭建详解第二章即可
 
## 3.安装pleroma
其实关于安装方面,pleroma的官方文档已经足够详尽了,一步一步照抄即可,没什么难度。
 
!以下仅说明OTP版本Pleroma,如过是从source安装,可以尝试参考[Pleroma:支持ActivityPub的社交网络服务器](https://lala.im/6254.html)
 
运行到
>\# Create the database schema
su pleroma -s $SHELL -lc “./bin/pleroma_ctl migrate”

这一步时,安装程序会问很多问题,并以此生成配置文件。只要域名没写错就没什么大问题,大部分后期都可以改。
- 个人建议这个阶段不要开启Pleroma in-database,有一定风险,没有必要现在开,调试好了再说
- ---
另外
OTP版安装文档中,编辑nginx config中有这么一句
$EDITOR path-to-nginx-config
初见的话有点摸不着头脑,其实就是去编辑上一步操作的/etc/nginx/pleroma.conf
 
按文档走完,访问域名就可以看见页面了。
+ 注意,最后一步创建管理员帐户后,终端中会给一段链接,通过浏览器访问这个链接设置密码
 
## 4.设置pleroma
设置文件在/etc/pleroma/config.exs
官方文档要求改动这个文件,但是个人建议不要改动,直接备份一份到其他地方去。
 
+ 使用这份设置文件绝对能打开pleroma,这一点非常重要。万一在Admin-FE输入了错误的设置导致pleroma无法打开,有它在还可以救一命
 
后续建议都在数据库模式下于Admin-FE操作。
+ *因为官方文档有关设置文件中各项参数的说明并不是很好理解,查阅他人留下的经验又很容易碰到过时的设置参数,导致出现错误却查无可查。而使用Admin-FE就不会遇到这种问题。即使不打算长期使用数据库模式,也只要从数据库导出一份设置文件就好*

另外,官方文档中的$ ./bin/pleroma_ctl
都是指进入/opt/pleroma目录后的操作
 

-----


<font size=5>**关于数据库模式下,输出设置中可能存在的bug**</font>

感谢@ShyKana@wuppo.allowed.org的帮助。

- pleroma版本 2.3.0-1-gb221d77a

> 实际使用发现,按照官方文档描述导出设置文件
执行$ ./bin/pleroma_ctl config migrate_from_db会出现错误,运行无效。

根据@ShyKana@wuppo.allowed.org的建议
可以尝试修改/etc/systemd/system/pleroma.service
>找到ProtectSystem=full,改为ProtectSystem=true

重新启动pleroma
```
systemctl daemon-reload systemctl restart pleroma
```
重新执行导出
```
$ ./bin/pleroma_ctl config migrate_from_db
```
可以发现/etc/pleroma目录下输出了文件 prod.exported_from_db.secret.exs

:fa-cog spin: **随后需要移走这个文件,目录下存在该文件会导致pleroma无法启动。**


 
>另外需要注意的是,只有在数据库模式下才可以导出设置文件,只有在设置文件模式下才可以导入数据库,导出的文件不可直接用于启动pleroma,请与config.exs合并之后再使用。
报错请先检查一下目前运行的模式。

  >检查方式即查找下文提到的config :pleroma, configurable_from_database
 
备份好了就可以开始了
 
### 4.1 修改pleroma为数据库模式
Admin-FE的大部分功能仅在数据库模式下有效。
>修改/etc/pleroma/config.exs
+ 在最底下找到config :pleroma, configurable_from_database
  + 将其改为true,重启pleroma
  ```
  systemctl restart pleroma
  ```
 
### 4.2 进入Admin-FE设置
+ 浏览器进入 https://你的域名/pleroma/admin/
登录管理员帐户,即可看到设置界面。
 
+ 大部分内容都是按需设置即可,可以省略大量查阅文档,磨配置文件的过程。
 
+ 如果不是很清楚的话,建议修改配置前使用vps的snapshots功能备份服务器,玩坏了也不用担心,整个恢复服务器就好。
  比如vultr就提供免费的snapshots,如果有这项功能的话务必用起来,一步操作一备份都没关系,这不是胆小,是安全至上(
 
开启了Media proxy后不显示缩略图的话,可以尝试开启Media preview proxy
 
### 4.3 部署媒体文件外置存储
这是一个难点,官方文档和官方gitlab中的issues中确实有相关内容,但是不是过于老旧就是说明不清,特别是关于使用AWS以外的兼容S3存储该如何设置。
当然,如果你的服务器没有存储方面的担忧,就可以跳过
关于如何选择外置存储方面可以参考长毛象社区搭建详解第六章
 

-----


这里以Scaleway为例
+ 注册并创建一个Bucket,存储桶大小设置成75G就可以白嫖,岂不美哉(
>长毛象社区搭建详解第六章 要求储存桶的名称务必填写媒体服务的子域名。理论上来说并不是必须这么做,但是照做,避免不必要的坑。

  Visibility设置为 Private。网络上有些教程要求设置为Public,但是实际上并不需要设置为 Public,否则任何人都可以看到媒体文件列表。
+ 获取所需的信息
 
 这里就照搬 长毛象社区搭建详解 了
 
 
 
注意这里的api key只会显示一次,请立即记录下来
 
 
接下来在Admin-FE中填写信息。
 
这是我的设置,直接套模板就行,media.shiina-rimo.cafe换成自己的域名,即之前填写的媒体服务的子域名
另外填上上一步获得的api key就好
 
官方文档的描述很难懂,特别是对于Truncated namespace的作用的说明模棱两可。但是Admin-FE上的描述就很合适。
Bucket namespace的内容,即文件上传到存储桶的目录。如填写media,文件上传到/media/
我是空着的,即直接上传到根目录。
 
接下来通过终端重启pleroma:
``` 
systemctl restart pleroma
 ```
### 4.4 设置DNS重定向
在cloudflare中添加一个DNS条目
类型CNAME,名称即之前填写的媒体服务的子域名。目标指向Bucket Endpoint,可在scaleway存储桶页面Bucket Settings选项卡查询。
 
回到pleroma,试着传一个图片文件,并发出嘟文,看看是否正常
如果有问题,可以试试删除Media proxy的Base URL看看是否恢复
 
一切正常,就大功告成!
 
补充:如需邮件功能,建议参考 技术小白如何搭建Mastodon实例 中的 邮件服务 章节设置邮箱,然后去Admin-FE的mailer中填写Pleroma.Emails.Mailer部分,Adapter选择STMP。
 
## 5.备份pleroma
pleroma需要备份的内容主要是数据库、设置文件、静态目录。
官方文档提到的config/setup_db.psql,目前OTP版本似乎不存在
 
+ 备份数据库
参考官方文档:
```
sudo -Hu postgres pg_dump -d <pleroma_db> –format=custom -f </path/to/backup_location/pleroma.pgdump>
``` 
+ 上述命令中,“postgres”是PostgreSQL 默认管理员用户名,没有自行修改则不需改动
  +  <pleroma_db>pleroma的数据库名。
  + </path/to/backup_location/pleroma.pgdump>填写备份文件输出目录
 
数据库名默认pleroma,不知道的话可以按照以下方法查看:
切换到数据库
```
sudo -u postgres psql
```
查看所有用户
```
\du
```
查看所有数据库
```
\l 
```
  默认的数据库应该是pleroma 
退出
```
\q 
  ```
  完整的postgresql常用命令可以看[这里](https://mozillazg.com/2014/06/hello-postgresql.html#)
  + 复制输出文件pleroma.pgdump  
  + 复制目录及所有文件/etc/pleroma/,这里是设置文件存放目录 
  + 复制目录及所有文件/var/lib/pleroma/,这里是使用过程中上传文件、表情包等默认存放位置 
  + (可选)复制/etc/nginx/sites-available/pleroma.conf 

  因为除了Pleroma外操作很少,备份和恢复操作我大多是继续使用VPS的Snapshots功能 所以恢复方面未实际操作过,就不写了,链接[官方文档 ](http://https://docs-develop.pleroma.social/backend/administration/backup/)