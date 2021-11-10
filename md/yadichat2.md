效果：

![image-20211101112912540](https://gitee.com/hqinglau/img/raw/master/img/20211101112912.png)

示例程序链接：[https://yadiChat.orzlinux.cn](https://yadichat.orzlinux.cn/)

github地址：https://github.com/hqingLau/YadiChat

**前文**：

一、[yadiChat-step1-登陆注册](https://orzlinux.cn/blog/yadichat1.html)

## 目录

![image-20211101103212411](https://gitee.com/hqinglau/img/raw/master/img/20211101103212.png)

## pom.xml

加入依赖，netty，序列化，会过期的map包（来管理session）。

```xml
<!--加入netty依赖-->
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
    <version>4.1.69.Final</version>
</dependency>

<!--Gson提供了 fromJson () 和 toJson () 两个直接用于解析和生成的方法，前者实现反序列化，后者实现了序列化-->
<dependency>
    <groupId>com.google.code.gson</groupId>
    <artifactId>gson</artifactId>
    <version>2.8.8</version>
</dependency>

<!--会过期的map-->
<dependency>
    <groupId>net.jodah</groupId>
    <artifactId>expiringmap</artifactId>
    <version>0.5.10</version>
</dependency>
```

## session管理

之前我们将session由spring管理，现在我们改为自己管理。

UserSession.java

```java
public class UserSession {
    private UserSession() {}
    public static void put(String sessionId,User user) {
        userMap.put(sessionId,user);
    }

    public static User get(String sessionId) {
        return userMap.getOrDefault(sessionId,null);
    }

    private static ExpiringMap<String, User> userMap = ExpiringMap.builder().variableExpiration()
            .expiration(24*60, TimeUnit.MINUTES).expirationPolicy(ExpirationPolicy.CREATED).build();

    public static void update(String value) {
        userMap.put(value,userMap.get(value));
    }
}
```

这里面主要就是封装了一个会过期的map，这里设置为24小时过期，用来管理session。这样一段时间后，session过期，就会自动提示登录界面。

更改登录和注册逻辑，如：

UserController.java

```java
@PostMapping("/login")
public String login(Model model, HttpServletRequest request, HttpServletResponse response) {
    String idname = request.getParameter("idname");
    String password = request.getParameter("password");
    User user = userService.login(new User(idname,null,password));
    if (user==null) {
        return "redirect:/signin";
    }
    String userUUID = idname;
    UserSession.put(userUUID,user);
    Cookie cookie = new Cookie("user",userUUID);
    cookie.setMaxAge(24*60*60); //一天过期
    // 配置cookie为主站cookie
    cookie.setDomain("orzlinux.cn");
    response.addCookie(cookie);
    return "redirect:/home";
}
```

session就是客户端放一个cookie作为sessionid，来服务器之后用这个sessionid查找对应的session。这里设置为主站cookie，这样在后期用到其它子域名的服务时cookie也能读取到，这里简单设置为用户idname。

接下来更改拦截策略：

UserInterceptor.java

```java
public class UserInterceptor implements HandlerInterceptor  {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String uri = request.getRequestURI();
        if("/".equals(uri)||"/error".equals(uri)||"/signin".equals(uri)||"/login".equals(uri)||"/signup".equals(uri)||"/regist".equals(uri)) {
            return true;
        }
        Cookie cookie = CookieUtils.getCookie(request.getCookies(),"user");
        if(cookie!=null) {
            UserSession.update(cookie.getValue());
            return true;
        }
        response.sendRedirect("/signin");
        return false;
    }
}
```

首先获取用户cookie，取UserSession查找，有就更新一下map，主要为了更新过期时间，没有就转到登录界面。

这里面需要获取cookie，为了方便使用封装成一个工具类：

CookieUtils.java

```java
public class CookieUtils {
    public static Cookie getCookie(javax.servlet.http.Cookie[] cookies, String key) {
        if(cookies != null) {
            for (javax.servlet.http.Cookie cookie : cookies) {
                if (cookie.getName().equals(key)) {
                    if (UserSession.get(cookie.getValue()) != null) {
                        return cookie;
                    }
                }
            }
        }
        return null;
    }
}
```

就是遍历，比较键值。

## TextMsgBean

接下来要定义信息：

```java
public class TextMsg {
    private String idname;
    private String nick_name;
    private String msg;
    private Date msgTime;
    // getter setter constructor
}
```

需要存储idname用户确定用户身份，昵称用于显示，消息具体内容，发送消息的时间。

## 全局消息数据库

```sql
# 建立全局聊天数据库
CREATE TABLE global_room_msg
(
    id INT NOT NULL AUTO_INCREMENT,
    id_name VARCHAR(128) NOT NULL ,
    nick_name VARCHAR(128) not null ,
    msg VARCHAR(10240) not null,
    msg_time DATETIME not null ,
    PRIMARY KEY (id),
    INDEX id_name(id_name)
);
```

同样，数据库配置。

## TextMsgDao

这里用到了两个方法，向数据库插入一条消息，和获取过去的100条消息用于显示。

```java
@Mapper
public interface TextMsgDao {
    int insert(TextMsg msg);
    List<TextMsg> getLast100Msg();
}
```

## TextMsgDao.xml

对应上面方法的数据库实现。

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="cn.orzlinux.step2_basic_communication.dao.TextMsgDao">
 <insert id="insert" parameterType="TextMsg" >
        insert into global_room_msg(id_name,nick_name,msg,msg_time) values(#{idname},#{nick_name},#{msg},#{msgTime})
    </insert>

    <select id="getLast100Msg" resultType="TextMsg">
        select
               id_name idname,
               nick_name nick_name,
               msg msg,
               msg_time msgTime
        from global_room_msg
        order by id DESC
        limit 100
    </select>
</mapper>
```

## netty server实现

netty内容见前文：[Netty概述](https://orzlinux.cn/blog/netty20211021.html)

```java
public class WebSocketServer {
    private static WebSocketServer server;

    private static final int READ_IDLE_TIME_OUT = 0;
    private static final int WRITE_IDLE_TIME_OUT = 0;
    private static final int ALL_IDLE_TIME_OUT = 5;

    private WebSocketServer() {}

    // 只能通过getInstance获取单例
    public static WebSocketServer getInstance() {
        if(server==null) {
            server = new WebSocketServer();
        }
        return server;
    }

    public void run(int port) {
        EventLoopGroup bossGroup = new NioEventLoopGroup(1); //只管接收
        EventLoopGroup workerGroup = new NioEventLoopGroup(); //默认cpu核数*2
        try {
            // 创建服务端的启动对象，配置参数
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(bossGroup,workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            ChannelPipeline pipeline = socketChannel.pipeline();
                            // netty的解码器和编码器
                            pipeline.addLast(new HttpServerCodec());
                            // 以块方式写，用于大数据分区传输
                            pipeline.addLast(new ChunkedWriteHandler());
                            // http在传输过程中分段，这里聚合多个段
                            pipeline.addLast(new HttpObjectAggregator(32*1024));
                            // wesocket数据压缩
                            pipeline.addLast(new WebSocketServerCompressionHandler());

                            //WebSocketServerProtocolHandler将http协议升级为ws协议，保持长连接
                            pipeline.addLast(new WebSocketServerProtocolHandler("/ws",null,true,10*1024));

                            // 当连接60s没有收到消息，就会触发idlestateevent事件,ChannelRead()方法未被调用
                            // 则触发一次MyTextWebSocketFrameHandler 的userEventTrigger()方法
                            // 心跳机制: 主要是用来检测远端是否存活，如果不存活或活跃则对空闲Socket连接进行处理避免资源的浪费
                            pipeline.addLast(new IdleStateHandler(READ_IDLE_TIME_OUT,WRITE_IDLE_TIME_OUT,ALL_IDLE_TIME_OUT, TimeUnit.SECONDS));

                            //自定义的handler ，处理业务逻辑
                            pipeline.addLast(new MyTextWebSocketFrameHandler());
                        }
                    });
            ChannelFuture channelFuture = bootstrap.bind(port).sync();
            channelFuture.channel().closeFuture().sync();


        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            SessionGroup.getInstance().shutdownGracefully();
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

这里面用到了一个自定义handler：MyTextWebSocketFrameHandler。还有websocket协议长链接的处理。

## SocketSession

建立`channel`之后，需要获取对应的用户。这里主要用了一个`AttributeKey`将`channel`和`user`绑定。

```java
/**
 * 对应用户服务端会话管理
 */
public class SocketSession {
    public static final AttributeKey<SocketSession> SESSION_KEY = AttributeKey.valueOf("SESSION_KEY");
    private Channel channel;
    private User user;

    public String getUserCookieId() {
        return userCookieId;
    }

    public void setUserCookieId(String userCookieId) {
        this.userCookieId = userCookieId;
    }

    private String userCookieId;
    private String group;
    // session中存储的属性值
    private Map<String,Object> map = new HashMap<>();

    public SocketSession(Channel channel) {
        this.channel = channel;
        // channel和session绑定
        channel.attr(SocketSession.SESSION_KEY).set(this);
    }

    // 返回ctx对应的SocketSession，ctx.channel->找session
    public static SocketSession getSession(ChannelHandlerContext ctx) {
        return ctx.channel().attr(SocketSession.SESSION_KEY).get();
    }
}
```

## MyTextWebSocketFrameHandler

要将消息插入数据库，需要对应service，这里并不能直接Autowire，可以用一个自定义的`SpringUtil`工具类实现：

```java
@ChannelHandler.Sharable
public class MyTextWebSocketFrameHandler extends SimpleChannelInboundHandler<TextWebSocketFrame> {
    private static TextMsgService textMsgService;

    static {
        textMsgService = SpringUtil.getBean(TextMsgService.class);
    }
    // ……
}
```

SpringUtil：

```java
@Component
public class SpringUtil implements ApplicationContextAware {
    private static ApplicationContext ac;

    @Override
    public void setApplicationContext(ApplicationContext arg0) throws BeansException {
        ac = arg0;
    }

    public static ApplicationContext getApplicationContext() {
        return ac;
    }

    /**
     * 通过class获取Bean
     */
    public static <T> T getBean(Class<T> clazz){
        return getApplicationContext().getBean(clazz);
    }


    /**
     * 如果BeanFactory包含所给名称匹配的bean返回true
     * @param name
     * @return boolean
     */
    public static boolean containsBean(String name) {
        return ac.containsBean(name);
    }

    /**
     * 判断注册的bean是singleton还是prototype。
     * 如果与给定名字相应的bean定义没有被找到，将会抛出一个异常（NoSuchBeanDefinitionException）
     * @param name
     * @return boolean
     */
    public static boolean isSingleton(String name) {
        return ac.isSingleton(name);
    }

    /**
     * @param name
     * @return Class 注册对象的类型
     */
    public static Class<?> getType(String name) {
        return ac.getType(name);
    }

}
```

ws协议事件触发：

```java
@Override
public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
    // 握手成功，升级为websocket协议
    if(evt== WebSocketServerProtocolHandler.ServerHandshakeStateEvent.HANDSHAKE_COMPLETE) {
        new SocketSession(ctx.channel());
    } else if (evt instanceof IdleStateEvent) {
        // TODO
    } else {
        super.userEventTriggered(ctx,evt);
    }

}
```

`websocket`连接成功，就新建一个session记录这个连接。这样，在数据到来的时候，直接从`context`的`channel`中获取`session`，读取对应`channel`的用户数据。

其中：

```java
public <T> T fromJson(@Nullable String json, reflect.Type typeOfT)
```

在将`json`读取为`hashmap`时，由于泛型无法直接获取`class`，需要用`TypeToken`做一个中介。

```java
@Override
protected void channelRead0(ChannelHandlerContext ctx, TextWebSocketFrame msg) throws Exception {
    SocketSession session = SocketSession.getSession(ctx);
    TypeToken<HashMap<String,String>> typeToken = new TypeToken<HashMap<String,String>>(){};
    Gson gson = new Gson();
    // 对于泛型无法直接获取class
    Map<String,String> map = gson.fromJson(msg.text(),typeToken.getType());
    User user = null;
    switch (map.get("type")) {
        case "msg":
            Map<String,String> result = new HashMap<>();
            user = session.getUser();
            result.put("type","msg");
            result.put("msg",map.get("msg"));
            result.put("sendUserCookieId",session.getUserCookieId());
            result.put("sendUserNickname",user.getNickName());
            TextMsg textMsg = new TextMsg(user.getIdName(),user.getNickName(),map.get("msg"),new Date());
            // 消息插入数据库
            textMsgService.insertMsg(textMsg);
            SessionGroup.getInstance().sendToOthers(result,session);
            break;
        case "init":
            //String room = map.getOrDefault("room",null);
            String room = "群聊";
            session.setGroup(room);
            String userCookieId = map.getOrDefault("user",null);
            user = UserSession.get(userCookieId);
            if(user==null) {
                return;
            }
            session.setUserCookieId(userCookieId);
            session.setUser(user);
            SessionGroup.getInstance().addSession(session);
            break;
    }
}
```

`init`情况为初始进入聊天室，需要确定用户身份，`session`的组，用于消息转发时给不同的组。在发送消息时`msg`，解析用户消息之后，也需要将消息转发给不同的组，这里用一个封装类`SessionGroup`完成此事。

## SessionGroup

详见注释：

```java
public class SessionGroup {
    private static SessionGroup instance;

    // 组名到组的映射
    private ConcurrentHashMap<String, ChannelGroup> groupMap = new ConcurrentHashMap<>();

    private SessionGroup() {}
    // 单例模式
    public static SessionGroup getInstance() {
        if(instance==null) {
            instance = new SessionGroup();
        }
        return instance;
    }
    
    // 发送给对应的channel组。
    public void sendToOthers(Map<String, String> result, SocketSession session) {
        ChannelGroup group = groupMap.get(session.getGroup());
        if(null == group) {
            return;
        }
        Gson gson = new Gson();
        String json = gson.toJson(result);
        ChannelGroupFuture future = group.writeAndFlush(new TextWebSocketFrame(json));
        future.addListener(f->{
            System.out.println("完成发送："+json);
        });
    }
	
    // 将session加入对应的group组名对应的channel组
    public void addSession(SocketSession session) {
        String groupName = session.getGroup();
        if(StringUtils.isNullOrEmpty(groupName)) {
            return;
        }
        ChannelGroup group = groupMap.get(groupName);
        if(null==group) {
            group = new DefaultChannelGroup(ImmediateEventExecutor.INSTANCE);
            groupMap.put(groupName,group);
        }
        group.add(session.getChannel());
    }

    /**
     * 关闭连接
     * @param session
     * @param echo
     */
    public void closeSession(SocketSession session,String echo) {
        ChannelFuture sendFuture = session.getChannel().writeAndFlush(new TextWebSocketFrame(echo));
        sendFuture.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture channelFuture) throws Exception {
                System.out.println("连接关闭："+echo);
                channelFuture.channel().close();
            }
        });
    }

    /**
     * 发送消息
     */
    public void sendMsg(ChannelHandlerContext ctx,String msg) {
        ChannelFuture sendFuture = ctx.writeAndFlush(new TextWebSocketFrame(msg));
        sendFuture.addListener(f->{
            System.out.println("对所有发送完成："+msg);
        });
    }

    public void shutdownGracefully() {
        Iterator<ChannelGroup> groupIterator = groupMap.values().iterator();
        while(groupIterator.hasNext()) {
            groupIterator.next().close();
        }
    }
}
```

## TextMsgController

Controller只需加入一个获取数据库前100条的url就可以：

```java
@Controller
public class TextMsgController {
    @Autowired
    TextMsgService service;

    @ResponseBody
    @RequestMapping(value = "/textMsg/getLast100Msg",produces = {"application/json;charset=UTF-8"})
    public List<TextMsg> getLast100Msg() {
        return service.getLast100Msg();
    }
}
```

还有进入全局聊天室的controller:

```java
@RequestMapping("/room")
public String room() {
    return "room";
}
```

## room.html

前端界面需要给定之前提到的用户数据发给后端，完成交互，代码如下：

```html
<!DOCTYPE HTML>
<html>
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8"/>
    <title>全局聊天室</title>
    <!-- 引入 Bootstrap -->
    <link href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css" rel="stylesheet">
    <script src="https://cdn.bootcss.com/jquery/3.4.1/jquery.js"></script>
    <style type="text/css">
        body{ padding-bottom:80px;
            max-width: 800px;
            margin: 0 auto;
        }
        .fixed{
            max-width: 800px;
            margin: 0 auto;
            position: fixed;
            bottom: 0px;
            width: 100%;
            height: 50px;
            /*background-color: #000;*/
            background-color: white;
            z-index: 9999;
        }
        .msg {
            margin-left: 30px;
            margin-right: 30px;
            margin-top: 8px;
            max-width: 500px;
        }
        .nickname {
            background: #31708f;
            color: white;
            border-radius: 10px;
            padding: 8px;
        }
        ul {
            overflow: auto;
            height: auto;
            margin: 0 auto;
        }

        li {
            float: left;
            width: 800px;
            margin: 0 auto;
        }

        .haha {
            background: #00bfff1c;
            border-radius: 5px;
            padding: 10px;
            width: fit-content;
            margin-top: 5px;
        }

        span {
            word-break: break-all;
        }
    </style>
</head>
<body>
<div class="page-header text-center text-primary">
    <h1>聊天室</h1>
</div>

<br /><br />
<!--<div id="message"></div>-->
<ul class="list-group" id="message">
    <li class="list-group-item"><span class="nickname">系统信息</span><div class="msg"> 欢迎入群</div></li>
</ul>
<div class="clear:both"></div>
<br /><br />


<div class="fixed">
    <button class="btn btn-primary" style="float: right;margin-right: 10px" type="button" onclick="send()">发送</button>
    <div style="overflow: hidden; padding-right: 10px">
    <textarea style="margin-left:10px;width: 100%; resize: none"
           id="text"
           name="send"
           placeholder="输入发送消息"
           rows="1"
           maxlength="1000"
           minlength="1"
           class="form-control"></textarea>
    </div>
    <div class="clear"></div>
</div>

<script type="text/javascript">
    var webSocket;
    var loadPreMsg = false;
    var wsConnect = false; //避免重复连接
    var url = "ws://ws-yadichat.orzlinux.cn/ws";
    // var url = "ws://localhost:10240/ws";
    if (window.WebSocket) {
        webSocket = new WebSocket(url);
    } else {
        alert("抱歉，您的浏览器不支持WebSocket协议!");
    }

    function setWebSocket() {
        //连通之后的回调事件
        webSocket.onopen = function() {
            console.log("已经连通了websocket");
            wsConnect = true;
            enter();
        }
        //连接发生错误的回调方法
        webSocket.onerror = function(event){
            console.log("出错了");
            wsConnect = false;
            reconnect(url);
//              setMessageInnerHTML("连接失败");
        };

        //连接关闭的回调方法
        webSocket.onclose = function(){
            console.log("连接已关闭...");
            wsConnect = false;
            reconnect(url);
        }

        //接收到消息的回调方法
        webSocket.onmessage = function(event){
            var data = JSON.parse(event.data)
            var msg = data.msg;
            var nick = data.sendUserNickname;
            var userId = data.sendUserCookieId;
            switch(data.type){
                case 'init':
                    console.log("init");
                    break;
                case 'msg':
                    console.log("msg");
                    setMessageInnerHTML(nick, msg,userId);
                    break;
                default:
                    break;
            }
        }
    }
    setWebSocket();
    function reconnect(url) {
        if(wsConnect) return;
        //没连接上会一直重连，设置延迟避免请求过多
        webSocket = new WebSocket(url);
        wsConnect = true;
        setWebSocket();
        console.log("正在重连，当前时间" + new Date());
    }


    function enter(){
        var map = new Map();
        map.set("type","init");
        map.set("user",getCookie("user"));
        var message = Map2Json(map);
        webSocket.send(message);
    //    获取最近100条消息
        if(!loadPreMsg) {
            loadPreMsg = true;

            $.ajax({
                url: '/textMsg/getLast100Msg',
                success: function (json) {
                    $.each(json, function (i, item) {
                        //循环获取数据
                        var nick = item.nick_name;
                        var msg = item.msg;
                        var userId = item.idname;

                        if (userId != getCookie("user")) {
                            // 别人
                            $("#message").last().append('<li class="list-group-item"><span class="nickname">' + nick +
                                '</span><div class="msg"><div class="haha"><span>' + msg + '</span></div></div></li>');
                        } else {
                            // 自己
                            $("#message").last().append('<li class="list-group-item">\n' +
                                '        <div style="float: right">\n' +
                                '            <span class="nickname" style="background: forestgreen;">' + nick + '</span>\n' +
                                '        </div>\n' +
                                '        <br />\n' +
                                '        <div class="msg" style="float: right"><div class="haha"><span> ' + msg + '</span></div></div></li>');
                        }

                    });
                    window.scrollTo(0, 99999999);

                }
            });
        }
    }

    function send() {
        var msg = document.getElementById('text').value;
        console.log("1:"+msg);
        if (msg != null && msg !== ""){
            var map = new Map();
            map.set("type","msg");
            map.set("user",getCookie("user"));
            map.set("msg",msg);
            var map2json=Map2Json(map);
            if (map2json.length < 8000){
                console.log("4:"+map2json);
                webSocket.send(map2json);
                document.getElementById('text').value='';
            }else {
                console.log("文本太长了，少写一点吧😭");
            }
        }
    }

    function getCookie(cookie_name) {
        var allcookies = document.cookie;
        //索引长度，开始索引的位置
        var cookie_pos = allcookies.indexOf(cookie_name);

        // 如果找到了索引，就代表cookie存在,否则不存在
        if (cookie_pos != -1) {
            // 把cookie_pos放在值的开始，只要给值加1即可
            //计算取cookie值得开始索引，加的1为“=”
            cookie_pos = cookie_pos + cookie_name.length + 1;
            //计算取cookie值得结束索引
            var cookie_end = allcookies.indexOf(";", cookie_pos);

            if (cookie_end == -1) {
                cookie_end = allcookies.length;

            }
            //得到想要的cookie的值
            var value = unescape(allcookies.substring(cookie_pos, cookie_end));
        }
        return value;
    }

    //将消息显示在网页上
    function setMessageInnerHTML(nick,msg,userId) {
        if(userId!=getCookie("user")){
            // 别人
            $("#message").last().append('<li class="list-group-item"><span class="nickname">' +nick+
                '</span><div class="msg"><div class="haha"><span>'+msg+ '</span></div></div></li>');
        }else {
            // 自己
            $("#message").last().append('<li class="list-group-item">\n' +
                '        <div style="float: right">\n' +
                '            <span class="nickname" style="background: forestgreen;">' + nick +'</span>\n' +
                '        </div>\n' +
                '        <br />\n' +
                '        <div class="msg" style="float: right"><div class="haha"><span> '+msg+ '</span></div></div></li>');
        }
        window.scrollTo(0,99999999);


    }

    function Map2Json(map) {
        var str = "{";
        map.forEach(function (value, key) {
            str += '"'+key+'"'+':'+ '"'+value+'",';
        })
        str = str.substring(0,str.length-1)
        str +="}";
        return str;
    }
</script>
<!--!&#45;&#45; jQuery文件。务必在bootstrap.min.js 之前引入 -->
<script src="https://cdn.staticfile.org/jquery/2.1.1/jquery.min.js"/>
<!-- 最新的 Bootstrap 核心 JavaScript 文件 -->
<script src="https://cdn.staticfile.org/twitter-bootstrap/3.3.7/js/bootstrap.min.js"/>
</body>
</html>
```

