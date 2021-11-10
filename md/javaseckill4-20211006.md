> 本文为[Java高并发秒杀API之高并发优化](https://www.imooc.com/learn/632)课程笔记。

编辑器：IDEA

java版本：java8

前文：

一、[秒杀系统环境搭建与DAO层设计](https://orzlinux.cn/blog/javaseckill1-20211004.html)

二、 [秒杀系统Service层](https://orzlinux.cn/blog/javaseckill2-20211005.html)

三、[秒杀系统web层](https://orzlinux.cn/blog/javaseckill3-20211005.html)

## 高并发优化分析

并发发生在哪？对一件商品秒杀，自然在具体商品详情页下面，会存在高并发瓶颈。

<img src="https://gitee.com/hqinglau/img/raw/master/img/20211006210620.png" alt="image-20211006210620584" style="zoom:80%;" />

> 红色部分为高并发发生点。

为什么要单独获取系统时间？因为资源不都是从服务器获取的。

<img src="https://gitee.com/hqinglau/img/raw/master/img/20211006211059.png" alt="image-20211006211059120" style="zoom:80%;" />

所以需要单独获取时间来明确服务器的当前时间。

CDN：内容分发网络，加速用户获取数据的系统。部署在离用户最近的网络结点上。命中CDN不需要访问后端服务器。

**获取系统时间不需要优化**。访问一次内存大概10ns，没有后端访问。

**获取秒杀地址**：无法使用CDN缓存，适合服务端缓存：redis等。一致性维护成本低。

<img src="https://gitee.com/hqinglau/img/raw/master/img/20211006235429.png" alt="image-20211006235429419" style="zoom:80%;" />

**执行秒杀操作**：无法使用CDN，后端缓存困难：库存问题，一行数据竞争：热点商品。

**秒杀方案**：

![image-20211006235908392](https://gitee.com/hqinglau/img/raw/master/img/20211006235908.png)

成本分析：运维成本和稳定：nosql,mq等。开发成本：数据一致性，回滚方案。幂等性难保证：重复秒杀问题。不适合新手的架构。

**为什么不用MySQL解决？**

一条update，MySQL可以QPS很高。

java控制事务行为分析：

<img src="https://gitee.com/hqinglau/img/raw/master/img/20211007000557.png" alt="image-20211007000557818" style="zoom:80%;" />



瓶颈分析：

<img src="https://gitee.com/hqinglau/img/raw/master/img/20211007001006.png" alt="image-20211007001006461" style="zoom:80%;" />

**优化分析**：行级锁在commit之后释放，所以优化方向在于减少行级锁的保持时间。

**延迟分析**：本地机房，可能1ms，异地机房：

<img src="https://gitee.com/hqinglau/img/raw/master/img/20211007001433.png" alt="image-20211007001433741" style="zoom:80%;" />

往返可能20ms，那并发最多就50QPS。

**优化思路：**

把客户端逻辑放端MySQL服务端，避免网络延迟和GC影响。

两种方案：

- 定制SQL方案：`update /*+[auto_commit]*/`，成功就成功，不成功就回滚，需要修改MySQL源码。
- 使用存储过程：整个事务在MySQL端完成。

**优化总结**：

- 前端控制：暴露接口，按钮防重复
- 动静态数据分离：CDN缓存（静态资源），后端缓存（如redis）
- 事务竞争优化：减少事务锁时间

## redis后端优化缓存编码

> 使用redis优化地址暴露接口

下载安装redis。

```shell
D:\Program Files (x86)\Renren.io\Redis>redis-cli.exe -h 127.0.0.1 -p 6379
127.0.0.1:6379> set myKey abc
OK
127.0.0.1:6379> get myKey
"abc"
```

配合java使用，pom.xml加入依赖：

```xml
<!--redis依赖引入-->
<dependency>
  <groupId>redis.clients</groupId>
  <artifactId>jedis</artifactId>
  <version>3.5.0</version>
</dependency>

<!--序列化操作-->
<dependency>
    <groupId>com.dyuproject.protostuff</groupId>
    <artifactId>protostuff-core</artifactId>
    <version>1.1.6</version>
</dependency>

<dependency>
    <groupId>com.dyuproject.protostuff</groupId>
    <artifactId>protostuff-runtime</artifactId>
    <version>1.1.6</version>
</dependency>
```

在SecKillServiceImpl.java文件中原本暴露url的代码为：

```java
/**
 * 秒杀开启时，输出秒杀接口地址
 * 否则输出系统时间和秒杀时间
 *
 * @param seckillId
 */
@Override
public Exposer exportSecKillUrl(long seckillId) {
    // 查数据库
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

在DAO文件夹下新建RedisDao.java，因为它也是和数据打交道的。

![image-20211007133647855](https://gitee.com/hqinglau/img/raw/master/img/20211007133647.png)

RedisDao.java

```java
public class RedisDao {
    private final Logger logger = LoggerFactory.getLogger(this.getClass());

    private JedisPool jedisPool;
    public RedisDao(String ip,int port) {
        jedisPool = new JedisPool(ip,port);
    }

    private RuntimeSchema<SecKill> schema = RuntimeSchema.createFrom(SecKill.class);

    public SecKill getSeckill(long seckillId) {
        // redis操作逻辑
        try {
            Jedis jedis = jedisPool.getResource();
            try {
                String key = "seckill:"+seckillId;
                // redis并没有实现内部序列化操作
                // get得到的是一个二进制数组byte[],通过反序列化-> Object(SecKill)
                // 采用自定义序列化 protostuff
                // protostuff:pojo 有get set这些方法
                byte[] bytes = jedis.get(key.getBytes(StandardCharsets.UTF_8));
                // 获取到了，需要protostuff转化
                // 需要字节数组和schema
                if (bytes != null) {
                    // 创建一个空对象来放反序列化生成的对象
                    SecKill secKill = schema.newMessage();
                    ProtostuffIOUtil.mergeFrom(bytes,secKill,schema);
                    // seckill被反序列化
                    return secKill;
                }
            } finally {
                jedis.close();
            }

        } catch (Exception e) {
            logger.error(e.getMessage(),e);
        }
        return null;
    }

    public String putSeckill(SecKill secKill) {
        // set: objest(SecKill) -> bytes[] 序列化操作
        try {
            Jedis jedis = jedisPool.getResource();
            try {
                String key = "seckill:"+secKill.getSeckillId();
                byte[] bytes = ProtostuffIOUtil.toByteArray(secKill,schema,
                        LinkedBuffer.allocate(LinkedBuffer.DEFAULT_BUFFER_SIZE));
                // 超时缓存
                int timeout = 60*60; // 1小时
                String result = jedis.setex(key.getBytes(StandardCharsets.UTF_8),timeout,bytes);
                return result; //加入缓存信息，成功还是失败
            } finally {
                jedis.close();
            }
        } catch (Exception e) {
            logger.error(e.getMessage(),e);
        }
        return null;
    }
}
```

这里面有两个方法，put和get，中间还牵扯到序列化，用的是protostuff。

单元测试之前，需要注入RedisDao，在spring-dao.xml中注入：

```xml
<!--需要自己配置redis dao-->
<bean id="redusDao" class="cn.orzlinux.dao.cache.RedisDao">
    <constructor-arg index="0" value="localhost" />
    <constructor-arg index="1" value="6379" />
</bean>
```

进行单元测试：

```java
@RunWith(SpringJUnit4ClassRunner.class)
//告诉junit spring配置文件
@ContextConfiguration({"classpath:spring/spring-dao.xml"})
public class RedisDaoTest {
    private long id = 1001;
    @Autowired
    private RedisDao redisDao;

    @Autowired
    private SecKillDao secKillDao;

    @Test
    public void testSeckill() {
        // get and put
        // 从缓存中拿
        SecKill secKill = redisDao.getSeckill(id);
        // 没有就从数据库中查
        // 查到后放回redis
        if(secKill==null) {
            secKill = secKillDao.queryById(id);
            if(secKill != null) {
                String result = redisDao.putSeckill(secKill);
                System.out.println(result);
                secKill = redisDao.getSeckill(id);
                System.out.println(secKill);
            }
        }
    }
}
```

可以通过在命令行查询redis验证一下，可以看出的确是放入了：

```shell
127.0.0.1:6379> get seckill:1001
"\b\xe9\a\x12\x11500\xe7\xa7\x92\xe6\x9d\x80iphone12\x18\xbc\x84=!\x00evL|\x01\x00\x00)\x00\xe4\xcd\xb3\xc5\x01\x00\x001\xf8z/N|\x01\x00\x00"
127.0.0.1:6379> get seckill:1002
(nil)
```

> 这里面能用redis缓存是因为秒杀一件商品可能有成千上万人，这些人访问这件商品的URL都是一样的，不需要频繁查找数据库，直接存缓存中拿就可以。

## 秒杀操作并发优化

事务执行：

<img src="https://gitee.com/hqinglau/img/raw/master/img/20211007141054.png" alt="image-20211007141053922" style="zoom:80%;" />

### 简单优化

<img src="https://gitee.com/hqinglau/img/raw/master/img/20211007141341.png" alt="image-20211007141341792" style="zoom:80%;" />

insert插入操作冲突概率低。服务端根据insert结果判断是否执行update，排除重复秒杀，不再update加锁。然后再是update行级锁，可以减少行级锁的持有时间。

源码更改数据库操作时间：

```java
@Override
@Transactional
public SeckillExecution executeSeckill(long seckillId, long userPhone, String md5) throws
        SeckillException, RepeatKillException, SeckillException {
    if(md5==null || !md5.equals(getMD5(seckillId))) {
        // 秒杀的数据被重写修改了
        throw new SeckillException("seckill data rewrite");
    }
    // 执行秒杀逻辑：减库存、加记录购买行为
    Date nowTime = new Date();

    try {
        // 记录购买行为
        int insertCount = successKilledDao.insertSuccessKilled(seckillId,userPhone);
        if(insertCount<=0) {
            // 重复秒杀
            throw new RepeatKillException("seckill repeated");
        } else {
            // 减库存。热点商品竞争
            int updateCount = secKillDao.reduceNumber(seckillId,nowTime);
            if(updateCount<=0) {
                // 没有更新记录,秒杀结束
                throw new SeckillCloseException("seckill is closed");
            } else {
                //秒杀成功
                SuccessKilled successKilled = successKilledDao.queryByIdWithSeckill(seckillId,userPhone);
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

### 深度优化

事务SQL在MySQL端执行（存储过程）。

**存储过程**

存储过程（Stored Procedure）是在大型数据库系统中，一组为了完成特定功能的SQL 语句集，它存储在数据库中，**一次编译后永久有效**，用户通过指定存储过程的名字并给出参数（如果该存储过程带有参数）来执行它。存储过程是数据库中的一个重要对象。

优点很明显，说一下缺点：难调试、可移植性差、如果业务数据模型变动，大型项目的存储过程更改很大。

存储过程优化的是事务行级锁的持有时间。不要过度依赖存储过程，简单的逻辑可以依靠存储过程。

定义一个存储过程：

```sql
-- 秒杀执行存储过程
DELIMITER $$ -- console ;转化为\$\$ 表示sql可以执行操作了
-- 定义存储过程
-- 参数：in 输入参数; out 输出参数
-- row_count(): 返回上一条修改类型sql的影响行数
-- row_count: 0未修改数据，>0 修改的行数，<0 sql错误或未执行

# SUCCESS(1,"秒杀成功"),
# END(0,"秒杀结束"),
# REPEAT_KILL(-1,"重复秒杀"),
# INNER_ERROR(-2,"系统异常"),
# DATA_REWRITE(-3,"数据篡改")
CREATE PROCEDURE `seckill`.`execute_seckill`
    (in v_seckill_id bigint, in v_phone bigint,
        in v_kill_time timestamp,out r_result int)
    BEGIN
        DECLARE insert_count int DEFAULT 0;
        START TRANSACTION;
        insert ignore into success_killed
            (seckill_id, user_phone,create_time)
            values (v_seckill_id,v_phone,v_kill_time);
        select row_count() into insert_count;
        IF (insert_count=0) THEN
            ROLLBACK;
            set r_result = -1;
        ELSEIF (insert_count<0) THEN
            ROLLBACK;
            set r_result = -2;
        ELSE
            update seckill
                set number = number-1
                where seckill_id = v_seckill_id
                    and end_time > v_kill_time
                    and start_time < v_kill_time
                    and number>0;
            select row_count() into insert_count;
            IF (insert_count = 0) THEN
                ROLLBACK;
                set r_result = 0;
            ELSEIF(insert_count<0) then
                ROLLBACK;
                set r_result = -2;
            ELSE
                COMMIT;
                set r_result = 1;
            end if;
        end if;
    END;
$$ -- 存储过程定义结束
delimiter ;

-- console定义变量
set @r_result=-3;
-- 执行存储过程
call execute_seckill(1001,19385937587,now(),@r_result);
-- 获取结果
select @r_result;
```

这样，在服务器端完成插入和update的操作。

要想使用这个存储过程，需要在SeckillDao.java加入新的方法：

```java
// 使用存储过程执行秒杀
void killByProcedure(Map<String,Object> paramMap);
```

然后通过xml实现sql语句：

```xml
<!--mybatis调用存储过程-->
<select id="killByProcedure" statementType="CALLABLE">
    call execute_seckill(
        #{seckillId,jdbcType=BIGINT,mode=IN},
        #{phone,jdbcType=BIGINT,mode=IN},
        #{killTime,jdbcType=TIMESTAMP,mode=IN},
        #{result,jdbcType=INTEGER,mode=OUT}
        )
</select>
```



SecKillService接口加入新的方法，然后在SecKillServiceImpl.java实现：

```java
/**
 * 存储过程执行秒杀
 *
 * @param seckillId
 * @param userPhone
 * @param md5
 * @return
 * @throws SeckillException
 * @throws RepeatKillException
 * @throws SeckillCloseException
 */
@Override
public SeckillExecution executeSeckillProcedure(long seckillId, long userPhone, String md5) {
    if(md5==null || !md5.equals(getMD5(seckillId))) {
        // 秒杀的数据被重写修改了
        throw new SeckillException("seckill data rewrite");
    }
    Date nowTime = new Date();
    Map<String,Object> map = new HashMap<>();
    map.put("seckillId",seckillId);
    map.put("phone",userPhone);
    map.put("killTime",nowTime);
    map.put("result",null);
    // 执行存储过程只有，result被赋值
    try {
        secKillDao.killByProcedure(map);
        // 获取result
        int result = MapUtils.getInteger(map,"result",-2);
        if(result == 1) {
            SuccessKilled sk = successKilledDao.queryByIdWithSeckill(seckillId,userPhone);
            return new SeckillExecution(seckillId,SecKillStatEnum.SUCCESS,sk);
        } else {
            return new SeckillExecution(seckillId,SecKillStatEnum.stateOf(result));
        }
    } catch (Exception e) {
        logger.error(e.getMessage(),e);
        return new SeckillExecution(seckillId,SecKillStatEnum.INNER_ERROR);
    }
}
```

最后再controller层将原有的执行秒杀方法换成这个。

