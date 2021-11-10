> 本文为[Java高并发秒杀API之Service层](http://www.imooc.com/learn/631)课程笔记。

编辑器：IDEA

java版本：java8

前文：[秒杀系统环境搭建与DAO层设计](https://orzlinux.cn/blog/javaseckill1-20211004.html)

## 秒杀业务接口与实现

DAO层：接口设计、SQL编写

Service：业务，DAO拼接等逻辑

代码和SQL分离，方便review。

### service接口设计

目录如下：

![image-20211005132704868](https://gitee.com/hqinglau/img/raw/master/img/20211005132705.png)

首先是SecKillService接口的设计：

```java
/**
 * 业务接口：站在使用者角度设计接口
 * 三个方面：
 * 方法定义粒度 - 方便调用
 * 参数 - 简练直接
 * 返回类型 - return（类型、异常）
 */
public interface SecKillService {
    /**
     * 查询所有秒杀记录
     * @return
     */
    List<SecKill> getSecKillList();

    SecKill getById(long seckillId);

    /**
     * 秒杀开启时，输出秒杀接口地址
     * 否则输出系统时间和秒杀时间
     * @param seckillId
     */
    Exposer exportSecKillUrl(long seckillId);

    // 执行秒杀操作验证MD5秒杀地址，抛三个异常（有继承关系）
    // 是为了更精确的抛出异常
    SeckillExecution executeSeckill(long seckillId, long userPhone, String md5)
        throws SeckillException, RepeatKillException,SeckillCloseException;

}
```

`exportSecKillUrl`函数用来暴露出接口的地址，用一个专门的dto类`Exposer`来实现：

```java
/**
 * 暴露秒杀地址DTO
 */
public class Exposer {
    // 是否开启秒杀
    private boolean exposed;
    // 加密措施
    private String md5;

    private long seckillId;
    // 系统当前时间
    private long now;

    // 秒杀开启结束时间
    private long start;
    private long end;
    // constructor getter setter
}
```

这里有一个md5，是为了构造秒杀地址，防止提前猜出秒杀地址，执行作弊手段。

> **MD5**是一个安全的散列算法，输入两个不同的明文不会得到相同的输出值，根据输出值，不能得到原始的明文，即其过程不可逆；所以要解密MD5没有现成的算法，只能用穷举法。

`executeSeckill`函数表示执行秒杀，应该返回执行的结果相关信息：

```java
/**
 * 封装秒杀执行后结果
 */
public class SeckillExecution {
    private long seckillId;
    // 秒杀结果状态
    private int state;

    // 状态信息
    private String stateInfo;

    // 秒杀成功对象
    private SuccessKilled successKilled;
    // constructor getter setter
}
```

执行过程中可能会抛出异常。这里面用到了几个异常：`SeckillException`, `RepeatKillException`, `SeckillCloseException`。

SeckillException.java，这个是其他两个的父类，除了那两个精确的异常，都可以返回这个异常。

```java
// 秒杀相关业务异常
public class SeckillException extends RuntimeException {
    public SeckillException(String message) {
        super(message);
    }

    public SeckillException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

RepeatKillException是重复秒杀异常，一个用户一件商品只能秒杀一次：

```java
// 重复秒杀异常，运行期异常
public class RepeatKillException extends SeckillException {
    public RepeatKillException(String message) {
        super(message);
    }

    public RepeatKillException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

同理，SeckillCloseException是秒杀关闭异常，秒杀结束了还在抢，返回异常。

```java
// 秒杀关闭异常，如时间到了，库存没了
public class SeckillCloseException extends SeckillException {
    public SeckillCloseException(String message) {
        super(message);
    }

    public SeckillCloseException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

### service接口实现

首先开启扫描。

spring-service.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd">
    <!--扫描service包下使用注解的类型-->
    <context:component-scan base-package="cn.orzlinux.service"/>

</beans>
```

SecKillServiceImpl实现类实现了SecKillService接口的方法：指定日志对象，DAO对象。

```java
@Service
public class SecKillServiceImpl implements SecKillService {
    // 使用指定类初始化日志对象，在日志输出的时候，可以打印出日志信息所在类
    private Logger logger = LoggerFactory.getLogger(this.getClass());

    // 需要两个 dao的配合
    // 注入service依赖
    @Resource
    private SecKillDao secKillDao;
    @Resource
    private SuccessKilledDao successKilledDao;
    
    //...
}
```

查询比较简单，直接调用DAO方法：

```java
@Override
public List<SecKill> getSecKillList() {
    // 这里因为只有四条秒杀商品
    return secKillDao.queryAll(0,4);
}

@Override
public SecKill getById(long seckillId) {
    return secKillDao.queryById(seckillId);
}
```

暴露秒杀接口函数：

```java
/**
 * 秒杀开启时，输出秒杀接口地址
 * 否则输出系统时间和秒杀时间
 *
 * @param seckillId
 */
@Override
public Exposer exportSecKillUrl(long seckillId) {
    SecKill secKill = secKillDao.queryById(seckillId);
    if(secKill == null) {
        // 查不到id，false
        return new Exposer(false,seckillId);
    }
    Date startTime = secKill.getStartTime();
    Date endTime = secKill.getEndTime();
    Date nowTime = new Date();
    if(nowTime.getTime()<startTime.getTime()
            || nowTime.getTime()>endTime.getTime()) {
        return new Exposer(false,seckillId, nowTime.getTime(),
                startTime.getTime(),endTime.getTime());
    }
    // 不可逆
    String md5 = getMD5(seckillId);
    return new Exposer(true,md5,seckillId);
}
```

getMD5是一个自定义函数：

```java
// 加入盐、混淆效果，如瞎打一下:
private final String slat="lf,ad.ga.dfgm;adrktpqerml[fasedfa]";
private String getMD5(long seckillId) {
    String base = seckillId+"/orzlinux.cn/"+slat;
    // spring已有实现
    String md5 = DigestUtils.md5DigestAsHex(base.getBytes(StandardCharsets.UTF_8));
    return md5;
}
```

> 如果直接对密码进行散列，那么黑客可以对通过获得这个密码散列值，然后通过查散列值字典（例如MD5密码破解网站），得到某用户的密码。加Salt可以一定程度上解决这一问题。所谓加Salt方法，就是加点”佐料”。其基本想法是这样的：当用户首次提供密码时（通常是注册时），由系统自动往这个密码里撒一些“佐料”，然后再散列。而当用户登录时，系统为用户提供的代码撒上同样的“佐料”，然后散列，再比较散列值，已确定密码是否正确。这里的“佐料”被称作“Salt值”，这个值是由系统随机生成的，并且只有系统知道。

执行秒杀函数，这里面牵扯到编译异常和运行时异常。

**异常**

编译时异常：编译成字节码过程中可能出现的异常。

运行时异常：将字节码加载到内存、运行类时出现的异常。

异常体系结构：

```shell
 * java.lang.Throwable
 * 		|-----java.lang.Error:一般不编写针对性的代码进行处理。
 * 		|-----java.lang.Exception:可以进行异常的处理
 * 			|------编译时异常(checked)
 * 					|-----IOException
 * 						|-----FileNotFoundException
 * 					|-----ClassNotFoundException
 * 			|------运行时异常(unchecked,RuntimeException)
 * 					|-----NullPointerException
 * 					|-----ArrayIndexOutOfBoundsException
 * 					|-----ClassCastException
 * 					|-----NumberFormatException
 * 					|-----InputMismatchException
 * 					|-----ArithmeticException
```

使用`try-catch-finally`处理编译时异常，是得程序在编译时就不再报错，但是运行时仍可能报错。相当于我们使用`try-catch-finally`将一个编译时可能出现的异常，延迟到运行时出现。开发中，由于运行时异常比较常见，所以我们通常就不针对运行时异常编写`try-catch-finally`了。针对于编译时异常，我们说一定要考虑异常的处理。

executeSeckill.java

```java
@Override
public SeckillExecution executeSeckill(long seckillId, long userPhone, String md5) throws
        SeckillException, RepeatKillException, SeckillException {
    if(md5==null || !md5.equals(getMD5(seckillId))) {
        // 秒杀的数据被重写修改了
        throw new SeckillException("seckill data rewrite");
    }
    // 执行秒杀逻辑：减库存、加记录购买行为
    Date nowTime = new Date();
    // 减库存
    int updateCount = secKillDao.reduceNumber(seckillId,nowTime);
    try {
        if(updateCount<=0) {
            // 没有更新记录,秒杀结束
            throw new SeckillCloseException("seckill is closed");
        } else {
            // 记录购买行为
            int insertCount = successKilledDao.insertSuccessKilled(seckillId,userPhone);
            if(insertCount<=0) {
                // 重复秒杀
                throw new RepeatKillException("seckill repeated");
            } else {
                //秒杀成功
                SuccessKilled successKilled = successKilledDao.queryByIdWithSeckill(seckillId,userPhone);
                //return new SeckillExecution(seckillId,1,"秒杀成功",successKilled);
                return new SeckillExecution(seckillId, SecKillStatEnum.SUCCESS,successKilled);
            }
        }
    } catch (SeckillCloseException | RepeatKillException e1){
        throw e1;
    } catch (Exception e) {
        logger.error(e.getMessage(),e);
        // 所有编译期异常转化为运行期异常，这样spring才能回滚
        throw new SeckillException("seckill inner error"+e.getMessage());
    }
}
```

`SeckillCloseException`和`RepeatKillException`都继承了运行时异常，所以这些操作把异常都转化为了运行时异常。这样spring才能回滚。数据库的修改才不会紊乱。

这里有一个操作就是枚举的使用。

```java
//return new SeckillExecution(seckillId,1,"秒杀成功",successKilled);
return new SeckillExecution(seckillId, SecKillStatEnum.SUCCESS,successKilled);
```

用第一行的方式割裂了状态和状态信息，很不优雅，而且后续要更改的话，这些代码分散在各个代码中，不易修改，所以用枚举代替。

```java
package cn.orzlinux.enums;

/**
 * 使用枚举表述常量数据字段
 */
public enum SecKillStatEnum {

    SUCCESS(1,"秒杀成功"),
    END(0,"秒杀结束"),
    REPEAT_KILL(-1,"重复秒杀"),
    INNER_ERROR(-2,"系统异常"),
    DATA_REWRITE(-3,"数据篡改")
    ;


    private int state;
    private String stateInfo;

    SecKillStatEnum(int state, String stateInfo) {
        this.state = state;
        this.stateInfo = stateInfo;
    }

    public int getState() {
        return state;
    }

    public String getStateInfo() {
        return stateInfo;
    }

    public static SecKillStatEnum stateOf(int index) {
        for(SecKillStatEnum statEnum:values()) {
            if(statEnum.getState()==index) {
                return statEnum;
            }
        }
        return null;
    }
}
```

### 声明式事务

<img src="https://gitee.com/hqinglau/img/raw/master/img/20211005154110.png" alt="image-20211005154110783" style="zoom:80%;" />

spring早期使用方式（2.0）：ProxyFactoryBean + XML

后来：tx:advice+aop命名空间，一次配置永久生效。

注解@Transactional，注解控制。（推荐）

支持事务方法嵌套。

**何时回滚事务？**抛出运行期异常，小心try/catch

具体配置：

在spring-service.xml添加：

```xml
<!--配置事务管理器-->
<bean id="transationManager"
      class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <!--注入数据库连接池-->
    <property name="dataSource" ref="dataSource"/>
</bean>

<!--配置基于注解的声明式事务-->
<!--默认使用注解管理事务行为-->
<tx:annotation-driven transaction-manager="transationManager"/>
```

在SecKillServiceImpl.java文件添加注解

```java
@Override
@Transactional
public SeckillExecution executeSeckill(long seckillId, long userPhone, String md5) 		throws SeckillException, RepeatKillException, SeckillException
...
```

**使用注解控制事务方法的优点**：

- 开发团队达成一致约定，明确标注事务方法的编程风格
- 保证事务方法的执行时间尽可能短，不要穿插其它网络操作，要剥离到事务外部
- 不是所有的方法都需要事务

### 集成测试

在resource文件夹下新建logback.xml，日志的配置文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <layout class="ch.qos.logback.classic.PatternLayout">
            <pattern>[%d{yyyy-MM-dd' 'HH:mm:ss.sss}] [%C] [%t] [%L] [%-5p] %m%n</pattern>
        </layout>
    </appender>

    <root level="DEBUG">
        <appender-ref ref="STDOUT"/>
    </root>
</configuration>
```

SecKillServiceTest.java

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration({
        "classpath:spring/spring-dao.xml",
        "classpath:/spring/spring-service.xml"
})
public class SecKillServiceTest {
    private final Logger logger = LoggerFactory.getLogger(this.getClass());

    @Autowired
    private SecKillService secKillService;
    @Test
    public void getSecKillList() {
        List<SecKill> list = secKillService.getSecKillList();;
        logger.info("list={}",list);
        // 输出信息： [main] [89] [DEBUG] JDBC Connection [com.mchange.v2.c3p0.impl.NewProxyConnection@5443d039] will not be managed by Spring
        //[DEBUG] ==>  Preparing: select seckill_id,name,number,start_time,end_time,create_time from seckill order by create_time DESC limit 0,?;
        //[DEBUG] ==> Parameters: 4(Integer)
        //[DEBUG] <==      Total: 4
        //[DEBUG]
        // ------------------Closing non transactional SqlSession ---------------------
        // [org.apache.ibatis.session.defaults.DefaultSqlSession@66c61024]
        //[INFO ] list=[SecKill{seckillId=1000, name='1000秒杀iphone13', number=99, startTime=Tue Oct 05 02:00:00 CST 2021, endTime=Wed Oct 06 02:00:00 CST 2021, createTime=Tue Oct 05 10:01:47 CST 2021}, SecKill{seckillId=1001, name='500秒杀iphone12', number=200, startTime=Tue Oct 05 02:00:00 CST 2021, endTime=Wed Oct 06 02:00:00 CST 2021, createTime=Tue Oct 05 10:01:47 CST 2021}, SecKill{seckillId=1002, name='300秒杀iphone11', number=300, startTime=Tue Oct 05 02:00:00 CST 2021, endTime=Wed Oct 06 02:00:00 CST 2021, createTime=Tue Oct 05 10:01:47 CST 2021}, SecKill{seckillId=1003, name='100秒杀iphone6', number=400, startTime=Tue Oct 05 02:00:00 CST 2021, endTime=Wed Oct 06 02:00:00 CST 2021, createTime=Tue Oct 05 10:01:47 CST 2021}]
    }

    @Test
    public void getById() {
        long id = 1000;
        SecKill secKill = secKillService.getById(id);
        logger.info("seckill={}",secKill);
        // seckill=SecKill{seckillId=1000, name='1000秒杀iphone13', n...}
    }

    @Test
    public void exportSecKillUrl() {
        long id = 1000;
        Exposer exposer = secKillService.exportSecKillUrl(id);
        logger.info("exposer={}",exposer);
        //exposer=Exposer{exposed=true, md5='c78a6784f8e8012796c934dbb3f76c03',
        //          seckillId=1000, now=0, start=0, end=0}
        // 表示在秒杀时间范围内
    }

    @Test
    public void executeSeckill() {
        long id = 1000;
        long phone = 10134256781L;
        String md5 = "c78a6784f8e8012796c934dbb3f76c03";

        // 重复测试会抛出异常，junit会认为测试失败，要把异常捕获一下更好看
        try {
            SeckillExecution seckillExecution = secKillService.executeSeckill(id,phone,md5);
            logger.info("result: {}",seckillExecution);
        } catch (RepeatKillException | SeckillCloseException e) {
            logger.error(e.getMessage());
            // 再运行一次： [ERROR] seckill repeated
        }


        // 有事务记录
        // Committing JDBC transaction on Connection [com.mchange.v2.c3p0.impl.NewProxyConnection@4b40f651]
        //[2021-10-05 16:45:00.000] [org.springframework.jdbc.datasource.DataSourceTransactionManager] [main] [384] [DEBUG] Releasing JDBC Connection [com.mchange.v2.c3p0.impl.NewProxyConnection@4b40f651] after transaction
        //result: SeckillExecution{seckillId=1000, state=1, stateInfo='秒杀成功', successKilled=SuccessKilled{seckillId=1000, userPhone=10134256781, state=0, createTime=Wed Oct 06 00:45:00 CST 2021}}
    }

    // 集成测试完整逻辑实现
    @Test
    public void testSeckillLogic() {
        long id = 1001;
        Exposer exposer = secKillService.exportSecKillUrl(id);
        if(exposer.isExposed()) {
            logger.info("exposer={}",exposer);

            long phone = 10134256781L;
            String md5 = exposer.getMd5();

            // 重复测试会抛出异常，junit会认为测试失败，要把异常捕获一下更好看
            try {
                SeckillExecution seckillExecution = secKillService.executeSeckill(id,phone,md5);
                logger.info("result: {}",seckillExecution);
            } catch (RepeatKillException | SeckillCloseException e) {
                logger.error(e.getMessage());
                // 再运行一次： [ERROR] seckill repeated
            }

        } else {
            // 秒杀未开启
            logger.warn("exposer={}",exposer);
        }
    }
}
```

