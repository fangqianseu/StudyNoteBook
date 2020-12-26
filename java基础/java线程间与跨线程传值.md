# 线程内传值-TheadLocal

TheadLocal常用于线程内传值 避免了繁琐的参数传递过程 切线程安全

## 用法

使用非常简单，定义如下

```java
public class WebHolder {

    static final ThreadLocal<Context> CONTEXTS = new ThreadLocal<>();

    public static Context get() {
        return CONTEXTS.get();
    }

    public static void set(Context context) {
        CONTEXTS.set(context);
    }

    public static void remove() {
        CONTEXTS.remove();
    }
}
```

使用:

```java
WebHolder.set(context);    // 赋值
context = WebHolder.get()  // 获取
```

+ 这样，在同一个线程中的任意一个地方，都可以访问ThreadLocal的值；
+ ThreadLocal 在多线程环境下也安全，每个线程都有自己的ThreadLocal变量，一个线程无法访问到其他线程的值

## 原理

+ 每个线程维护一个 `ThreadLocalMap` 映射表，这个映射表的 key 是 `ThreadLocal`实例本身，value 是真正需要存储的 `Object`。
+  `ThreadLocal` 本身并不存储值，它只是作为一个 `key` 来让线程从 `ThreadLocalMap` 获取该线程对应的 `value`

### get()

这是get()方法的源码

```java
public T get() {
  // ThreadLocalMap 存储在线程中
  Thread t = Thread.currentThread();
  ThreadLocalMap map = getMap(t);
  if (map != null) {
    ThreadLocalMap.Entry e = map.getEntry(this);
    if (e != null) {
      @SuppressWarnings("unchecked")
      T result = (T)e.value;
      return result;
    }
  }
  return setInitialValue();
}
```

可以很清楚的看出，`ThreadLocal`的值存储在了与线程相关的`ThreadLocalMap`中；

- 获取当前线程t
- 利用当前线程，获取一个ThreadLocalMap的对象
- 如果ThreadLocalMap对象不为空，则设置值，否则创建这个ThreadLocalMap对象并设置值

其中`getEntry()`方法使用了`AtomicInteger`对象的`threadLocalHashCode`作为key保证不冲突

### set()

set()方法源码如下

```java
public void set(T value) {
  Thread t = Thread.currentThread();
  ThreadLocalMap map = getMap(t);
  if (map != null)
    map.set(this, value);
  else
    createMap(t, value);
}
```

可以看出，与get的流程类似，也是将将value存储到 `ThreadLocalMap`中

## 内存泄漏

+ `ThreadLocalMap`使用`ThreadLocal`的弱引用作为`key`，如果一个`ThreadLocal`没有外部强引用来引用它，那么系统 GC 的时候，这个`ThreadLocal`势必会被回收，
+ 这样，`ThreadLocalMap`中就会出现`key`为`null`的`Entry`，由于`ThreadLocalMap`的生命周期与线程一样，这些`key`为`null`的`Entry`的`value`就会一直存在一条强引用链：`Thread -> ThreaLocalMap -> Entry -> value`，无法回收，造成内存泄漏，特别是在线程池场景。
+ 引用类型：
  + 强引用: 声明例如 `int a = 1`
  + 软引用: 内存空间足够不清除
  + 弱引用: 内存空间足够也清除
  + 虚引用: 追踪垃圾回收状态
+ 在`ThreadLocal`的`get()`,`set()`,`remove()`的时候都会清除线程`ThreadLocalMap`里所有`key`为`null`的`value`。
+ 在使用 `threadlocal` 的时候 也要像使用 lock 一样，**使用后进行remove()**

# 线程间传值

在java中，线程间值得传递主要通过 `InheritableThreadLocal`， 是 `ThreadLocal`的子类，可以实现父子线程之间的数据传递，在子线程中能够父线程中的`ThreadLocal`变量。

## 使用

```java
public static void main(String[] args) {
  ThreadLocal<String> local = new InheritableThreadLocal<>();
  local.set("主线程");

  Thread thread = new Thread(()->{
    System.out.println(local.get());
  });
  thread.run();
}

// 输出 ”from 主线程"
```

这样就实现了数据在线程之间的传递

## 源码分析

看一下InheritableThreadLocal的源码

![image-20200924201711954](https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20200924201719.png)

 可以看到 很简单 重写了3个方法。对比threadlocal的代码，发现主要的变化是`createMap()、getMap()`两个方法的对象不同，由`t.threadLocals`变成了 `t.inheritableThreadLocals`。回到 Thread中，发现在`init()`方法中的处理

![image-20200924202149814](https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20200924202149.png)

![image-20200924202245536](https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20200924202245.png)

可以看出，子线程会获取父线程的中`inherThreadLocals`的值，对自己的`inherThreadLocals`通过`createInheritedMap()`方法进行赋值。这样就完成了线程间的传值。

## 问题

在线程池的场景，使用 `InheritableThreadLocal`会产生问题,例如

```java
public static void main(String[] args) {
  ThreadLocal<String> local = new InheritableThreadLocal<>();
  local.set("from 主线程1");

  ExecutorService executorService = Executors.newFixedThreadPool(1);

  executorService.submit(() -> {
    System.out.println("1" + local.get());
  });

  local.set("from 主线程2");
  executorService.submit(() -> {
    System.out.println("2" + local.get());
  });
}
/** 
1from 主线程1
2from 主线程1
**/
```

可以看出，两次赋值后，ThreadLocal的值没有改变。

这是因为线程池是对已有线程的复用，`init()`方法只会执行一次，只能得到初始化时得到的父线程的值

解决方法  `TransmittableThreadLocal`   [参考](https://github.com/alibaba/transmittable-thread-local)

