---
title: 'Ubuntu 14.04 安装LAMP开发环境'
date: 2015-02-12 22:48:00
categories: ['工具, 库与框架', LAMP]
tags: [Ubuntu, LAMP, PHP]
---

最近工作需要用到一点PHP，因此首先了解了以下如何在Ubuntu下准备开发环境，期间查阅了不少文章，也遇到了不少问题，特记录如下，希望对其他开发人员能够有所帮助：

## 安装基础包

http://howtoubuntu.org/how-to-install-lamp-on-ubuntu

参照以上步骤就行。

## 将apache2的localhost默认路径指向你需要的开发路径

默认路径在/var/www/html下，相信大多数开发人员都不会直接将该目录下进行开发，通过以下步骤改变该默认路径：

<!-- More -->

### 修改/etc/apache2/sites-available/000-default.conf 配置文件

为了防止修改错误，首先可以对其备份：

```bash
sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/000-default.conf.bak
```

然后开始对其进行修改：

```bash
sudo vim /etc/apache2/sites-available/000-default.conf 
```

以下配置文件省略了注释部分：

```
<VirtualHost *:80>  
        ServerName localhost  
  
        ServerAdmin webmaster@localhost  
        DocumentRoot /home/destiny1020/Site/test  
  
        ErrorLog ${APACHE_LOG_DIR}/error.log  
        CustomLog ${APACHE_LOG_DIR}/access.log combined  
</VirtualHost>
```

实际上只需要修改几个地方：

1. 加上ServerName localhost --- 这是为了避免在启动apache2的时候出现一个关于ServerName不存在的警告。
2. 修改DocumentRoot --- 修改成你的目标开发路径。

### 修改apache.conf

```bash
sudo vim /etc/apache2/apache2.conf
```

找到定义Directory的部分，其中一个定义的路径是/var/www/html，将其路径修改成目标路径(也就是在第一步中指定的那个路径)，其他的不需要改变：

```
<Directory /home/destiny1020/Site/test/>  
        Options Indexes FollowSymLinks  
        AllowOverride None  
        Require all granted  
</Directory>
```

这里需要注意的是路径需要以"/"结尾。(我并没有去求证是否真的有必要这样做)

## 安装phpmyadmin

```bash
sudo apt-get install phpmyadmin
```

中途会让你配置需要为哪个server启用，选择apache2。

然后还会问你要不要进行DB配置，如果之前没有配置过，那么选择Yes，然后会向你要MySQL admin的密码。

紧接着，会让你输入一个密码，该密码用来向DB注册phpmyadmin，如果你直接Enter，那么会生成一个随机密码。这里我输入了一个自定义的密码。因为担心生成了随机密码后有什么不良后果。

完成安装后，为了让http://localhost/phpmyadmin/ 能够被访问，还需要进行如下配置：

### 修改/etc/apache2/apache2.conf

```bash
sudo vim /etc/apache2/apache2.conf
```

在该文件的最后添加：

```
Include /etc/phpmyadmin/apache.conf
```

### 最后重启apache2以确保修改生效

```bash
sudo /etc/init.d/apache2 restart
```

访问http://localhost/phpmyadmin/ 并登录。

这里需要输入的就是MySQL的用户名和密码。因此使用root和相应的密码即可登录。

到这里，Ubuntu下的LAMP环境就准备完毕了。

如果发现了任何问题，欢迎给我留言。

谢谢！
