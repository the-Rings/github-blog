---
title: network-http-message
date: 2022-07-31 22:50:04
categories:
- Network
---

##### Web的应用层协议是HTTP(HyperText Transfer Protocol, HTTP)，HTTP使用TCP作为支撑它的传输层协议，定义在`[RFC 1945]`和`[RFC 2616]`

##### HTTP定义了浏览器和Web服务器之间的传输的message(报文)格式和序列，HTTP服务器不保存关于客户端的任何信息，它一种无状态协议(stateless protocol)
1. HTTP协议是在应用程序中实现的

##### 在HTTP/1.1中，每个请求和响应是经一个单独的TCP连接发送，还是所有的请求及响应经相同的TCP连接发送，采用前一种，被称为non-persistent connection(非持续连接)，后一种被称为persistent connection(持续连接)
1. 非持续连接，必须为每一个请求的对象建立和维护一个全新的TCP连接
2. 在采用HTTP/1.1持续连接(Connection:keep-alive)的情况下，服务器在发送响应后保持该TCP连接打开，后续的请求和响应报文能够通过相同的连接进行传送，如果一个连接经过一定时间间隔(一个可配置的超时间隔)仍未被使用，HTTP服务器就关闭该连接
3. HTTP/1.1的持续连接与WebSocket协议是完全不同的，前者是在HTTP/1.0的基础上进行了优化，减少连接的建立次数来提高性能，但仍然是请求-响应式的。而WebSocket是一种全双工通信，可以进行双向的实时通信，不需要每次都发送HTTP请求和响应的繁琐报文

##### HTTP报文格式的分为请求报文和相应报文
1. 请求报文的格式：
- request line
- header line
- entity body
```http
GET /somedir/page.html HTTP/1.1
Host: www.some.com
Connection: close8724
User-agent: Mozilia/5.0
Accept-language: fr
```
2. 响应报文的格式：
- status line
- header line
- entiry body
```http
HTTP/1.1 200 OK
Connection: close
Date: Tue, 18 Aug 2015 15:44:01 GMT
Server: Apache/2.2.3 (CentOS)
Last-Modified: Tue, 18 Aug 2015 15:11:03 GMT
Content-Length: 6821
Content-Type: text/html

(data data data ...)
```
