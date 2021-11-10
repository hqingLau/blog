## SpringMVC

### 小例子

目录：

![image-20210927111129822](https://gitee.com/hqinglau/img/raw/master/img/20210927111129.png)

**流程图**：

![image-20210927111632522](https://gitee.com/hqinglau/img/raw/master/img/20210927111632.png)

springmvc-servlet.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="simpleUrlHandlerMapping" class="org.springframework.web.servlet.handler.SimpleUrlHandlerMapping">
        <property name="mappings">
            <props>
                <prop key="/index">indexController</prop>
            </props>
        </property>
    </bean>
    <bean id="indexController" class="controller.IndexController"></bean>
</beans>
```

web.xml用来做映射：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">
    <servlet>
        <servlet-name>springmvc</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>
    
    <servlet-mapping>
        <servlet-name>springmvc</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>
```

controller:

```java
package controller;

import org.springframework.web.servlet.ModelAndView;
import org.springframework.web.servlet.mvc.Controller;

public class IndexController implements Controller {
    @Override
    public ModelAndView handleRequest(javax.servlet.http.HttpServletRequest httpServletRequest, javax.servlet.http.HttpServletResponse httpServletResponse) throws Exception {
        ModelAndView mav = new ModelAndView("index.jsp");
        mav.addObject("message","HelloSpring MVC");
        mav.addObject("Title","my defined title");
        return mav;
    }
}
```

包问题：试试这个

![image-20210927110900247](https://gitee.com/hqinglau/img/raw/master/img/20210927110900.png)

### 视图定位

把视图约定在`/WEB-INF/page/*.jsp`。

修改 springmvc-servlet.xml:

```xml
<bean id="viewResolver" class="org.springframework.web.servlet.view.InternalResourceViewResolver">
    <property name="prefix" value="/WEB-INF/page/"/>
    <property name="suffix" value=".jsp"/>
</bean>
```

controller：

```java
ModelAndView mav = new ModelAndView("index");
```

![image-20210927112224072](https://gitee.com/hqinglau/img/raw/master/img/20210927112224.png)

### 注解方式

显示报错：通配符的匹配很全面, 但无法找到元素 'context:component-scan' 的声明。因为`beans`这里不全：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans-4.2.xsd
    http://www.springframework.org/schema/mvc
    http://www.springframework.org/schema/mvc/spring-mvc-4.2.xsd
    http://www.springframework.org/schema/context
    http://www.springframework.org/schema/context/spring-context-4.2.xsd">
<!--    <bean id="simpleUrlHandlerMapping" class="org.springframework.web.servlet.handler.SimpleUrlHandlerMapping">-->
<!--        <property name="mappings">-->
<!--            <props>-->
<!--                <prop key="/index">indexController</prop>-->
<!--            </props>-->
<!--        </property>-->
<!--    </bean>-->
<!--    <bean id="indexController" class="controller.IndexController"></bean>-->

    <bean id="viewResolver" class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="prefix" value="/WEB-INF/page/"/>
        <property name="suffix" value=".jsp"/>
    </bean>
    <!--    自动扫描-->
    <context:component-scan base-package="controller"/>
</beans>
```

注解：

```java
@Controller
public class IndexController{

    @RequestMapping("/index")
    public ModelAndView handleRequest(javax.servlet.http.HttpServletRequest httpServletRequest, javax.servlet.http.HttpServletResponse httpServletResponse) throws Exception {
        ModelAndView mav = new ModelAndView("index");
        mav.addObject("message","HelloSpring MVC");
        mav.addObject("Title","my defined title");
        return mav;
    }
}
```

### 提交表单

```java

public class Product {
    private int id;
    private String name;
    private float price;
    。。getter setter
}
===========================
@Controller
public class ProductController {

    // addProduct.jsp 提交的name和price会自动注入到参数 product里
    // 注： 参数product会默认被当做值加入到ModelAndView 中，相当于：
    // mav.addObject("product",product);
    @RequestMapping("/addProduct")
    public ModelAndView add(Product product) throws Exception {
        ModelAndView mav = new ModelAndView("/showProduct.jsp");
        return mav;
    }
}
```





![image-20210927120237570](https://gitee.com/hqinglau/img/raw/master/img/20210927120237.png)





![image-20210927120228314](https://gitee.com/hqinglau/img/raw/master/img/20210927120228.png)



### 客户端跳转

之前是服务端跳转，url不变的，如：

![image-20210927120555185](https://gitee.com/hqinglau/img/raw/master/img/20210927120555.png)

跳转到了index.jsp。

客户端跳转，url会变：

```java
@RequestMapping("/jump")
public ModelAndView jump() {
    // redirect:/index 即表示客户端跳转的意思
    ModelAndView mav = new ModelAndView("redirect:/index");
    return mav;
}
```

### Session

spring mvc默认可以直接用`HttpSession`

```java
@RequestMapping("/check")
public ModelAndView check(HttpSession session) {
    Integer i = (Integer) session.getAttribute("count");
    if(i==null) i=0;
    i++;
    session.setAttribute("count",i);
    ModelAndView mav = new ModelAndView("check");
    return mav;
}
```

查看cookie，保留了sessionId：

![image-20210927122011540](https://gitee.com/hqinglau/img/raw/master/img/20210927122011.png)

删除后，再访问：

![image-20210927122039914](https://gitee.com/hqinglau/img/raw/master/img/20210927122039.png)

服务器就认不得它了，重新计数。

### 中文

直接在web.xml添加：

```xml
<filter>  
    <filter-name>CharacterEncodingFilter</filter-name>  
    <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>  
    <init-param>  
        <param-name>encoding</param-name>  
        <param-value>utf-8</param-value>  
    </init-param>  
</filter>  
<filter-mapping>  
    <filter-name>CharacterEncodingFilter</filter-name>  
    <url-pattern>/*</url-pattern>  
</filter-mapping> 
```

否则post会乱码：

![image-20210927122902987](https://gitee.com/hqinglau/img/raw/master/img/20210927122903.png)

### 上传文件

效果：

![image-20210927134705174](https://gitee.com/hqinglau/img/raw/master/img/20210927134705.png)

jsp:

```html
<form action="uploadImage" method="post" enctype="multipart/form-data">
  选择图片:<input type="file" name="img" accept="image/*" /> <br>
  <input type="submit" value="上传">
</form>
```

这里的img要对应pojo的img：

```java
public class UploadImageFile {
    MultipartFile img;

    public MultipartFile getImg() {
        return img;
    }

    public void setImg(MultipartFile img) {
        this.img = img;
    }
}
```

否则会对应不上。

controller

```java
@Controller
public class UploadController {
    @RequestMapping("/uploadImage")
    //若没有@RequestParam("此处对应Form表单input的name属性值") MultipartFile
    public ModelAndView upload(HttpServletRequest request,UploadImageFile file) throws IOException {
        String name = RandomStringUtils.randomAlphabetic(10);
        String newFileName = name+".jpg";
        File newFile = new File(request.getServletContext().getRealPath("/image"),newFileName);
        newFile.getParentFile().mkdirs();
        file.getImg().transferTo(newFile);
        ModelAndView mav = new ModelAndView("showUploadedFile");
        mav.addObject("imageName", newFileName);
        return mav;
    }
}
```

遇到的问题：

```xml
org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'multipartResolver': Lookup method resolution failed; nested exception is java.lang.IllegalStateException: Failed to introspect Class [org.springframework.web.multipart.commons.CommonsMultipartResolver] from ClassLoader [ParallelWebappClassLoader
  context: ROOT
```

原因是没有文件上传的jar包，引入就可以了。

```xml
<dependency>
    <groupId>commons-fileupload</groupId>
    <artifactId>commons-fileupload</artifactId>
    <version>1.3.3</version>
</dependency>
```

不行就再put一下：

![image-20210927125138355](https://gitee.com/hqinglau/img/raw/master/img/20210927125138.png)

### 拦截器

```java
public class IndexInterceptor implements HandlerInterceptor {
    /**
     * 在业务处理器处理请求之前被调用
     * 如果返回false
     *     从当前的拦截器往回执行所有拦截器的afterCompletion(),再退出拦截器链
     * 如果返回true
     *    执行下一个拦截器,直到所有的拦截器都执行完毕
     *    再执行被拦截的Controller
     *    然后进入拦截器链,
     *    从最后一个拦截器往回执行所有的postHandle()
     *    接着再从最后一个拦截器往回执行所有的afterCompletion()
     */
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        System.out.println("preHandle(), 在访问Controller之前被调用");
        return true;
    }

    /**
     * 在业务处理器处理请求执行完成后,生成视图之前执行的动作
     * 可在modelAndView中加入数据，比如当前时间
     */
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        System.out.println("postHandle(), 在访问Controller之后，访问视图之前被调用,这里可以注入一个时间到modelAndView中，用于后续视图显示");
        assert modelAndView != null;
        modelAndView.addObject("date","由拦截器生成的时间:" + new Date());
    }

    /**
     * 在DispatcherServlet完全处理完请求后被调用,可用于清理资源等
     * 当有拦截器抛出异常时,会从当前拦截器往回执行所有的拦截器的afterCompletion()
     */
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        System.out.println("afterCompletion(), 在访问视图之后被调用");
    }
}
```

配置：

```xml
<mvc:interceptors>
    <mvc:interceptor>
        <!--  /** 拦截所有-->
        <!--  /category/** 拦截/category路径下的所有-->
        <mvc:mapping path="/index"/>
        <bean class="interceptor.IndexInterceptor"/>
    </mvc:interceptor>
    <!-- 当设置多个拦截器时，先按顺序调用preHandle方法，然后逆序调用每个拦截器的postHandle和afterCompletion方法 -->
</mvc:interceptors>
```

## SSM例子



![image-20210927193254616](https://gitee.com/hqinglau/img/raw/master/img/20210927193254.png)

![image-20210927194504157](https://gitee.com/hqinglau/img/raw/master/img/20210927194504.png)

![image-20210927200425325](https://gitee.com/hqinglau/img/raw/master/img/20210927200425.png)



跟着代码具体走一下：

### web.xml

```xml
<!DOCTYPE web-app PUBLIC
 "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
 "http://java.sun.com/dtd/web-app_2_3.dtd" >

<web-app>
  <display-name>Archetype Created Web Application</display-name>
<!--  spring配置文件-->
  <context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:applicationContext.xml</param-value>
  </context-param>
  <listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>

  <!-- spring mvc核心：分发servlet -->
  <servlet>
    <servlet-name>mvc-dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <!-- spring mvc的配置文件 -->
    <init-param>
      <param-name>contextConfigLocation</param-name>
      <param-value>classpath:springMVC.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
  </servlet>
  <servlet-mapping>
    <servlet-name>mvc-dispatcher</servlet-name>
    <url-pattern>/</url-pattern>
  </servlet-mapping>
</web-app>
```

### springmvc.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans-4.2.xsd
    http://www.springframework.org/schema/mvc
    http://www.springframework.org/schema/mvc/spring-mvc-4.2.xsd
    http://www.springframework.org/schema/context
    http://www.springframework.org/schema/context/spring-context-4.2.xsd">
    <context:annotation-config/>
    <context:component-scan base-package="cn.orzlinux.controller">
        <context:include-filter type="annotation"
                                expression="org.springframework.stereotype.Controller"/>
    </context:component-scan>

    <mvc:annotation-driven />

    <mvc:default-servlet-handler />

    <!-- 视图定位 -->
    <bean
            class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="viewClass"
                  value="org.springframework.web.servlet.view.JstlView" />
        <property name="prefix" value="/WEB-INF/jsp/" />
        <property name="suffix" value=".jsp" />
    </bean>
</beans>
```

#### context:annotation-config

本节来自：[here](https://blog.csdn.net/fox_bert/article/details/80793030)

< context:annotation-config> 是用于激活那些已经在spring容器里注册过的bean上面的注解，也就是显示的向Spring注册

```xml
AutowiredAnnotationBeanPostProcessor
CommonAnnotationBeanPostProcessor
PersistenceAnnotationBeanPostProcessor
RequiredAnnotationBeanPostProcessor
```

这四个Processor，注册这4个BeanPostProcessor的作用，就是为了你的系统能够识别相应的注解。BeanPostProcessor就是处理注解的处理器。

- 比如我们要使用@Autowired注解，那么就必须事先在 Spring 容器中声明 AutowiredAnnotationBeanPostProcessor Bean。传统声明方式如下

```xml
<bean class="org.springframework.beans.factory.annotation. AutowiredAnnotationBeanPostProcessor "/>
```

- 如果想使用@ Resource 、@ PostConstruct、@ PreDestroy等注解就必须声明CommonAnnotationBeanPostProcessor。传统声明方式如下

```xml
<bean class="org.springframework.beans.factory.annotation. CommonAnnotationBeanPostProcessor"/> 
```

- 如果想使用@PersistenceContext注解，就必须声明PersistenceAnnotationBeanPostProcessor的Bean。

```xml
<bean class="org.springframework.beans.factory.annotation.PersistenceAnnotationBeanPostProcessor"/> 
```

- 如果想使用 @Required的注解，就必须声明RequiredAnnotationBeanPostProcessor的Bean。

同样，传统的声明方式如下：

```xml
<bean class="org.springframework.beans.factory.annotation.RequiredAnnotationBeanPostProcessor"/> 
```

一般来说，像@ Resource 、@ PostConstruct、@Antowired这些注解在自动注入还是比较常用，所以如果总是需要按照传统的方式一条一条配置显得有些繁琐和没有必要，于是spring给我们提供< context:annotation-config/>的简化配置方式，自动帮你完成声明。

思考1：假如我们要使用如@Component、@Controller、@Service等这些注解，使用能否激活这些注解呢?

答案：单纯使用< context:annotation-config/>对上面这些注解无效，不能激活！

#### context:component-scan

Spring 提供了context:component-scan配置，如下

```html
<context:component-scan base-package=”XX.XX”/> 
```

该配置项其实也包含了自动注入上述 四个processor 的功能，因此当使用 `< context:component-scan/>` 后，就可以将 `< context:annotation-config/>` 移除了。 

通过对**base-package**配置，就可以把controller包下 service包下 dao包下的注解全部扫描到了！

#### mvc:default-servlet-handler

默认 Servlet 的 RequestDispatcher 必须通过名称而不是路径来检索。 换句话说就是 Spring MVC 将接收到的所有请求都看作是一个普通的请求，包括对于静态资源的请求。这样以来，所有对于静态资源的请求都会被看作是一个普通的后台控制器请求，导致请求找不到而报 404 异常错误。查看 tomcat 的日志会报一个警告：


对于这个问题 Spring MVC 在全局配置文件中提供了一个`<mvc:default-servlet-handler/>`标签。在 WEB 容器启动的时候会在上下文中定义一个 DefaultServletHttpRequestHandler，它会对DispatcherServlet的请求进行处理，如果该请求已经作了映射，那么会接着交给后台对应的处理程序，如果没有作映射，就交给 WEB 应用服务器默认的 Servlet 处理，从而找到对应的静态资源，只有再找不到资源时才会报错。


#### 总结

（1）**< context:annotation-config />：**仅能够在已经在已经注册过的bean上面起作用。对于没有在spring容器中注册的bean，它并不能执行任何操作。 
（2）**< context:component-scan base-package="XX.XX"/>** ：除了具有上面的功能之外，还具有自动将带有@component,@service,@Repository等注解的对象注册到spring容器中的功能。 

思考2: 如果同时使用这两个配置会不会出现重复注入的情况呢？

答案：因为`< context:annotation-config />`和 `< context:component-scan>`同时存在的时候，前者会被忽略。如@autowire，@resource等注入注解只会被注入一次！

#### mvc:annotation-driven

`<mvc:annotation-driven/>`会自动注册RequestMappingHandlerMapping与RequestMappingHandlerAdapter两个Bean,这是Spring MVC为@Controller分发请求所必需的。



### CategoryController.java

![image-20210927200425325](https://gitee.com/hqinglau/img/raw/master/img/20210927202246.png)

```java
@Controller
@RequestMapping("")
public class CategoryController {
    @Autowired
    CategoryService categoryService;

    @RequestMapping("listCategory")
    public ModelAndView listCategory() {
        ModelAndView mav = new ModelAndView();
        List<Category> cs = categoryService.list();
        mav.addObject("cs",cs);
        mav.setViewName("listCategory");
        return mav;
    }
}
```

通过springmvc.xml的操作，service已经注册到容器了，准群来说是其实现：

![image-20210927202538987](https://gitee.com/hqinglau/img/raw/master/img/20210927202539.png)

要调用service的list方法，得去实现里面找，找到之后就加到mav的object里面传给显示界面。

### CategoryServiceImpl.java

```java
@Service
public class CategoryServiceImpl implements CategoryService {
    @Autowired
    public CategoryMapper categoryMapper;

    @Override
    public List<Category> list() {
        return categoryMapper.list();
    }
}
```

这个找就要找mapper了，而mapper和xml的映射和数据库连接已经在applicationContext.xml处理了。

### applicationContext.xml

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:tx="http://www.springframework.org/schema/tx" xmlns:jdbc="http://www.springframework.org/schema/jdbc"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="
     http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.0.xsd
     http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
     http://www.springframework.org/schema/jdbc http://www.springframework.org/schema/jdbc/spring-jdbc-3.0.xsd
     http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-3.0.xsd
     http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-3.0.xsd
     http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc.xsd">

<!--    通过注解，将Service的生命周期纳入Spring的管理-->
    <context:annotation-config/>
    <context:component-scan base-package="cn.orzlinux.service"/>

<!--    配置数据源-->
    <bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="driverClassName">
            <value>com.mysql.cj.jdbc.Driver</value>
        </property>
        <property name="url">
            <value>jdbc:mysql://localhost:3306/how2java?characterEncoding=UTF-8</value>
        </property>
        <property name="username">
            <value>root</value>
        </property>
        <property name="password">
            <value>===</value>
        </property>
    </bean>

    <bean id="sqlSession" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="typeAliasesPackage" value="cn.orzlinux.pojo"/>
        <property name="dataSource" ref="dataSource"/>
        <property name="mapperLocations" value="classpath:cn/orzlinux/mapper/*.xml"/>
    </bean>

    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="basePackage" value="cn.orzlinux.mapper"/>
    </bean>
</beans>
```

在 MyBatis 中，可以使用 SqlSessionFactory 来创建 SqlSession。一旦你获得一个 session 之后，可以使用它来执行映射语句、提交或回滚连接，最后，当不再需要它的时 候，可以关闭 session。

所以，controller就拿到了数据库里的数据，可以显示了。

### catagory.java

```java
public class Category {
    private int id;
    private String name;
    。。getter setter
}
```

### CategoryMapper.java

```java
public interface CategoryMapper {
    int add(Category category);
    void delete(int id);
    Category get(int id);
    int update(Category category);
    List<Category> list();
    int count();
}
```

### CategoryMapper.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="cn.orzlinux.mapper.CategoryMapper">
    <insert id="add" parameterType="Category" >
        insert into category_ ( name ) values (#{name})
    </insert>

    <delete id="delete" parameterType="Category" >
        delete from category_ where id= #{id}
    </delete>

    <select id="get" parameterType="_int" resultType="Category">
        select * from   category_  where id= #{id}
    </select>

    <update id="update" parameterType="Category" >
        update category_ set name=#{name} where id=#{id}
    </update>
    <select id="list" resultType="Category">
        select * from   category_
    </select>

</mapper>
```

### listCategory.jsp

```html
<%--
  Created by IntelliJ IDEA.
  User: PC
  Date: 2021/9/27
  Time: 16:53
  To change this template use File | Settings | File Templates.
--%>
<%@ page contentType="text/html;charset=UTF-8" language="java" isELIgnored="false" %>
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c"%>

<html>
<head>
    <title>Title</title>
</head>
<body>


<table align='center' border='1' cellspacing='0'>
  <tr>
    <td>id</td>
    <td>name</td>
  </tr>
  <c:forEach items="${cs}" var="c" varStatus="st">
    <tr>
      <td>${c.id}</td>
      <td>${c.name}</td>

    </tr>
  </c:forEach>
</table>

</body>
</html>
```



### Autowired

@Autowired原理：
举例：

```java
@Autowired
private BookService bookService;
```



先按照类型去容器中找到对应的组件；`bookService = ioc.getBean(BookService.class)`

找到一个：找到就赋值，没找到就报异常。

按照类型可以找到多个？找到多个如何装配上？类型一样就按照变量名为ID继续匹配，匹配上就装配。没有匹配？报错。原因：因为我们按照变量名作为id。继续匹配的使用@Qualifier指定一个新的id，找到就匹配。

### 遇到的问题

#### Mybatis问题

org.apache.ibatis.session java.lang.NoSuchMethodError，左右解决不了。应该是缓存冲突啥之类的问题。

解决办法如下：

**一：**

<img src="https://gitee.com/hqinglau/img/raw/master/img/20210927184532.png" alt="image-20210927184532738" style="zoom:80%;" />

**二：点击蓝色的选项**

<img src="https://gitee.com/hqinglau/img/raw/master/img/20210927184600.png" alt="image-20210927184600464" style="zoom:80%;" />

**三：重启**

![image-20210927184927654](https://gitee.com/hqinglau/img/raw/master/img/20210927184927.png)

- 分别输入以下命令：
- mvn idea:idea
- mvn clean
- mvn test
- 执行完上述命令后一般就会解决问题

我的也解决了。

#### IDEA xml问题

IDEA并不会将`src\main\java\`目录下的配置文件（.xml）和资源文件(.properties)搬运到target目录下，解决办法是修改pom.xml文件，添加以下代码：

```xml
<build>
  <!--解决Intellij构建项目时，target/classes目录下不存在mapper.xml文件-->
    <!--解决Intellij构建项目时，target/classes目录下不存在mapper.xml文件-->
    <resources>
      <resource>
        <directory>src/main/java</directory>
        <includes>
          <include>**/*.xml</include>
        </includes>
      </resource>
    </resources>
  。。。
```

![image-20210927192046713](https://gitee.com/hqinglau/img/raw/master/img/20210927192046.png)

然后target就有了

![image-20210927192618881](https://gitee.com/hqinglau/img/raw/master/img/20210927192618.png)

#### jstl foreach不生效问题

在page directive中的isELIgnored属性用来指定是否忽略。格式为： ＜%@ page isELIgnored＝"true|false"%＞ 如果设定为真，那么JSP中的表达式被当成字符串处理。比如下面这个表达式\${2000 % 20}, 在isELIgnored＝"true"时输出为\${2000 % 20}，而isELIgnored＝"false"时输出为100。

![image-20210927193239749](https://gitee.com/hqinglau/img/raw/master/img/20210927193239.png)

![image-20210927194108074](https://gitee.com/hqinglau/img/raw/master/img/20210927194108.png)





![image-20210927193254616](https://gitee.com/hqinglau/img/raw/master/img/20210927193254.png)

## SSM其他

### 分页

搞一个页面结构体

```java
public class Page {
    int start = 0;
    int count = 5;
    int last = 0;
}
```

更改jsp：

```html
<div style="text-align:center">
  <a href="?start=0">首  页</a>
  <a href="?start=${page.start-page.count}">上一页</a>
  <a href="?start=${page.start+page.count}">下一页</a>
  <a href="?start=${page.last}">末  页</a>
</div>
```

更改controller

```java
@RequestMapping("listCategory")
public ModelAndView listCategory(Page page) {
    ModelAndView mav = new ModelAndView();
    List<Category> cs = categoryService.list(page);
    int total = categoryService.total();

    page.calculateLast(total);
    mav.addObject("cs",cs);
    mav.setViewName("listCategory");
    return mav;
}
```

mapper、service添加。

![image-20210927210552763](https://gitee.com/hqinglau/img/raw/master/img/20210927210552.png)

### 连接池

```xml
<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource" init-method="init" destroy-method="close">
    <!-- 基本属性 url、user、password -->
    <property name="url" value="jdbc:mysql://localhost:3306/how2java?characterEncoding=UTF-8" />
    <property name="username" value="root" />
    <property name="password" value="===" />
    <property name="driverClassName" value="com.mysql.cj.jdbc.Driver" />

    <!-- 配置初始化大小、最小、最大 -->
    <property name="initialSize" value="3" />
    <property name="minIdle" value="3" />
    <property name="maxActive" value="20" />

    <!-- 配置获取连接等待超时的时间 -->
    <property name="maxWait" value="60000" />

    <!-- 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒 -->
    <property name="timeBetweenEvictionRunsMillis" value="60000" />

    <!-- 配置一个连接在池中最小生存的时间，单位是毫秒 -->
    <property name="minEvictableIdleTimeMillis" value="300000" />

    <property name="validationQuery" value="SELECT 1" />
    <property name="testWhileIdle" value="true" />
    <property name="testOnBorrow" value="false" />
    <property name="testOnReturn" value="false" />

    <!-- 打开PSCache，并且指定每个连接上PSCache的大小 -->
    <property name="poolPreparedStatements" value="true" />
    <property name="maxPoolPreparedStatementPerConnectionSize" value="20" />
</bean>
```

### maven更新

点一下就行了。

![image-20210927211645798](https://gitee.com/hqinglau/img/raw/master/img/20210927211645.png)



### 事务

```java
@Override
@Transactional(propagation= Propagation.REQUIRED,rollbackForClassName="Exception")
public void addTwo() {
    Category c1 = new Category();
    c1.setName("短的名字");
    categoryMapper.add(c1);

    Category c2 = new Category();
    c2.setName("名字长对应字段放不下,名字长对应字段放不下,名字长对应字段放不下,名字长对应字段放不下,名字长对应字段放不下,名字长对应字段放不下,名字长对应字段放不下,名字长对应字段放不下,");
    categoryMapper.add(c2);
}
```

配置：

```xml
<tx:annotation-driven transaction-manager="transactionManager"/>
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource" />
</bean>
```

**事物传播行为介绍:** 

```shell
@Transactional(propagation=Propagation.REQUIRED) ：如果有事务, 那么加入事务, 没有的话新建一个(默认情况下)
@Transactional(propagation=Propagation.NOT_SUPPORTED) ：容器不为这个方法开启事务
@Transactional(propagation=Propagation.REQUIRES_NEW) ：不管是否存在事务,都创建一个新的事务,原来的挂起,新的执行完毕,继续执行老的事务
@Transactional(propagation=Propagation.MANDATORY) ：必须在一个已有的事务中执行,否则抛出异常
@Transactional(propagation=Propagation.NEVER) ：必须在一个没有的事务中执行,否则抛出异常(与Propagation.MANDATORY相反)
@Transactional(propagation=Propagation.SUPPORTS) ：如果其他bean调用这个方法,在其他bean中声明事务,那就用事务.如果其他bean没有声明事务,那就不用事务.
```



## 参考

[SPRING MVC](https://how2j.cn/module/98.html)

 [SSM](https://how2j.cn/module/126.html) 

[★spring@Autowired注解原理，通俗易懂（笔记）](https://blog.csdn.net/awodwde/article/details/107705778)

[Mybatis 出现org.apache.ibatis.io不存在，org.apache.ibatis.session不存在等错误](https://blog.csdn.net/qq_41740004/article/details/106661212)

[IDEA中Maven项目target目录下缺少包及mapper.xml文件](https://blog.csdn.net/liliuqing/article/details/85045384)

[spring配置注解context:annotation-config和context:component-scan区别](https://blog.csdn.net/fox_bert/article/details/80793030)

[mvc:default-servlet-handler标签的作用](https://blog.csdn.net/codejas/article/details/80055608)