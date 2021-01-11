# 1、SpringMVC自动配置概览

Spring Boot provides auto-configuration for Spring MVC that **works well with most applications.(大多场景我们都无需自定义配置)**

The auto-configuration adds the following features on top of Spring’s defaults:

- Inclusion of `ContentNegotiatingViewResolver` and `BeanNameViewResolver` beans.

    - 内容协商视图解析器和BeanName视图解析器
- Support for serving static resources, including support for WebJars (covered [later in this document](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-spring-mvc-static-content))).
- 静态资源（包括webjars）
- Automatic registration of `Converter`, `GenericConverter`, and `Formatter` beans.
  - 自动注册 `Converter，GenericConverter，Formatter `
- Support for `HttpMessageConverters` (covered [later in this document](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-spring-mvc-message-converters)).
  - 支持 `HttpMessageConverters` （后来我们配合内容协商理解原理）
- Automatic registration of `MessageCodesResolver` (covered [later in this document](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-spring-message-codes)).
  - 自动注册 `MessageCodesResolver` （国际化用）
- Static `index.html` support.
  - 静态index.html 页支持
- Custom `Favicon` support (covered [later in this document](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-spring-mvc-favicon)).
  - 自定义 `Favicon`  
- Automatic use of a `ConfigurableWebBindingInitializer` bean (covered [later in this document](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-spring-mvc-web-binding-initializer)).
  - 自动使用 `ConfigurableWebBindingInitializer` ，（DataBinder负责将请求数据绑定到JavaBean上）

## 扩展方

> If you want to keep those Spring Boot MVC customizations and make more [MVC customizations](https://docs.spring.io/spring/docs/5.2.9.RELEASE/spring-framework-reference/web.html#mvc) (interceptors, formatters, view controllers, and other features), you can add your own `@Configuration` class of type `WebMvcConfigurer` but **without** `@EnableWebMvc`.
>
> **不用@EnableWebMvc注解。使用** **`@Configuration`** **+** **`WebMvcConfigurer`** **自定义规则**

例如：

+ @Bean方式

```java
@Configuration(proxyBeanMethods = false)
public class WebConfig {
  	//注入 WebMvcConfigurer对象
    @Bean
    public WebMvcConfigurer webMvcConfigurer(){
        return new WebMvcConfigurer() {

            @Override
            public void configurePathMatch(PathMatchConfigurer configurer) {
                UrlPathHelper urlPathHelper = new UrlPathHelper();
                // 不移除；后面的内容。矩阵变量功能就可以生效
                urlPathHelper.setRemoveSemicolonContent(false);
                configurer.setUrlPathHelper(urlPathHelper);
            }
        }
    }
}
```

+ 直接实现 WebMvcConfigurer 接口

```java
@Configuration(proxyBeanMethods = false)
public class WebConfig implements WebMvcConfigurer {
      @Override
   public void configurePathMatch(PathMatchConfigurer configurer) {

        UrlPathHelper urlPathHelper = new UrlPathHelper();
        // 不移除；后面的内容。矩阵变量功能就可以生效
        urlPathHelper.setRemoveSemicolonContent(false);
        configurer.setUrlPathHelper(urlPathHelper);
   }
}
```



> If you want to provide custom instances of `RequestMappingHandlerMapping`, `RequestMappingHandlerAdapter`, or `ExceptionHandlerExceptionResolver`, and still keep the Spring Boot MVC customizations, you can declare a bean of type `WebMvcRegistrations` and use it to provide custom instances of those components.
>
> **声明** **`WebMvcRegistrations`** **改变默认底层组件** 

```java
@Bean
public WebMvcRegistrations webMvcRegistrations(){
		return new WebMvcRegistrations(){
      @Override
      public RequestMappingHandlerMapping getRequestMappingHandlerMapping() {
        return null;
      }
    };
}
```

> If you want to take complete control of Spring MVC, you can add your own `@Configuration` annotated with `@EnableWebMvc`, or alternatively add your own `@Configuration`-annotated `DelegatingWebMvcConfiguration` as described in the Javadoc of `@EnableWebMvc`.
>
> **使用** **`@EnableWebMvc+@Configuration+DelegatingWebMvcConfiguration 全面接管SpringMVC`**

**@EnableWebMvc + WebMvcConfigurer +@Bean**  可以全面接管SpringMVC，所有规则全部自己重新配置； 实现定制和扩展功能

原理

+ WebMvcAutoConfiguration  默认的SpringMVC的自动配置功能类。静态资源、欢迎页.....

+ 一旦使用 @EnableWebMvc 、。会  @Import(DelegatingWebMvcConfiguration.class)
  **DelegatingWebMvcConfiguration** 的 作用，只保证SpringMVC最基本的使用

  + 把所有系统中的 WebMvcConfigurer 拿过来。所有功能的定制都是这些 WebMvcConfigurer  合起来一起生效
  + 自动配置了一些非常底层的组件。**RequestMappingHandlerMapping**、这些组件依赖的组件都是从容器中获取
  + 声明为  **public class** DelegatingWebMvcConfiguration **extends** **WebMvcConfigurationSupport**

+ **WebMvcAutoConfiguration** 里面的配置要能生效 必须  

  `@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)` ，而DelegatingWebMvcConfiguration 则继承了WebMvcConfigurationSupport，**@EnableWebMvc  导致了 WebMvcAutoConfiguration  没有生效**。



# 2、简单功能分析

## 2.1、静态资源访问

### 1、静态资源目录

只要静态资源放在类路径下： called `/static` (or `/public` or `/resources` or `/META-INF/resources`

访问 ： 当前项目根路径/ + 静态资源名 

原理： 静态映射/**。

请求进来，先去找Controller看能不能处理。不能处理的所有请求又都交给静态资源处理器。静态资源也找不到则响应404页面

改变默认的静态资源路径

```yaml
spring:
  mvc:
  	# 自定义 静态资源目录
    static-path-pattern: /res/**

  resources:
  	# 自定义 静态资源访问前缀
    static-locations: [classpath:/haha/]
```

### 2、静态资源访问前缀

默认无前缀

```
spring:
  mvc:
    static-path-pattern: /res/**
```

当前项目 + static-path-pattern + 静态资源名 = 静态资源文件夹下找

###  3 静态资源 自动配置原理

- SpringBoot启动默认加载  xxxAutoConfiguration 类（自动配置类）
- SpringMVC功能的自动配置类 WebMvcAutoConfiguration



# 3、请求参数处理

## 0、请求映射

### 1、rest使用与原理

- @xxxMapping；
- Rest风格支持（*使用**HTTP**请求方式动词来表示对资源的操作*）

  核心Filter: HiddenHttpMethodFilter  只作用于表单这种只有 post、put 的场景

  - 用法： 表单method=post，隐藏域 _method=put 指明真实请求动作

  - SpringBoot中手动开启`spring.mvc.hiddenmethod.filter.enabled=true   #开启页面表单的Rest功能`
    
  - Rest原理（表单提交要使用REST的时候）

    表单提交会带上**_method=PUT** 请求过来被**HiddenHttpMethodFilter**拦截，判断请求是否正常，并且是POST

    + 获取到**_method**的值。
    + 原生request（post），包装模式requesWrapper重写了getMethod方法，返回的是传入的值。
    + 过滤器链放行的时候用wrapper。以后的方法调用getMethod是调用requesWrapper的
    + 兼容以下请求；**PUT**.**DELETE**.**PATCH**

### 2、请求映射原理

![image.png](https://img-1252710297.cos.ap-nanjing.myqcloud.com/img/20210103100002.png)

SpringMVC功能分析都从 **org.springframework.web.servlet.DispatcherServlet**-》doDispatch（）

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
// 省略
// 找到当前请求使用哪个Handler（Controller的方法）处理
mappedHandler = getHandler(processedRequest);
```
getHandler() 方法中，使用了HandlerMapping：处理器映射Map 处理请求的映射

![image.png](https://img-1252710297.cos.ap-nanjing.myqcloud.com/img/20210103100008.png)

+ WelcomePageHandlerMapping ： SpringBoot自动配置欢迎页 访问 /能访问到index.html；

+  **RequestMappingHandlerMapping** ：保存了所有@RequestMapping 和handler的映射规则。

![image.png](https://img-1252710297.cos.ap-nanjing.myqcloud.com/img/20210103100013.png)

​		SpringBoot自动配置了默认 的 RequestMappingHandlerMapping；请求进来，挨个尝试所有的HandlerMapping看是否有请求信息。如果有就找到这个请求对应的handle，如果没有就是下一个 HandlerMapping

- 我们需要一些自定义的映射处理，我们也可以自己给容器中放**HandlerMapping**。

  自定义 **HandlerMapping**

```java
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
  if (this.handlerMappings != null) {
    for (HandlerMapping mapping : this.handlerMappings) {
      HandlerExecutionChain handler = mapping.getHandler(request);
      if (handler != null) {
        return handler;
      }
    }
  }
  return null;
}
```

## 1、普通参数与基本注解

### 1.1、注解：

+ @PathVariable：获取请求路径中的值
+ @RequestHeader：请求头
+ @RequestParam：请求参数

以上可以直接用 Map<sting,string> 类型变量进行承接  ： `@PathVariable Map<String,String> pv,`

+ @CookieValue：cookie值

  `@CookieValue("_ga") String _ga,@CookieValue("_ga") Cookie cookie`

+ @RequestBody：直接返回数据，而不是视图名： 标注方法上

+ @ModelAttribute

+ @MatrixVariable：矩阵参数 

  + 用于cookie禁用场景，对url进行重新，`;`分割，与常规请求区分开
+ 必须与 PathVariable 进行绑定，是 PathVariable 的一部分，不能单独存在
  
  ```java
  // url: /boss/1;age=20/2;age=10
  @GetMapping("/boss/{bossId}/{empId}")
  // 此时 bossId 与  empId 都有同样的 矩阵参数 age
  public Map carsSell(@MatrixVariable(value = "age",pathVar = "bossId") Integer bossAge,
                      @MatrixVariable(value = "age",pathVar = "empId") Integer empAge){
  ```
```
  
  + springboot默认禁用，需要手动开启：UrlPathHelper进行处理，重新改对象的 removeSemicolonContent 属性
  
  ```java
  @Configuration(proxyBeanMethods = false)
  public class WebConfig implements WebMvcConfigurer {
        @Override
     public void configurePathMatch(PathMatchConfigurer configurer) {
  
          UrlPathHelper urlPathHelper = new UrlPathHelper();
          // 不移除；后面的内容。矩阵变量功能就可以生效
          urlPathHelper.setRemoveSemicolonContent(false);
          configurer.setUrlPathHelper(urlPathHelper);
     }
  }
```

###  1.2、Servlet API：

WebRequest、ServletRequest、MultipartRequest、 HttpSession、javax.servlet.http.PushBuilder、Principal、InputStream、Reader、HttpMethod、Locale、TimeZone、ZoneId

对应的ArgumentResolver为  **ServletRequestMethodArgumentResolver**

### 1.3、复杂参数：

+ **Map**、**Model**、**HttpServletRequest**: 里面的数据会被放在request的请求域  相当于 request.setAttribute
+ **RedirectAttributes**:重定向携带数据
+ **ServletResponse**、**HttpServletResponse**：为 response
+ SessionStatus、UriComponentsBuilder、ServletUriComponentsBuilder
+ Errors/BindingResult

**Map、Model类型的参数**，会返回 mavContainer.getModel（）；---> BindingAwareModelMap 是Model 也是Map

### 1.4、自定义对象参数：

可以自动类型转换与格式化，可以级联封装。

由 **ServletModelAttributeMethodProcessor** 处理

## 3、参数处理原理

都发生在 `DispatcherServlet -- doDispatch()`中

```java
// Actually invoke the handler.
//DispatcherServlet -- doDispatch

// 获取HandlerAdapter 处理各种入参
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

//执行目标方法
mav = invokeHandlerMethod(request, response, handlerMethod); 


//ServletInvocableHandlerMethod
Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
//获取方法的参数值
Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
```

总体流程：

- HandlerMapping中找到能处理请求的Handler（Controller.method()）
- 为当前Handler 找一个适配器 HandlerAdapter： **RequestMappingHandlerAdapter** 处理mvc的各种出入参
- 适配器执行目标方法并确定方法参数的每一个值

### 1、获取HandlerAdapter

`mv = ha.handle(processedRequest, response, mappedHandler.getHandler());`

便利所有的 HandlerAdapter，判断是否支持当前的Handler（Controller.method()）

![image.png](https://img-1252710297.cos.ap-nanjing.myqcloud.com/img/20210103164946.png)

+ 0 - 支持方法上标注@RequestMapping 

+ 1 - 支持函数式编程的

一般情况下 第一个**RequestMappingHandlerAdapter** 就支持，故获取

### 2、执行目标方法

 DispatcherServlet -- doDispatch()中

`mv = ha.handle(processedRequest, response, mappedHandler.getHandler());`
 执行RequestMappingHandlerAdapter的方法：

之后进入 RequestMappingHandlerAdapter#handleInternal() 中，执行

`mav = invokeHandlerMethod(request, response, handlerMethod);`

源码为

```java
@Nullable
	protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
			
    // 新建 invocableMethod
    ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
    // 参数解析器 共享RequestMappingHandlerAdapter的
			if (this.argumentResolvers != null) {
				invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
			}
    // 返回值处理器 共享RequestMappingHandlerAdapter的
			if (this.returnValueHandlers != null) {
				invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
			}
    // 真正执行方法
   invocableMethod.invokeAndHandle(webRequest, mavContainer);
  }

```

可以看到有 参数解析器 与 返回值处理器

### 3、参数解析器 - HandlerMethodArgumentResolver

确定将要执行的目标方法的每一个参数的值是什么;

SpringMVC目标方法能写多少种参数类型。取决于参数解析器。

![image.png](https://img-1252710297.cos.ap-nanjing.myqcloud.com/img/20210103165024.png)

每个Resolver都实现了这个接口 HandlerMethodArgumentResolver，有如下2个方法

![image.png](https://img-1252710297.cos.ap-nanjing.myqcloud.com/img/20210103165121.png)

- 当前解析器是否支持解析传入的参数
- 进行参数解析： resolveArgument



### 4、返回值处理器

![image.png](https://img-1252710297.cos.ap-nanjing.myqcloud.com/img/20210103165212.png)

controller层方法中可以返回的对象类型

### 5、确定目标方法每一个参数的值

之后 invocableMethod.invokeAndHandle()方法中，执行的是 ServletInvocableHandlerMethod#invokeAndHandle()，源码为

```java
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {
		// 真正调用方法
		Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
		setResponseStatus(webRequest);
}

public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {
  // 获取spring mvc 自动解析的出入参
		Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
  	.....
}
```

参数的解析 主要实现在 InvocableHandlerMethod#getMethodArgumentValues() 方法中

```java
protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
            Object... providedArgs) throws Exception {

        MethodParameter[] parameters = getMethodParameters();
        if (ObjectUtils.isEmpty(parameters)) {
            return EMPTY_ARGS;
        }

        Object[] args = new Object[parameters.length];
        for (int i = 0; i < parameters.length; i++) {
            MethodParameter parameter = parameters[i];
            parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
            args[i] = findProvidedArgument(parameter, providedArgs);
            if (args[i] != null) {
                continue;
            }
            if (!this.resolvers.supportsParameter(parameter)) {
                throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
            }
            try {
           // 这里调用参数解析器的具体解析方法
                args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
            }
            catch (Exception ex) {
                // Leave stack trace for later, exception may actually be resolved and handled...
                if (logger.isDebugEnabled()) {
                    String exMsg = ex.getMessage();
                    if (exMsg != null && !exMsg.contains(parameter.getExecutable().toGenericString())) {
                        logger.debug(formatArgumentError(parameter, exMsg));
                    }
                }
                throw ex;
            }
        }
        return args;
    }
```

#### 5.1、挨个判断所有参数解析器那个支持解析这个参数

InvocableHandlerMethod 类中的 有HandlerMethodArgumentResolverComposite对象，是 RequestMappingHandlerAdapter 传入的

```java
public class HandlerMethodArgumentResolverComposite implements HandlerMethodArgumentResolver {

	@Deprecated
	protected final Log logger = LogFactory.getLog(getClass());

	private final List<HandlerMethodArgumentResolver> argumentResolvers = new LinkedList<>();
	
  // 保存 参数解析器的缓存
	private final Map<MethodParameter, HandlerMethodArgumentResolver> argumentResolverCache =
			new ConcurrentHashMap<>(256);
```

使用了缓存，第一次会for循环获取相应的参数解析器，较慢，之后会存入缓存，多次请求共用同一个缓存

```java
    @Nullable
    private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
      
      // 使用了缓存
        HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);
        if (result == null) {
          
          // 第一次获取，循环查找 放入缓存中
            for (HandlerMethodArgumentResolver resolver : this.argumentResolvers) {
                if (resolver.supportsParameter(parameter)) {
                    result = resolver;
                    this.argumentResolverCache.put(parameter, result);
                    break;
                }
            }
        }
        return result;
    }
```

#### 5.2、解析这个参数的值

调用各自 HandlerMethodArgumentResolver 的 resolveArgument 方法即可

#### 5.3、自定义类型参数 封装POJO

**ServletModelAttributeMethodProcessor** 处理满足以下条件的参数：

+ 包含@ModelAttribute
+ 不为简单类型，即为 POJO对象

解析参数源码如下：

```java
@Override
@Nullable
public final Object resolveArgument(...) throws Exception {
  	//省略
  
  // attribute 为新建的 POJO对象 所有属性为null
  // binder 为 web数据绑定器，将请求参数的值绑定到指定的JavaBean里面
  WebDataBinder binder = binderFactory.createBinder(webRequest, attribute, name);
  
  // WebDataBinder 利用它里面的 Converters 将请求数据转成指定的数据类型,封装到attribute中
  // 通过 <传入类型，传出类型>寻找对应的 Converter
  bindRequestParameters(binder, webRequest);
}

return attribute;
}
```

![image.png](https://img-1252710297.cos.ap-nanjing.myqcloud.com/img/20210103170339.png)

**自定义 Converter**

```java
    //1、WebMvcConfigurer定制化SpringMVC的功能
    @Bean
    public WebMvcConfigurer webMvcConfigurer(){
        return new WebMvcConfigurer() {

            @Override
            public void addFormatters(FormatterRegistry registry) {
              
              // public interface** Converter<S, T>
                registry.addConverter(new Converter<String, Pet>() {

                    @Override
                    public Pet convert(String source) {
                        if(!StringUtils.isEmpty(source)){
                            Pet pet = new Pet();
                            String[] split = source.split(",");
                            pet.setName(split[0]);
                            pet.setAge(Integer.parseInt(split[1]));
                            return pet;
                        }
                        return null;
                    }
                });
            }
        };
    }
```

### 6、目标方法执行完成

将所有的数据都放在 **ModelAndViewContainer**；包含要去的页面地址View。还包含Model数据。

### 7、处理派发结果

**processDispatchResult**(processedRequest, response, mappedHandler, mv, dispatchException);

# 4、数据响应与内容协商

## 1、响应JSON

### jackson.jar+@ResponseBody

#### 1、返回值解析器

**RequestMappingHandlerAdapter** 中默认存在的返回值解析器，实现了  **HandlerMethodReturnValueHandler**接口

![image.png](https://img-1252710297.cos.ap-nanjing.myqcloud.com/img/20210103220120.png)

支持的返回值

```
同步
    ModelAndView
    Model
    View
    ResponseEntity 
    ResponseBodyEmitter
    StreamingResponseBody
    HttpEntity
    HttpHeaders
    有 @ModelAttribute 且为对象类型的
    @ResponseBody 注解 ---> RequestResponseBodyMethodProcessor 中 HTTPMessageConverter处理
异步
    Callable
    DeferredResult
    ListenableFuture
    CompletionStage
    WebAsyncTask

```

在传入参数处理完成、目标方法执行完成之后，执行返回值解析器的逻辑

org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod#invokeAndHandle() 中的

```java
this.returnValueHandlers.handleReturnValue(returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
```

#### 2、返回值解析器原理--Json ![image.png](https://img-1252710297.cos.ap-nanjing.myqcloud.com/img/20210103220445.png)

- 返回值处理器判断是否支持这种类型返回值    supportsReturnType()

- 返回值处理器调用 handleReturnValue() 进行处理

- RequestResponseBodyMethodProcessor 可以处理返回值标了@ResponseBody 注解的。

  + 根据内容协商(浏览器想要接受什么、服务器可以产生什么)，选择返回的内容类型数据

    SpringMVC会挨个遍历所有容器底层的 **HttpMessageConverter** ，看看哪个支持解析返回类型；
  
    之后，将改converter支持的 媒体类型(MediaType) 放在List中，汇总，
  
    根据浏览器的接受的媒体类型，选择满足要求的那一个
  
  + 再次便利 **HttpMessageConverter** ，获取支持处理该媒体类型、支持处理返回值 的converter。调用它进行转化
  
    此时 使用 MappingJackson2HttpMessageConverter 处理为json并返回

 

### HTTPMessageConverter

**HttpMessageConverter**接口 ： 看是否支持将 此 Class类型的对象，转为MediaType类型的数据。

![image.png](https://img-1252710297.cos.ap-nanjing.myqcloud.com/img/20210103222856.png)

+ canRead：是否能读入对象 @RequestBody
+ canWrite：是否能写出对象 @ResponseBody
+ getSupportedMediaTypes：支持的媒体类型

例子：Person对象转为JSON。或者 JSON转为Person

默认的MessageConverter

![image.png](https://img-1252710297.cos.ap-nanjing.myqcloud.com/img/20210103222907.png)

最终 MappingJackson2HttpMessageConverter  把对象转为JSON（利用底层的jackson的objectMapper转换的）

路径：` @ResponseBody -> RequestResponseBodyMethodProcessor -> messageConverter`

### 自定义MessageConverter

```java
public class GuiguMessageConverter implements HttpMessageConverter<Person> {

  	// @RequestBody 作用于传入参数
    @Override
    public boolean canRead(Class<?> clazz, MediaType mediaType) {
        return false;
    }

  	// @ResponseBody 作用于写出参数
    @Override
    public boolean canWrite(Class<?> clazz, MediaType mediaType) {
        return clazz.isAssignableFrom(Person.class);
    }

    /**
     * 服务器要统计所有MessageConverter都能写出哪些内容类型
     *
     * application/x-guigu
     * @return
     */
    @Override
    public List<MediaType> getSupportedMediaTypes() {
        return MediaType.parseMediaTypes("application/x-guigu");
    }

    @Override
    public Person read(Class<? extends Person> clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException {
        return null;
    }

    @Override
    public void write(Person person, MediaType contentType, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException {
        //自定义协议数据的写出
        String data = person.getUserName()+";"+person.getAge()+";"+person.getBirth();


        //写出去
        OutputStream body = outputMessage.getBody();
        body.write(data.getBytes());
    }
}
```

添加到spring mvc

```java
@Bean
    public WebMvcConfigurer webMvcConfigurer(){
        return new WebMvcConfigurer() {

            /**
             * 自定义内容协商策略   formart 方式支持自定义格式
             * @param configurer
             */
            @Override
            public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
                //Map<String, MediaType> mediaTypes
                Map<String, MediaType> mediaTypes = new HashMap<>();
                mediaTypes.put("json",MediaType.APPLICATION_JSON);
                mediaTypes.put("xml",MediaType.APPLICATION_XML);
                mediaTypes.put("gg",MediaType.parseMediaType("application/x-guigu"));
                //指定支持解析哪些参数对应的哪些媒体类型
                ParameterContentNegotiationStrategy parameterStrategy = new ParameterContentNegotiationStrategy(mediaTypes);
//                parameterStrategy.setParameterName("ff");

                HeaderContentNegotiationStrategy headeStrategy = new HeaderContentNegotiationStrategy();

                configurer.strategies(Arrays.asList(parameterStrategy,headeStrategy));
            }

            @Override
            public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
                converters.add(new GuiguMessageConverter());
            }

```

# 5、视图解析与模板引擎

## 视图解析原理流程

**原理**

1. 目标方法处理的过程中，所有数据都会被放在 **ModelAndViewContainer** 。包括数据和视图地址

2. 方法的参数是一个自定义类型对象（从请求参数中确定的），把他重新放在**ModelAndViewContainer** 

3. 任何目标方法执行完成以后都会返回 ModelAndView（数据和视图地址）。

4. processDispatchResult  处理派发结果（页面改如何响应）

   1. **render**(**mv**, request, response); 进行页面渲染逻辑

   2. 根据方法的String返回值得到 **View** 对象【定义了页面的渲染逻辑】

      1. 所有的视图解析器尝试是否能根据当前返回值得到**View**对象

         ![image.png](https://img-1252710297.cos.ap-nanjing.myqcloud.com/img/20210109164711.png)

      2. 得到了  **redirect:/main.html** --> Thymeleaf new **RedirectView**()

      3. ContentNegotiationViewResolver 里面包含了所有的视图解析器，内部还是利用所有视图解析器得到视图对象。

      4. view.render(mv.getModelInternal(), request, response);  视图对象调用自定义的render进行页面渲染工作

         **RedirectView 如何渲染【重定向到一个页面】**
      
         + 获取目标url地址
         + response.sendRedirect(encodedURL);

**视图解析总结**

+ **返回值以 forward: 开始** ： new InternalResourceView(forwardUrl); -->  转发*request.getRequestDispatcher(path).forward(request, response);

+ **返回值以** **redirect:** 开始：new RedirectView() --》 render就是重定向 
+  **返回值是普通字符串** ： new ThymeleafView（）--->

# 拦截器

## HandlerInterceptor 接口

```java
/**
 * 登录检查
 * 1、配置好拦截器要拦截哪些请求
 * 2、把这些配置放在容器中
 */
@Slf4j
public class LoginInterceptor implements HandlerInterceptor {

    /**
     * 目标方法执行之前
     */
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {

        String requestURI = request.getRequestURI();
        log.info("preHandle拦截的请求路径是{}",requestURI);

        //登录检查逻辑
        HttpSession session = request.getSession();

        Object loginUser = session.getAttribute("loginUser");

        if(loginUser != null){
            //放行
            return true;
        }

        //拦截住。未登录。跳转到登录页
        request.setAttribute("msg","请先登录");
//        re.sendRedirect("/");
        request.getRequestDispatcher("/").forward(request,response);
        return false;
    }

    /**
     * 目标方法执行完成以后
     */
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        log.info("postHandle执行{}",modelAndView);
    }

    /**
     * 页面渲染以后
     */
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        log.info("afterCompletion执行异常{}",ex);
    }
}
```

## 配置

```java
/**
 * 1、编写一个拦截器实现HandlerInterceptor接口
 * 2、拦截器注册到容器中（实现WebMvcConfigurer的addInterceptors）
 * 3、指定拦截规则【如果是拦截所有，静态资源也会被拦截】
 */
@Configuration
public class AdminWebConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoginInterceptor())
                .addPathPatterns("/**")  //所有请求都被拦截包括静态资源
                .excludePathPatterns("/","/login","/css/**","/fonts/**","/images/**","/js/**"); //放行的请求
    }
}
```

## 原理

1. 根据当前请求，找到**HandlerExecutionChain【**可以处理请求的handler以及handler的所有 拦截器，在找到 Handler的时候一并找到】

   ![image.png](https://img-1252710297.cos.ap-nanjing.myqcloud.com/img/20210109171803.png)

2. 先来**顺序执行** 所有拦截器的 preHandle方法

   1. 如果当前拦截器prehandler返回为true。则执行下一个拦截器的preHandle
   2. 如果当前拦截器返回为false。直接   倒序执行所有已经执行了的拦截器的  afterCompletion；

3. **如果任何一个拦截器返回false。直接跳出不执行目标方法**

4. 所有拦截器都返回True。执行目标方法

5. **倒序执行所有拦截器的postHandle方法。**

6. **前面的步骤有任何异常都会直接倒序触发** afterCompletion

7. 页面成功渲染完成以后，也会倒序触发 afterCompletion

![image.png](https://img-1252710297.cos.ap-nanjing.myqcloud.com/img/20210109171431.png)

# 文件上传

代码

```java
@PostMapping("/upload")
public String upload(@RequestParam("email") String email,
                     @RequestParam("username") String username,
                     @RequestPart("headerImg") MultipartFile headerImg,
                     @RequestPart("photos") MultipartFile[] photos) throws IOException {

    String originalFilename = headerImg.getOriginalFilename();
    headerImg.transferTo(new File("H:\\cache\\"+originalFilename));
  
  return "main";
}
```

# 异常处理

## 默认规则

- 默认情况下，Spring Boot提供`/error`处理所有错误的映射

  + 对于机器客户端，它将生成JSON响应，其中包含错误，HTTP状态和异常消息的详细信息。

  + 对于浏览器客户端，响应一个“ whitelabel”错误视图，以HTML格式呈现相同的数据

  + 要对其进行自定义，添加 `View` 解析为`error`

    要完全替换默认行为，可以实现 `ErrorController `并注册该类型的Bean定义，或添加`ErrorAttributes类型的组件`以使用现有机制但替换其内容。

- error/下的4xx，5xx页面会被自动解析；

## 定制错误处理逻辑

- 自定义错误页

- error/404.html  error/5xx.html: 有精确的错误状态码页面就匹配精确，没有就找 4xx.html；如果都没有就触发白页

- @ControllerAdvice+@ExceptionHandler处理全局异常

  ```java
  /**
   * 处理整个web controller的异常
   */
  @Slf4j
  @ControllerAdvice
  public class GlobalExceptionHandler {
  
      @ExceptionHandler({ArithmeticException.class,NullPointerException.class})  //处理异常
      public String handleArithException(Exception e){
  
          log.error("异常是：{}",e);
          return "login"; //视图地址
      }
  }
  ```

  底层是 **ExceptionHandlerExceptionResolver 支持的**

- @ResponseStatus+自定义异常 :

  ```java
  @ResponseStatus(value= HttpStatus.FORBIDDEN,reason = "用户数量太多")
  public class UserTooManyException extends RuntimeException {
  ```

  底层是 **ResponseStatusExceptionResolver** ，把responsestatus注解的信息底层调用response.sendError(statusCode, resolvedReason)；tomcat发送的/error

- Spring底层的异常，如 参数类型转换异常；**DefaultHandlerExceptionResolver 处理框架底层的异常。**

  response.sendError(HttpServletResponse.SC_BAD_REQUEST, ex.getMessage()); 


- 自定义实现 HandlerExceptionResolver 处理异常；

  可以作为默认的全局异常处理规则：需要定义 @Order(value= Ordered.HIGHEST_PRECEDENCE) ，先被发现，先处理异常

  ```java
  @Order(value= Ordered.HIGHEST_PRECEDENCE)  //优先级，数字越小优先级越高 不然扫描到的时间晚，被默认的异常解析器提前处理了
  @Component
  public class CustomerHandlerExceptionResolver implements HandlerExceptionResolver {
      @Override
      public ModelAndView resolveException(HttpServletRequest request,
                                           HttpServletResponse response,
                                           Object handler, Exception ex) {
  
          try {
              response.sendError(511,"我喜欢的错误");
          } catch (IOException e) {
              e.printStackTrace();
          }
          return new ModelAndView();
      }
  }
  ```

  ![image.png](https://img-1252710297.cos.ap-nanjing.myqcloud.com/img/20210109223610.png)

- **ErrorViewResolver**  实现自定义处理异常；

  + response.sendError :error请求就会转给controller

  + 你的异常没有任何人能处理。tomcat底层 response.sendError。error请求就会转给controller

  **basicErrorController 要去的页面地址是** **ErrorViewResolver**  ；

## 异常处理自动配置原理

通过**ErrorMvcAutoConfiguration  自动配置异常处理规则**

容器中的组件：

 + **errorAttributes** 类型：DefaultErrorAttributes

   public class DefaultErrorAttributes implements ErrorAttributes, HandlerExceptionResolver

 + **basicErrorController**：  类型  **BasicErrorController**

   + json+白页 适配响应处理默认 /error 路径的请求；

   + 页面响应 new ModelAndView("error", model)；容器中有组件 View->id是error；（响应默认错误页）

   + 容器中放组件 BeanNameViewResolver（视图解析器）；按照返回的视图名作为组件的id去容器中找View对象。

+ **conventionErrorViewResolver： 类型：**DefaultErrorViewResolver 

  + 解析框架产生的异常
  
  + 会以HTTP的状态码 作为视图页地址（viewName），找到真正的页面 error/404、5xx.html

## 异常处理步骤流程

+ 执行目标方法，目标方法运行期间有任何异常都会被catch、而且标志当前请求结束；并且用 **dispatchException** 

+ 进入视图解析流程（页面渲染？） 

  processDispatchResult(processedRequest, response, mappedHandler, **mv**, **dispatchException**);

+ **mv** = **processHandlerException**；处理handler发生的异常，处理完成返回ModelAndView；
  + 遍历所有的 **handlerExceptionResolvers，看谁能处理当前异常【HandlerExceptionResolver处理器异常解析器】**

    ![image.png](https://img-1252710297.cos.ap-nanjing.myqcloud.com/img/20210109224213.png)

    系统默认的  异常解析器

    ![image.png](https://img-1252710297.cos.ap-nanjing.myqcloud.com/img/20210109224348.png)

  + - DefaultErrorAttributes先来处理异常：把异常信息保存到rrequest域，并且返回null；

    - HandlerExceptionResolverComposite：聚合了3个异常解析器

      - ExceptionHandlerExceptionResolver：`@ExceptionHandler` 定义的异常解析器
      - ResponseStatusExceptionResolver:  处理 @ResponseStatus 异常 
      - DefaultErrorViewResolver :自动配置的， 处理框架本身的异常，发送 response.sendError(),转换为各种异常页面返回

    - 如果没有任何人能处理异常，异常会被抛出，最终底层就会重新发送 /error 请求被底层的BasicErrorController处理

      + 解析错误视图；遍历所有的  ErrorViewResolver  看谁能解析。

      - 默认的 DefaultErrorViewResolver ,作用是把响应状态码作为错误页的地址，error/500.html 
      - 模板引擎最终响应这个页面 error/500.html 

# Web原生组件注入（Servlet、Filter、Listener）

### 注解方式

  若要生效 需要 `@ServletComponentScan(basePackages="com.atguigu.admin")` 指明 扫描的原生组件的地址

  + @WebServlet：

    ```java
    @WebServlet(urlPatterns = "/my")
    public class MyServlet extends HttpServlet {
    
        @Override
        protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
            resp.getWriter().write("66666");
        }
    }
    ```

  + @WebFilter

    ```java
    @Slf4j
    @WebFilter(urlPatterns={"/css/*","/images/*"}) 
    public class MyFilter implements Filter {
        @Override
        public void init(FilterConfig filterConfig) throws ServletException {
            log.info("MyFilter初始化完成");
        }
    
        @Override
        public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
            log.info("MyFilter工作");
            chain.doFilter(request,response);
        }
    
        @Override
        public void destroy() {
            log.info("MyFilter销毁");
        }
    }
    ```

  + @WebListener

    ```java
    @Slf4j
    @WebListener
    public class MySwervletContextListener implements ServletContextListener {
        @Override
        public void contextInitialized(ServletContextEvent sce) {
            log.info("MySwervletContextListener监听到项目初始化完成");
        }
        @Override
        public void contextDestroyed(ServletContextEvent sce) {
            log.info("MySwervletContextListener监听到项目销毁");
        }
    }
    ```

    

  ###  使用RegistrationBean

  + ServletRegistrationBean
  + FilterRegistrationBean
  + ServletListenerRegistrationBean

  ```java
  @Configuration
  public class MyRegistConfig {
  
      @Bean
      public ServletRegistrationBean myServlet(){
          MyServlet myServlet = new MyServlet();
          return new ServletRegistrationBean(myServlet,"/my","/my02");
      }
  
      @Bean
      public FilterRegistrationBean myFilter(){
          MyFilter myFilter = new MyFilter();
  //        return new FilterRegistrationBean(myFilter,myServlet());  只拦截 myServlet 路径
          FilterRegistrationBean filterRegistrationBean = new FilterRegistrationBean(myFilter);
          filterRegistrationBean.setUrlPatterns(Arrays.asList("/my","/css/*"));
          return filterRegistrationBean;
      }
  
      @Bean
      public ServletListenerRegistrationBean myListener(){
          MySwervletContextListener mySwervletContextListener = new MySwervletContextListener();
          return new ServletListenerRegistrationBean(mySwervletContextListener);
      }
  }
  ```

  扩展：DispatchServlet 如何注册进来

- 容器中自动配置了  DispatcherServlet  属性绑定到 WebMvcProperties；对应的配置文件配置项是 **spring.mvc。**
- **通过** **ServletRegistrationBean**<DispatcherServlet> 把 DispatcherServlet  配置进来。
- 默认映射的是 / 路径。

