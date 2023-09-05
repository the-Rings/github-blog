---
title: network-socket-program
date: 2022-08-05 17:05:18
categories:
- Network
---


##### Socket编程样例
>TCPClient.py
```python
from socket import *
serverName = 'servername'
serverPort = 12000
clientSocket = socket(AF_INET, SOCK_STREAM)
# 建立连接的过程                                                                            [1]
clientSocket.connect(serverName, serverPort)
sentence = raw_input('input lowercase sentence:')
clientSocket.send(sentence.encode())
modifiedSentence = clientSocket.recv(1024)
print('From Server: ', modifiedSentence.decode())
clientSocket.close();
```

>TCPServer.py
```python
from socket import *
serverName = 'servername'
serverPort = 12000
socketSocket = socket(AF_INET, SOCK_STREAM)
socketSocket.bind('', serverPort)
serverSocket.listen(1)
print('The Server is ready to receive')
while True:
	# server socket调用其accept方法，在服务器中专门创建了一个connection socket由特定客户端使用  [2]
	connectionSocket, addr = serverSocket.accept()
	sentence = connectionSocket.recv(1024).decode()
	capitalizedSentence = sentence.upper()
	connectionSocket.send(capitalizedSentence.encode())
	connectionSocket.close()
```

以上两端代码模拟了传输层socket通信的样例
- [1] 表示三次握手，建立socket连接
- [2] 表示了最原始的"阻塞式IO"处理客户端的消息的过程，这是一种比较原始且低效的做法，高效的做法是*IO多路复用*的做法参照{% post_link network-io-multiplex '图解多路复用' %}
