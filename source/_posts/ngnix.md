---
title: 记录一次Nginx的严重错误
date: 2022-12-24 17:51:55
tags: Nginx
---

nginx监听80端口代理网关应用，对应的nginx配置conf.d/gateway.conf
```nginx
server {
    listen       80;
    server_name  192.168.56.10;

    location / {
        proxy_set_header Host $host;  #[1]
        proxy_pass http://192.168.100.2:88;
    }

}
```
spring-cloud-gateway应用的配置如下
```yaml
spring:
  application:
    name: foomall-gateway
  cloud:
    gateway:
      routes:
        - id: foomall_host_route
          uri: lb://foomall-product
          predicates:
            - Host=192.168.56.10
```
此时，在地址栏输入`http://192.168.56.10`后显示404，从配置得知gateway通过Host参数来进行路由，也就是HTTP Header中的Host参数，说明其未生效。
原因就是Nginx在代理`http://192.168.56.10`请求时，将Host参数弄丢了，所以在`[1]`的位置加上`proxy_set_header Host $host;`，除了这个参数，Nginx在代理的过程中会丢失很多参数
