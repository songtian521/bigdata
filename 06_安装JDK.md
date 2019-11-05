# 06_安装JDK

## 1.卸载现有JDK

1. 查询是否安装java

   ```
   rpm -qa | grep java
   ```

2. 如果已安装的jdk版本低于1.7，卸载该jdk

   ```
    rpm -e 软件包名字
   ```

## 2.在opt目录下创建两个子文件

```
mkdir /opt/module /opt/software
```

作用：一个存放安装包，一个是安装目录

注：使用命令`yum install lrzsz`可以直接将文件拖入Xshell中上传至虚拟机

## 3.解压JDK至/opt/module目录下

```
tar -zxvf jdk-8u144-linux-x64.tar.gz -C /opt/module/
```

## 4.配置JDK环境变量

1. 进入配置文件

   ```
   vi /etc/profile
   ```

2. 文件的末尾添加如下内容

   ```
   export JAVA_HOME=/opt/module/jdk1.8.0_144
   export PATH=$PATH:$JAVA_HOME/bin
   ```

3. 格式化配置文件

   ```
   source /etc/profile
   ```

4. 测试JDK是否安装成功

   ```
   java -version
   java version "1.8.0_144"
   ```

   ```
   [root@bigData111 module]# java -version
   java version "1.8.0_144"
   Java(TM) SE Runtime Environment (build 1.8.0_144-b01)
   Java HotSpot(TM) 64-Bit Server VM (build 25.144-b01, mixed mode)
   ```