# Centos7上安装MySQL8教程

>在这个教程中，你将会学习如何一步一步的在Centos7上面安装MySQL8

## Step 1. 设置Yum存储库

执行下面的命令将会在Centos上启用MySQL的Yum存储库

```shell
rpm -Uvh https://repo.mysql.com/mysql80-community-release-el7-3.noarch.rpm
```

## Step 2.安装MySQL 8 Community Server

因为MySQL Yum存储库有多个MySQL版本的多个存储库配置，所以需要禁用mysql repo文件中的所有存储库

```shell
sed -i 's/enabled=1/enabled=0/' /etc/yum.repos.d/mysql-community.repo
```

使用下面的命令执行安装 MySQL8

```shell
yum --enablerepo=mysql80-community install mysql-community-server
```

在执行上面的命令的时候，可能为出现如下的错误提示:

```tex
警告：/var/cache/yum/x86_64/7/mysql80-community/packages/mysql-community-common-8.0.30-1.el7.x86_64.rpm: 头V4 RSA/SHA256 Signature, 密钥 ID 3a79bd29: NOKEY
从 file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql 检索密钥


源 "MySQL 8.0 Community Server" 的 GPG 密钥已安装，但是不适用于此软件包。请检查源的公钥 URL 是否配置正确。


 失败的软件包是：mysql-community-common-8.0.30-1.el7.x86_64
 GPG  密钥配置为：file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql
```

提示我们 GPG密钥和yum源包里面的下载链接的URL不匹配，请先使用下面命令，后重新执行MySQL安装命令

```shell
rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
```



## Step 3.开启MySQL服务

使用下面的命令开启MySQL服务

```shell
service mysqld start
```

## Step 4.查看root用户的默认密码

当成功安装完成MySQL8后，系统会自动为`root`用户设置一个临时的密码。使用下面命令可以查看`root`用户的临时密码

```SHELL
grep "A temporary password" /var/log/mysqld.log
```

将会有如下的输出结果：

```TXT
[Note] A temporary password is generated for root@localhost: hjkygMukj5+t783
```

**注意**：你的临时密码和上面输出结果中临时密码将会是不一样的哦。后续可以使用这个临时密码，修改`root`用户的密码

## Step 5.MySQL安全配置

执行下面的命令开启MySQL的安全配置向导

```shell
mysql_secure_installation
```

提示你需要输入`root`用户的当前密码（就是在Setp 4 中查看到的临时密码）

```TXT
Enter password for user root:
```

输入临时密码按<kbd>Enter</kbd>. 将会有如下的提示信息:

```tex
The existing password for the user account root has expired. Please set a new password.

New password:
Re-enter new password:
```

> 原来的临时密码已经过期了，需要重新给root用户设置新的密码.	

你需要输入两次root用户的密码.将会提示你下面的一些问题，建议所有都回答 yes (y)

```tex
# 是否删除匿名用户
Remove anonymous users? (Press y|Y for Yes, any other key for No) : y
# 禁止root用户远程登录
Disallow root login remotely? (Press y|Y for Yes, any other key for No) : y
# 删除test数据库和对test数据库的访问权限
Remove test database and access to it? (Press y|Y for Yes, any other key for No) : y
# 刷新表的权限
Reload privilege tables now? (Press y|Y for Yes, any other key for No) : y
```

## Step 6.重启和开机自启动MySQL服务

使用下面命令重启MySQL服务

```shell
service mysqld restart
```

设置MySQL开机自启动

```shell
chkconfig mysqld on
```

## Step 7.连接MySQL

使用下面的命令连接MySQL服务

```shell
mysql -u root -p
```

将提示你输入 root 用户的密码，输入对应的密码即可。

就会看到`mysql`的命令框了

```SHELL
mysql>
```

使用 `SHOW DATABASES`展示当前数据库实例中的全部数据库

```mysql
mysql> show databases;
```

```shell
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.05 sec)
```

## Step 8.设置远程连接和登录

进入MySQL使用mysql库

```SHELL
mysql -uroot -p
```

使用MySQL库

```MYSQL
use mysql;
```

修改允许连接的IP地址（host='%'）表示所有Ip地址都能够连接，你也可以只指定某个ip

```mysql
update user set host = '%' where user = 'root';
```

刷新权限

```mysql
FLUSH PRIVILEGES;
```

