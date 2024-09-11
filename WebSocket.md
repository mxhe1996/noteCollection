# WebSocket
##  介绍
由于http是一种无状态的、单向的应用层,采用了请求/响应模型.只能由客户端发起,服务端对请求作出回答.
所有存在一个弊端: Http协议无法实现服务器主动向客户端发起请求
**那么如果服务端存在连续的变化,那么客户端要获取通知是非常麻烦的**,大多数的web应用程序频繁使用异步调用Ajax实现长轮询,导致效率过低,浪费资源



#### 握手阶段
客户端-->服务端,服务端返回一个101协议
握手阶段基于**http**协议:

```xml
    GET ws://localhost/chat HTTP/1.1
    Host: localhost
    Upgrade: webSocket # 协议升级为webSocket
    sec-WebSocket-Key: # 客户端采用base64的24位随机序列,服务端接收客户端http协议升级的证明
    sec-WebSocket-Extensions:
    sec-WebSocket-Version:
```

来自服务器的握手:

```xml
Http/1.1 101 switching Protocols
Upgrade: websocket   # 升级为websocket
Connection: Upgrade  # 标识这是一个协议升级请求
sec-WebSocket-Accept: # 针对客户端发起的sec-websocket-Key, 响应一个对应加密的信息头作为应答
sec-Websocket-Extensions
```



### 数据传输/交互阶段
可以由任意一端率先发起数据

#### 客户端实现

| 事件    | 处理程序   | 描述                     |
| ------- | ---------- | ------------------------ |
| open    | .onopen    | 连接建立时触发           |
| message | .onmessage | 接收服务器端数据时候触发 |
| error   | .onerror   |                          |
| close   | .onclose   |                          |



| 方法     | 描述     |
| -------- | -------- |
| `send()` | 发送信息 |


#### 服务端实现
Java Websocket应用由一系列的WebSokcetEndPoint组成
EndPoint是一个java类,代表WebSocket链接的一端,对服务端,可以视为具体websocket消息的接口.每个EndPoint与一个客户端对应

两种实现方式:
+ 编程式,继承EndPointlei
+ 注解式,定义个POJO,并添加@ServerEndpoint相关注解

需要标记`@Component`,`@ServerEndpoint`注解


**服务端如何接受客户端发送的数据**
编程式,通过Session添加MessageHandler消息处理器接收消息
注解式,则通过`@OnMessage`注解指定接收消息的方法,并实现转发

**服务端如何推送数据给客户端**
发送消息由`RemoteEndpoint`完成,实例由session维护,根据使用情况,`Session.getBasicRemote/getAsyncRemote`获取同步/异步消息发送的实例,然后调用sendxxx(),发送消息

编写configuration对象
```java
@configuration
public class WebSocketConfig{
	@Bean // 注入该ServerEndpointExporter bean对象,自动注册使用了@ServerEndpoint注解的bean
	public ServerEndpointExporter serverEndpointExporter(){
		return new ServerEndpointExporter();
	}
	//**<u>在拦截器里面,需要将websocket的url放开</u>**
}
```

在Websocket类
```java
@Component
@ServerEndpoint("")
public class SocketServer{
  // 该Map主要存储所有(在线)的 姓名-客户端 信息,static公用,如果系统本身采用分布式的架构,可以考虑将该Map存储在redis中
	private static HashMap<String, Session> sessionMap = new CurrentHashMap<>;
	// 为每个客户端存储一个session对象,用于发送消息(可不用)
	private session;
  // 声明一个httpSession类,在该对象中存储用户名(可不用)
	private HttpSession httpSession;

}
```

### 消息格式(不唯一)
客户端发送给服务器的格式:
```json
{"toName":"","message":"txt"}
```
服务端发给客户端的格式
```json
{"isSystem":true,"fromName":"","message":"txt"}
```

可以生成对应的class对象或者直接按照`JsonObject()`传递