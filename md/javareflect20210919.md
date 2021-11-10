**注解提供了一种安全的类似注释的机制，用来将任何的信息或元数据（metadata）与程序元素（类、方法、成员变量等）进行关联。**

**JDK中预定义的一些注解**

`@Override`：检测被该注解标注的方法是否是继承自父类（接口）的

`@Deprecated`：该注解标注的方法表示该方法已过时

`@SuppressWarnings`: 压制警告的注解,一般传递的参数为all，如：@SuppressWarnings("all")

**自定义注解**

格式：

```java
public @interface 注解名称{
     属性列表;
}
```


本质：注解本质上使用一个接口，该接口默认继承Annotation接口

**属性：接口中的抽象方法**

要求：

属性（抽象方法）的返回值类型需要是以下几种类型

- 基本数据类型
- String
- 枚举
- 注解
- 以上类型的数组

定义了属性，在使用时需要给属性进行赋值：

- 如果定义属性时，使用default关键字给属性默认的初始值，则使用注解时，可以不进行属性的赋值

- 如果只有一个属性需要赋值，且属性的名称是value，则value可以省略，直接定义值即可。
- 数组赋值时，值使用{}包裹，如果数组汇总只有一个值，则{}可以省略

**元注解：用于描述注解的注解**

```java
@Target：描述注解能够作用的位置，如：类、方法等
	ElementType取值：
        TYPE:可以作用在类上
        METHOD:可以作用在方法上
        FIELD:可以作用在成员变量上
@Retention：描述注解被保留的阶段
    RetentionPolicy取值
        @Retention(RetentionPolicy.RUNTIME)
        （1）RUNTIME:一般自定义的注解取RUNTIME即可，表示表描述的注解作用于运行时阶段
@Documented:描述注解是否被抽取到api文档中
@Inherited:描述注解是否会被子类继承
```

>  **注解的常见用途**：
>
> 生成文档的注解，如@author，@param。
>
> 跟踪代码依赖性，实现替代配置文件功能，如spring mvc的注解。
>
> 编译时进行格式检查，如@override。
>
> 编译时进行代码生成补全，如lombok插件的@Data。

## 自定义注解代替配置文件

**创建一个普通类：**

```java
package com.lhq.annotation;

public class Person {
    private String name;
    private int age;

    ...

    public void eat(){
        System.out.println("调用eat方法！");
    }
}
```

**创建注解：**

```java
@Target(ElementType.TYPE) // 只作用在类上
@Retention(RetentionPolicy.RUNTIME)
public @interface pro {
    String className();
    String methodName();
}
```

**使用一：**

```java
@pro(className = "com.lhq.annotation.Person",methodName = "eat")
public class Demo {
    public static void main(String[] args) throws ClassNotFoundException, IllegalAccessException, InstantiationException, NoSuchMethodException, InvocationTargetException {
        // 获取该类的字节码文件
        Class demoClass = Demo.class;
        pro p = (pro)demoClass.getAnnotation(pro.class);
        // 这行代码相当于在内存中生产了一个注解接口的子类实现对象，相当于如下代码：

    /*class proImpl implements pro{
        @Override
        public String className() {
            return "com.lhq.annotation.Person";
        }
        @Override
        public String methodName() {
            return "eat";
        }
     */
        String className = p.className();
        String methodName = p.methodName();

        // 根据反射机制类名获取类对象
        Class cls = Class.forName(className);
        // 根据类对象创建对象
        Object o = cls.newInstance();
        Method method = cls.getMethod(methodName);
        //调用invoke方法执行对象中的method方法
        method.invoke(o); // output :调用eat方法！
    }
}
```

**使用二：配置文件**

```java
// yadi.properties
// className=com.lhq.annotation.Person
// methodName=eat

public class Demo2 {
    public static void main(String[] args) throws ClassNotFoundException, IllegalAccessException, InstantiationException, NoSuchMethodException, InvocationTargetException, IOException {
        // 创建Properties对象
        Properties pro = new Properties();

        //通过本类的类加载器将properties文件中的内容加载到pro对象中
        ClassLoader classLoader = Demo2.class.getClassLoader();
        InputStream is = classLoader.getResourceAsStream("com/lhq/annotation/yadi.properties");
        pro.load(is);

        //利用properties对象中getProperty方法利用key获取value
        String className = pro.getProperty("className");
        String methodName = pro.getProperty("methodName");

        // 根据反射机制类名获取类对象
        Class cls = Class.forName(className);
        // 根据类对象创建对象
        Object o = cls.newInstance();
        Method method = cls.getMethod(methodName);
        //调用invoke方法执行对象中的method方法
        method.invoke(o); //output: 调用eat方法！
    }
}

```

## 通过注解进行赋值和校验

**赋值注解：**

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD,ElementType.METHOD})
@Inherited
public @interface InitSex {
    enum SEX_TYPE {MAN,WOMAN};
    SEX_TYPE sex() default SEX_TYPE.MAN;
}
```

**校验注解：**

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD,ElementType.METHOD})
@Inherited
public @interface ValidateAge {
    int min() default 18;
    int max() default 99;
    int value() default 20;
}

```

**普通类：**使用注解

```java
public class User {
    private String username;
    @ValidateAge(min = 20,max = 35,value = 22)
    private int age;
    @InitSex(sex = InitSex.SEX_TYPE.MAN)
    private String sex;
	...
}
```

**测试：**

```java
public class Test {
    public static void main(String[] args) throws IllegalAccessException {
        User user = new User();
        initUser(user);
        // 年龄为0，校验为通过情况
        // output: 完成属性值的修改，修改值为:MAN
        boolean checkResult = checkUser(user);
        // output: 年龄值不符合条件
        printResult(checkResult);
        // output: 校验未通过
        // 重新设置年龄，校验通过情况
        user.setAge(22);
        checkResult = checkUser(user);
        printResult(checkResult);
        // output: 校验通过
    }

    static void initUser(User user) throws IllegalAccessException {
        // 获取User类中所有的属性(getFields无法获得private属性: )
        // getFields: reflecting all the accessible public fields of the class
        // getDeclaredFields: objects reflecting all the fields declared by the class or interface
        Field[] fields = User.class.getDeclaredFields();
        for(Field field:fields) {
            if(field.isAnnotationPresent(InitSex.class)) {
                InitSex init = field.getAnnotation(InitSex.class);
                field.setAccessible(true);
                // 设置属性性别
                field.set(user,init.sex().toString());
                System.out.println("完成属性值的修改，修改值为:" + init.sex().toString());
            }
        }
    }

    static boolean checkUser(User user) throws IllegalAccessException {
        // 获取User类中所有的属性(getFields无法获得private属性)
        Field[] fields = User.class.getDeclaredFields();
        boolean result = true;
        // 遍历所有属性
        for (Field field : fields) {
            // 如果属性上有此注解，则进行赋值操作
            if (field.isAnnotationPresent(ValidateAge.class)) {
                ValidateAge validateAge = field.getAnnotation(ValidateAge.class);
                field.setAccessible(true);
                int age = (Integer) field.get(user);
                if (age < validateAge.min() || age > validateAge.max()) {
                    result = false;
                    System.out.println("年龄值不符合条件");
                }
            }
        }
        return result;
    }

    static void printResult(boolean checkResult) {
        if (checkResult) {
            System.out.println("校验通过");
        } else {
            System.out.println("校验未通过");
        }
    }
}
```

## 给private属性赋值

```java
public class PrivateSet {
    private String read;
    public String getReadOnly() {
        return read;
    }

    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException {
        PrivateSet t = new PrivateSet();
        Field f = t.getClass().getDeclaredField("read");
        f.setAccessible(true);
        f.set(t,"write something...");
        System.out.println(t.getReadOnly());
        // output: write something...
    }
}
```

## 为啥都说慢

`invoke`源码如下：

```java
public final class Method extends Executable {
    
    ...
        
    @CallerSensitive
    public Object invoke(Object obj, Object... args)
        throws IllegalAccessException, IllegalArgumentException,
           InvocationTargetException
    {
        // 权限检查
        if (!override) {
            if (!Reflection.quickCheckMemberAccess(clazz, modifiers)) {
                Class<?> caller = Reflection.getCallerClass();
                checkAccess(caller, clazz, obj, modifiers);
            }
        }
        // ============================================
        MethodAccessor ma = methodAccessor;             // read volatile
        if (ma == null) {
            ma = acquireMethodAccessor();
        }
        return ma.invoke(obj, args);
        // ============================================
    }
}
```

![image-20210919214739409](https://gitee.com/hqinglau/img/raw/master/img/20210919214741.png)

其实在`Method`对象内部维护了一个接口`MethodAccessor`，该接口有二个实现类，其中`NativeMethodAccessorImpl`用来实现本地`native`调用。而`DelegatingMethodAccessorImpl`顾名思义，是一个委派实现类，该方法将`invoke`操作委派给了`native`方法。

```java
public class ReflectSlow {
    public static void target(int i) {
        new Exception("#"+i).printStackTrace();
    }

    public static void main(String[] args) throws ClassNotFoundException, NoSuchMethodException, InvocationTargetException, IllegalAccessException {
        Class<?> kclass = Class.forName("com.lhq.annotation2.ReflectSlow");
        // 首先Class.forName属于native方法，native方法就要经过语言执行层面转换。也就是java到c再到java的切换。
        Method method = kclass.getMethod("target", int.class);
        // 而getMethod这个操作则会遍历该类的公有方法，如果没有命中，则还要去父类中查找。并且返回该
        // method对象的一份copy。在查找成功之后，这份copy对象，则会占用堆空间，而无法进行内联优化，
        // 相反还会引起gc频率的提高。对性能也是一份影响。
        method.invoke(null,0);
    }
}
```

输出：

```java
/** output:
* java.lang.Exception: #0
* 	at com.lhq.annotation2.ReflectSlow.target(ReflectSlow.java:8)
* 	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
* 	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
* 	at sun.reflect.DelegatingMethodAccessorImpl. invoke(DelegatingMethodAccessorImpl.java:43)
* 	at java.lang.reflect.Method.invoke(Method.java:498)
* 	at com.lhq.annotation2.ReflectSlow.main(ReflectSlow.java:14)
*/
```

换成循环多次`method.invoke(null,i);`：

![image-20210919215203623](https://gitee.com/hqinglau/img/raw/master/img/20210919215205.png)

在程序调用第16次的时候，调用栈更改成了`GeneratedMethodAccessor1`，而不再是`native`方法。这是因为 Jvm 维护了一个阈值`-Dsun.reflect.inflationThreshold`，默认为15。当反射`native`调用超过15次就会触发 Jvm 的动态生成字节码，之后的操作，全部都会调用该动态实现。

动态实现与`native`实现相比，动态实现的效率要快的多，这是因为`native`的实现要在 Java 语言层面切换到 C 语言，然后再次切换到 Java 语言。但是，因为动态实现第一次生成的时候要生成字节码，而这个操作是比较耗时的。所以相比较起来单独一次调用的时候`native`反而要比动态实现快的多。

> 总结一下就是 委派实现，用的少了就直接 native 。虽然 native 方法要切换，但是生成字节码更慢，invoke用的多了生成字节码就值的了，就生成字节码，省去了切换，就快了。二者都是：慢。

还有**注解处理器**，另开一篇吧。

## 参考

[一篇文章，全面掌握Java自定义注解（Annontation）](https://zhuanlan.zhihu.com/p/60730622)

[Java注解之自定义](https://juejin.cn/post/6844903773060464654)

[Java 使用自定义注解替代配置文件案例（注解介绍）](https://blog.csdn.net/guanmao4322/article/details/106043794)

[反射为什么慢](https://blog.csdn.net/xqlovetyj/article/details/82798864)

[反射调用为什么慢？细推反射细节！](https://blog.csdn.net/zhang6622056/article/details/98950855)