### 前言
后面开始实现 AOP 的内容，AOP 在日常开发的过程中，相对来说是用的比较多的，比如

- 用户权限检查
- 数据库分库分表
- 用户请求上下文 

等等，各种场景都可以使用AOP去做扩展，而且还不会影响到内部业务，相对来说，这是一种很好的解决方法，所以学习这个得到的回报还是比较高的。
下面是一些 关于 AOP 的一些基本信息，先简单了解一下AOP 都有哪些东西，都是干什么的。
### 术语

- `通知 (Advice)`: AOP 框架中的增强处理。通知描述了切面何时执行以及如何执行增强处理。通知的类型有：后置通知、返回通知、异常通知、环绕通知、前置通知。
- `连接点 (Joint Point)`：连接点满足切入点识别，是在应用执行过程中能够插入切面（Aspect）的一个点。广义上来讲，方法、异常处理块、字段这些程序调用过程中可以抽像成一个执行步骤（或者说执行点）的单元。可以是调用方法时、甚至修改一个字段时。
- `切入点 (Pointcut)`：切入点是特定连接点的描述定义，是指通知（Advice）所要织入（Weaving）的具体位置；Spring AOP 通过切入点来定位到哪些连接点。切点表达式语言就是切点用来定义连接点的语法。
- `切面 (Aspect)`：切面是通知和切入点的结合，通知规定了在什么时机干什么事，切入点规定了在什么位置。
- `引入 (Introduction)`：引入允许我们向现有的类添加新的方法或者属性。
- `织入 (weaving)`: 织入是把切面应用到目标对象并创建代理对象的过程。切点在指定的连接点(切点)被织入到目标对象中。在目标的生命周期中，织入的时机有多种：编译期、类加载期、运行期
### 通知类型
| 注解 | 描述 | 说明 |
| --- | --- | --- |
| @Before | 在目标方法执行之前调用 | 如果抛出异常，会影响业务方法 |
| @AfterReturning | 在目标方法返回后调用 | 无 |
| @AfterThrowing | 在目标方法异常后调用 | 无 |
| @After | 在目标方法返回或者异常后调用 | 无论业务方法是否正常都会执行 |
| @Around | 把目标方法封装起来（包装一层） | 封装后通过 proceed() 方法执行业务方法 |

### 切点表达式
| 切点指示器 | 描述 |
| --- | --- |
| arg() | 限制连接点匹配参数为指定类型的执行方法 |
| @arg() | 限制连接点匹配参数由指定注解标注的执行方法 |
| execution() | 用于匹配是连接点的执行方法 |
| this() | 限制连接点匹配 AOP 代理的 Bean 应用为指定类型的类 |
| target() | 限制连接点匹配目标对象为指定类型的类 |
| @target() | 限制连接点匹配特点的执行对象，这些对象对应的类要具备指定类型的注解 |
| within() | 限制连接点匹配特点的执行对象，这些对象对应的类要具备指定类型的注解 |
| @within() | 限制连接点匹配指定注解所标注的类型（当使用 Spring AOP 时，方法定义在由指定的注解所标注的类里） |
| @annotation() | 限制匹配带有指定注解连接点 |

#### 切点表单式的使用
```java
arg(param-pattern)
param-pattern：参数类型的全路径    

// 拦截参数类型是string的方法 搭配 其他的拦截表达式一起使用
@Pointcut("args(java.lang.String)")
public void intercept(){}


@arg(param-pattern)
param-pattern：注解参数类型的全路径

// 拦截参数被 注解 param-pattern 标识的方法，该表达式表示拦截 被 @RequestParam 修饰的方法
@Pointcut("@args(org.springframework.web.bind.annotation.RequestParam)")
public void intercept(){}


// 问号 ? 表示该项可以没有
execution(modifiers-pattern? ret-type-pattern declaring-type-pattern?name-pattern(param-pattern) throws-pattern?)
modifiers-pattern：方法的可见性修饰符，如 public，protected，private；
ret-type-pattern：方法的返回值类型，如 int，void 等；
declaring-type-pattern：方法所在类的全路径名，如 com.spring.Aspect；
name-pattern：方法名，如 getOrderDetail()；
param-pattern：方法的参数类型，如 java.lang.String；
throws-pattern：方法抛出的异常类型，如 java.lang.Exception；

// 匹配目标类的所有 public 方法，第一个 * 代表返回类型，第二个 * 代表方法名，..代表方法的参数
execution(public * *(..))
// 匹配目标类所有以 User 为后缀的方法。第一个 * 代表返回类型，*User 代表以 User 为后缀的方法
execution(* *User(..))
// 匹配 User 类里的所有方法
execution(* com.test.demo.User.*(..))
// 匹配 User 类及其子类的所有方法
execution(* com.test.demo.User+.*(..)) :
// 匹配 com.test 包下的所有类的所有方法
execution(* com.test.*.*(..))


// 用来限制代理的对象 必须是 指定类型的类 this 表示使用了cglib来代理的时候获取的类，
// target 表示使用jdk动态代理的时候获取的类
this() && target()
// 表示 被拦截的类必须 被配置的注解标识           
@target(anntation-type) 

// 拦截被RestController的类          
@Pointcut("@target(org.springframework.web.bind.annotation.RestController)")
public void intercept(){}          

// 表示拦截 pattern 全类名，下的所有方法， @within() 拦截的是 pattern 注解标识的类
within(pattern) && @within(pattern)

@Pointcut("@within(org.springframework.web.bind.annotation.RestController)")
public void intercept(){}          

// 拦截被 注解修饰的类          
@annotation(annotation-type)          
```
