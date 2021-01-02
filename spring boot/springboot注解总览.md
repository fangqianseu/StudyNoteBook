# 容器注入

## @configuration

- 基本使用
  - 配置类里面使用@Bean标注在方法上给容器注册组件，默认 单实例的
  
  - 配置类本身也是组件   内嵌了  `@Component` 注解

  - `proxyBeanMethods ` 属性
  
    - `@Configuration(proxyBeanMethods = false)` 每次主动调用 该配置类的方法 是否返回被代理的对象：
  
    默认`True` 被代理，返回容器内的对象；`False`不会被代理，每次新生成对象
  
    ```java
    //  configbean 为容器中的配置类对象
    MyConfig configbean = run.getBean(MyConfig.class);
    
    //如果@Configuration(proxyBeanMethods = true)代理对象调用方法。SpringBoot总会检查这个组件是否在容器中有。
    //保持组件单实例
    User user = configbean.user01();
    ```
  
  - **Full模式与Lite模式**
  
    - Full  `proxyBeanMethods = true` : 保证每个@Bean方法被调用多少次返回的组件都是单实例的
    - Lite  `proxyBeanMethods = false`每个@Bean方法被调用多少次返回的组件都是新创建的
  
    配置 类组件之间无依赖关系用 **Lite模式加速容器启动**过程，**减少判断**
  
    配置类组件之间有依赖关系，方法会被调用得到之前单实例组件，用Full模式
  

## @Bean

+ 给容器中添加组件。
+ 默认以方法名作为组件的id，也可以 `@Bean("name")`指定
+ 给@Bean标注的方法传入了对象参数，这个参数的值就会从容器中找（先name、后type，不需要 @autowired注解）

## @Component、@Controller、@Service、@Repository

都可以声明为组件

## @Import

+ 必须写在 组件 的类上，默认组件的名字就是  **全类名**

  ```java
  @Import({User.class, DBHelper.class})
  @Configuration(proxyBeanMethods = false)
  // @Component、@Controller、@Service、@Repository 皆可
  public class MyConfig {
  }
  ```

## @Conditional

+ 条件装配：满足Conditional指定的条件，则进行组件注入
+ 可以标注在 类 上，对该类下的所有 `@Bean`生效

![image.png](https://img-1252710297.cos.ap-nanjing.myqcloud.com/img/20210102225353.png)

+ 常用的有

  + Class相关： @ConditionalOnClass、@ConditionalOnMissingClass

  + Bean相关：@ConditionalOnBean、@ConditionalOnMissingBean  根据类型判断；如果要根据name，指明 name属性

  + 属性值相关：@ConditionalOnProperty

    例如 `@ConditionalOnProperty( prefix = "spring.redis", name="enabled",havingValue ="true",MatchIfMissing = true}`  MatchIfMissing = true 没有配置 也生效

## @ImportResource

+ 用于 原生配置文件 (`spring.xml`)引入

  ```java
  @ImportResource("classpath:beans.xml")
  public class MyConfig {}
  ```

  

# 配置绑定

## @ConfigurationProperties

+ `@ConfigurationProperties(prefix = "prefix")` 指定配置文件的前缀名

+ 默认不被spring扫描，需要写在 `@Component`注解上

  ```java
  @Component //需要暴露在容器中
  @ConfigurationProperties(prefix = "mycar")
  public class Car {
      private String brand;
      private Integer price;
  ```

## @EnableConfigurationProperties

+ `@EnableConfigurationProperties(Properties.class)` 

+ 如果一个配置类只配置@ConfigurationProperties注解，而没有使用@Component，那么在IOC容器中是获取不到properties 配置文件转化的bean。

  通过 @EnableConfigurationProperties 可以把第三方  @ConfigurationProperties 的类进行注入。

