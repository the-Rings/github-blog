---
title: Docker镜像安装
date: 2022-01-22 16:22:12
categories:
- Docker
- Vagrant
---

## 安装MySQL
1. 登录docker hub > 搜索mysql
```shell
sudo docker pull mysql:5.7

# sudo docker images 查看本地镜像
```
2. 启动容shell器，并配置端口映射、挂载文件
```shell
sudo docker run -p 3306:3306 --name mysql \
-v /mydata/mysql/log:/var/log/mysql \
-v /mydata/mysql/data:/var/lib/mysql \
-v /mydata/mysql/conf:/etc/mysql \
-e MYSQL_ROOT_PASSWORD=root \
-d mysql:5.7
```
注：
-p 是将容器的3306映射到主机的3306
-v /mydata/mysql/log:/var/log/mysql 将日志文件挂载到主机
-e 初始化root用户密码

3. 查看容器启动情况
```shell
sudo docker ps
```
4. 进入docker容器内部查看
```shell
sudo docker exec -it [container-id] /bin/bash
```
5. mysql字符编码配置
进入主机挂载mysql的文件目录
```shell
cd /mydata/mysql/conf
vim my.cnf

[mysqld]
init_connect='SET NAMES utf8mb4'
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci
```
6. Always Start
```shell
sudo docker update mysql --restart=always
# 设置容器的开机启动
```


## 安装Redis
1. 登录docker hub > 搜索reids
```shell
sudo docker pull redis

# sudo docker images 查看本地镜像
```
2. 启动实例
```shell
sudo mkdir -p /mydata/redis/conf/
sudo touch /mydata/redis/conf/redis.conf

sudo docker run -p 6379:6379 --name redis \
-v /mydata/redis/data:/data \
-v /mydata/redis/conf/redis.conf:/etc/redis/redis.conf \
-d redis redis-server /etc/redis/redis.conf
```
redis配置文件描述：
https://raw.githubusercontent.com/antirez/redis/4.0/redis.conf

3. 验证
不再进入/bin/bash，直接进入redis-cli命令行即可
```
sudo docker exec -it redis redis-cli
```
4. Always Start
```shell
sudo docker update redis --restart=always
# 设置容器的开机启动
```


## 安装Elasticsearch
1. 下载安装
```shell
sudo docker pull elasticsearch:7.4.2
sudo docker pull kibana:7.4.2
```
2. 创建文件
```shell
sudo mkdir -p /mydata/elasticsearch/config
cd /mydata/elasticsearch/config
touch elasticsearch.yml
sudo mkdir -p /mydata/elasticsearch/data
duso mkdir -p /mydata/elasticsearch/plugins
```
3. 赋权限
```shell
sudo chmod -R /mydata/elasticsearch/
```
4. 配置文件写入
```shell
sudo echo "http.host: 0.0.0.0" >> /mydata/elasticsearch/config/elasticsearch.yml
```
5. 启动
```shell
sudo docker run --name elasticsearch -p 9200:9200 -p 9300:9300 \
-e "discovery.type=single-node" \
-e ES_JAVA_OPTS="-Xms64m -Xmx128m" \ #设置内存大小占用64~128M（很重要）
-v /mydata/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticserach.yml \
-v /mydata/elasticsearch/data:/usr/share/elasticsearch/data \ # -v 挂载文件，这样只用修改宿主机的文件，就可以在容器中得到应用
-v /mydata/elasticsearch/plugins:/usr/share/elasticsearch/plugins \
-d elasticsearch:7.4.2
```
6. Always Start
```shell
sudo docker update elasticsearch --restart=always
# 设置容器的开机启动
```


## 安装Kibaba
1. 下载
```shell
sudo docker pull kibana:7.4.2
sudo docker run --name kibana -e ELASTICSEARCH_HOSTS=http://192.168.56.10:9200 -p 5601:5601 -d kibana:7.4.2
```
2. Always Start
```shell
sudo docker update kibana --restart=always
# 设置容器的开机启动
```


## 安装Nginx
1. 下载
```
sudo docker pull nginx:1.20.2
```
2. 试运行并复制出配置
```shell
sudo docker run -p 80:80 --name nginx -d nginx:1.20.2
sudo mkdir -p /mydata/nginx/conf
# 复制nginx的默认配置到mydata文件下
sudo docker container cp nginx:/etc/nginx/. /mydata/nginx/conf/
# 停止删除nginx容器
sudo docker stop nginx
sudo docker rm nginx
```
3. 配置启动nginx
```shell
sudo docker run -p 80:80 --name nginx \
-v /mydata/nginx/html:/usr/share/nginx/html \
-v /mydata/nginx/logs:/var/log/nginx \
-v /mydata/nginx/conf:/etc/nginx \
-d nginx:1.20.2
```
4. Always Start
```shell
sudo docker update nginx --restart=always
# 设置容器的开机启动
```


## 安装Nacos
1. 下载
```shell
sudo docker pull nacos/nacos-server:2.0.3
```
nacos版本要与spring-cloud-alibaba/spring-cloud/spring-boot的版本是兼容的，可以查看https://github.com/alibaba/spring-cloud-alibaba/wiki/版本说明
2. 创建文件夹
```shell
mkdir -p /mydata/nacos/logs
mkdir -p /mydata/nacos/init.d
```
3. 启动
```shell
sudo docker  run --name nacos \
-p 8848:8848 \
-p 9848:9848 \
-p 9849:9849 \
--privileged=true \
--restart=always \
-e JVM_XMS=256m \
-e JVM_XMX=256m \
-e MODE=standalone \
-v /mydata/nacos/logs:/mydata/nacos/logs \
-v /mydata/nacos/init.d/custom.properties:/mydata/nacos/init.d/custom.properties \
-d nacos/nacos-server:2.0.3
```
其中`-e PREFER_HOST_MODE=hostname`可以将其配置为支持域名模式，默认是ip模式
8848端口用来HTTP服务，9848和9849用来服务发现和注册，所以也是需要映射到宿主机的端口的
4. Always Start
```shell
sudo docker update nacos --restart=always
# 设置容器的开机启动
```

## 安装mongodb
1. sudo docker pull mongo:5.0.22
2. ```shell
sudo docker run -p 27017:27017 --name mongodb \
 -v /mydata/mongo/data:/data/db \
 -d mongo:5.0.22
```
3. sudo docker update mongodb --restart=always
4. ```JavaScript
db.createUser(
	{
		user: "admin",
		pwd: "123456",
		roles: [{role: "root", db: "admin"}]
	}
)
```
## 安装Kafka
1. 安装zookeepr


## 安装Zipkin




