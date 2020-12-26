工作中需要开发对hbase操作的api，尝试使用了springboot的自动配置方法，进行加载的可配置化

# springboot的自动配置原理解析

springboot最大的特点是免去了spring繁琐的配置，下面从springboot的入口函数看起：

```java
@SpringBootApplication
public class SamApplication {
    public static void main(String[] args) throws IOException {
        SpringApplication.run(SamApplication.class, args);
    }
}
```

只用了一个 `@SpringBootApplication` 注解，就自动进行了配置。进入`@SpringBootApplication`内部，

![image-20201013151440110](https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20201013151440.png)

可以看到有个 `@EnableAutoConfiguration`注解，意为 开启自动配置。进入源码 可以看到有一句 `@Import(AutoConfigurationImportSelector.class)` 这便是是自动配置的关键类。

注意到selectImports()方法

```java
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
  if (!isEnabled(annotationMetadata)) {
    return NO_IMPORTS;
  }
  AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
    .loadMetadata(this.beanClassLoader);
  AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(autoConfigurationMetadata,
                                                                            annotationMetadata);
  return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
}
```

该方法主要做了这些事情：

+ 判断环境变量中`spring.boot.enableautoconfiguration`是否为`true` （默认为true）。如果被主动设置为false，则不进行自动配置

+ 加载`"META-INF/" + "spring-autoconfigure-metadata.properties"`文件，为自动配置的源数据

+ getAutoConfigurationEntry()加载具体的需要自动配置的对象，通过调用链
  
  `getAutoConfigurationEntry() -> getCandidateConfigurations() ->SpringFactoriesLoader.loadFactoryNames() -> loadSpringFactories()`
  
  其中，loadSpringFactories() 中进行了如下的操作：
  
  + 从当前类路径中获取所有 `META-INF/spring.factories` 文件。该文件存储了很多关于自动配置的k-v，对，封装成一个 Map 。
  + 从返回的 Map 中，获取 `org.springframework.boot.autoconfigure.EnableAutoConfiguration`下的所有值。

这是截取的一部分获取的EnableAutoConfiguration中的信息，根据名字可以看出，这就是springboot提供的关于自动配置的类，通过导入这些类，即可完成自动配置

```properties
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration,\
org.springframework.boot.autoconfigure.cloud.CloudServiceConnectorsAutoConfiguration,\
org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration,\
org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration,\
org.springframework.boot.autoconfigure.couchbase.CouchbaseAutoConfiguration,\
```

# 具体的自动配置类解析

以 `RedisAutoConfiguration` 为例。这是这个类的源码

```java
@Configuration
@ConditionalOnClass(RedisOperations.class)
@EnableConfigurationProperties(RedisProperties.class)
public class RedisAutoConfiguration {

  @Bean
  @ConditionalOnMissingBean(name = "redisTemplate")
  public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory)
    throws UnknownHostException {
    RedisTemplate<Object, Object> template = new RedisTemplate<>();
    template.setConnectionFactory(redisConnectionFactory);
    return template;
  }

  @Bean
  @ConditionalOnMissingBean
  public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory redisConnectionFactory)
    throws UnknownHostException {
    StringRedisTemplate template = new StringRedisTemplate();
    template.setConnectionFactory(redisConnectionFactory);
    return template;
  }
}
```

这个自动配置类使用了以下注解：

+ @Configuration：标记为配置类，可以被spring容器扫描到

+ @ConditionalOnClass：指定的类存在时，该配置类才生效。这样可以在让用户自定义的配置存在时，默认配置不生效

+ @ConditionalOnMissingBean：spring容器中没有这个bean是，才会生成这个bean。作用和ConditionalOnClass类似

+ @EnableConfigurationProperties({RedisProperties.class})：
  
  如果一个配置类只配置`@ConfigurationProperties`注解，而没有使用`@Component`，那么在IOC容器中是获取不到这个配置文件的。通过 `@EnableConfigurationProperties` 可以把第三方 `@ConfigurationProperties `的类进行ioc注入， 加入到 IOC 容器中，后续可以获取其中配置的值。

例如 RedisProperties 配置文件的源码

```java
@ConfigurationProperties(prefix = "spring.redis")
public class RedisProperties {

    /**
     * Database index used by the connection factory.
     */
    private int database = 0;
```

`@ConfigurationProperties(prefix = "spring.redis" )` 声明了前缀，可以在配置文件中进行自定义配置，替换默认的配置

除此之外，还有一些常用的注解

+ @ConditionalOnProperty：主配置文件中存在指定的属性才生效。例如
  
  `@ConditionalOnProperty( prefix = "spring.redis.enabled", havingValue ="true"}`
  
  只有在配置文件配置`spring.redis.enabled=ture` 之后，该自动配置类才会生效

仿照自动配置类，自己的hbase api类编写如下

HbaseClient 文件

```java
@Configuration
@ConditionalOnProperty(value = "hbase.enabled", havingValue = "true")
@EnableConfigurationProperties(HbaseProperties.class)
public class HbaseClient implements InitializingBean {
    private final HbaseProperties hbaseProperties;

    private Connection connection;

    public HbaseClient(HbaseProperties hbaseProperties) {
        this.hbaseProperties = hbaseProperties;
    }

    @Override
    public void afterPropertiesSet() {
        org.apache.hadoop.conf.Configuration configuration = HBaseConfiguration.create();
        configuration.set("hbase.zookeeper.property.clientPort", hbaseProperties.getClientPort());
        configuration.set("hbase.zookeeper.quorum", hbaseProperties.getAddress());
        try {
            this.connection = ConnectionFactory.createConnection(configuration);
        } catch (IOException e) {
            throw new RuntimeException("hbase Connection 启动失败");
        }
    }
  // 省略其他方法
}
```

对应的配置文件

```java
@ConfigurationProperties(prefix = "hbase")
public class HbaseProperties {
    private String clientPort = "2181";
    private String address = "localhost";

    // 省略get set方法
}
```
