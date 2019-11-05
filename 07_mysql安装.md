# 07_mysql安装

## 1.检测本地是否安装mysql已存在的包

```
rpm -qa | grep mysql
```

## 2.检测本地是否有mariadb已存在的包

```
rpm -qa | grep mariadb
```

存在：

```
[root@bigData111 module]# rpm -qa | grep mariadb
mariadb-libs-5.5.56-2.el7.x86_64
```

如果存在则删除：

```
yum -y remove mariadb-libs-5.5.56-2.el7.x86_64
```

## 3.创建一个文件夹，上传jar包到/opt/software/mysql

```
mkdir /opt/software/mysql
```

## 4.**解压mysql** jar包

```
tar -xvf mysql-5.7.19-1.el7.x86_64.rpm-bundle.tar
```

## 5.安装mysql的 server、client、common、libs、lib-compat

依次使用如下命令：

```
rpm -ivh --nodeps mysql-community-server-5.7.19-1.el7.x86_64.rpm
rpm -ivh --nodeps mysql-community-client-5.7.19-1.el7.x86_64.rpm
rpm -ivh mysql-community-common-5.7.19-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-5.7.19-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-compat-5.7.19-1.el7.x86_64.rpm
```

## 6.查看mysql的服务是否启动

```
systemctl status mysqld
```

如下则为未启动状态  **Active: inactive (dead)**

```
[root@bigData111 software]# systemctl status mysqld
● mysqld.service - MySQL Server
   Loaded: loaded (/usr/lib/systemd/system/mysqld.service; enabled; vendor preset: disabled)
   Active: inactive (dead)
     Docs: man:mysqld(8)
           http://dev.mysql.com/doc/refman/en/using-systemd.html
```

启动mysql服务：

```
systemctl start mysqld
```

再次检查mysql的服务是否启动

```
systemctl status mysqld
```

如下则为已启动状态：**Active: active (running) since Sun 2019-09-29 15:53:20 CST; 4s ago**

```
[root@bigData111 software]# systemctl status mysqld
● mysqld.service - MySQL Server
   Loaded: loaded (/usr/lib/systemd/system/mysqld.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2019-09-29 15:53:20 CST; 4s ago
     Docs: man:mysqld(8)
           http://dev.mysql.com/doc/refman/en/using-systemd.html
  Process: 1898 ExecStart=/usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid $MYSQLD_OPTS (code=exited, status=0/SUCCESS)
  Process: 1825 ExecStartPre=/usr/bin/mysqld_pre_systemd (code=exited, status=0/SUCCESS)
 Main PID: 1901 (mysqld)
   CGroup: /system.slice/mysqld.service
           └─1901 /usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid

Sep 29 15:53:13 bigData111 systemd[1]: Starting MySQL Server...
Sep 29 15:53:20 bigData111 systemd[1]: Started MySQL Server.
```

## 7.查看默认生成的密码

```
cat /var/log/mysqld.log | grep password
```

如下：

```
[root@bigData111 software]# cat /var/log/mysqld.log | grep password
2019-09-29T07:53:15.860473Z 1 [Note] A temporary password is generated for root@localhost: 3g:ieXDlZFjg
```

## 8.登录mysql服务

```
mysql -uroot -p   然后粘贴上密码
```

注意：-p后面直接加密码，不要有空格

## 9.修改mysql密码规则

目的：更改mysql登录密码

注意事项：以下修改为临时修改，但是利用更改后规则设置的密码是永久的。

1. 密码强度检查等级，0/LOW、1/MEDIUM、2/STRONG

   ```
   mysql>  set global validate_password_policy=0;
   Query OK, 0 rows affected (0.00 sec)
   ```

2. 密码至少要包含的小写字母个数和大写字母个数

   ```
   mysql> set global validate_password_mixed_case_count=0;
   Query OK, 0 rows affected (0.00 sec)
   ```

3. 密码至少要包含的数字个数 

   ```
   mysql> set global validate_password_number_count=3;
   Query OK, 0 rows affected (0.00 sec)
   ```

4. 密码至少要包含的特殊字符数

   ```
   mysql> set global validate_password_special_char_count=0;
   Query OK, 0 rows affected (0.00 sec)
   ```

5. 密码最小长度，参数默认为8，

   ```
   mysql> set global validate_password_length=3;
   Query OK, 0 rows affected (0.00 sec)
   ```

6. 验证是否临时修改成功

   ```
   mysql> SHOW VARIABLES LIKE 'validate_password%';
   +--------------------------------------+-------+
   | Variable_name                        | Value |
   +--------------------------------------+-------+
   | validate_password_check_user_name    | OFF   |
   | validate_password_dictionary_file    |       |
   | validate_password_length             | 3     |
   | validate_password_mixed_case_count   | 0     |
   | validate_password_number_count       | 3     |
   | validate_password_policy             | LOW   |
   | validate_password_special_char_count | 0     |
   +--------------------------------------+-------+
   7 rows in set (0.00 sec)
   ```

   

## 10.修改mysql登录密码为000000

```
mysql> SET PASSWORD FOR 'root'@'localhost' = PASSWORD('000000');
Query OK, 0 rows affected, 1 warning (0.00 sec)
```

## 11.修改远程登录权限

1. 查询当前user表内root的登录权限：

   ```
   select host,user from mysql.user;
   ```

   ```
   mysql> select host,user from user;
   +-----------+---------------+
   | host      | user          |
   +-----------+---------------+
   | localhost | mysql.session |
   | localhost | mysql.sys     |
   | localhost | root          |
   +-----------+---------------+
   3 rows in set (0.00 sec)
   ```

2. 修改权限为所有%：

   ```
   update mysql.user set host = '%' where user = 'root';
   ```

   ```
   mysql> update mysql.user set host = '%' where user = 'root';
   Query OK, 1 row affected (0.00 sec)
   Rows matched: 1  Changed: 1  Warnings: 0
   
   mysql> select host,user from user;
   +-----------+---------------+
   | host      | user          |
   +-----------+---------------+
   | %         | root          |
   | localhost | mysql.session |
   | localhost | mysql.sys     |
   +-----------+---------------+
   3 rows in set (0.00 sec)
   ```

3. 刷新缓存：

   ```
   flush privileges;
   ```

   ```
   mysql> flush privileges;
   Query OK, 0 rows affected (0.00 sec)
   ```

   