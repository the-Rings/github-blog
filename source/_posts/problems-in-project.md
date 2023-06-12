---
title: 项目中遇到的问题
date: 2023-05-16 22:11:28
tags:
---

问题1：交易记录的表，交易id，日期，金额，买卖方向，债券代码，债券类型。但是某个债券可能会从属于多个债券类型。这时，如何进行分组统计每种债券类型的交易量？比如，我要查询2023-02-23日期，PFB、CGB和RVB对应的交易量分别是多少，由于债券类型之间有交集，所以我们不能简单的`select sum(qty) from table1 where biz_date = '2023-02-03' group by bond_type`。那么我在设计表的时候就要注意了，多个债券类型如何存储，使用多个字段还是，用MySQL的json还是别的？
首先，我们存储的时候，债券类型字段，采用逗号分隔即可，比如："PFB,CGB,RVB"
|order_id|biz_date  |bond_code|qty |bond_type|
|1       |2023-02-23|2012.IB  |100 |PFB,CGB,RVB|
|2       |2023-02-23|2013.IB  |103 |GB,CGB|
|3       |2023-02-23|2023.IB  |190 |LGB,CGB|
|4       |2023-02-23|2079.IB  |212 |PFB,RVB|
使用sql查询时，查询多个债券类型时，无法使用`bond_type in ('PFB', 'RVB')`可以使用MySQL中的`find_is_set`函数
```SQL
-- 
select * from table1 where (find_in_set('PFB', bond_type) or find_in_set('RVB', bond_type))
```
接下来我们需要借助代码，在程序里，将多个债券类型的同一个订单计算多次，准确地计算出每种类型的交易金额


问题2：文本框模糊检索，输入名称的同时，给出提示，比如输入当前的债券发行人，比如输入“银行”，会检索出“上海银行”，“工商银行股份有限公司”等，不能检索出“银河车行”。
MySQL中对应的表字段建立全文索引，MySQL5.7支持中文的全文索引，使用全文索引的语法进行查询 `MATCH(issuer_name) AGAINST( '银行' IN BOOLEAN MODE)`


问题3：MySQL的一个表有20个字段，上百万的数据，存储的是交易数据，包括：交易id，日期，金额，买卖方向，债券代码，债券类型，剩余期限。如何在几百毫秒内，计算出最近15天，各种债券类型，各种剩余期限的净成交序列。
1. 按照某种类型拆分表
2. 使用MySQL分区表
3. 使用覆盖索引，将所需要查询的字段都罗列出来，一起建立索引，查询时只能包含覆盖索引中的字段，这样索引中的值就可以满足查询要求，不用根据主键回表


问题4：40万条数据，进行for循环，耗时10ms以内，如果在里边加上`LocalDate.format(DateTimeFormatter.ISO_DATE)`，为什么耗时飙升到了2000ms？
```Java
// 耗时2000+ms
for (Object obj: objList) {
	LocalDate now = LocalDate.now()
	now.format(DateTimeFormatter.ISO_DATE)
	// nothing ...
}
```
这里要使用`LocalDate.format(DateTimeFormatter.ofPattern("yyyy-MM-dd"))`就大幅提高了效率，结果就成了100+ms


问题5：`@KafkaLinstner`监听并消费消息，经过计算发送websocket消息到客户端页面。后端有两个实例，同一个账户两个浏览器登录，为什么两个客户端收到的消息不一致？
1. 接受到新的消息，经过一些列逻辑判断（是否是新订单，是否旧的订单新发生了变化），将变化更新到Redis集群中，并发送通知。
2. 发送通知时，会遍历当前服务器的所有的session发送通知，并不是广播
3. 两台服务器都收到了kafka消息，但是必然有先后顺序，就会导致先收到消息的一台服务器，进行了Redis集群中订单的update，第二台服务器收到消息后查询Redis发现订单无变化，那么第二台服务器就不会对当前服务器上的建立websocket连接的用户发送通知，然而第一台服务器上的用户却收到了变化，所以两个客户端上显示数据不一致
解决方案是：同时使用本地缓存来存储订单信息，同时也在Redis中存储订单信息，如果服务器重启，在重启过程中，读取Redis并更新本地缓存。
技术点：
- 对每台服务器配置不同的消费者组ID
- 服务器启动过程中，运行更新本地缓存的代码


