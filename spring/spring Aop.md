导入依赖

```
 <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
            <version>1.5.19.RELEASE</version>
        </dependency>
```



开启配置注解

`@EnableAspectJAutoProxy`  只有添加这个注解才能使用自己写的 aop 切面



简单的配置

```java
@Aspect
@Component
public class MyAop {
    Logger log = LoggerFactory.getLogger(MyAop.class);

    @Pointcut("execution(* com.spring.aop.DoSting.*(..))")
    public void pointCut() {
    }

    @Around("pointCut()")
    public Object around(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        Object res = proceedingJoinPoint.proceed();
        log.info("cost {}", System.currentTimeMillis() - start);
        return res;
    }
}

```

+ `@Aspect` 与  `@Component` 必须同时存在 否则无法识别

 

