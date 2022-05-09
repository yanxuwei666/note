# 前提

- 本文档基于 Centos7 系统进行配置。
- 若无其他必要，均使用 docker 或 docker-compose 进行配置。

# Linux环境安装

## Docker安装

```sh
# 1、卸载旧版本
yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine

# 2、设置稳定的存储库，使用阿里云的
yum install -y yum-utils
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 3、安装最新版本docker引擎和容器
yum -y install docker-ce docker-ce-cli containerd.io

# 4、docker完成后查看docker版本
docker -v

# 5、设置docker自启动
systemctl start docker
systemctl enable docker

# 6、配置阿里云镜像加速器
mkdir -p /etc/docker
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://pu5sdck7.mirror.aliyuncs.com"]
}
EOF

# 7、设置docker默认存储目录，默认为/var/lib/docker，修改到home目录下（我这里没修改）
vim /usr/lib/systemd/system/docker.service

ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock –graph /home/jack/docker/home/

# 8、重启docker，查看详细信息
systemctl restart docker
docker info

# 9、其他命令，设置容器自启动
docker update --restart=always 镜像ID
```

## Docker-Compose安装

```sh
# 1、安装命令如下
wget -c  https://github.com/docker/compose/releases/download/1.25.5/docker-compose-Linux-x86_64
mv docker-compose-Linux-x86_64  /usr/bin/docker-compose
chmod  a+x /usr/bin/docker-compose 
docker-compose  --version

# 2、常用命令如下
# 在后台所有启动服务，使用-f 指定使用的Compose模板文件，默认为docker-compose.yml，可以多次指定
docker-compose up -d
docker-compose -f docker-compose.yml up -d
# 列出项目中目前的所有容器
docker-compose ps
# 停止正在运行的容器
docker-compose stop
```

# 数据库环境安装

## MySQL安装

```sh
# 1、拉取mysql镜像，不指定版本默认最新
docker pull mysql 
docker pull mysql:5.7

# 2、创建mysql容器，指定密码123456，设置root用户权限，设置开机自启动
docker run -d -p 3306:3306 --privileged=true --restart=always  -e MYSQL_ROOT_PASSWORD=123456 --name docker_mysql_latest mysql:latest --character-set-server=utf8mb4 --collation-server=utf8mb4_general_ci

# 3、进入mysql容器里
docker exec -it docker_mysql_8 bash
mysql -uroot -p123456
```

## Oracle安装

```sh
# 1、拉取oracle镜像
docker pull registry.cn-hangzhou.aliyuncs.com/helowin/oracle_11g

# 2、创建oracle容器
docker run -d -p 1521:1521 --restart=always --name docker_oracle_11g registry.cn-hangzhou.aliyuncs.com/helowin/oracle_11g

# 3、进行oracle容器里,设置oracle用户及口令
docker exec -it docker_oracle_11g /bin/bash 
cd /home/oracle
source .bash_profile
sqlplus /nolog
conn /as sysdba
alter user system identified by 123;

# 4、通过navicat连接oracle，数据库名为helowin。
```

## Redis安装

```sh
# 1、拉取redis镜像
docker pull redis

# 2、创建redis容器
docker run -d -p 6379:6379 --restart=always --name docker_redis_latest redis:latest --requirepass "12345"

# 3、进入redis客户端
docker exec -it docker_redis_latest redis-cli

# 4、认证密码
auth 12345
```

## MongoDB安装

```sh
# 1、拉取镜像 
docker pull mongo:latest

# 2、创建mongodb实例
docker run -itd --name docker_mongo_latest --restart=always -p 27017:27017 mongo:latest --auth	

# 3、进入mongodb容器内部
docker exec -it docker_mongo_latest mongo admin

# 4、创建mongodb用户
user admin
db.createUser({ user:'admin',pwd:'12345',roles:[ { role:'userAdminAnyDatabase', db: 'admin'},"readWriteAnyDatabase"]})
db.auth('admin', '123456')

# 5、连接db
docker exec -it docker_mongo_latest mongo admin
db.auth('admin', '123456')
show dbs
```

# Java基础环境安装

## JDK安装

```sh
# 1、在/usr/local/下创建java文件夹
mkdir /usr/local/java && cd /usr/local/java

# 2、将JDK安装包解压到当前目录，推荐JDK11
tar -zxvf jdk-11.0.8_linux-x64_bin.tar.gz

# 3、在线安装JDK11，默认安装在/usr/lib/jvm文件下
yum search java-11-openjdk	
yum install -y java-11-openjdk java-11-openjdk-devel

# 4、配置jdk环境变量
vim /etc/profile

export JAVA_HOME=/usr/local/java/jdk11
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin

# 5、使文件生效
source /etc/profile

# 6、jdk版本验证
java -version
```

## Maven安装

```sh
# 1、下载maven包
wget https://archive.apache.org/dist/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.tar.gz

# 2、将Maven安装包放到/opt/maven目录下，并解压
tar -zxvf apache-maven-3.6.3-bin.tar.gz

# 3、修改/opt/maven/apache-maven-3.6.3/conf/settings.xml，配置maven加速镜像源
<mirror>
    <id>alimaven</id>
    <name>aliyun maven</name>
    <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
    <mirrorOf>central</mirrorOf>
</mirror>

# 4、配置maven环境变量，并刷新使其生效 /etc/profile
export MAVEN_HOME=/opt/maven/apache-maven-3.6.3
export PATH=$MAVEN_HOME/bin:$PATH

# 5、查看maven版本
mvn -v
```

## Nginx安装

```sh
# 1、拉取nginx镜像
docker pull nginx:latest

# 2、随便启动一个nginx实例
docker run -p 80:80 --name nginx -d nginx:latest

# 3、将容器内的配置文件拷贝到/opt/nginx/conf目录下，并创建html和logs目录
cd /opt/nginx/
docker container cp nginx:/etc/nginx .
mkdir html
mkdir logs

# 4、在html目录中定义index.html

# 5、重新创建一个新的nginx实例，指定本地文件目录
docker run -p 80:80 --name docker_nginx_latest --restart=always -v /opt/nginx/html:/usr/share/nginx/html -v /opt/nginx/logs:/var/log/nginx -v /opt/nginx/conf:/etc/nginx -d nginx:latest
```

## Tomcat安装

```sh
# 1、拉取Tomcat镜像
docker pull tomcat:7-jre7

# 2、创建并运行容器
docker run -di --name=tomcat7 -p 9000:8080 --restart=always tomcat:7-jre7 -v /usr/local/webapps:/usr/local/tomcat/webapps
```





