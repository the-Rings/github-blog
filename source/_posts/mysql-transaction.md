---
title: MySQL事务
date: 2022-11-29 21:30:22
tags:
---

# 事务隔离级别
MySQL中事务的默认隔离级别是Repeatable read（可重复读），开发中如果遇到调试的问题时，要看当前未提交的数据时，我们可以在图形客户端（如：DBeaver）中打开一个会话，在这个会话中临时修改当前的数据库隔离级别：
```SQL
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
```
