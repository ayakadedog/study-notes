## 1，WebSocket

### 1.1 WebSocket介绍

WebSocket 是一种网络通信协议。RFC6455 定义了它的通信标准。

WebSocket 是 HTML5 开始提供的一种在单个 TCP 连接上进行全双工通讯的协议。

HTTP 协议是一种无状态的、无连接的、单向的应用层协议。它采用了请求/响应模型。通信请求只能由客户端发起，服务端对请求做出应答处理。

这种通信模型有一个弊端：HTTP 协议无法实现服务器主动向客户端发起消息。

这种单向请求的特点，注定了如果服务器有连续的状态变化，客户端要获知就非常麻烦。大多数 Web 应用程序将通过频繁的异步 AJAX 请求实现长轮询。轮询的效率低，非常浪费资源（因为必须不停连接，或者 HTTP 连接始终打开）。

http协议：

<img src="img/image-20200519150832143.png" alt="image-20200519150832143" style="zoom:70%;" />

websocket协议：

<img src="img/image-20200519150920536.png" alt="image-20200519150920536" style="zoom:67%;" />

### 1.2 websocket协议

本协议有两部分：握手和数据传输。

握手是基于http协议的。

来自客户端的握手看起来像如下形式：

```http
GET ws://localhost/chat HTTP/1.1
Host: localhost
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Extensions: permessage-deflate
Sec-WebSocket-Version: 13
```

来自服务器的握手看起来像如下形式：

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Extensions: permessage-deflate
```

字段说明：

| 头名称                    | 说明                                                         |
| :------------------------ | ------------------------------------------------------------ |
| Connection：Upgrade       | 标识该HTTP请求是一个协议升级请求                             |
| Upgrade: WebSocket        | 协议升级为WebSocket协议                                      |
| Sec-WebSocket-Version: 13 | 客户端支持WebSocket的版本                                    |
| Sec-WebSocket-Key：       | 客户端采用base64编码的24位随机字符序列，服务器接受客户端HTTP协议升级的证明。要求服务端响应一个对应加密的Sec-WebSocket-Accept头信息作为应答 |
| Sec-WebSocket-Extensions  | 协议扩展类型                                                 |



### 1.3 客户端（浏览器）实现

#### 1.3.1 websocket对象

实现 WebSockets 的 Web 浏览器将通过 WebSocket 对象公开所有必需的客户端功能（主要指支持 Html5 的浏览器）。

以下 API 用于创建 WebSocket 对象：

```js
var ws = new WebSocket(url);
```

> 参数url格式说明： ws://ip地址:端口号/资源名称

#### 1.3.2 websocket事件

WebSocket 对象的相关事件

| 事件    | 事件处理程序            | 描述                       |
| ------- | ----------------------- | -------------------------- |
| open    | websocket对象.onopen    | 连接建立时触发             |
| message | websocket对象.onmessage | 客户端接收服务端数据时触发 |
| error   | websocket对象.onerror   | 通信发生错误时触发         |
| close   | websocket对象.onclose   | 连接关闭时触发             |

#### 1.3.3 WebSocket方法

WebSocket 对象的相关方法:

| 方法   | 描述             |
| ------ | ---------------- |
| send() | 使用连接发送数据 |



### 1.4 服务端实现

Tomcat的7.0.5 版本开始支持WebSocket,并且实现了Java WebSocket规范(JSR356)。

Java WebSocket应用由一系列的WebSocketEndpoint组成。Endpoint 是一个java对象，代表WebSocket链接的一端，对于服务端，我们可以视为处理具体WebSocket消息的接口， 就像Servlet之与http请求一样。

我们可以通过两种方式定义Endpoint:

* 第一种是编程式， 即继承类 javax.websocket.Endpoint并实现其方法。 

* 第二种是注解式, 即定义一个POJO, 并添加 @ServerEndpoint相关注解。



Endpoint实例在WebSocket握手时创建，并在客户端与服务端链接过程中有效，最后在链接关闭时结束。在Endpoint接口中明确定义了与其生命周期相关的方法， 规范实现者确保生命周期的各个阶段调用实例的相关方法。生命周期方法如下：

| 方法    | 含义描述                                                     | 注解     |
| ------- | ------------------------------------------------------------ | -------- |
| onClose | 当会话关闭时调用。                                           | @OnClose |
| onOpen  | 当开启一个新的会话时调用, 该方法是客户端与服务端握手成功后调用的方法。 | @OnOpen  |
| onError | 当连接过程中异常时调用。                                     | @OnError |

**服务端如何接收客户端发送的数据呢？**

通过为 Session 添加 MessageHandler 消息处理器来接收消息，当采用注解方式定义Endpoint时，我们还可以通过 @OnMessage 注解指定接收消息的方法。

**服务端如何推送数据给客户端呢？**

发送消息则由 RemoteEndpoint 完成， 其实例由 Session 维护， 根据使用情况， 我们可以通过Session.getBasicRemote 获取同步消息发送的实例 ， 然后调用其 sendXxx()方法就可以发送消息， 可以通过Session.getAsyncRemote 获取异步消息发送实例。

服务端代码：

```java
@ServerEndpoint("/robin")
public class ChatEndPoint {

    private static Set<ChatEndPoint> webSocketSet = new HashSet<>();

    private Session session;

    @OnMessage
    public void onMessage(String message, Session session) throws IOException {
        System.out.println("接收的消息是：" + message);
        System.out.println(session);
        //将消息发送给其他的用户
        for (Chat chat : webSocketSet) {
            if(chat != this) {
                chat.session.getBasicRemote().sendText(message);
            }
        }
    }

    @OnOpen
    public void onOpen(Session session) {
        this.session = session;
        webSocketSet.add(this);
    }

    @OnClose
    public void onClose(Session seesion) {
        System.out.println("连接关闭了。。。");
    }

    @OnError
    public void onError(Session session,Throwable error) {
        System.out.println("出错了。。。。" + error.getMessage());
    }
}
```



## 2 基于WebSocket的网页聊天室

### 2.1 需求

通过 websocket 实现一个简易的聊天室功能 。

1）. 登陆聊天室

<img src="img/image-20200519160300978.png" alt="image-20200519160300978" style="zoom:60%;" />

2）. 登陆之后，进入聊天界面进行聊天

登陆成功后，呈现出以后的效果：

<img src="img/image-20200519160527484.png" alt="image-20200519160527484" style="zoom:50%;" />

当我们想和李四聊天时就可以点击 `好友列表` 中的 `李四`，效果如下：

<img src="img/image-20200519160650665.png" alt="image-20200519160650665" style="zoom:50%;" />

接下来就可以进行聊天了，`张三` 的界面如下：

<img src="img/image-20200519161039307.png" alt="image-20200519161039307" style="zoom:50%;" />

`李四` 的界面如下：

<img src="img/image-20200519161114611.png" alt="image-20200519161114611" style="zoom:50%;" />

### 2.2 实现流程

<img src="img/流程图.png" style="zoom:70%;" />

### 2.3 消息格式

* 客户端 --> 服务端

  {"toName":"张三","message":"你好"}

* 服务端 --> 客户端

  * 系统消息格式：{"isSystem":true,"fromName":null,"message"：["李四","王五"]}
  * 推送给某一个的消息格式：{"isSystem":false,"fromName":"张三","message"："你好"}

### 2.4 功能实现

#### 2.4.1 创建项目，导入相关jar包的坐标

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.1.5.RELEASE</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-websocket</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!--devtools热部署-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <optional>true</optional>
        <scope>true</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <!-- 打jar包时如果不配置该插件，打出来的jar包没有清单文件 -->
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

#### 2.4.2 引入静态资源文件

<img src="img/image-20200519163317764.png" alt="image-20200519163317764" style="zoom:70%;" /> 

#### 2.4.3 引入公共资源

* pojo类

  ```java
  /**
   * @version v1.0
   * @ClassName: Message
   * @Description: 浏览器发送给服务器的websocket数据
   * @Author: 黑马程序员
   */
  public class Message {
      private String toName;
      private String message;
  
      public String getToName() {
          return toName;
      }
  
      public void setToName(String toName) {
          this.toName = toName;
      }
  
      public String getMessage() {
          return message;
      }
  
      public void setMessage(String message) {
          this.message = message;
      }
  }
  ```

  ```java
  /**
   * @version v1.0
   * @ClassName: ResultMessage
   * @Description: 服务器发送给浏览器的websocket数据
   * @Author: 黑马程序员
   */
  public class ResultMessage {
  
      private boolean isSystem;
      private String fromName;
      private Object message;//如果是系统消息是数组
  
      public boolean getIsSystem() {
          return isSystem;
      }
  
      public void setIsSystem(boolean isSystem) {
          this.isSystem = isSystem;
      }
  
      public String getFromName() {
          return fromName;
      }
  
      public void setFromName(String fromName) {
          this.fromName = fromName;
      }
  
      public Object getMessage() {
          return message;
      }
  
      public void setMessage(Object message) {
          this.message = message;
      }
  }
  ```

  ```java
  /**
   * @version v1.0
   * @ClassName: Result
   * @Description: 用于登陆响应回给浏览器的数据
   * @Author: 黑马程序员
   */
  public class Result {
      private boolean flag;
      private String message;
  
      public boolean isFlag() {
          return flag;
      }
  
      public void setFlag(boolean flag) {
          this.flag = flag;
      }
  
      public String getMessage() {
          return message;
      }
  
      public void setMessage(String message) {
          this.message = message;
      }
  }
  ```

  

* MessageUtils工具类

  ```java
  /**
   * @version v1.0
   * @ClassName: MessageUtils
   * @Description: 用来封装消息的工具类
   * @Author: 黑马程序员
   */
  public class MessageUtils {
  
      public static String getMessage(boolean isSystemMessage,String fromName, Object message) {
          try {
              ResultMessage result = new ResultMessage();
              result.setIsSystem(isSystemMessage);
              result.setMessage(message);
              if(fromName != null) {
                  result.setFromName(fromName);
              }
              ObjectMapper mapper = new ObjectMapper();
  
              return mapper.writeValueAsString(result);
          } catch (JsonProcessingException e) {
              e.printStackTrace();
          }
          return null;
      }
  }
  ```



#### 2.4.4 登陆功能实现

* login.html：使用异步进行请求发送

  ```js
  $(function() {
      $("#btn").click(function() {
          $.get("login",$("#loginForm").serialize(),function(res) {
              if(res.flag) {
                  //跳转到 main.html页面
                  location.href = "main.html";
              } else {
                  $("#err_msg").html(res.message);
              }
          },"json");
      });
  })
  ```

* UserController：进行登陆逻辑处理

  ```java
  @RestController
  public class UserController {
  
      @RequestMapping("/login")
      public Result login(User user, HttpSession session) {
          Result result = new Result();
          if(user != null && "123".equals(user.getPassword())) {
              result.setFlag(true);
              //将用户名存储到session对象中
              session.setAttribute("user",user.getUsername());
          } else {
              result.setFlag(false);
              result.setMessage("登陆失败");
          }
  
          return result;
      }
  }
  ```



#### 2.4.5 获取当前登录的用户名

* main.html：页面加载完毕后，发送请求获取当前登录的用户名

  ```js
  var username;
  $(function() {
      $.ajax({
          url:"getUsername",
          success:function(res) {
              username = res;
              $("#userName").html("用户：" + res + "<span style='float: right;color: green'>在线</span>");
          },
          async:false
      });
  }
  ```

* UserController 

  在UserController中添加一个getUsername方法，用来从session中获取当前登录的用户名并响应回给浏览器
  
  ```java
  @RequestMapping("/getUsername")
  public String getUsername(HttpSession session) {
      String username = (String) session.getAttribute("user");
      return username;
  }
  ```

#### 2.5.6 聊天室功能

* 客户端实现

  在main.html页面实现前端代码：

  ```js
  var toName;
          var username;
          function showChat(name) {
              toName = name;
              //清除聊天区的数据
              $("#msgs").html("");
              //现在聊天对话框
              $("#chatArea").css("display","inline");
  			//显示“正在和谁聊天”
              $("#chatMes").html("正在和 <font face=\"楷体\">"+toName+"</font> 聊天");
  
              //切换用户，需要将聊天记录渲染到聊天区
              var storeData = sessionStorage.getItem(toName);
              if(storeData != null) {
                  $("#msgs").html(storeData);
              }
          }
  
          $(function() {
              $.ajax({
                  url:"getUsername",
                  success:function(res) {
                      username = res;
                      //显示在线信息
                      $("#userName").html(" 用户："+res+"<span style='float: right;color: green'>在线</span>");
                  },
                  async: false
              })
  
              //创建websocket
              var ws;
              if(window.WebSocket) {
                  ws = new WebSocket("ws://localhost/chat");
              }
  
              //绑定事件
              ws.onopen = function(evt) {
                  //显示在线信息
                  $("#userName").html(" 用户："+username+"<span style='float: right;color: green'>在线</span>");
              }
  
              ws.onmessage = function(evt) {
                  //接收服务器推送的消息
                  var data = evt.data;
                  //将该字符串数据转换为json
                  var res = JSON.parse(data);
                  //判断是系统消息还是推送给个人的消息
                  if(res.isSystem) {
                      //系统消息
                      var names = res.message;
                      var userListStr = "";
                      var broadcastStr = "";
                      for(var name of names) {
                          if(name != username) {
                              userListStr += "<li class=\"rel-item\"><a onclick='showChat(\""+name+"\")'>"+name+"</a></li>";
                              broadcastStr += "<li class=\"rel-item\" style=\"color: #9d9d9d;font-family: 宋体\">您的好友 "+name+" 已上线</li>";
                          }
                      }
                      //将数据渲染到页面
                      $("#userlist").html(userListStr);
                      $("#broadcastList").html(broadcastStr);
                  } else {
                      //非系统消息
                      var content = res.message;
  
                      //拼接聊天区展示的数据
                      var str = "<div class=\"msg robot\"><div class=\"msg-left\" worker=\"\"><div class=\"msg-host photo\" style=\"background-image: url(img/avatar/Member002.jpg)\"></div><div class=\"msg-ball\">"+content+"</div></div></div>";
  
  
                      //有可能现在不是和指定用户的聊天框，所以需要进行判断
                      var storeData = sessionStorage.getItem(res.fromName);
                      if(storeData != null) {
                          storeData += str;
                      } else {
                          storeData = str;
                      }
                      sessionStorage.setItem(res.fromName,storeData);
                      if(toName == res.fromName) {
                          //将数据追加到聊天区
                          $("#msgs").append(str);
                      }
                  }
              }
  
              ws.onclose = function() {
                  //显示在线信息
                  $("#userName").html(" 用户："+username+"<span style='float: right;color: red'>离线</span>");
              }
  
              //给发送按钮绑定点击事件
              $("#submit").click(function() {
                  //获取输入的内容
                  var data = $("#context_text").val();
                  //将该文本框清空
                  $("#context_text").val("");
                  //拼接消息
                  var str = "<div class=\"msg guest\"><div class=\"msg-right\"><div class=\"msg-host headDefault\"></div><div class=\"msg-ball\">"+data+"</div></div></div>";
                  $("#msgs").append(str);
                  //将聊天记录进行存储sessionStorage
                  var storeData = sessionStorage.getItem(toName);
                  if(storeData != null) {
                      //将此次的内容拼接到storeData中
                      str = storeData + str;
                  }
                  //将消息存储到sessionStorage中
                  sessionStorage.setItem(toName,str);
  
                  //定义服务端需要的数据格式
                  var message = {toName:toName,message:data};
                  //将输入的数据发送给服务器
                  ws.send(JSON.stringify(message));
              });
          })
  ```

* 服务端代码实现

  `WebSocketConfig` 类实现

  开启 springboot 对websocket的支持

  ```java
  @Configuration
  public class WebSocketConfig {
  
      @Bean
      //注入ServerEndpointExporter，自动注册使用@ServerEndpoint注解的
      public ServerEndpointExporter serverEndpointExporter() {
          return new ServerEndpointExporter();
      }
  }
  ```

  `ChatEndPoint` 类实现

  ```java
  @ServerEndpoint(value = "/chat",configurator = GetHttpSessionConfigurator.class)
  @Component
  public class ChatEndpoint {
  
      //用来存储每一个客户端对象对应的ChatEndpoint对象
      private static Map<String,ChatEndpoint> onlineUsers = new ConcurrentHashMap<>();
  
      //和某个客户端连接对象，需要通过他来给客户端发送数据
      private Session session;
  
      //httpSession中存储着当前登录的用户名
      private HttpSession httpSession;
  
      @OnOpen
      //连接建立成功调用
      public void onOpen(Session session, EndpointConfig config) {
          //需要通知其他的客户端，将所有的用户的用户名发送给客户端
          this.session = session;
          //获取HttpSession对象
          HttpSession httpSession = (HttpSession) config.getUserProperties().get(HttpSession.class.getName());
          //将该httpSession赋值给成员httpSession
          this.httpSession = httpSession;
          //获取用户名
          String username = (String) httpSession.getAttribute("user");
          //存储该链接对象
          onlineUsers.put(username,this);
          //获取需要推送的消息
          String message = MessageUtils.getMessage(true, null, getNames());
          //广播给所有的用户
          broadcastAllUsers(message);
      }
  
      private void broadcastAllUsers(String message) {
          try {
              //遍历 onlineUsers 集合
              Set<String> names = onlineUsers.keySet();
              for (String name : names) {
                  //获取该用户对应的ChatEndpoint对象
                  ChatEndpoint chatEndpoint = onlineUsers.get(name);
                  //发送消息
                  chatEndpoint.session.getBasicRemote().sendText(message);
              }
          } catch (Exception e) {
              e.printStackTrace();
          }
      }
  
      private Set<String> getNames() {
          return onlineUsers.keySet();
      }
  
      @OnMessage
      //接收到消息时调用
      public void onMessage(String message,Session session) {
          try {
              //获取客户端发送来的数据  {"toName":"张三","message":"你好"}
              ObjectMapper mapper = new ObjectMapper();
              Message mess = mapper.readValue(message, Message.class);
              //获取当前登录的用户名
              String username = (String) httpSession.getAttribute("user");
              //拼接推送的消息
              String data = MessageUtils.getMessage(false, username, mess.getMessage());
              //将数据推送给指定的客户端
              ChatEndpoint chatEndpoint = onlineUsers.get(mess.getToName());
              chatEndpoint.session.getBasicRemote().sendText(data);
          } catch (Exception e) {
              e.printStackTrace();
          }
      }
  
      @OnClose
      //连接关闭时调用
      public void onClose(Session session) {
          //获取用户名
          String username = (String) httpSession.getAttribute("user");
          //移除连接对象
          onlineUsers.remove(username);
          //获取需要推送的消息
          String message = MessageUtils.getMessage(true, null, getNames());
          //广播给所有的用户
          broadcastAllUsers(message);
      }
  }
  ```
  
* `GetHttpSessionConfigurator` 配置类实现

  ```java
  public class GetHttpSessionConfigurator extends ServerEndpointConfig.Configurator {
      @Override
      public void modifyHandshake(ServerEndpointConfig config, HandshakeRequest request, HandshakeResponse response) {
          HttpSession httpSession = (HttpSession) request.getHttpSession();
          config.getUserProperties().put(HttpSession.class.getName(),httpSession);
      }
  }
  ```

  