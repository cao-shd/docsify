# JAVA 安装

## 环境检查

运行命令

```shell
java -version
```

显示结果如下，证明已安装过

```shell
openjdk version "1.8.0_161"
OpenJdk Runtime Environment (build 1.8.0_161-b14)
OpenJdk 64-Bit Server VM (build 25.161-b14, mixed mode)
```

如果对 Java 环境无特殊要求，可结束后续操作

## 卸载 OpenJdk

检查 OpenJdk 安装包，运行如下命令：

```shell
rpm -qa | grep java
```

系统显示结果如下：

```shell
java-1.8.0-openjdk-headless-1.8.0.161-2.b14.el7.x86_64
tzdata-java-2018c-1.el7.noarch
python-javapackages-3.4.1-11.el7.noarch
javapackages-tools-3.4.1-11.el7.noarch
java-1.8.0-openjdk-1.8.0.161-2.b14.el7.x86_64
```

卸载OpenJdk，运行如下命令：

```shell
yum remove *openjdk* -y
```

查看卸载情况，再次输入上述检查命令：

```shell
rpm -qa | grep java
```

若显示结果如下，则证明已成功卸载 OpenJdk

```shell
tzdata-java-2018c-1.el7.noarch
python-javapackages-3.4.1-11.el7.noarch
javapackages-tools-3.4.1-11.el7.noarch
```

> 以上内容因不同的发型版本显示结果会有些许差异， 请自行判断

## 安装Jdk

下载 JDK 安装包 上传到指定目录


创建路径

```shell
mkdir -p /usr/local/java
```

解压安装

```shell
tar -zxvf jdk-8u161-linux-x64.tar.gz -C /usr/local/java
cd /usr/local/java && mv jdk1.8.0_161 8u161
```

设置环境变量

```shell
vi /etc/profile

export JAVA_HOME=/usr/local/java/8u341
export PATH=$PATH:$JAVA_HOME/bin
```

刷新环境变量

```shell
source /etc/profile
```

问题及解决方法：

```shell
# 执行 source 命令可能出现如下问题
# sudo: source: command not found

# 解决办法 注意使用拥有 root 权限的用户
# 执行命令
locate source /etc/profile

# 显示如下
locate: can not stat () `/var/lib/mlocate/mlocate.db': No such file or directory

# 继续执行
updatedb
```

检查是否安装成功。

```shell
java -version
```

查看环境变量

```shell
echo $JAVA_HOME
```
