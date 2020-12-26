# spi

## 什么是spi

+ 已知接口，动态加载第三方提供的接口的实现类

+ 例子：JDBC、日志门面接口

+ 原理：使用类加载器 `classloader`：
  
  + SPI 的接口由 Java 核心库负责加载（启动类加载器Bootstrap Classloader）
  + 而 SPI 的实现代码则是作为 Java 应用所依赖的 jar 包被包含进类路径（CLASSPATH）里，是由系统类加载器(System ClassLoader）加载
  + 启动类加载器是无法找到 SPI 的实现类的，因为依照 双亲委派模型，BootstrapClassloader无法委派AppClassLoader来加载类
  + 解决：线程上下文类加载器（破坏了“双亲委派模型”，可以在执行线程中抛弃双亲委派加载链模式，使程序可以逆向使用类加载器）

+ **通俗的讲**：JDK提供了一种帮助第三方实现者加载服务（如数据库驱动、日志库）的便捷方式，只要第三方遵循约定（把类名写在/META-INF/services里），当服务启动时就会去扫描所有jar包里符合约定的类名，再调用forName加载，由于启动类加载器没法加载实现类，就把加载它的任务交给了线程上下文类加载器。

## 使用介绍

+ 当服务提供者提供了接口的一种具体实现后，在jar包的`META-INF/services`目录下创建一个以“接口全限定名”为命名的文件，内容为实现类的全限定名；**如果ServiceLoader在load时提供Classloader**，则可以从其他的目录读取。
+ 接口实现类所在的jar包放在主程序的classpath中；
+ 主程序通过java.util.ServiceLoder动态装载实现模块，它通过扫描META-INF/services目录下的配置文件找到实现类的全限定名，把类加载到JVM；
+ SPI的实现类必须携带一个不带参数的构造方法；

## 源码分析

```java
private static final String PREFIX = "META-INF/services/";

// 要加载的接口
private Class<S> service;

// The class loader used to locate, load, and instantiate providers
private ClassLoader loader;

// 用于缓存已经加载的接口实现类，其中key为实现类的完整类名
private LinkedHashMap<String,S> providers = new LinkedHashMap<>();

// 用于延迟加载接口的实现类
private LazyIterator lookupIterator;

public void reload() {
    providers.clear();
    lookupIterator = new LazyIterator(service, loader);
}
private ServiceLoader(Class<S> svc, ClassLoader cl) {
    service = Objects.requireNonNull(svc, "Service interface cannot be null");
    loader = (cl == null) ? ClassLoader.getSystemClassLoader() : cl;
    acc = (System.getSecurityManager() != null) ? AccessController.getContext() : null;
    reload();
}

private class LazyIterator implements Iterator<S> {

    Class<S> service;
    ClassLoader loader;
    Enumeration<URL> configs = null;
    Iterator<String> pending = null;
    String nextName = null;

    private LazyIterator(Class<S> service, ClassLoader loader) {
        this.service = service;
        this.loader = loader;
    }

    public boolean hasNext() {
        if (nextName != null) {
            return true;
        }
        if (configs == null) {
            try {
                String fullName = PREFIX + service.getName();
                if (loader == null)
                    configs = ClassLoader.getSystemResources(fullName);
                else
                   configs = loader.getResources(fullName);
            } catch (IOException x) {
                fail(service, "Error locating configuration files", x);
            }
        }
        while ((pending == null) || !pending.hasNext()) {
            if (!configs.hasMoreElements()) {
                return false;
            }
            pending = parse(service, configs.nextElement());
        }
        nextName = pending.next();
        return true;
    }

    public S next() {
        if (!hasNext()) {
            throw new NoSuchElementException();
        }
        String cn = nextName;
        nextName = null;
        Class<?> c = null;
        try {
            // 遍历时，查找类对象
            c = Class.forName(cn, false, loader);
        } catch (ClassNotFoundException x) {
            fail(service,  "Provider " + cn + " not found");
        }
        if (!service.isAssignableFrom(c)) {
            fail(service, "Provider " + cn  + " not a subtype");
        }
        try {
            // 遍历时，才会初始化类实例对象
            S p = service.cast(c.newInstance());
            providers.put(cn, p);
            return p;
        } catch (Throwable x) {
            fail(service, "Provider " + cn + " could not be instantiated", x);
        }
        throw new Error();
    }

    public void remove() {
        throw new UnsupportedOperationException();
    }
}
```

**首先，**ServiceLoader实现了Iterable接口，所以它有迭代器的属性，这里主要都是实现了迭代器的hasNext和next方法。这里主要都是调用的lookupIterator的相应hasNext和next方法，lookupIterator是懒加载迭代器。

**其次，**LazyIterator中的hasNext方法，静态变量PREFIX就是”META-INF/services/”目录，这也就是为什么需要在classpath下的META-INF/services/目录里创建一个以服务接口命名的文件。

**最后，**在遍历时，通过反射方法Class.forName()加载类对象，并用newInstance方法将类实例化，并把实例化后的类缓存到providers对象中，(LinkedHashMap<String,S>类型） 然后返回实例对象。

## 缺点

+ 虽然ServiceLoader也算是使用的延迟加载，**但是基本只能通过遍历全部获取，也就是接口的实现类全部加载并实例化一遍**。如果你并不想用某些实现类，它也被加载并实例化了，这就造成了浪费。
+ 获取某个实现类的方式不够灵活，**只能通过Iterator形式获取**，不能根据某个参数来获取对应的实现类
+ java spi 不能使用 aop 与 ioc
+ JAVA SPI可能会丢失加载扩展点异常信息，导致追踪问题很困难

# dubbo SPI

## 总览

SPI(Service Provider Interface)是服务发现机制，Dubbo没有使用jdk SPI而对其增强和扩展：

1. **jdk SPI仅通过接口类名获取所有实现，但是Duboo SPI可以根据接口类名和key值获取具体一个实现**
2. **可以对扩展类实例的属性进行依赖注入，即IOC**
3. **可以采用装饰器模式实现AOP功能**
