> 本文为[Java高并发秒杀API之web层](https://www.imooc.com/learn/630)课程笔记。

编辑器：IDEA

java版本：java8

前文：

一、[秒杀系统环境搭建与DAO层设计](https://orzlinux.cn/blog/javaseckill1-20211004.html)

二、 [秒杀系统Service层](https://orzlinux.cn/blog/javaseckill2-20211005.html)

## 设计Restful接口

根据需求设计前端交互流程。

三个职位：

- 产品：解读用户需求，搞出需求文档
- 前端：不同平台的页面展示
- 后端：存储、展示、处理数据

前端页面流程：

<img src="https://gitee.com/hqinglau/img/raw/master/img/20211005214607.png" alt="image-20211005214607039" style="zoom:80%;" />

详情页流程逻辑：

<img src="https://gitee.com/hqinglau/img/raw/master/img/20211005214858.png" alt="image-20211005214858360" style="zoom:80%;" />

标准系统时间从服务器获取。

Restful：一种优雅的URI表述方式、资源的状态和状态转移。

Restful规范：

- GET 查询操作
- POST 添加/修改操作（非幂等）
- PUT 修改操作（幂等，没有太严格区分）
- DELETE 删除操作

URL设计：

```shell
/模块/资源/{标示}/集合/...
/user/{uid}/friends -> 好友列表
/user/{uid}/followers -> 关注者列表
```

秒杀API的URL设计

```shell
GET /seckill/list 秒杀列表
GET /seckill/{id}/detail 详情页
GET /seckill/time/now 系统时间
POST /seckill/{id}/exposer 暴露秒杀
POST /seckill/{id}/{md5}/execution 执行秒杀
```

下一步就是如何实现这些URL接口。

## SpringMVC

### 理论

![image-20211005234650630](https://gitee.com/hqinglau/img/raw/master/img/20211005234650.png)

**适配器模式（Adapter Pattern），把一个类的接口变换成客户端所期待的另一种接口， Adapter模式使原本因接口不匹配（或者不兼容）而无法在一起工作的两个类能够在一起工作。**

SpringMVC的`handler`（`Controller`，`HttpRequestHandler`，`Servlet`等）有多种实现方式，例如继承`Controller`的，基于注解控制器方式的，`HttpRequestHandler`方式的。由于实现方式不一样，调用方式就不确定了。

看HandlerAdapter接口有三个方法：

```java
// 判断该适配器是否支持这个HandlerMethod
boolean supports(Object handler);
// 用来执行控制器处理函数，获取ModelAndView 。就是根据该适配器调用规则执行handler方法。
ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;
long getLastModified(HttpServletRequest request, Object handler);
```

访问流程如上图，用户访问一个请求，首先经过`DispatcherServlet`转发。利用`HandlerMapping`得到想要的`HandlerExecutionChain`（里面包含`handler`和一堆拦截器）。然后利用`handler`，得到`HandlerAdapter`，遍历所有注入的`HandlerAdapter`，依次使用`supports`方法寻找适合这个`handler`的适配器子类。最后通过这个获取的适配器子类运用`handle`方法调用控制器函数，返回ModelAndView。

**注解映射技巧**

- 支持标准的URL
- ？和*和**等字符，如`/usr/*/creation`会匹配`/usr/AAA/creation`和`/usr/BBB/creation`等。`/usr/**/creation`会匹配`/usr/creation`和`/usr/AAA/BBB/creation`等URL。
- 带{xxx}占位符的URL。如`/usr/{userid}`匹配`/usr/123`、`/usr/abc`等URL.

**请求方法细节处理**

- 请求参数绑定

- 请求方式限制

- 请求转发和重定向

- 数据模型赋值

- 返回json数据

- cookie访问

![image-20211006002757781](https://gitee.com/hqinglau/img/raw/master/img/20211006002757.png)

**返回json数据**

![image-20211006002918369](https://gitee.com/hqinglau/img/raw/master/img/20211006002918.png)

**cookie访问**：

![image-20211006002952177](https://gitee.com/hqinglau/img/raw/master/img/20211006002952.png)

### 项目整合SpringMVC

web.xml下配置springmvc需要加载的配置文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
                      http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
         version="3.1" metadata-complete="true">
  <!--修改servlet版本为3.1-->

  <!--配置DispatcherServlet-->
  <servlet>
    <servlet-name>seckill-dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <!--配置springmvc需要加载的配置文件
        spring-dao.xml spring-service.xml spring-web.xml-->
    <!--整合：mybatis -> spring -> springmvc-->
    <init-param>
      <param-name>contextConfigLocation</param-name>
      <param-value>classpath:spring/spring-*.xml</param-value>
    </init-param>
  </servlet>
  <servlet-mapping>
    <servlet-name>seckill-dispatcher</servlet-name>
    <url-pattern>/</url-pattern>
  </servlet-mapping>
</web-app>
```

在resources文件夹下的spring文件夹添加spring-web.xml文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:tx="http://www.springframework.org/schema/tx" xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd http://www.springframework.org/schema/cache http://www.springframework.org/schema/cache/spring-cache.xsd http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx.xsd http://www.springframework.org/schema/mvc https://www.springframework.org/schema/mvc/spring-mvc.xsd">
<!--配置springmvc-->
    <!--1. 开启springmvc注解模式-->
    <!-- 简化配置，
        自动注册handlermapping,handleradapter
        默认提供了一系列功能：数据绑定，数字和日期的format,xml和json的读写支持
     -->
    <mvc:annotation-driven/>

     <!--servlet-mapping 映射路径："/"-->
    <!--2. 静态资源默认servlet配置
        静态资源处理：js,gif,png..
        允许使用/做整体映射
    -->
    <mvc:default-servlet-handler/>

    <!--3. jsp的显示viewResolver-->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="viewClass" value="org.springframework.web.servlet.view.JstlView"/>
        <property name="prefix" value="/WEB-INF/jsp"/>
        <property name="suffix" value=".jsp"/>
    </bean>

    <!--4. 扫描web相关的bean-->
    <context:component-scan base-package="cn.orzlinux.web"/>
</beans>
```

## 使用SpringMVC实现Restful接口

新建文件：

![image-20211006161515136](https://gitee.com/hqinglau/img/raw/master/img/20211006161515.png)

首先是SeckillResult.java，这个保存controller的返回结果，做一个封装。

```java
// 所有ajax请求返回类型，封装json结果
public class SeckillResult<T> {
    private boolean success; //是否执行成功
    private T data; // 携带数据
    private String error; // 错误信息
    // getter setter contructor
}
```

在Seckillcontroller.java中，实现了我们之前定义的几个URL：

```shell
GET /seckill/list 秒杀列表
GET /seckill/{id}/detail 详情页
GET /seckill/time/now 系统时间
POST /seckill/{id}/exposer 暴露秒杀
POST /seckill/{id}/{md5}/execution 执行秒杀
```

具体代码如下：

```java
@Controller // @Service @Component放入spring容器
@RequestMapping("/seckill") // url:模块/资源/{id}/细分
public class SeckillController {
    private final Logger logger = LoggerFactory.getLogger(this.getClass());
    @Autowired
    private SecKillService secKillService;
    @RequestMapping(value = "/list",method = RequestMethod.GET)
    public String list(Model model) {
        // list.jsp + model = modelandview
        List<SecKill> list = secKillService.getSecKillList();
        model.addAttribute("list",list);
        return "list";
    }

    @RequestMapping(value = "/{seckillId}/detail", method = RequestMethod.GET)
    public String detail(@PathVariable("seckillId") Long seckillId, Model model) {
        if (seckillId == null) {
            // 0. 不存在就重定向到list
            // 1. 重定向访问服务器两次
            // 2. 重定向可以重定义到任意资源路径。
            // 3. 重定向会产生一个新的request，不能共享request域信息与请求参数
            return "redrict:/seckill/list";

        }
        SecKill secKill = secKillService.getById(seckillId);
        if (secKill == null) {
            // 0. 为了展示效果用forward
            // 1. 转发只访问服务器一次。
            // 2. 转发只能转发到自己的web应用内
            // 3. 转发相当于服务器跳转，相当于方法调用，在执行当前文件的过程中转向执行目标文件，
            //      两个文件(当前文件和目标文件)属于同一次请求，前后页 共用一个request，可以通
            //      过此来传递一些数据或者session信息
            return "forward:/seckill/list";
        }
        model.addAttribute("seckill",secKill);
        return "detail";
    }

    // ajax json
    @RequestMapping(value = "/{seckillId}/exposer",
            method = RequestMethod.POST,
            produces = {"application/json;charset=UTF8"})
    @ResponseBody
    public SeckillResult<Exposer> exposer(Long seckillId) {
        SeckillResult<Exposer> result;
        try {
            Exposer exposer = secKillService.exportSecKillUrl(seckillId);
            result = new SeckillResult<Exposer>(true,exposer);
        } catch (Exception e) {
            logger.error(e.getMessage(),e);
            result = new SeckillResult<>(false,e.getMessage());
        }
        return result;
    }

    @RequestMapping(value = "/{seckillId}/{md5}/execution",
        method = RequestMethod.POST,
        produces = {"application/json;charset=UTF8"})
    public SeckillResult<SeckillExecution> execute(
            @PathVariable("seckillId") Long seckillId,
            // required = false表示cookie逻辑由我们程序处理，springmvc不要报错
            @CookieValue(value = "killPhone",required = false) Long userPhone,
            @PathVariable("md5") String md5) {
        if (userPhone == null) {
            return new SeckillResult<SeckillExecution>(false, "未注册");
        }
        SeckillResult<SeckillExecution> result;
        try {
            SeckillExecution execution = secKillService.executeSeckill(seckillId, userPhone, md5);
            result = new SeckillResult<SeckillExecution>(true, execution);
            return result;
        } catch (SeckillCloseException e) { // 秒杀关闭
            SeckillExecution execution = new SeckillExecution(seckillId, SecKillStatEnum.END);
            return new SeckillResult<SeckillExecution>(false,execution);
        } catch (RepeatKillException e) { // 重复秒杀
            SeckillExecution execution = new SeckillExecution(seckillId, SecKillStatEnum.REPEAT_KILL);
            return new SeckillResult<SeckillExecution>(false,execution);
        } catch (Exception e) {
            // 不是重复秒杀或秒杀结束，就返回内部错误
            logger.error(e.getMessage(), e);
            SeckillExecution execution = new SeckillExecution(seckillId, SecKillStatEnum.INNER_ERROR);
            return new SeckillResult<SeckillExecution>(false,execution);
        }

    }

    @RequestMapping(value = "/time/now",method = RequestMethod.GET)
    @ResponseBody
    public SeckillResult<Long> time() {
        Date now = new Date();
        return new SeckillResult<Long>(true,now.getTime());
    }
}
```

## 页面

![image-20211006182314439](https://gitee.com/hqinglau/img/raw/master/img/20211006182314.png)

这里修改数据库为合适的时间来测试我们的代码。

点击后跳转到详情页。

![image-20211006173025166](https://gitee.com/hqinglau/img/raw/master/img/20211006173025.png)

详情页涉及到比较多的交互逻辑，如cookie，秒杀成功失败等等。放到**逻辑交互**一节来说。

运行时发现jackson版本出现问题，pom.xml修改为:

```xml
<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-databind</artifactId>
  <version>2.10.2</version>
</dependency>
```

list.jsp代码为：

```html
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%--引入jstl--%>
<%--标签通用头，写在一个具体文件，直接静态包含--%>
<%@include file="common/tag.jsp"%>
<!DOCTYPE html>
<html>
<head>
    <title>Bootstrap 模板</title>
    <%--静态包含：会合并过来放到这，和当前文件一起作为整个输出--%>
    <%@include file="common/head.jsp"%>
</head>
<body>
    <%--页面显示部分--%>
    <div class="container">
        <div class="panel panel-default">
            <div class="panel panel-heading text-center">
                <h1>秒杀列表</h1>
            </div>
            <div class="panel-body">
                <table class="table table-hover" >
                    <thead>
                        <tr>
                            <th>名称</th>
                            <th>库存</th>
                            <th>开始时间</th>
                            <th>结束时间</th>
                            <th>创建时间</th>
                            <th>详情页</th>
                        </tr>
                    </thead>
                    <tbody>
                        <c:forEach var="sk" items="${list}">
                            <tr>
                                <td>${sk.name}</td>
                                <td>${sk.number}</td>
                                <td>
                                    <fmt:formatDate value="${sk.startTime}" pattern="yyyy-MM-dd HH:mm:ss"/>
                                </td>
                                <td>
                                    <fmt:formatDate value="${sk.endTime}" pattern="yyyy-MM-dd HH:mm:ss"/>
                                </td>
                                <td>
                                    <fmt:formatDate value="${sk.createTime}" pattern="yyyy-MM-dd HH:mm:ss"/>
                                </td>
                                <td>
                                    <a class="btn btn-info" href="/seckill/${sk.seckillId}/detail" target="_blank">
                                        link
                                    </a>
                                </td>
                            </tr>
                        </c:forEach>
                    </tbody>
                </table>
            </div>
        </div>
    </div>

<!-- jQuery (Bootstrap 的 JavaScript 插件需要引入 jQuery) -->
<script src="https://code.jquery.com/jquery.js"></script>
<!-- 包括所有已编译的插件 -->
<script src="js/bootstrap.min.js"></script>
</body>
</html>
```

![image-20211006172715018](https://gitee.com/hqinglau/img/raw/master/img/20211006172715.png)



## 逻辑交互

### 身份认证

cookie中没有手机号要弹窗，手机号不正确（11位数字）要提示错误：

![image-20211006191323525](https://gitee.com/hqinglau/img/raw/master/img/20211006191323.png)

选择提交之后要能够在cookie中看到：

![image-20211006191309641](https://gitee.com/hqinglau/img/raw/master/img/20211006191309.png)

目前为止detail.jsp:

```html
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<!DOCTYPE html>
<html>
<head>
    <title>秒杀详情页</title>
    <%--静态包含：会合并过来放到这，和当前文件一起作为整个输出--%>
    <%@include file="common/head.jsp"%>
    <link href="https://cdn.bootcdn.net/ajax/libs/jquery-countdown/2.1.0/css/jquery.countdown.css" rel="stylesheet">
</head>
<body>
    <%--<input type="hidden" id="basePath" value="${basePath}" />--%>
    <div class="container">
        <div class="panel panel-default text-center">
            <h1>
                <div class="panel-heading">${seckill.name}</div>
            </h1>

        </div>
        <div class="panel-body">
            <h2 class="text-danger">
                <!-- 显示time图标 -->
                <span class="glyphicon glyphicon-time"></span>
                <!-- 展示倒计时 -->
                <span class="glyphicon" id="seckillBox"></span>
            </h2>
        </div>
    </div>
    <!-- 登录弹出层，输入电话 bootstrap里面的-->
    <div id="killPhoneModal" class="modal fade bs-example-modal-lg">
        <div class="modal-dialog">
            <div class="modal-content">
                <div class="modal-header">
                    <h3 class="modal-title text-center">
                        <span class="glyphicon glyphicon-phone"></span>秒杀电话：
                    </h3>
                </div>
                <div class="modal-body">
                    <div class="row">
                        <div class="col-xs-8 col-xs-offset-2">
                            <input type="text" name="killphone" id="killphoneKey"
                                   placeholder="填手机号^O^" class="form-control" />
                        </div>
                    </div>
                </div>
                <div class="modal-footer">
                    <span id="killphoneMessage" class="glyphicon"></span>
                    <button type="button" id="killPhoneBtn" class="btn btn-success">
                        <span class="glyphicon glyphicon-phone"></span> Submit
                    </button>
                </div>
            </div>
        </div>
    </div>


    <!-- jQuery文件。务必在bootstrap.min.js 之前引入 -->
    <script src="//cdn.bootcss.com/jquery/1.11.3/jquery.min.js"></script>
    <!-- 最新的 Bootstrap 核心 JavaScript 文件 -->
    <script src="//cdn.bootcss.com/bootstrap/3.3.5/js/bootstrap.min.js"></script>
    <!-- jQuery cookie操作插件 -->
    <script src="//cdn.bootcss.com/jquery-cookie/1.4.1/jquery.cookie.min.js"></script>
    <!-- jQery countDonw倒计时插件  -->
    <script src="//cdn.bootcss.com/jquery.countdown/2.1.0/jquery.countdown.min.js"></script>

<%--开始写交互逻辑--%>
<script src="/resources/script/seckill.js" type="text/javascript"></script>
<script type="text/javascript">
    $(function () {
        seckill.detail.init({
            seckillId: ${seckill.seckillId},
            startTime: ${seckill.startTime.time}, // 转化为毫秒，方便比较
            endTime: ${seckill.endTime.time},
        });
    });
</script>

</body>
</html>
```

我们的逻辑主要写在另外的js文件中：

seckill.js

```javascript
// 存放主要交互逻辑js
// javascript 模块化
var seckill={
    // 封装秒杀相关ajax的URL
    URL:{

    },
    // 验证手机号
    validatePhone: function (phone) {
        if(phone && phone.length==11 && !isNaN(phone)) {
            return true;
        } else {
            return false;
        }
    },
    // 详情页秒杀逻辑
    detail: {
        // 详情页初始化
        init: function (params) {
            // 手机验证和登录，计时交互
            // 规划交互流程
            // 在cookie中查找手机号
            var killPhone = $.cookie('killPhone');
            var startTime = params['startTime'];
            var endTime = params['endTime'];
            var seckillId = params['seckillId'];
            // 验证手机号
            if(!seckill.validatePhone(killPhone)) {
                // 绑定手机号，获取弹窗输入手机号的div id
                var killPhoneModal = $('#killPhoneModal');
                killPhoneModal.modal({
                    show: true, //显示弹出层
                    backdrop: 'static',//禁止位置关闭
                    keyboard: false, //关闭键盘事件
                });
                $('#killPhoneBtn').click(function () {
                    var inputPhone = $('#killphoneKey').val();
                    // 输入格式什么的ok了就刷新页面
                    if(seckill.validatePhone(inputPhone)) {
                        // 将电话写入cookie
                        $.cookie('killPhone',inputPhone,{expires:7,path:'/seckill'});
                        window.location.reload();
                    } else {
                        // 更好的方式是把字符串写入字典再用
                        $('#killphoneMessage').hide().html('<label class="label label-danger">手机号格式错误</label>').show(500);
                    }
                });
            }
            // 已经登录

        }
    }
}
```

### 计时面板

在登录完成后，处理计时操作：

```javascript
// 已经登录
// 计时交互
$.get(seckill.URL.now(),{},function (result) {
    if(result && result['success']) {
        var nowTime = result['data'];
        // 写到函数里处理
        seckill.countdown(seckillId,nowTime,startTime,endTime);
    } else {
        console.log('result: '+result);
    }
});
```

在countdown函数里，有三个判断，未开始、已经开始、结束。

```javascript
URL:{
    now: function () {
        return '/seckill/time/now';
    }
},

handleSeckill: function () {
    // 处理秒杀逻辑
},

countdown: function (seckillId,nowTime,startTime,endTime) {
    var seckillBox = $('#seckillBox');
    if(nowTime>endTime) {
        seckillBox.html('秒杀结束！');
    } else  if(nowTime<startTime) {
        // 秒杀未开始，计时
        var killTime = new Date(startTime + 1000);
        seckillBox.countdown(killTime,function (event) {
            // 控制时间格式
            var format = event.strftime('秒杀开始倒计时：%D天 %H时 %M分 %S秒');
            seckillBox.html(format);
            // 时间完成后回调事件
        }).on('finish.countdown', function () {
            // 获取秒杀地址，控制显示逻辑，执行秒杀
            seckill.handleSeckill();
        })
    } else {
        // 秒杀开始
        seckill.handleSeckill();
    }
},
```

总体就是一个显示操作，用了jquery的countdown倒计时插件。

<img src="https://gitee.com/hqinglau/img/raw/master/img/20211006194407.png" alt="image-20211006194407145" style="zoom:67%;" />

### 秒杀交互



秒杀之前：

![image-20211006202253376](https://gitee.com/hqinglau/img/raw/master/img/20211006202253.png)

详情页：

<img src="https://gitee.com/hqinglau/img/raw/master/img/20211006201149.png" alt="image-20211006201149488" style="zoom:80%;" />

点击开始秒杀：

<img src="https://gitee.com/hqinglau/img/raw/master/img/20211006202320.png" alt="image-20211006202320137" style="zoom:80%;" />

列表页刷新：

![image-20211006202306300](https://gitee.com/hqinglau/img/raw/master/img/20211006202306.png)



运行时发现controller忘了写`@ResponseBody`了，这里返回的不是jsp是json，需要加上。

```java
@ResponseBody
public SeckillResult<SeckillExecution> execute(
        @PathVariable("seckillId") Long seckillId,
        // required = false表示cookie逻辑由我们程序处理，springmvc不要报错
        @CookieValue(value = "killPhone",required = false) Long userPhone,
        @PathVariable("md5") String md5)
```

在seckill.js中，补全秒杀逻辑：

```javascript
// 封装秒杀相关ajax的URL
URL:{
    now: function () {
        return '/seckill/time/now';
    },
    exposer: function(seckillId) {
        return '/seckill/'+seckillId+'/exposer';
    },

    execution: function (seckillId,md5) {
        return '/seckill/'+seckillId+'/'+md5+'/execution';
    }
},


// id和显示计时的那个模块
handleSeckill: function (seckillId,node) {
    // 处理秒杀逻辑
    // 在计时的地方显示一个秒杀按钮
    node.hide()
        .html('<button class="btn btn-primary btn-lg" id="killBtn">开始秒杀</button>');
    // 获取秒杀地址
    $.post(seckill.URL.exposer(),{seckillId},function (result) {
        if(result && result['success']) {
            var exposer = result['data'];
            if(exposer['exposed']) {
                // 如果开启了秒杀
                // 获取秒杀地址
                var md5 = exposer['md5'];
                var killUrl = seckill.URL.execution(seckillId,md5);
                console.log("killurl: "+killUrl);
               // click永远绑定，one只绑定一次
                $('#killBtn').one('click',function () {
                    // 执行秒杀请求操作
                    // 先禁用按钮
                    $(this).addClass('disabled');
                    // 发送秒杀请求
                    $.post(killUrl,{},function (result) {
                        if(result) {

                            var killResult = result['data'];
                            var state = killResult['state'];
                            var stateInfo = killResult['stateInfo'];
                            // 显示秒杀结果
                            if(result['success']) {
                                node.html('<span class="label label-success">'+stateInfo+'</span>');
                            } else {
                                node.html('<span class="label label-danger">'+stateInfo+'</span>');
                            }


                        }
                        console.log(result);
                    })
                });
                node.show();
            } else {
                // 未开始秒杀，这里是因为本机显示时间和服务器时间不一致
                // 可能浏览器认为开始了，服务器其实还没开始
                var now = exposer['now'];
                var start = exposer['start'];
                var end = exposer['end'];
                // 重新进入倒计时逻辑
                seckill.countdown(seckillId,now,start,end);
            }
        } else {
            console.log('result='+result);
        }
    })
},
```

秒杀成功后再次进行秒杀则不成功：

<img src="https://gitee.com/hqinglau/img/raw/master/img/20211006203710.png" alt="image-20211006203710505" style="zoom:80%;" />

输出：

![image-20211006203126546](https://gitee.com/hqinglau/img/raw/master/img/20211006203126.png)

在库存不够时也返回秒杀结束：

![image-20211006203238807](https://gitee.com/hqinglau/img/raw/master/img/20211006203238.png)

<img src="https://gitee.com/hqinglau/img/raw/master/img/20211006203736.png" alt="image-20211006203736790" style="zoom:80%;" />



至此，功能方面已经实现了，后面还剩下优化部分。