Servlet（Server Applet），全称 Java Servlet，是用Java编写的服务器端程序。其主要功能在于交互式地浏览和修改数据，生成动态Web内容。

## 工作模式

客户端发送请求至服务器，服务器启动并调用 Servlet，Servlet 根据客户端请求生成响应内容传给服务器，服务器将响应传给客户端。

![image-20210915122442895](https://gitee.com/hqinglau/img/raw/master/img/20210915122443.png)

Tomcat 是 Web 应用服务器,是一个 Servlet/JSP 容器。 Tomcat 作为 Servlet 容器，负责处理客户请求，把请求传送给 Servlet，并将Servlet 的响应传送回给客户。而 Servlet 是一种运行在支持Java语言的服务器上的组件。 Servlet 最常见的用途是扩展 Java Web 服务器功能,提供非常安全的，可移植的，易于使用的 CGI 替代品。

![image-20210915122823232](https://gitee.com/hqinglau/img/raw/master/img/20210915122823.png)

## 示例一

```java
public class HelloServlet extends HttpServlet {
    public void doGet(HttpServletRequest request, HttpServletResponse response) throws IOException {
        System.out.println("get");
    }
    public void doPost(HttpServletRequest request, HttpServletResponse response) throws IOException {
        System.out.println("post");
    }
}
```

web.xml：

```xml
<servlet>
    <!--servlet的名字，什么都可以，一般与我们编写的名字相同-->
    <servlet-name>HelloServlet</servlet-name>
    <servlet-class>com.lhq.javaweblhq.HelloServlet</servlet-class>
</servlet>

<servlet-mapping>
    <!-- 上面起的名字-->
    <servlet-name>HelloServlet</servlet-name>
    <!--  通过url找到servlet，/必须要加-->
    <url-pattern>/hello</url-pattern>
</servlet-mapping>
```

访问链接之后：

```shell
# http://localhost:8080/javaweblhq_war_exploded/hello
[2021-09-15 12:50:09,347] Artifact javaweblhq:war exploded: Deploy took 523 milliseconds
get
15-Sep-2021 12:50:18.427
```

## Servlet 原理

`Servlet`  **生命周期**：

<img src="https://gitee.com/hqinglau/img/raw/master/img/20210915125913.png" alt="image-20210915125913452" style="zoom: 75%;" />

```java
protected void service(HttpServletRequest req, HttpServletResponse resp)
        throws ServletException, IOException
{
    String method = req.getMethod();

    if (method.equals(METHOD_GET)) {
        long lastModified = getLastModified(req);
        if (lastModified == -1) {
            // servlet doesn't support if-modified-since, no reason
            // to go through further expensive logic
            doGet(req, resp);
        } else {
            long ifModifiedSince = req.getDateHeader(HEADER_IFMODSINCE);
            if (ifModifiedSince < lastModified) {
                // If the servlet mod time is later, call doGet()
                // Round down to the nearest second for a proper compare
                // A ifModifiedSince of -1 will always be less
                maybeSetLastModified(resp, lastModified);
                doGet(req, resp);
            } else {
                resp.setStatus(HttpServletResponse.SC_NOT_MODIFIED);
            }
        }

    } else if (method.equals(METHOD_HEAD)) {
        long lastModified = getLastModified(req);
        maybeSetLastModified(resp, lastModified);
        doHead(req, resp);

    } else if (method.equals(METHOD_POST)) {
        doPost(req, resp);

    } else if (method.equals(METHOD_PUT)) {
        doPut(req, resp);

    } else if (method.equals(METHOD_DELETE)) {
        doDelete(req, resp);

    } else if (method.equals(METHOD_OPTIONS)) {
        doOptions(req,resp);

    } else if (method.equals(METHOD_TRACE)) {
        doTrace(req,resp);

    } else {
        //
        // Note that this means NO servlet supports whatever
        // method was requested, anywhere on this server.
        //

        String errMsg = lStrings.getString("http.method_not_implemented");
        Object[] errArgs = new Object[1];
        errArgs[0] = method;
        errMsg = MessageFormat.format(errMsg, errArgs);

        resp.sendError(HttpServletResponse.SC_NOT_IMPLEMENTED, errMsg);
    }
}
```

以示例一为例，继承结构：

![image-20210915130925328](https://gitee.com/hqinglau/img/raw/master/img/20210915130925.png)

Servlet 接口内容：

![image-20210915131548152](https://gitee.com/hqinglau/img/raw/master/img/20210915131548.png)

`Servlet`生命周期的三个关键方法，`init`、`service`、`destroy`。还有另外两个方法，一个`getServletConfig`方法来获取`ServletConfig`对象，`ServletConfig`对象可以获取到`Servlet`的一些信息，`ServletName`、`ServletContext`、`InitParameter`、`InitParameterNames`。

`ServletContext`对象是`servlet`上下文对象，功能有很多，获得了`ServletContext`对象，就能获取大部分我们需要的信息，比如获取`servlet`的路径等方法。

可以通过`Servlet`的`getServletConfig`方法获取`ServletConfig`。

![image-20210915132221319](https://gitee.com/hqinglau/img/raw/master/img/20210915132221.png)

web.xml配置如下：

![img](https://gitee.com/hqinglau/img/raw/master/img/20210915132506.png)

java：

```java
public class HelloServlet extends HttpServlet {
    public void doGet(HttpServletRequest request, HttpServletResponse response) throws IOException {
        System.out.println("get");
        System.out.println("这样也行"+this.getServletName());
        // 继承了genericservlet,它实现了这个，见下图
        // 输出：这样也行HelloServlet
        // 或者 System.out.println(this.getServletConfig().getServletName());
        System.out.println(this.getServletConfig().getInitParameter("yadiparam"));
        // 输出
        // get
        // HelloServlet
        // 初始化参数测试
    }
    public void doPost(HttpServletRequest request, HttpServletResponse response) throws IOException {
        System.out.println("post");
    }
}
```

![image-20210915133404949](https://gitee.com/hqinglau/img/raw/master/img/20210915133404.png)

tomcat 为每个 web 项目都**创建一个`ServletContext`实例**，tomcat 在启动时创建，服务器关闭时销毁，在一个 web 项目中共享数据，管理 web 项目资源，为整个 web 配置公共信息等，通俗点讲，就是一个 web 项目，就存在一个`ServletContext`实例，每个 `Servlet` 读可以访问到它。

```xml
<context-param>
    <param-name>xixi</param-name>
    <param-value>web项目的初始化参数</param-value>
</context-param>
```

java:

```java
System.out.println(getServletContext().getInitParameter("xixi"));
```

**请求转发**

```java
request.getRequestDispatcher(String path).forward(request,response);
```

path：转发后跳转的页面，这里不管用不用"/"开头，都是以web项目根开始，因为这是请求转发，请求转发只局限与在同一个web项目下使用，所以这里一直都是从web项目根下开始的。

特点：浏览器中url不会改变，也就是浏览器不知道服务器做了什么，是服务器帮我们跳转页面的，并且在转发后的页面，能够继续使用原先的request，因为是原先的request，所以request域中的属性都可以继续获取到。

```java
response.setHeader("Refresh","3");
response.getOutputStream().print(request.getRequestURI()+" "+request.getServerName()+" "+new Date().getSeconds());
// 输出 ：/javaweblhq_war_exploded/hello localhost 21
// 3s刷新一次
```

`PrintWriter getWriter()`

获得字符流，通过字符流的`write(String s)`方法可以将字符串设置到 response   缓冲区中，随后 Tomcat 会将 response 缓冲区中的内容组装成 Http 响应返回给浏览器端。

`ServletOutputStream getOutputStream()`

获得字节流，通过该字节流的`write(byte[] bytes)`可以向 response 缓冲区中写入字节，再由 Tomcat 服务器将字节内容组成 Http 响应返回给浏览器。

response 对象的`getOutSream()`和`getWriter()`方法都可以发送响应消息体，但是他们之间相互排斥，不可以同时使用，否则会发生异常。

**中文异常问题：**

![image-20210915160751227](https://gitee.com/hqinglau/img/raw/master/img/20210915160751.png)

```java
response.setContentType("text/html;charset=utf-8");
response.getWriter().print(request.getRequestURI()+" "+request.getServerName()+"中文"+new Date().getSeconds());
```

<img src="https://gitee.com/hqinglau/img/raw/master/img/20210915160924.png" alt="image-20210915160924867" style="zoom: 70%;" />

## 参考

[JavaWeb——Servlet（全网最详细教程包括Servlet源码分析）](https://blog.csdn.net/qq_19782019/article/details/80292110)

[Java Web(一) Servlet详解！！](https://www.cnblogs.com/whgk/p/6399262.html)


