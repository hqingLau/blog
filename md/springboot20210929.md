## restful

Representational State Transfer

![image-20210929095807025](https://gitee.com/hqinglau/img/raw/master/img/20210929095814.png)



```java
@Controller
public class CategoryController {
    @Autowired
    CategoryMapper categoryMapper;

    @GetMapping("/categories")
    //在参数里接受当前是第几页 start ，以及每页显示多少条数据 size。 默认值分别是0和5。
    public String listCategory(Model m, @RequestParam(value = "start", defaultValue = "0") int start,
                               @RequestParam(value = "size", defaultValue = "5") int size) throws Exception {
        //根据start,size进行分页，并且设置id 倒排序
        PageHelper.startPage(start,size,"id desc");
        //因为PageHelper的作用，这里就会返回当前分页的集合了
        List<Category> cs=categoryMapper.findAll();
        //根据返回的集合，创建PageInfo对象
        PageInfo<Category> page = new PageInfo<>(cs);
        m.addAttribute("page", page);
        return "listCategory";
    }
    @DeleteMapping("/categories/{id}")
    public String deleteCategory(Category c) throws Exception {
        categoryMapper.delete(c.getId());
        return "redirect:/categories";
    }
}
```

jsp:

```html
<%@ page language="java" contentType="text/html; charset=UTF-8"
         pageEncoding="UTF-8"%>

<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c"%>
<script type="text/javascript" src="js/jquery.min.js"></script>

<script type="text/javascript">
    /*将post method 改变为delete*/
    $(function(){
        $(".delete").click(function(){
            var href=$(this).attr("href");
            $("#formdelete").attr("action",href).submit();
            return false;
        })
    })
</script>
<div align="center">

</div>

<div style="width:500px;margin:20px auto;text-align: center">
    <table align='center' border='1' cellspacing='0'>
        <tr>
            <td>id</td>
            <td>name</td>
            <td>编辑</td>
            <td>删除</td>
        </tr>
        <c:forEach items="${page.list}" var="c" varStatus="st">
            <tr>
                <td>${c.id}</td>
                <td>${c.name}</td>
                <td><a href="categories/${c.id}">编辑</a></td>
                <td><a class="delete" href="categories/${c.id}">删除</a></td>
            </tr>
        </c:forEach>

    </table>
    <br>
    <div>
        <a href="?start=1">[首  页]</a>
        <a href="?start=${page.pageNum-1}">[上一页]</a>
        <a href="?start=${page.pageNum+1}">[下一页]</a>
        <a href="?start=${page.pages}">[末  页]</a>
    </div>
    <br>
    <form action="categories" method="post">

        name: <input name="name"> <br>
        <button type="submit">提交</button>

    </form>

    <form id="formdelete" action="" method="POST" >
        <input type="hidden" name="_method" value="DELETE">
    </form>
</div>
```

发现在表单中添加

```html
<input type="hidden" name="_method" value="delete"/>
```

后台用`@deleteMapping（/xxx/{id}）`这种方式会报405（不允许的访问方式）。在 Spring Boot 的 META-INF/ spring-configuration-metadata.json 配置文件中，默认是关闭Spring 的 hiddenmethod 过滤器的

然后我们需要在springBoot的配置文件中将它手动开启即可；

```shell
spring.mvc.hiddenmethod.filter.enabled=true
```

效果：

![image-20210929105740838](https://gitee.com/hqinglau/img/raw/master/img/20210929105740.png)

## jQuery AJAX

```javascript
// 查找<div id="abc">:
var div = $('#abc');

var ps = $('p'); // 返回所有<p>节点

var a = $('.red'); // 所有节点包含`class="red"`都将返回
// 例如:
// <div class="red">...</div>
// <p class="green red">...</p>

// 按属性查找
var email = $('[name=email]'); // 找出<??? name="email">
var passwordInput = $('[type=password]'); // 找出<??? type="password">
var a = $('[items="A B"]'); // 找出<??? items="A B">
```

AJAX = 异步 JavaScript 和 XML（Asynchronous JavaScript and XML）。

简短地说，在不重载整个网页的情况下，AJAX 通过后台加载数据，并在网页上进行显示。

demo_test.txt

```html
<h2>jQuery AJAX 是个非常棒的功能！</h2>
<p id="p1">这是段落的一些文本。</p>
```

html

```html
<script>
$(document).ready(function(){
  $("button").click(function(){
    // 下面的例子会把文件 "demo_test.txt" 的内容加载到指定的 <div> 元素中
    //$("#div1").load("/try/ajax/demo_test.txt");
    // 下面的例子会把文件 "demo_test.txt" id=p1 的内容加载到指定的 <div> 元素中
    $("#div1").load("/try/ajax/demo_test.txt #p1");
  });
});
</script>
</head>
<body>

<div id="div1"><h2>使用 jQuery AJAX 修改文本</h2></div>
<button>获取外部文本</button>

</body>
```

可以设置回调函数，参数：

- responseTxt 调用成功的结果内容
- statusTxt 调用的状态
- xhr 包含XMLHttpRequest对象

```javascript
$(document).ready(function(){
  $("button").click(function(){
    $("#div1").load("/try/ajax/demo_test.txt",function(responseTxt,statusTxt,xhr){
      if(statusTxt=="success")
        alert("外部内容加载成功!");
      if(statusTxt=="error")
        alert("Error: "+xhr.status+": "+xhr.statusText);
    });
  });
});
```

**get方法**

```javascript
$("button").click(function(){
// <?php
// echo '这是个从PHP文件中读取的数据。';
// ?>
  $.get("demo_test.php",function(data,status){
    alert("数据: " + data + "\n状态: " + status);
  });
});
```

**post**

```javascript
$("button").click(function(){
    $.post("/try/ajax/demo_test_post.php",
    {
        name:"hqinglau",
        url:"https://orzlinux.cn"
    },
    function(data,status){
        alert("数据: \n" + data + "\n状态: " + status);
    });
});
```

## SpringBoot前后端分离

html

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
    <script type="text/javascript" src="js/jquery.min.js"></script>
</head>
<body>
<form >
    id：<input type="text" id="id" value="123" /><br/>
    名称：<input type="text" id="name" value="category xxx"/><br/>
    <input type="button" value="提交" id="sender">
</form>
<div id="messageDiv"></div>

<script>
    $('#sender').click(function (){
        var id = document.getElementById('id').value;
        var name = document.getElementById('name').value;
        var category = {"name":name,"id":id};
        var jsonData = JSON.stringify(category);
        var page="category";
        $.ajax({
            type:"put",
            url:page,
            data:jsonData,
            dataType:"json",
            contentType:"application/json;charset=UTF-8",
            success:function (result) {

            }
        });
        alert("chenggong");
    })
</script>
</body>
</html>
```

controller

```java
@RestController
public class CategoryController {
    @Autowired
    CategoryMapper categoryMapper;

    @GetMapping("/category")
    //在参数里接受当前是第几页 start ，以及每页显示多少条数据 size。 默认值分别是0和5。
    public List<Category> listCategory(Model m, @RequestParam(value = "start", defaultValue = "1") int start,
                                       @RequestParam(value = "size", defaultValue = "5") int size) throws Exception {
        //根据start,size进行分页
        PageHelper.startPage(start,size,"id");
        //因为PageHelper的作用，这里就会返回当前分页的集合了
        List<Category> cs=categoryMapper.findAll();
        //根据返回的集合，创建PageInfo对象
        PageInfo<Category> page = new PageInfo<>(cs);
//        m.addAttribute("page", page);
        return page.getList();
    }
    @GetMapping("/category/{id}")
    public Category getCategory(@PathVariable("id") int id) throws Exception {
        Category c = categoryMapper.get(id);
        System.out.println(c);
        return c;
    }

    @PutMapping("/category")
    public void addCategory(@RequestBody Category category) throws Exception {
        System.out.println("收到数据："+category);
    }
}
```

![image-20210929131457438](https://gitee.com/hqinglau/img/raw/master/img/20210929131457.png)

## Thymeleaf

**/ˈtaɪmˌlɪːf/**

Thymeleaf是用来开发Web和独立环境项目的服务器端的Java模版引擎

Spring官方支持的服务的渲染模板中，并不包含jsp。而是Thymeleaf和Freemarker等，而Thymeleaf与SpringMVC的视图技术，及SpringBoot的自动化配置集成非常完美，几乎没有任何成本，你只用关注Thymeleaf的语法即可。

**动静结合**：Thymeleaf 在有网络和无网络的环境下皆可运行，即它可以让美工在浏览器查看页面的静态效果，也可以让程序员在服务器查看带数据的动态页面效果。这是由于它支持 html 原型，然后在 html 标签里增加额外的属性来达到模板+数据的展示方式。浏览器解释 html 时会忽略未定义的标签属性，所以 thymeleaf 的模板可以静态地运行；当有数据返回到页面时，Thymeleaf 标签会动态地替换掉静态内容，使页面动态显示。

### 示例

引入依赖后，添加配置（有默认）：

```shell
spring.thymeleaf.encoding=utf-8
spring.thymeleaf.prefix=classpath:/templates/
spring.thymeleaf.suffix=.html
spring.thymeleaf.cache=false
```

**classpath**

**target->classes即为classpath，任何我们需要在classpath前缀中获取的资源都必须在target->classes文件夹中找到**。但是在idea项目中只有被标记为`Resource Folders`的文件夹下的文件才会被添加至target->classes。

Tomcat下的Web应用有两个预置的classpath :

- WEB-INF/classes

- WEB-INF/lib

classpath\*：不仅包含class路径，还包括jar文件中(class路径)进行查找。

![image-20210929134410886](https://gitee.com/hqinglau/img/raw/master/img/20210929134410.png)

hello1.html放到templates文件夹下：

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    <h1 th:text="${msg}">默认数字</h1>
</body>
</html>
```

controller:

```java
@Controller
public class ThemeleafController {
    @RequestMapping("/test")
    public String test(Model model) {
        model.addAttribute("msg","orzlinux.cn");
        return "hello1";
    }
}
```

测试：

![image-20210929135311225](https://gitee.com/hqinglau/img/raw/master/img/20210929135311.png)

也可以是对象：

![image-20210929141141001](https://gitee.com/hqinglau/img/raw/master/img/20210929141141.png)

常用语法

Thymeleaf的主要作用是把model中的数据渲染到html中，因此其语法主要是如何解析model中的数据。

`th:text`指令出于安全考虑，会把表达式读取到的值进行处理，防止html的注入。

例如，`<p>你好</p>`将会被格式化输出为`$lt;p$gt;你好$lt;/p$lt;`。

如果想要不进行格式化输出，而是要输出原始内容，则使用`th:utext`来代替.

### 方法调用

```html
<div th:object="${site}">
    <!--1. 方法调用-->
    <!--split()方法接受的是正则表达式，所以传入的”.”就变成了正则表达式的关键字，表示除换行符之外的任意字符。所以，这里需要转义”\.”或”[.]”。-->
    <!--用'.'会访问报错-->
    <p> Name: <span th:text="*{name.split('\.')[0]}">hqinglau</span></p>
    <p> id: <span th:text="*{id}">1</span></p>
</div>
```

![image-20210929143955076](https://gitee.com/hqinglau/img/raw/master/img/20210929143955.png)

### 内置对象

环境对象

![image-20210929144049277](https://gitee.com/hqinglau/img/raw/master/img/20210929144049.png)

全局对象

![image-20210929144112004](https://gitee.com/hqinglau/img/raw/master/img/20210929144112.png)

```html
<div>
    <!--controller：model.addAttribute("today",new Date());-->
    今天是：<span th:text="${#dates.format(today,'yyyy-MM-dd')}">1900-01-01</span>
    <!--输出：今天是：2021-09-29-->
</div>
```

### th:object

频繁读取对象情况，可以简写：

```html
<h1>
    <p> Name: <span th:text="${site.name}">hqinglau</span>
    <p> Name: <span th:text="${site.id}">1</span></p>
</h1>

<div>======================</div>
<h1 th:object="${site}">
    <p> Name: <span th:text="*{name}">hqinglau</span>
    <p> Name: <span th:text="*{id}">1</span></p>
</h1>
```

![image-20210929141738233](https://gitee.com/hqinglau/img/raw/master/img/20210929141738.png)



![image-20210929155115604](https://gitee.com/hqinglau/img/raw/master/img/20210929155115.png)

### el 表达式 和 ognl表达式

el (expression language)

用法：

```html
<some:tag value="${expr}"/>
<another:tag value="#{expr}"/>
```

OGNL是Object Graph Navigation Language的简称

<img src="https://gitee.com/hqinglau/img/raw/master/img/20210929140416.png" alt="image-20210929140416116" style="zoom:80%;" />



## 参考

 [SPRINGBOOT](https://how2j.cn/module/143.html) 

[SpringBoot@DeleteMapping(/xxx/{id})请求报405解决方案](https://blog.csdn.net/qq_33879627/article/details/106554777)

[jQuery选择器](https://www.liaoxuefeng.com/wiki/1022910821149312/1023023555539648)

[jQuery Ajax](https://www.runoob.com/jquery/jquery-ajax-intro.html)

[Thymeleaf入门到吃灰](https://www.cnblogs.com/msi-chen/p/10974009.html)

[el 表达式 和 ognl表达式](https://www.cnblogs.com/caiyao/p/4198667.html)