# SpringBoot原理

Spring原理【[Spring注解](https://www.bilibili.com/video/BV1gW411W7wy?p=1)】、**SpringMVC**原理、**自动配置原理**、SpringBoot原理

## SpringBoot启动过程

- 创建 **SpringApplication**

- 保存一些信息。

- - 判定当前应用的类型。ClassUtils。Servlet
  - **==bootstrappers==**：初始启动引导器（List<Bootstrapper>）：去spring.factories文件中找 org.springframework.boot.Bootstrapper
  - 找  ==**ApplicationContextInitializer**==；去spring.factories 找 ApplicationContextInitializer

- - - 存入 List<ApplicationContextInitializer<?>> initializers

- - 找 **==ApplicationListener==**  ；应用监听器。去spring.factories**找** ApplicationListener

- - - 存入 List<ApplicationListener<?>> listeners

- 运行 **SpringApplication**

- - **StopWatch**：记录应用的启动时间
  - 创建引导上下文（Context环境）createBootstrapContext()

- - - 获取到所有之前的 **bootstrappers 挨个执行** intitialize() 来完成对引导启动器上下文环境设置

- - 让当前应用进入**headless**模式。**java.awt.headless**
  - 获取所有 ==**RunListener**==（运行监听器）【为了方便所有Listener进行事件感知】

- - - getSpringFactoriesInstances 去spring.factories**找** SpringApplicationRunListener. 

- - 遍历 ==SpringApplicationRunListener 调用 starting== 方法；

- - - 相当于通知所有感兴趣系统正在启动过程的人，项目正在 starting。

- - 保存命令行参数；ApplicationArguments
  - 准备环境 prepareEnvironment（）;

- - - 返回或者创建基础环境信息对象。StandardServletEnvironment
    - 配置环境信息对象。

- - - - 读取所有的配置源的配置属性值。

- - - 绑定环境信息
    - ==监听器调用 listener.environmentPrepared()==；通知所有的监听器当前环境准备完成

- - 创建IOC容器（createApplicationContext（））

- - - 根据项目类型（Servlet）创建容器

      当前会创建 AnnotationConfigServletWebServerApplicationContext

- - 准备ApplicationContext IOC容器的基本信息   **prepareContext()**

- - - 保存环境信息
    - IOC容器的后置处理流程。
    - 应用初始化器 applyInitializers：创建 **SpringApplication** 时保存的 **initializers**

- - - - 遍历所有的 ==ApplicationContextInitializer 。调用 initialize.==。来对ioc容器进行初始化扩展功能
      - 遍历所有的 ==listener 调用 contextPrepared==。EventPublishRunListenr；通知所有的监听器contextPrepared

- - - 所有的==监听器 调用 contextLoaded==。通知所有的监听器 contextLoaded；

- - 刷新IOC容器。refreshContext

- - - 创建容器中的所有组件（Spring注解）

- - 容器刷新完成后工作？afterRefresh
  - ==所有监听器 调用 listeners.started(context==); 通知所有的监听器 started
  - 调用所有runners；callRunners()

- - - 获取容器中的 ApplicationRunner 

    - 获取容器中的  CommandLineRunner

    - 合并所有runner并且按照@Order进行排序

    - ==遍历所有的runner。调用 run 方法== 

      ApplicationRunner 先于 CommandLineRunner 执行

- - 如果以上有异常，

- - - ==调用Listener 的 failed==

- - ==调用所有监听器的 running 方法==  listeners.running(context); 通知所有的监听器 running 
  - ==running如果有问题。继续通知 failed== 。调用所有 Listener 的 failed；通知所有的监听器 failed



**SpringApplicationRunListener** 接口：对springboot启动进行事件监听回调

![image.png](https://img-1252710297.cos.ap-nanjing.myqcloud.com/img/20210117163508.png)

执行顺序：starting -》 environmentPrepared-》contextPrepared-》  contextLoaded-》 started -》 running

如果有异常，执行 failed