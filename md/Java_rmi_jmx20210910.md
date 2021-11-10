## RMI

远程方法调用（Remote Method Invocation），用于不同虚拟机之间的通信，可以在相同或不同的主机上。

![image-20210910151930494](https://gitee.com/hqinglau/img/raw/master/img/20210910151930.png)

> 每一个远程对象同时也包含一个 skeleton 对象，skeleton 运行在远程对象所在的虚拟机上，接受来自 stub 对象的调用。当skeleton 接收到来自 stub 对象的调用请求后， skeleton 会作如下工作：
>
> 1. 解包stub传来的参数 
> 2. 调用远程对象匹配的方法 
> 3. 打包返回值或错误发送给stub对象 
>
> jdk1.2以后的RMI可以通过反射API可以直接将请求发送给真实类，所以不需要 skeleton 类了

IRemoteMath:

```java
package rmi;

import java.rmi.Remote;
import java.rmi.RemoteException;

public interface IRemoteMath extends Remote {
    public double add(double a,double b) throws RemoteException;
    public double sub(double a,double b) throws RemoteException;
}
```

远程方法调用的本质依然是网络通信，只不过隐藏了底层实现，网络通信是经常会出现异常的，所以接口的所有方法都必须抛出`RemoteException`以说明该方法是有风险的

RemoteMath:

```java
package rmi;

import java.rmi.RemoteException;
import java.rmi.server.UnicastRemoteObject;

/**
 * 服务器端实现远程接口。
 * 必须继承UnicastRemoteObject，以允许JVM创建远程的存根/代理。
 */
public class RemoteMath extends UnicastRemoteObject implements IRemoteMath {
    private int numberOfComputations;
    public RemoteMath() throws RemoteException {
        numberOfComputations = 0;
    }

    @Override
    public double add(double a, double b) throws RemoteException {
        numberOfComputations++;
        System.out.println("Number of computations performed so far = "
                + numberOfComputations);
        return (a+b);
    }

    @Override
    public double sub(double a, double b) throws RemoteException {
        numberOfComputations++;
        System.out.println("Number of computations performed so far = "
                + numberOfComputations);
        return (a-b);
    }
}
```

由于方法参数与返回值最终都将在网络上传输，故必须是可序列化的。

server:

```java
package rmiServ;

import rmi.IRemoteMath;
import rmi.RemoteMath;

import java.rmi.AlreadyBoundException;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class RMIServer {
    public static void main(String[] args) {
        try {
            IRemoteMath remoteMath = new RemoteMath();
            LocateRegistry.createRegistry(1099);
            Registry registry = LocateRegistry.getRegistry();
            registry.bind("Compute",remoteMath);
            System.out.println("Math Server Ready.");
        } catch (RemoteException | AlreadyBoundException e) {
            e.printStackTrace();
        }
    }
}
/*
输出：
Math Server Ready.
Number of computations performed so far = 1
Number of computations performed so far = 2

 */
```

client:

```java
import rmi.IRemoteMath;

import java.rmi.AccessException;
import java.rmi.NotBoundException;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class rmiCli {
    public static void main(String[] args) {
        try {
            Registry registry = LocateRegistry.getRegistry("localhost");
            IRemoteMath remoteMath = (IRemoteMath) registry.lookup("Compute");
            double addResult = remoteMath.add(5.0, 3.0);
            System.out.println("5.0 + 3.0 = " + addResult);
            double subResult = remoteMath.sub(5.0, 3.0);
            System.out.println("5.0 - 3.0 = " + subResult);
        } catch (AccessException e) {
            e.printStackTrace();
        } catch (NotBoundException e) {
            e.printStackTrace();
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }
}
/*
输出：
5.0 + 3.0 = 8.0
5.0 - 3.0 = 2.0
 */
```

RMI 依赖于 Java 远程消息交换协议 JRMP（Java Remote Messaging Protocol），该协议为 Java 定制，要求服务端与客户端都为 Java 编写。

## Java bean

就是一个普通的 Java 类，要求如下：

1. 这个类需要是`public` 的， 然后需要有个无参数的构造函数

2. 这个类的属性应该是`private` 的， 通过`setXXX()`和`getXXX()`来访问

3. 这个类需要能支持”事件“， 例如`addXXXXListener(XXXEvent e)`,  事件可以是`Click`事件，`Keyboard`事件等等，也支持自定义的事件。 

4. 我们得提供一个所谓的自省/反射机制， 这样能在运行时查看`java bean` 的各种信息

5. 这个类应该是可以序列化的， 即可以把`bean`的状态保存的硬盘上， 以便以后恢复。 

## JMX

JMX（Java Management Extensions）是一个为应用程序植入管理功能的框架。基本框架：

![image-20210910192818520](https://gitee.com/hqinglau/img/raw/master/img/20210910192818.png)

MBean：是 Managed Bean 的简称，可以翻译为“管理构件”。在JMX中MBean代表一个被管理的资源实例，通过 MBean 中暴露的方法和属性，外界可以获取被管理的资源的状态和操纵 MBean 的行为。事实上，MBean 就是一个 Java Object，同 JavaBean 模型一样，外界使用自省和反射来获取 Object 的值和调用 Object 的方法，只是 MBean 更为复杂和高级一些。MBean 通过公共方法以及遵从特定的设计模式封装了属性和操作，以便暴露给管理应用程序。例如，一个只读属性在管理构件中只有 Get 方法，既有 Get 又有 Set 方法表示是一个可读写的属性。一共有四种类型的 MBean: Standard MBean, Dynamic MBean, Open MBean, Model MBean。

MBeanServer：MBean 生存在一个 MBeanServer 中。MBeanServer 管理这些 MBean，并且代理外界对它们的访问。并且MBeanServer 提供了一种注册机制，是的外界可以通过名字来得到相应的 MBean 实例。

JMX Agent：Agent 只是一个Java进程，它包括这个 MBeanServer 和一系列附加的 MbeanService。当然这些 Service 也是通过 MBean的形式来发布。

Protocol Adapters and Connectors：MBeanServer 依赖于 Protocol Adapters 和 Connectors 来和运行该代理的Java虚拟机之外的管理应用程序进行通信。Protocol Adapters 通过特定的协议提供了一张注册在 MBeanServer 的 MBean 的视图。例如，一个 HTML Adapter可以将所有注册过的 MBean 显示在 Web 页面上。不同的协议，提供不同的视图。Connectors 还必须提供管理应用一方的接口以使代理和管理应用程序进行通信，即针对不同的协议，Connectors 必须提供同样的远程接口来封装通信过程。当远程应用程序使用这个接口时，就可以通过网络透明的和代理进行交互，而忽略协议本身。Adapters 和 Connectors 使 MBean 服务器与管理应用程序能进行通信。因此，一个代理要被管理，它必须提供至少一个 Protocol Adapter 或者 Connector。面临多种管理应用时，代理可以包含各种不同的Protocol Adapters 和 Connectors。当前已经实现和将要实现的 Protocol Adapters 和 Connectors 包括： RMI Connector, SNMP Adapter, IIOP Adapter, HTML Adapter, HTTP Connector。

检查Standard MBean接口和应用设计模式的过程被称为内省（Introspection）。JMX代理通过内省来查看每一个注册在MBeanServer上的MBean的方法和超类，看它是否遵从一定设计模式，决定它是否代表了一个MBean，并辨认出它的属性和操作。

```java
// ========================= HelloMBean========================================
package jmx;

public interface HelloMBean {
    public String getName();
    public void setName(String name);
    public void printHello();
    public void printHello(String whoName);
}

// =========================== Hello =====================================
package jmx;

public class Hello implements HelloMBean{
    private String name;

    @Override
    public String getName() {
        return name;
    }

    @Override
    public void setName(String name) {
        this.name = name;
    }

    @Override
    public void printHello() {
        System.out.println("Hello, "+name);
    }

    @Override
    public void printHello(String whoName) {
        System.out.println("Hello, "+whoName);
    }
}

// ======================== HelloAgent ======================================
package jmx;

import com.sun.jdmk.comm.HtmlAdaptorServer;

import javax.management.*;
import javax.management.remote.JMXConnectorServer;
import javax.management.remote.JMXConnectorServerFactory;
import javax.management.remote.JMXServiceURL;
import java.io.IOException;
import java.lang.management.ManagementFactory;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class HelloAgent {
    public static void main(String[] args) throws MalformedObjectNameException, NotCompliantMBeanException, InstanceAlreadyExistsException, MBeanRegistrationException, IOException {
        // 首先建立一个MBeanServer,MBeanServer用来管理我们的MBean,通常是通过MBeanServer来获取我们MBean的信息，间接
        // 调用MBean的方法，然后生产我们的资源的一个对象。
        MBeanServer mbs = ManagementFactory.getPlatformMBeanServer();
        String domainName = "MyMBean";

        //为MBean（下面的new Hello()）创建ObjectName实例
        ObjectName helloName = new ObjectName(domainName+":name=HelloWorld");
        // 将new Hello()这个对象注册到MBeanServer上去
        mbs.registerMBean(new Hello(),helloName);
        // Distributed Layer, 提供了一个HtmlAdaptor。支持Http访问协议，并且有一个不错的HTML界面，这里的Hello就是用这个作为远端管理的界面
        // 事实上HtmlAdaptor是一个简单的HttpServer，它将Http请求转换为JMX Agent的请求
        ObjectName adapterName = new ObjectName(domainName+":name=htmladapter,port=8082");
        HtmlAdaptorServer adapter = new HtmlAdaptorServer();
        adapter.start();
        mbs.registerMBean(adapter,adapterName);
        int rmiPort = 1099;
        Registry registry = LocateRegistry.createRegistry(rmiPort);

        JMXServiceURL url = new JMXServiceURL("service:jmx:rmi:///jndi/rmi://localhost:"+rmiPort+"/"+domainName);
        JMXConnectorServer jmxConnector = JMXConnectorServerFactory.newJMXConnectorServer(url, null, mbs);
        jmxConnector.start();
    }
}
```

jmxtools需要另外下载，版权原因，maven没有。

效果如下：

![image-20210910205557426](https://gitee.com/hqinglau/img/raw/master/img/20210910205557.png)

![image-20210910205453649](https://gitee.com/hqinglau/img/raw/master/img/20210910205453.png)

```java
// 按下printHello
// Agent输出
Hello, 3
Hello, lyd
```

## 参考

[JAVA RMI 原理和使用浅析](https://blog.csdn.net/qq_28081453/article/details/83279066)

[Java RMI详解](https://blog.csdn.net/a19881029/article/details/9465663)

[从零开始玩转JMX(一)——简介和Standard MBean](https://blog.csdn.net/u013256816/article/details/52800742)