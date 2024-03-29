---
layout: post
title: 我所理解的JDK系列·第4篇·JavaSPI是如何工作的
categories: [JDK]
keywords: Java, JDK, SPI
---



本文实现了一个基于 Java SPI 机制的的简单 demo，然后分析了 ServiceLoader 类基于线程上下文类加载器实现 Java SPI 的源码。



![jdk-4-封面](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/jdk/jdk-4-封面.77nzpdzmv4k0.jpg)



## 1. 开篇词

本文着重于实现一个基于 Java SPI 的 demo 以及对其实现原理的解析，即 ServiceLoader 类源码分析。


其实最初想写这篇文章的原因是在之前的一次面试中，被面试官问到关于 Java SPI 的问题，但没能说出让他满意的答案，所以才想着整理一篇 SPI 的文章，顺便也巩固一下双亲委派机制的知识。



## 2. SPI 简述


SPI 的全称是 Service Provider Interface，翻译过来就是服务提供方接口。它是 Java 内置的一种服务提供发现机制，只需要在环境变量中添加相应接口的实现，程序就能自动装载该类并使用它。


SPI 有良好的扩展性，框架制定了规则（接口），具体的厂商提供实现（实现接口），如果想切换实现方案，只需要把另一个厂商的实现放在环境变量中就可以，也不需要修改代码，显然这是一种策略模式。


下面，我们就来实现一个简单的基于 SPI 的 demo。



## 2. Java 实现 SPI 的步骤


### 2.1 定义接口


首先需要定义一个接口，这就是所谓的“标准”，而厂商就是根据这个标准接口进行实现的，比如关于 JDBC 标准接口有 MySQL 的实现和 Oracle 的实现。


而在这个 demo 中，我所定义的 HelloSpi 接口的标准是需要实现 say 方法，其核心功能就是基于 say 方法输出一段文字。


```java
public interface HelloSpi {
    /**
     * spi接口的方法
     */
    void say();
}
```



### 2.2 创建接口的实现类


创建两个实现类 HelloInEnglish 和 HelloInChinese，分别输出一行关于 hello 的语句。


```java
public class HelloInChinese implements HelloSpi {
    @Override
    public void say() {
        System.out.println("from HelloInChinese: 你好");
    }
}

public class HelloInEnglish implements HelloSpi {
    @Override
    public void say() {
        System.out.println("from HelloInEnglish: hello");
    }
}
```


在这里有一个注意点，**实现类必须要有无参构造函数**，否则会报错，因为在 ServiceLoader 在创建实现类实例的时候会通过无参构造函数来实现，具体的代码在后面会分析。



### 2.3 创建接口全限定名配置元文件


接下来就是在 resources 目录下创建一个 **META-INF/services** 文件夹，继而创建一个名称为 **HelloSpi 接口全限定名**的文件，在我项目中就是 org.walker.planes.spi.HelloSpi。


而文件的内容就是刚刚创建的两个实现类的全限定名，文件中每行代表一个实现类。


```
org.walker.planes.spi.HelloInEnglish
org.walker.planes.spi.HelloInChinese
```



### 2.4 使用ServiceLoader加载配置文件中的类


创建一个有 main 方法的测试类，调用 ServiceLoader#load(Class) 方法加载对应的类，并执行。


```java
public class SpiMain {
    public static void main(String[] args) {
        // 加载 HelloSpi 接口的实现类
        ServiceLoader<HelloSpi> shouts = ServiceLoader.load(HelloSpi.class);
        // 执行say方法
        for (HelloSpi s : shouts) {
            s.say();
        }
    }
}

执行结果为：
from HelloInEnglish: hello
from HelloInChinese: 你好
```

至此，就基于 Java SPI 机制实现了一个简单的 demo，

至此，一个简单的基于 Java SPI 机制实现的 demo 就完成了。在例子中，我们可以知道，除了需要加一个配置文件外，比较重要的类是 ServiceLoader，通过调用 ServiceLoader#load(Class) 方法程序加载到了接口实现类，并运行了 say 方法。

由此可知，大概率 JavaSPI 就是基于 ServiceLoader 类来工作的。下面我们就来分析一下 ServiceLoader 类加载接口实现类的原理。




## 3. ServiceLoader 源码分析


通过上面的一个实现 Java SPI 的 demo，我想你已经知道它的工作流程了，下面就来说说 ServiceLoader 类是如何根据接口类找到并加载实现类的。




### 3.1 ServiceLoader#load(Class)


首先我们来看上面例子中的第4步，通过调用 ServiceLoader#load(Class) 方法加载对应的类，来看看 load 方法的源码。


```java
// 示例 main 方法中的 load 方法
public static <S> ServiceLoader<S> load(Class<S> service) {
    // 获取线程上下文类加载器
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    // 调用重载方法，通过 cl 线程上下文类加载器加载 service 目标类
    return ServiceLoader.load(service, cl);
}

// load 重载方法
public static <S> ServiceLoader<S> load(Class<S> service, ClassLoader loader) {
    return new ServiceLoader<>(service, loader);
}
```


正如上面代码中注释锁描述的那样，通过调用 ServiceLoader#load(Class) 方法，首先会获取线程上下文类加载器，然后通过线程上下文类加载器加载目标类。

说到这里，就不得不提一下 JVM 双亲委派机制和线程上下文类加载器(Thread Context ClassLoader)。




### 3.2 JVM 双亲委派机制


相信大家都知道 JVM 的双亲委派机制，即当类加载器接收到类加载的请求时，它不会自己去尝试加载这个类，而是把这个请求委派给父加载器去完成，只有当父类加载器反馈自己无法完成这个加载请求时，子加载器才会尝试自己去加载类。


一般我们所说的类加载器包括启动类加载器(Bootstrap ClassLoader)、拓展类加载器(Extension ClassLoader)、应用类加载器(AppClassLoader)、自定义类加载器(Custom ClassLoader)。


其中在本文中需要了解的是，启动类加载器负责加载 Java 的核心类，包括：


- 位于 JAVA_HOME\lib 中的类
- 被 -Xbootclasspath 参数所指定的路径中
- 虚拟机能够识别的类库，如 rt.jar、tools.jar

还有应用类加载器，它负责加载用户类路径 classpath 上所指定的类库。


至此我们已经知道了双亲委派机制和启动类加载器，再来说回 SPI。前文中也提到过，所谓的 SPI 就是 JDK 提供的一些标准接口，由厂商来进行实现，当环境变量中存在该实现时，就能够自动进行类的加载，从而使用厂商的实现。


值得注意的是，JDK 提供标准 SPI 接口一般是和核心代码库放在一起的，比如 JDBC 驱动 Driver.class 接口就是在 rt.jar 包中。也就是说，SPI接口是被启动类加载器加载的，如果基于传统的双亲委派机制，其实是通过启动类类加载器来加载厂商的实现类的，这个时候就会发现，厂商实现类是在 classpath 中，应该被应用类加载器加载，而不能被启动类加载器加载。

基于这样一种困境，线程上下文类加载应运而生，它也是用来打破双亲委派机制的一种方法。




### 3.3 线程上下文类加载器


线程上下文类加载器是 Thread 类的一个属性，它是用来缓存当前线程的类加载器的。


在 ServiceLoader#load(Class) 方法中，首先获取了当前线程的线程上下文类加载器，在示例代码中，执行此方法的线程是 main 线程，加载 main 线程的类加载器是应用类加载器。


![jdk-4-1](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/jdk/jdk-4-1.w906t6a3mls.jpg)


应用类加载器负责加载 classpath 上所指定的类库，当前工程当然属于 classpath 路径，所以使用应用类加载器加载 SPI 接口的实现类是可以加载成功的。

接下来言归正传，我们继续分析 ServiceLoader 类基于线程上下文类加载加载 SPI 接口实现类的过程。




### 3.4 源码追踪


在 load 的重载方法中，其实是创建了一个 ServiceLoader 类实例，传入的参数是目标接口 HelloSpi.class 和类加载器 AppClassLoader。


![jdk-4-2](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/jdk/jdk-4-2.3iv651aa6iw0.jpg)


而在这个 ServiceLoader 类的构造方法中，也进行了一些操作，具体见下面代码的注释。


```java
private ServiceLoader(Class<S> svc, ClassLoader cl) {
    // 检查目标接口类是否为空，若为空，抛出 NullPointerException
    service = Objects.requireNonNull(svc, "Service interface cannot be null");
    
    // 若入参 cl 为空，默认使用系统类加载器（应用类加载器）
    loader = (cl == null) ? ClassLoader.getSystemClassLoader() : cl;
    
    // Java 安全管理相关，本文不详细说明
    acc = (System.getSecurityManager() != null) ? AccessController.getContext() : null;
    
    // 清空缓存提供者 providers ，重新加载所有 SPI 接口实现类
    reload();
}
```


ServiceLoader#reload() 方法实际上是对 ServiceLoader 类的私有成员变量 providers 做一个清空的操作，同时创建一个懒加载的迭代器 LazyIterator 对象，传入的参数是目标接口类和类加载器。


```java
// Cached providers, in instantiation order
// 缓存提供者，实际上是存储 SPI 接口实现类对象，key 为实现类类名，value 为 SPI 接口实现类实例
private LinkedHashMap<String,S> providers = new LinkedHashMap<>();

// The current lazy-lookup iterator
// 当前懒加载迭代器，开始迭代时创建对象，然后放入 providers 中
private LazyIterator lookupIterator;
    
public void reload() {
    // 清空 providers
    providers.clear();
    // 创建懒加载迭代器
    lookupIterator = new LazyIterator(service, loader);
}
```


至此，ServiceLoader 类已经做好了加载 SPI 接口实现类的前置准备，当程序在 for 循环中遍历 ServiceLoader 对象时，实际上是调用了 Iterator 接口的 hasNext 方法，最终调用的是上一步提到的 LazyIterator#hasNext() 方法，如果返回 true ，就调用 next 方法开始进行迭代。


```java
public boolean hasNext() {
    // Java 访问控制上下文为null时，调用 hasNextService 方法，这里就是调用了 hasNextService 方法
    if (acc == null) {
        return hasNextService();
    } else {
        PrivilegedAction<Boolean> action = new PrivilegedAction<Boolean>() {
            public Boolean run() { return hasNextService(); }
        };
        return AccessController.doPrivileged(action, acc);
    }
}
```


而一般情况下 acc == null 的判断会是 true，继而会调用 LazyIterator#hasNextService() 方法，这个方法实际上是解析 META-INF/services 目录下的文件，生成 SPI 接口实现类迭代器，设置 nextName 属性，具体逻辑见下面代码中的注释。


```java
// Github：https://github.com/Planeswalker23
private boolean hasNextService() {
    // 第一次进入此方法时 nextName == null
    if (nextName != null) {
        return true;
    }
    // configs 属性表示的是 URL 类型的 Enumeration 对象，第一次进入此方法时为null
    if (configs == null) {
        try {
            // PREFIX = "META-INF/services/"，这也解释了为什么要在 META-INF/services/ 目录下建立接口全路径名的文件
            // service 属性就是创建懒加载迭代器 LazyIterator 对象时传入的 service 对象，即 SPI 接口类 HelloSpi.class
            // 所以 fullName 变量就是 META-INF/services/HelloSpi ，代表了在该目录下创建的文件名
            String fullName = PREFIX + service.getName();
            // 若类加载器 loader 为 null，会通过系统类加载器加载配置文件
            // 若不为 null，则通过该类加载器加载配置文件
            if (loader == null)
                configs = ClassLoader.getSystemResources(fullName);
            else
                configs = loader.getResources(fullName);
        } catch (IOException x) {
            fail(service, "Error locating configuration files", x);
        }
    }
    // 第一次进入此方法时，pending 迭代器为 null
    while ((pending == null) || !pending.hasNext()) {
        // 若配置文件中没有数据，返回 false
        if (!configs.hasMoreElements()) {
            return false;
        }
        // 将配置文件中的全路径类名转换为迭代器的方式存储到 pending 属性中，每行代表 pending 属性中的一个元素
        pending = parse(service, configs.nextElement());
    }
    // 将 nextName 属性设置为 pending 迭代器的下一个元素
    nextName = pending.next();
    return true;
}
```


在执行完 hasNextService 方法后，已经完成了 SPI 配置文件的解析、生成了 SPI 标准接口实现类的迭代器、完成了下一个实现类类名 nextName 的赋值，最终返回 true，表示迭代器有 next 值，然后会调用迭代器的 next 方法，最终调用 LazyIterator#next() 方法，这个方法其实调用的是 LazyIterator#nextService() 方法，在这个方法中，完成了类的加载，类的实例化等工作，具体内容见下面代码注释。


```java
// Github：https://github.com/Planeswalker23
private S nextService() {
    // 不是第一次调用 hasNextService 方法，作用是将 pending.next() 赋值给 nextName
    if (!hasNextService())
        throw new NoSuchElementException();
    // 标记了当前要进行加载和实例化的类名
    String cn = nextName;
    // 然后将 nextName 属性置为 null
    nextName = null;
    Class<?> c = null;
    try {
        // 通过类名 cn 、false 属性（代表不进行初始化）、类加载器 loader（创建 LazyIterator 时传入的线程上下文类加载器，即应用类加载器）这三个参数调用 Class#forName 方法进行类的加载工作
        c = Class.forName(cn, false, loader);
    } catch (ClassNotFoundException x) {
        fail(service, "Provider " + cn + " not found");
    }
    if (!service.isAssignableFrom(c)) {
        fail(service, "Provider " + cn  + " not a subtype");
    }
    try {
        // 将加载成功的类进行实例化，然后强转为 HelloSpi 类型
        S p = service.cast(c.newInstance());
        // 将实例化后的对象放到 providers 属性中
        providers.put(cn, p);
        // 返回实例化的对象，在 for 循环中通过调用 say 方法执行实现类的逻辑
        return p;
    } catch (Throwable x) {
        fail(service, "Provider " + cn + " could not be instantiated", x);
    }
    throw new Error();          // This cannot happen
}
```

至此，就完成了 ServiceLoader 类基于线程上下文类加载器实现 Java SPI 机制的整个流程的源码分析。




### 3.5 为什么 SPI 接口实现类需要一个无参构造器


在 2.2 创建接口的实现类部分中，我提到过一句话：**实现类必须要有无参构造函数**，接下来分析一下提到这句话的原因。


其实很简单，在 LazyIterator#nextService() 方法中实例化加载完成的类时，是通过 Class#newInstance() 方法完成实例化的，这个方法的源码如下所示。


```java
@CallerSensitive
public T newInstance() {
    // 省略大部分代码...
    
    Class<?>[] empty = {};
    // 调用 getConstructor0 获取无参构造函数，若没获取到会报 NoSuchMethodException
    final Constructor<T> c = getConstructor0(empty, Member.DECLARED);
    
    // 省略大部分代码...
}

// 获取无参构造函数 Class#getConstructor0
private Constructor<T> getConstructor0(Class<?>[] parameterTypes, int which) throws NoSuchMethodException {
    // 获取该类的所有构造函数，进行遍历
    Constructor<T>[] constructors = privateGetDeclaredConstructors((which == Member.PUBLIC));
    for (Constructor<T> constructor : constructors) {
        // 返回无参构造函数
        if (arrayContentsEq(parameterTypes, constructor.getParameterTypes())) {
            return getReflectionFactory().copyConstructor(constructor);
        }
    }
    throw new NoSuchMethodException(getName() + ".<init>" + argumentTypesToString(parameterTypes));
}
```

正如上面的代码所示，在进行类的实例化时，是通过调用 Class#getConstructor0 方法来获取构造器的，而这个方法获取的正是无参构造器，这也就是**实现类必须要有无参构造函数**的原因。




## 4. 小结


本文实现了一个基于 Java SPI 机制的的简单 demo，然后分析了 ServiceLoader 类基于线程上下文类加载器实现 Java SPI 的源码。


关于 Java SPI 的部分就总结到这里，各位看官如果有其他补充的希望能够告诉我，一起学习学习。


希望能够帮助到大家。




## 5. 参考资料


- [高级开发必须理解的Java中SPI机制](https://www.jianshu.com/p/46b42f7f593c)
- [深入理解Java虚拟机](https://book.douban.com/subject/24722612/)

最后，本文收录于个人语雀知识库: [我所理解的后端技术](https://www.yuque.com/planeswalker/bankend)，欢迎来访。