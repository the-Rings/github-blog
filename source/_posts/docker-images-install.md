---
title: 配置安装docker以及镜像
date: 2022-01-22 16:22:12
tags:
- Docker
---

# Docker install
下载路径https://docs.docker.com > Get Docker > Docker Engine > Install On CentOS
读文档只选择重要的步骤：
1. 删除旧docker
```shell
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```
2. 配置repo
```shell
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```
3. 安装docker
```shell
sudo yum install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```
4. 启动docker
```shell
sudo systemctl start docker
# 配置开机启动
sudo systemctl enable docker
```
5. 配置阿里云docker镜像加速
登录阿里云 > 控制台 > 产品与服务 > 容器服务 > 容器镜像服务 > 镜像工具 > 镜像加速器 > CentOS > 配置镜像加速


## 安装mysql
1. 登录docker hub > 搜索mysql
```shell
docker pull mysql:5.7

# sudo docker images 查看本地镜像
```
2. 启动容器，并配置端口映射、挂载文件
```
docker run -p 3306:3306 --name mysql \
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
```
sudo docker ps
```
4. 进入docker容器内部查看
```
sudo docker exec -it [container-id] /bin/bash
```
5. mysql字符编码配置
进入主机挂载mysql的文件目录
```
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


## 安装redis
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
```
sudo docker pull kibana:7.4.2
sudo docker run --name kibana -e ELASTICSEARCH_HOSTS=http://192.168.56.10:9200 -p 5601:5601 -d kibana:7.4.2
```
2. Always Start
```shell
sudo docker update kibana --restart=always
# 设置容器的开机启动
```
