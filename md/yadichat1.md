示例程序链接：[https://yadiChat.orzlinux.cn](https://yadichat.orzlinux.cn)

github地址：https://github.com/hqingLau/YadiChat

## 目录

![image-20211024114515289](https://gitee.com/hqinglau/img/raw/master/img/20211024114515.png)

## pom.xml

pom文件加入依赖，需要springboot，mybatis，mysql，数据库连接池druid，测试junit。

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-thymeleaf</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>druid</artifactId>
        <version>1.2.7</version>
    </dependency>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>8.0.26</version>
    </dependency>

    <dependency>
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-starter</artifactId>
        <version>2.1.3</version>
    </dependency>
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.13.2</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

## 数据库

新建数据库`YadiChat`，新建表`user`存放用户基本信息。包括`id`，`id_name`（唯一用于指代用户身份的id名，不可重复），`nick_name`（昵称，随意，可重复），密码（数据库存放MD5加密后的密码，不应存放明文密码）。

为了方便查找，将`id_name`设置唯一索引，`nick_name`也设置索引。

```sql
# 新建数据库 YadiChat
# 新建表user存放用户基本信息
CREATE TABLE user
(
    id INT NOT NULL AUTO_INCREMENT,
    id_name VARCHAR(128) NOT NULL, # 唯一的id名字
    nick_name VARCHAR(128) NOT NULL, # 昵称
    password VARCHAR(32) NOT NULL, # md5加密后的密码
    PRIMARY KEY(id),
    UNIQUE INDEX id_name(id_name),
    INDEX nick_name(nick_name)
);
```

`user`表如下所示：

```shell
mysql> select * from user;
+----+----------+-----------+----------------------------------+
| id | id_name  | nick_name | password                         |
+----+----------+-----------+----------------------------------+
|  1 | orzlinux | hqinglau  | 5b5091dcd75fcbc5b7d29ac59fe6b377 |
+----+----------+-----------+----------------------------------+
1 row in set (0.00 sec)
```

这样即使泄露了数据库，密码也不会泄露。因为MD5不能逆向计算。

## bean

和数据库字段对应，id设置了自增非空，也不看，没必要加入`User`类。

User.java:

```java
public class User {
    private  String idName; // 唯一名id
    private String nickName;
    private String password;
    // getter setter constructor
}
```

## Dao

数据库中用到了两个函数，一个是根据`id_name`查找，一个是插入用户。

```java
@Mapper
public interface UserDao {
    User selectUserByIdname(String idname);
    int insertUser(User user);
}
```

## application.yml

配置数据库连接参数，static路径，thymeleaf路径，mybatis参数，包括`mapper-locations`（mapper路径）和`type-alises-package`（mapper的xml中就不用写全类名了）。

```yaml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    type: com.alibaba.druid.pool.DruidDataSource
    url: jdbc:mysql://localhost:3306/YadiChat?serverTimezone=UTC&useUnicode=true&characterEncoding=utf8&useSSL=false
    username: root
    password: fahkdfh

  mvc:
    static-path-pattern: /static/**
  thymeleaf:
    prefix: classpath:/templates/
    suffix: .html
    cache: false

mybatis:
  type-aliases-package: cn.orzlinux.step1_signup_in.bean
  mapper-locations: classpath:mapper/*.xml
```

## UserDao.xml

数据库字段和User属性不对应时，可以用sql的as 别名。

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="cn.orzlinux.step1_signup_in.dao.UserDao">
    <select id="selectUserByIdname" resultType="User" parameterType="String">
        SELECT id_name idName, nick_name nickName,password FROM user WHERE id_name = #{idname}
    </select>

    <insert id="insertUser" parameterType="User" >
        insert into user(id_name,nick_name,password) values(#{idName},#{nickName},#{password})
    </insert>
</mapper>
```

## MD5

```java
public class MD5 {
    private final static String salt = "hqinglau@is;a^good_man";

    /**
     * 生成32位的MD5
     * @param msg
     * @return
     */
    public static String getMd5(String msg) {
        String base = msg+"/"+salt;
        return DigestUtils.md5DigestAsHex(base.getBytes(StandardCharsets.UTF_8));
    }
}
```



## UserService.java

`selectUserByIdname`根据`idname`从数据库中查找用户，`login`处理用户在表单中提交的登陆信息，如果查到了就返回用户。匹配需要匹配MD5后的密码。`regist`处理表单提交的注册请求。

```java
@Service
public class UserService {
    @Autowired
    UserDao userDao;

    public User selectUserByIdname(String idname) {
        return userDao.selectUserByIdname(idname);
    }

    public User login(User user) {
        String idname = user.getIdName();
        String password = user.getPassword();

        User u1 = userDao.selectUserByIdname(idname);
        if(u1==null) {
            return null;
        }
        if (u1.getPassword().equals(MD5.getMd5(password))) {
            return u1;
        }
        return null;
    }

    public boolean regist(User user) {
        user.setPassword(MD5.getMd5(user.getPassword()));
        int x = userDao.insertUser(user);
        return x>0;
    }
}
```

## UserController.java

```java
@Controller
public class UserController {
    @Autowired
    UserService userService;

    // 仅仅返回登录界面
    @RequestMapping("/signin")
    public String signin() {
        return "sign-in";
    }

    @RequestMapping("/")
    public String index() {
        return "index";
    }

    @RequestMapping("/home")
    public String home() {
        return "index";
    }

    @PostMapping("/login")
    public String login(Model model, HttpServletRequest request, HttpServletResponse response) {
        String idname = request.getParameter("idname");
        String password = request.getParameter("password");
        User user = userService.login(new User(idname,null,password));
        if (user==null) {
            return "redirect:/signin";
        }
        // 随便造一个session，放入
        String sess = MD5.getMd5(idname+"=^8&"+MD5.getMd5(password));
        request.getSession().setAttribute("userid", sess);
        model.addAttribute("userid",sess);
        return "redirect:/home";
    }

    // 注册
    @GetMapping("/signup")
    public String signup() {
        return "sign-up";
    }

    @PostMapping("/regist")
    public String regist(Model model, HttpServletRequest request, HttpServletResponse response) {
        String idname = request.getParameter("idname");
        String password = request.getParameter("password");
        String nickname = request.getParameter("nickname");
        try {
            boolean ret = userService.regist(new User(idname,nickname,password));
            if (!ret) {
                return "redirect:/signup";
            }
            // 随便造一个session，放入
            String sess = idname;
            request.getSession().setAttribute("userid", sess);
            return "redirect:/";
        } catch (Exception e) {
            return "redirect:/signup";
        }

    }

    @RequestMapping("/somepage")
    @ResponseBody
    public String somepage(HttpServletRequest request, HttpServletResponse response) {
        return "private msg";
    }
}
```

其中，登陆、注册成功都需要加入用户session。

> 服务器创建session出来后，会把session的id号，以cookie的形式回写给客户机，这样，只要客户机的浏览器不关，再去访问服务器时，都会带着session的id号去，服务器发现客户机浏览器带session id过来了，就会使用内存中与之对应的session为之服务。

这篇文章不错：[session的到底是做什么的？](https://blog.csdn.net/h19910518/article/details/79348051)

## interceptor

SessionConfiguration.java

```java
@Configuration
public class SessionConfiguration implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new UserInterceptor()).addPathPatterns("/**").excludePathPatterns("/static/**");
    }
}
```

UserInterceptor.java

```java
public class UserInterceptor implements HandlerInterceptor  {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String uri = request.getRequestURI();
        if("/".equals(uri)||"/error".equals(uri)||"/signin".equals(uri)||"/login".equals(uri)||"/signup".equals(uri)||"/regist".equals(uri)) {
            return true;
        }
        Object o = request.getSession().getAttribute("userid");
        if(null == o) {
            response.sendRedirect("/signin");
            return false;
        }
        return true;
    }
}
```

如果session存在并且有userid属性，认为是已登陆用户的操作，否则重定向到登陆页面。

## 效果

首页：

![image-20211024121348487](https://gitee.com/hqinglau/img/raw/master/img/20211024121348.png)

点击somepage，若未登陆会跳转到登陆界面：

![image-20211024121415216](https://gitee.com/hqinglau/img/raw/master/img/20211024121415.png)

若未注册点击右下角注册，进入注册界面：

![image-20211024121454555](https://gitee.com/hqinglau/img/raw/master/img/20211024121454.png)

登陆之后，主页点击somepage，进入：

![image-20211024121604856](https://gitee.com/hqinglau/img/raw/master/img/20211024121604.png)

示例程序链接：[https://yadiChat.orzlinux.cn](https://yadichat.orzlinux.cn)

github地址：https://github.com/hqingLau/YadiChat



