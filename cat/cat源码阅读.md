# cat代码与架构

- cat-client: 客户端，上报监控数据 。
  
  待监控的项目引入cat-client模块，对异常、事件进行打点统计

- cat-consumer: 服务端，收集监控数据进行统计分析，构建丰富的统计报表。
  
  对于cat客户端发送的的各种类型数据，在这里进行处理

- cat-alarm: 实时告警，提供报表指标的监控告警
  
  其中开放了spi接口，可以对告警方式、告警内容进行自定义设置

- cat-hadoop: 数据存储，logview 存储至 Hdfs

- cat-home: 管理端，报表展示、配置管理等
  
  对cat web端的大盘显示做定制化开发，就是在这里进行的，比如异常的显示内容定制化

![image-20200927165636835](https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20200927165636.png)

上图是cat的整体关键架构。为了提升性能，其中大量运用了异步的方式

+ 在client上报数据到server时，采用的是基于Netty框架的NIO方式，提高吞吐率；
+ 在处理server端收到的各种类型的消息时，采用了一个异步队列进行异步削峰，发送给不同的MessageAnalyzer，把消息接收与消息处理进一步解耦；同时，MessageAnalyzer也可以设置为线程池，进一步加强消息的处理能力

# cat client

## 初始化流程

这是客户端手动打点的示例代码

```java
Transaction t=Cat.newTransaction("logTransaction", "logTransaction");
TimeUnit.SECONDS.sleep(30);
t.setStatus("0");
t.complete();
```

通过 `Cat.newTransaction()`创建一个`Transaction` 对象，即可进行手动埋点

下面查看源码

<img src="https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20200927193658.png" alt="image-20200927193658114" style="zoom:50%;" />

进入 `getProducer()`方法，源码如下

<img src="https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20200927193801.png" alt="image-20200927193801963" style="zoom:50%;" />

可以看出，该方法主要做了2件事

1. 判断cat的配置是否需要初始化
   
   `configNeedInit` cat是否需要初始化，用于开关cat监控系统
   
   `configInit` cat是否已经初始化，是cat是否初始化的标志位

2. 调用 `checkAndInitialize()` 进行初始化，源代码如下

<img src="https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20200927194942.png" alt="image-20200927194942086" style="zoom:50%;" />

<img src="/Users/fangqian/Library/Application%20Support/typora-user-images/image-20200927195001998.png" alt="image-20200927195001998" style="zoom:50%;" />

该方法主要做了一下几件事

+ 利用双重检测，保证 initialize方法只执行一次，进行单次初始化加载
+ 在 CatHome路径下，获取 client.xml 文件，获取配置的初始化参数
+ 使用了 `PlexusContainer` 容器，加载配置文件`/META-INF/plexus/plexus.xml`，完成IOC容器的初始化。
+ 加载 `com.dianping.cat.CatClientModule`文件，其中，启动了2个线程
  + `StatusUpdateTask` : 每秒钟上报客户端的基础信息，如JVM信息，既心跳信息
  + `LocalAggregator.DataUploader` :  每秒钟上报本地已有的Transaction与Event

至此 初始化流程完成

## 构造消息

从 `Cat.newTransaction()`入手

<img src="https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20200927202048.png" alt="image-20200927202048331" style="zoom:50%;" />

`setup()`方法是对manager的初始化，主要是新建了 `com.dianping.cat.message.internal.DefaultMessageManager.Context` 对象：

<img src="https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20200927202534.png" alt="image-20200927202534179" style="zoom:50%;" />

Context类是DefaultMessageManager包私有的内部类，存储了一次Transaction的相关数据，如线程id、线程名、domain、主机名、ip等。其中，m_tree为当前的消息体，m_stack为多个嵌套消息体填充的栈。 

再向后看m_manager.start()方法，其中第一行调用了 `com.dianping.cat.message.internal.DefaultMessageManager#getContext()` 方法：

<img src="https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20200927203436.png" alt="image-20200927203436567" style="zoom:50%;" />

其中，m_context为 `private ThreadLocal<Context> m_context = new ThreadLocal<Context>();` 一个 ThreadLocal对象。通过使用`ThreadLocal<Context>`，使Context中实际的消息保证了线程安全。

在之后的start方法中，<img src="https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20200927203719.png" alt="image-20200927203719576" style="zoom:50%;" />

如果当前Context的栈m_stack不为空,那么接着之前的消息后边，将当前消息构造为一个孩子结点。如果当前消息之前没有其他消息当前消息就为父节点。

至此，消息体构造完毕。

## 消息上报

`t.complete();`方法会触发消息的上报。该方法通过`com.dianping.cat.message.internal.DefaultMessageManager#end()`，最终调用了 `com.dianping.cat.message.internal.DefaultMessageManager.Context#end()` 方法，具体源码如下

<img src="https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20200927204811.png" alt="image-20200927204811689" style="zoom:50%;" />

从stack遍历，判断当前的transaction是否为最初的根transaction；如果是，则证明所有的transaction都已完成，需要发送当前的MessageTree集合，调用`flush()`-> `sender.send(tree)` -> `offer()`:

<img src="https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20200927210123.png" alt="image-20200927210123513" style="zoom:50%;" />

塞入 `MessageQueue` 中，等待`TcpSocketSender`的异步发送。

# cat Alert

alert是对告警处理的模块，用于发送告警信息

该模块的核心类为 `com.dianping.cat.alarm.spi.AlertManager`，主要功能有2个

+ 根据告警类型，发送告警信息
+ 根据恢复时间，发送告警恢复信息

首先是初始化方法 

<img src="https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20200927214749.png" alt="image-20200927214749093" style="zoom:50%;" />

初始化方法定义了2个线程：

+ `SendExecutor` 发送告警信息
+ `RecoveryAnnouncer` 发送告警恢复信息

首先进入SendExecutor

<img src="https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20200927214859.png" alt="image-20200927214859054" style="zoom:50%;" />

该类很简单，就是每5毫秒从 `new LinkedBlockingDeque<>(10000)` 中获取待发送的告警信息，调用`send`方法发送

send()方法源码如下

```java
private boolean send(AlertEntity alert) {
  // 告警的具体类型
  boolean result = false;
  String type = alert.getType().getName();
  String group = alert.getGroup();
  String level = alert.getLevel().getLevel();
  String alertKey = alert.getKey();

  // 查询告警走的告警通道与静默时间
  List<AlertChannel> channels = m_policyManager.queryChannels(type, group, level);
  int suspendMinute = m_policyManager.querySuspendMinute(type, group, level);

  // 添加到告警恢复信息集合
  m_unrecoveredAlerts.put(alertKey, alert);

  Pair<String, String> pair = m_decoratorManager.generateTitleAndContent(alert);
  String title = pair.getKey();

  /*
    判断当前告警是否在静默时间之内
    遇到一个新的 alert之后，会放到 m_sendedAlerts之中，
    如果在遇到第二次出现，判断上次出现的时间加上静默时间是否在当前时间之后
  */
  if (suspendMinute > 0) {
    if (isSuspend(alertKey, suspendMinute)) {
      return true;
    } else {
      m_sendedAlerts.put(alertKey, alert);
    }
  }


  SendMessageEntity message = null;

  // 获取告警对应的频道 对每个频道发送告警信息
  for (AlertChannel channel : channels) {
    String contactGroup = alert.getContactGroup();
    List<String> receivers = m_contactorManager.queryReceivers(contactGroup, channel, type);
    //去重
    removeDuplicate(receivers);

    if (receivers.size() > 0) {
      String rawContent = pair.getValue();

      if (suspendMinute > 0) {
        rawContent = rawContent + "<br/>[告警间隔时间]" + suspendMinute + "分钟";
      }
      String content = m_splitterManager.process(rawContent, channel);
      message = new SendMessageEntity(group, title, type, content, receivers);

      if (m_senderManager.sendAlert(channel, message)) {
        result = true;
      }
    } else {
      Cat.logEvent("NoneReceiver:" + channel, type + ":" + contactGroup, Event.SUCCESS, null);
    }
  }

  String dbContent = Pattern.compile("<div.*(?=</div>)</div>", Pattern.DOTALL).matcher(pair.getValue()).replaceAll("");

  // 发送一次告警之后 存储到数据库
  if (message == null) {
    message = new SendMessageEntity(group, title, type, "", null);
  }
  message.setContent(dbContent);
  m_alertService.insert(alert, message);
  return result;
}
```

接下来查看RecoveryAnnouncer

```java
public void run() {
  while (true) {
    long current = System.currentTimeMillis();
    String currentStr = m_sdf.format(new Date(current));

    // 已经发送恢复告警信息的集合
    List<String> recoveredItems = new ArrayList<String>();

    // 获取在发送告警线程中 添加到告警恢复集合中的所有信息
    for (Entry<String, AlertEntity> entry : m_unrecoveredAlerts.entrySet()) {
      try {
        String key = entry.getKey();
        AlertEntity alert = entry.getValue();
        // 设置的恢复时间
        int recoverMinute = queryRecoverMinute(alert);
        long alertTime = alert.getDate().getTime();
        // 已经恢复的时间
        int alreadyMinutes = (int) ((current - alertTime) / MILLIS1MINUTE);

        // 当然时间超过了设置的告警恢复时间
        if (alreadyMinutes >= recoverMinute) {
          recoveredItems.add(key);
          sendRecoveryMessage(alert, currentStr);
        }
      } catch (Exception e) {
        Cat.logError(e);
      }
    }

    // 从待发送的告警恢复信息中删除已发送的信息
    for (String key : recoveredItems) {
      m_unrecoveredAlerts.remove(key);
    }


    // 告警恢复线程 每1分钟执行一次
    long duration = System.currentTimeMillis() - current;
    if (duration < MILLIS1MINUTE) {
      long lackMills = MILLIS1MINUTE - duration;

      try {
        TimeUnit.MILLISECONDS.sleep(lackMills);
      } catch (InterruptedException e) {
        Cat.logError(e);
      }
    }
  }
}
```

下面介绍一下如何自定义sender

cat内置了一个抽象类 `AbstractSender`，实现了基础的告警发送方法，默认是使用http请求，指定请求url、请求方法、请求内容、已经响应成功码，即可发送告警信息

![image-20200928101050604](https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20200928101050.png)

下图是一段post请求的核心代码

![image-20200928101509149](https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20200928101509.png)

主要实现了2个功能

+ 建立http链接，并发送content中的内容
+ 获取响应信息，判断响应信息中是否包含配置的成功码

**注意**：这里有个需要注意的，配置的successCode必须是返回的内容中的一部分，而不是响应状态码。比如如果配置的successcode是200，则还需要接口返回时，包含 200 这个字符串，否则会判断发送失败。

# cat server

server端接受client上报服务，解析、持久化。在cat中，使用的是war包的形式，即cat-home模块，以Servlet方式的形式启动。下面分析下启动流程

由于是web服务，所以看看`web.xml`文件中的内容。可以看到，主要定义了2个Servlet服务：`CatServlet`与`MVC`

![image-20200930114628009](https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20200930114628.png)

按照优先级的不同，`CatServlet`会先于`MVC`进行初始化

首先看看`CatServlet`

`CatServlet extends AbstractContainerServlet` 继承了AbstractContainerServlet，重写了了`initComponents()`方法，用于将加载`cat-client-xml`和`cat-server-xml`文件，具体的init逻辑还是在 `AbstractContainerServlet#init()`

```java
public void init(ServletConfig config) throws ServletException {
  super.init(config);

  // ioc容器 plexus 初始化
  try {
    if (m_container == null) {
      m_container = ContainerLoader.getDefaultContainer();
    }

   // 日志对象初始化
    m_logger = ((DefaultPlexusContainer) m_container).getLoggerManager().getLoggerForComponent(
      getClass().getName());

    // 初始化组件模块 这个方法就是在 CatServlet 重写的方法
    initComponents(config);

  } catch (Exception e) {
    if (m_logger != null) {
      m_logger.error("Servlet initializing failed. " + e, e);
    } else {
      System.out.println("Servlet initializing failed. " + e);
      e.printStackTrace(System.out);
    }

    throw new ServletException("Servlet initializing failed. " + e, e);
  }
}
```

重点看看初始化组件模块的方法 initComponents()

![image-20200930141908807](https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20200930141908.png)

这个方法主要是通过`ctx.lookup(ModuleInitializer.class);`找到所有相关`ModuleInitializer`的类，并执行`execute`方法。具体的执行逻辑在`org.unidal.initialization.DefaultModuleInitializer`中

```java
@Override
public void execute(ModuleContext ctx) {
  // 获取顶层 Module 即 cat-home
  Module[] modules = m_manager.getTopLevelModules();

  // 执行初始化方法
  execute(ctx, modules);
}

@Override
public void execute(ModuleContext ctx, Module... modules) {
  Set<Module> all = new LinkedHashSet<Module>();

  info(ctx, "Initializing top level modules:");

  for (Module module : modules) {
    info(ctx, "   " + module.getClass().getName());
  }

  try {
    // 根据顶层Module,获取所有依赖到的modules,调用他们的setup方法
    expandAll(ctx, modules, all);

    // 再次检测是否初始化
    for (Module module : all) {
      if (!module.isInitialized()) {
        executeModule(ctx, module, m_index++);
      }
    }
  } catch (Exception e) {
    throw new RuntimeException("Error when initializing modules! Exception: " + e, e);
  }
}

private void expandAll(ModuleContext ctx, Module[] modules, Set<Module> all) throws Exception {
      if (modules != null) {
         for (Module module : modules) {

            expandAll(ctx, module.getDependencies(ctx), all);

            if (!all.contains(module)) {
               if (module instanceof AbstractModule) {
        // 调用各个module实现类的setup
                  ((AbstractModule) module).setup(ctx);
               }

               all.add(module);
            }
         }
      }
   }
```

![](https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20200930143050.png)

这是所有的 AbstractModule 的子类，每个类都有`getDependencies()`方法，用于定义自己依赖的模块，梳理如下

```
CatHomeModule  -> CatConsumerModule   -> CatCoreModule -> CatClientModule
                            |                     |
                             -> CatHadoopModule -> 
```

![image-20200930143320014](https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20200930143320.png)

通过debug可以看出，TopLevelModels是 catHomeModule；

![image-20200930143523471](https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20200930143523.png)

而catHomeModule的依赖module，依次为CatClientModule、CatClientModule、CatCoreModule、CatConsumerModule。

catHomeModule比较特殊，除了`execute`方法之外，唯一实现了 `setup`方法的类

<img src="https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20200930144755.png" alt="image-20200930144754937" style="zoom:50%;" />

这个方法启动了`TCPSocketReceiver`,是一个 netty 事件服务器，用来接收客户端的上报的数据连接连接请求:

```java
// 开启service接收服务器
public synchronized void startServer(int port) throws InterruptedException {
        boolean linux = getOSMatches("Linux") || getOSMatches("LINUX");
        int threads = 24;
        ServerBootstrap bootstrap = new ServerBootstrap();

  // 如果是linux系统 开启Epoll模式
        m_bossGroup = linux ? new EpollEventLoopGroup(threads) : new NioEventLoopGroup(threads);
        m_workerGroup = linux ? new EpollEventLoopGroup(threads) : new NioEventLoopGroup(threads);
        bootstrap.group(m_bossGroup, m_workerGroup);
        bootstrap.channel(linux ? EpollServerSocketChannel.class : NioServerSocketChannel.class);

  // 增加编解码器
        bootstrap.childHandler(new ChannelInitializer<SocketChannel>() {
            @Override
            protected void initChannel(SocketChannel ch) throws Exception {
                ChannelPipeline pipeline = ch.pipeline();

                pipeline.addLast("decode", new MessageDecoder());
                pipeline.addLast("encode", new ClientMessageEncoder());
            }
        });

        bootstrap.childOption(ChannelOption.SO_REUSEADDR, true);
        bootstrap.childOption(ChannelOption.TCP_NODELAY, true);
        bootstrap.childOption(ChannelOption.SO_KEEPALIVE, true);
        bootstrap.childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);

        try {
            m_future = bootstrap.bind(port).sync();
            m_logger.info("start netty server!");
        } catch (Exception e) {
            m_logger.error("Started Netty Server Failed:" + port, e);
        }
    }

    /*
     * MessageConsumer按每个period(整小时一个period)组合了多个解析器，用来解析生产多个报表（12个，如：Transaction、Event、Problem等等）。一个解析器对象-一个有界队列-一个整小时时间组合了一个PeriodTask，轮询的处理这个有界队列中的消息
     */
    public class MessageDecoder extends ByteToMessageDecoder {
        private long m_processCount;

        @Override
        protected void decode(ChannelHandlerContext ctx, ByteBuf buffer, List<Object> out) throws Exception {
            if (buffer.readableBytes() < 4) {
                return;
            }
            buffer.markReaderIndex();
            int length = buffer.readInt();
            buffer.resetReaderIndex();
            if (buffer.readableBytes() < length + 4) {
                return;
            }
            try {
                if (length > 0) {
                    ByteBuf readBytes = buffer.readBytes(length + 4);

                    readBytes.markReaderIndex();
                    //readBytes.readInt();

                    DefaultMessageTree tree = (DefaultMessageTree) CodecHandler.decode(readBytes);

                    // readBytes.retain();
                    readBytes.resetReaderIndex();
                    tree.setBuffer(readBytes);
                    m_handler.handle(tree); //消息消费
                    m_processCount++;

                    long flag = m_processCount % CatConstants.SUCCESS_COUNT;

                    if (flag == 0) {
                        m_serverStateManager.addMessageTotal(CatConstants.SUCCESS_COUNT);
                    }
                } else {
                    // client message is error
                    buffer.readBytes(length);
                    BufReleaseHelper.release(buffer);
                }
            } catch (Exception e) {
                m_serverStateManager.addMessageTotalLoss(1);
                m_logger.error(e.getMessage(), e);
            }
        }
    }
```

总结如下：

cat服务端是使用了war的打包方式，执行入口在`CatServlet`， 从cat-home这一顶层模块开始，找到其他依赖模块
CatClientModule\CatConsumerModule\CatCoreModule），并且依次调用setup方法、execute方法，完成初始化。

# cat consumer

consumer主要是对cat client上报的数据进行处理。在cat中，上报的数据类型不同，会对应不同的数据处理器，这个过程是经由queue进行分发的，且每个处理器都是一个线程池，可以根据数据量进行横向扩容

`TCPSocketReceiver#decode()` 接受的数据，经由 `MessageHandler#handle()` -> `RealtimeConsumer.consume()` 进行处理：

```java
public void consume(MessageTree tree) {
        long timestamp = tree.getMessage().getTimestamp();
        Period period = m_periodManager.findPeriod(timestamp);

        if (period != null) {
            period.distribute(tree);
        } else {
            m_serverStateManager.addNetworkTimeError(1);
        }
    }
```

这段代码出现了重要概念 periodManager。

PeriodManager 是管理每小时的各种类型解析器的管理工具。对于每个小时，periodManager都会生成一批全新的消息解析器，并将上一个小时的解析器销毁。

上文的代码中，获取当前时间戳timestamp，再根据当前时间戳获取当前激活的Period，调用distribute方法进行消息分发。

```java
public void distribute(MessageTree tree) {
        m_serverStateManager.addMessageTotal(tree.getDomain(), 1);
        boolean success = true;
        String domain = tree.getDomain();

  // List<PeriodTask> 是所有消息类型的分析器集合,分发消息时回向所有的消息解析器分发
        for (Entry<String, List<PeriodTask>> entry : m_tasks.entrySet()) {
      // 一种类型报表的解析器 可能有多个线程
            List<PeriodTask> tasks = entry.getValue();
            int length = tasks.size();
            int index = 0;
            boolean manyTasks = length > 1;

      // 从所有的消息解析器中选择一个投递
            if (manyTasks) {
                index = Math.abs(domain.hashCode()) % length;
            }
            PeriodTask task = tasks.get(index);
            boolean enqueue = task.enqueue(tree);

            if (!enqueue) {
                if (manyTasks) {
                    task = tasks.get((index + 1) % length);
          // 进行消息投递
                    enqueue = task.enqueue(tree);

                    if (!enqueue) {
                        success = false;
                    }
                } else {
                    success = false;
                }
            }
        }

      //如果投递没有成功 统计没有成功的次数
        if ((!success) && (!tree.isProcessLoss())) {
            m_serverStateManager.addMessageTotalLoss(tree.getDomain(), 1);
            tree.setProcessLoss(true);
        }
    }
```

刚刚分析了消息的投递流程，下面看一下消息的消费。`AbstractMessageAnalyzer`是所有消息分析器的父类，用于处理各种消息，这点从`com.dianping.cat.analysis.PeriodTask#run`可以看出

<img src="https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20200930161533.png" alt="image-20200930161533367" style="zoom:50%;" />

analyze() 源码如下

```java
public void analyze(MessageQueue queue) {
  // 在当前周期内，不停自旋 处理消息
  while (!isTimeout() && isActive()) {
    //  非阻塞式获取消息
    MessageTree tree = queue.poll();

    if (tree != null) {
      try {
        // 消息解析
        process(tree);
      } catch (Throwable e) {
        m_errors++;

        if (m_errors == 1 || m_errors % 10000 == 0) {
          Cat.logError(e);
        }
      }
    }
  }

  // 如果不在当前周期 或者关闭，处理完队列里的消息再关闭
  while (true) {
    MessageTree tree = queue.poll();

    if (tree != null) {
      try {
        process(tree);
      } catch (Throwable e) {
        m_errors++;

        if (m_errors == 1 || m_errors % 10000 == 0) {
          Cat.logError(e);
        }
      }
    } else {
      break;
    }
  }
}
```

总结一下：

+ 当客户端消息发送到服务端，会存在periodManager对象，检查一下当前小时对应的Period对象是否需要创建；如果存在，那么消息会被分发给各个PeriodTask（每个消息处理器都会收到同一个消息），
+ 每个PeriodTask对应了一种消息解析器MessageAnalyzer，且PeriodTask也可以横向扩展。在MessageAnalyzer中实现了对消息的处理。这里大量使用了Queue，实现了解耦与延迟异步消费。
+ 如果时间进入下一个小时，periodManager会创建一个新的当前小时的Period，并且销毁之前的Period

# 总结

+ cat的结构很清晰，各个模块分工明确，在阅读源码时帮助很大
+ 为了吞吐量和性能，cat中大量使用了异步+线程池的方式，比如连接时使用netty的异步NIO模式，在消息处理时采用queue进行中转解耦，同时在处理消息的MessageAnalyzer中支持横向扩展，提升写入能力
+ cat使用的ioc容器较为小众，初上手时较为劝退，配置文件、注入与spring相比还是更加有门槛的
