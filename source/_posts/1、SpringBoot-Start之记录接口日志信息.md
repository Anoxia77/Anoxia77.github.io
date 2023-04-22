### 前言
> 不积跬步无以至千里，不积小流，无以成江海

在公司一般来说，都只会接触一些CRUD的业务，很多时候可能你想设计很多的代码结构，但是时间不允许。近期接到一个项目，领导说要扩展、要兼容、等等等。然后我就想着那我就花点时间把代码结构设计一下，然后给出排期，当领导看到我的排期时间的时候，他跟我说，`我们要小步快跑`，我觉得他在CPU我，明明上一次还再说要扩展要兼容的。
自己去完成一些springboot-start的功能实现， 我觉得是一个很锻炼人的方式，一是可以让自己多去思考自己的代码改怎么组织，再一个是springboot-start一般都是抽出一些公共的逻辑，所以这对我们理解项目的能力也有一定的要求。springboot 存在很多的start组件，他们可以很方便的帮我们注入一些我们需要的功能，不需要的时候，直接删除依赖就可以了，开发出来的组件 可以拔插式的接入项目。

### 目标
基于 AOP 实现系统监控，主要是通过aop切面功能来增强方法，实现监控。
### 设计
项目结构
```java
cn-anoxia-start-log
└── src
    ├── main
    │   └── java
    │       ├── cn.anoxia.log
    │       │   ├── annotation
    │       │   │   └── LogCheck.java
    │       │   ├── config
    │       │   │   └── LogAutoConfigure.java
    │       │   └── LogCheckJoinPoint.java
    │       └── resources
    │           └── META-INF 
    │               └── spring.factories
    └── test
        └── java
            └── cn.anoxia.log.test
                └── ApiTest.java

```
实现过程主要是通过AOP拦截注解，然后对方法进行处理

- LogCheck  自定义注解，主要作用就是添加到需要监控的方法上。
- LogAutoConfigure  配置类，对一些类做初始化操作。
- LogCheckJoinPoint  核心类，负责对拦截的方法做逻辑处理。
- spring.factories  spring-boot 自动注入的配置文件。

springboot 在启动的时候 读取`spring.factories`里面的内容，然后把配置类添加到spring容器中。
使用 springboot的自动注入的功能完成配置的加载。
```java
org.springframework.boot.autoconfigure.EnableAutoConfiguration=cn.anoxia.log.config.LogAutoConfigure
```
 自定义拦截注解`LogCheck`
```java
/**
 * @description: 方法耗时检测注解
 * @author：huangle
 * @date: 2022/7/22
 */
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE,ElementType.METHOD})
public @interface LogCheck {
    String key() default "";
    String desc() default "";
}
```
AOP 处理类, 定义切点为 注解，然后使用 环绕处理 对方法进行增强。
```java
/**
 * @description: aop拦截注解，进行方法增强
 * @author：huangle
 * @date: 2022/7/22
 */
@Aspect
@Component
public class LogCheckJoinPoint {

    private final Logger logger = LoggerFactory.getLogger(LogCheckJoinPoint.class);

    @Pointcut("@annotation(cn.anoxia.log.annotation.LogCheck)")
    public void aopPoint(){
        // 定义切点
    }

    @Around("aopPoint() && @annotation(logCheck)")
    public void doCheck(ProceedingJoinPoint joinPoint, LogCheck logCheck) throws Throwable {
        // 执行前增强
        logger.info("当前执行的类：{}",joinPoint.getClass());
        Method method = getMethod(joinPoint);
        Long start = System.currentTimeMillis();
        try {
            // 执行目标方法
            joinPoint.proceed();
        }finally {
            System.out.println("监控 - Begin By AOP");
            System.out.println("监控索引：" + logCheck.key());
            System.out.println("监控描述：" + logCheck.desc());
            System.out.println("方法名称：" + method.getName());
            System.out.println("方法耗时：" + (System.currentTimeMillis() - start) + "ms");
            System.out.println("监控 - End\r\n");
        }
    }

    private Method getMethod(JoinPoint jp) throws NoSuchMethodException {
        Signature sig = jp.getSignature();
        MethodSignature methodSignature = (MethodSignature) sig;
        return jp.getTarget().getClass().getMethod(methodSignature.getName(), methodSignature.getParameterTypes());
    }

}

```
配置类里面的内容, 对核心类进行初始化，并且添加到spring容器
```java
@Configuration
public class LogAutoConfigure implements EnvironmentAware {

    private final Logger logger = LoggerFactory.getLogger(LogAutoConfigure.class);

    private final Map<String,Object> logConfigMap = new HashMap<>();

    @Bean
    @ConditionalOnMissingBean
    public LogCheckJoinPoint point(){
        return new LogCheckJoinPoint();
    }
}
```
### 测试
创建一个项目，然后导入我们创建的`start`
```xml
<dependency>
            <groupId>cn.anoxia</groupId>
            <artifactId>anoxia-spring-start-log</artifactId>
            <version>1.0.0-SNAPSHOT</version>
        </dependency>
```
添加注解，拦截方法
```java
@LogCheck(key = "cn.anoxia.demo.controller.TestController",desc = "获取用户信息")
    @RequestMapping("/v1/info")
    public String testController(){
        return "hello";
    }
```
测试结果,可以获取到执行方法的一些信息，并且对方法进行增强。
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2869098/1658474804394-23d1aeff-e515-419b-991a-588aa9e27a1c.png#averageHue=%232d2c2c&clientId=udb09d950-4756-4&from=paste&height=204&id=ue97881b9&name=image.png&originHeight=408&originWidth=1958&originalType=binary&ratio=1&rotation=0&showTitle=false&size=65951&status=done&style=none&taskId=udf48678e-3630-4277-8253-cb0ead25322&title=&width=979)
### 总结
在我们开发组件的时候，主要是添加一个配置文件 `spring.factories`让spring-boot在启动的时候，把我们添加的类注入进去，然后直接就在项目中使用。

