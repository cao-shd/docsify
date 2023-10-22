# MAVEN 安装

## 解压

```shell
tar zvxf apache-maven-3.6.1-bin.tar.gz
```

创建路径

```shell
mkdir -p /usr/local/maven/repo && chmod 777 -R /usr/local/maven/
```

安装

```shell
cp -r apache-maven-3.6.1 /usr/local/maven/3
```

## 配置

编辑配置文件

```shell
vi /usr/local/maven/3/conf/settings.xml
```

修改本地仓库位置

```shell
<localRepository>/usr/local/maven/3/repo</localRepository>
```

修改 maven 镜像 (使用阿里镜像)

```xml
<mirror>
  <id>aliyun</id>
  <name>aliyun maven</name>
  <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
  <mirrorOf>central</mirrorOf>
</mirror>
```

修改默认JDK版本

```shell
    <profile>
      <id>jdk8</id>
      <activation>
        <activeByDefault>true</activeByDefault>
        <jdk>1.8</jdk>
      </activation>
      <properties>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
        <maven.compiler.compilerVersion>1.8</maven.compiler.compilerVersion>
      </properties>
    </profile>

  <activeProfiles>
    <activeProfile>jdk8</activeProfile>
  </activeProfiles>
```

修改环境变量

```shell
vim /etc/profile

# 在最开始添加如下内容
export MAVEN_HOME=/usr/local/maven/3
export PATH=$PATH:$MAVEN_HOME/bin
```

使环境变量立即生效

```shell
source /etc/profile
```

查看是否安装成功

```shell
mvn -v
```

查看 mvn 路径

```shell
echo $MAVEN_HOME
```
