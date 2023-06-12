---
title: 实时交易数据
date: 2023-02-21 21:40:08
categories: 
- WebSocket
---

最近遇到一个查看实时交易数据的问题。数据来自于上游的Kafka，然后在服务端经过些许处理，发送到WebSocket，前端页面实时刷新，这就是整个链路。
Kafka像这样：
```Java
public KafkaDealConsumer {

	@KafkaLintener(topic = "${topic}", containerFactory = "kafkaListenerFactory")
	public void sendDealMessage(ConsumerRecord<String, String> record) {
		String source = record.value();
		doSendDealMessage(source);
	}

	public void doSendDealMessage(String source) {
		String message;
		// 
		// handle the record ...
		//
		dealWsServer.sendInfoToAll(message);
	}
}

public class DealWsServer {
	private final Object lockObj = new Object();
	ConcurrentHashMap<String, DealWsServer> webSocketSet = new ConcurrentHashMap<>();
	public void sendMessage(WebSocketServer wsServer, String message) {
		wsServer.seesion.getBasicRemote().sendText(message);
	}

	@OnOpen
	public void onOpen(@PathParam(value = "sessionId") String sessoinId, Session session) {
		this.sessionId = sessionId;
		this.session = session;
		webSocketSet.put(seesionId, this);
		DealWsServer.onlineCount.getAndIncrement();
	}

	@OnMessage
	public void onMessage(String requestMessage) {
		boolean inited = false;
		// some codes ...
		// 这里判断是否是初次建立连接，或者发送的是心跳
		// 
		if (!inited) {
			init();
			return;
		}
		// 如果是前端发送过来的参数信息，那么后端进行查询并返回结果
		synchronized (lockObj) {
			RequestDto request = JSONObject.parseObject(requestMessage);
			String message;
			// 处理request并生成message;
			sendMessage(this, message);
		}
	}

	@OnClose
	public void onClose() {
		webSocketSet.remove(sessionId);
		DealWsServer.onlineCount.getAndDecrement();
	}

	@OnError
	public void onError(Session sesson, Throwable error) {
		// ...
	}

	public void sendInfoToAll(String message) {
		// 遍历每一个session
		for (Map.Entry<String, DealWsServer> entry : webSocketSet.entrySet()) {
			DealWsServer dealWsServer = entry.getValue();
			synchronized (dealWsServer.lockObj) {
				sendMessage(wsServer, message);
			}
		}
	}
}
```
