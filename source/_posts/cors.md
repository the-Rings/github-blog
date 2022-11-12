---
title: Cross-Origin Resource Sharing
date: 2022-06-22 16:22:12
tags:
- CORS
---

所有跨域问题一文讲清楚

https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS
https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CORS

> 跨源资源共享 (CORS)（或通俗地译为跨域资源共享）是一种基于 HTTP 头的机制，该机制通过允许服务器标示除了它自己以外的其它 origin（域，协议和端口），使得浏览器允许这些 origin 访问加载自己的资源。跨源资源共享还通过一种机制来检查服务器是否会允许要发送的真实请求，该机制通过浏览器发起一个到服务器托管的跨源资源的"预检(preflight)"请求。在预检中，浏览器发送的头中标示有 HTTP 方法和真实请求中会用到的头。


> 跨源 HTTP 请求的一个例子：运行在 https://domain-a.com 的 JavaScript 代码使用 XMLHttpRequest 来发起一个到 https://domain-b.com/data.json 的请求。

> 出于安全性，浏览器限制脚本内发起的跨源 HTTP 请求。 例如，XMLHttpRequest 和 Fetch API 遵循同源策略。这意味着使用这些 API 的 Web 应用程序只能从加载应用程序的同一个域请求 HTTP 资源，除非响应报文包含了正确 CORS 响应头。


注：preflight(预检)请求是一个OPTIONS方法的请求，如果OPTIONS请求得到了正确的响应头，那么浏览器就认为可以跨域。所以解决跨域问题有两种思路，一种是部署在同一个域，第二种是让OPTIONS请求带上“正确”的响应头。通常大家都采用第二种方式，比如，在后端gateway中配置corsFilter，让所有的请求都都加上正确的响应头。

