## 前言
`**面向切面编程 - Aspect Oriented Programming (AOP)**`** 是什么：  **
> AOP最早是AOP联盟的组织提出的,指定的一套规范,spring将AOP的思想引入框架之中,通过**预编译方式**和**运行期间动态代理**实现程序的统一维护的一种技术。

**如何理解AOP？**
> AOP的本质也是为了** 解耦**，它是一种设计思想； 提供了一种在代码运行期间实现代码动态织入的能力，在不影响业务代码的基础上，对义务代码的功能逻辑做统一的处理。重点在  解耦 和 复用

**怎么用？**
> AOP 主要是通过代理来实现的，在运行期间，通过代理对需要增强的类进行扩展。

## JDK & Cglib 动态代理
代理区分了 JDK 动态代理、Cglib 动态代理，两种方式的区别在于

- JDK 动态代理 只能代理接口，所以被jdk代理的类需要实现一个接口
- Cglib 可以代理接口、也可以直接代理类，Cglib 的通过 ASM 操作 底层字节码实现的代理，所以没有什么限制。

**为什么JDK只能代理接口？**
这是一个被面试问了很多次的问题，很多时候我们只是知道他就是不能，总是`知其然，不知所以然`。我们知道的是，JDK代理对象，是通过生成一个实现了接口的新类，然后对这个新类做的扩展。然后执行业务逻辑的生活还是由原来的业务类执行。就是说`代理类`，其实在执行业务逻辑前，跟`实现类`是没有很大的关系的。两个类都是存在于 spring的容器中的。
![](https://cdn.nlark.com/yuque/0/2023/jpeg/2869098/1679280383904-85baf21f-9cef-42e0-a6a8-d0eefbc90b1c.jpeg)
```java
@Test
    public void test_cglib(){
        IUserDao userDao = (IUserDao) Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(),
                new Class[]{IUserDao.class}, (proxy, method, args) -> "你被代理了！");
        String name = userDao.queryName();
        String user = userDao.queryUser();
        System.out.println("测试结果："+name+","+user);
    }
```
使用JDK动态代理去代理一个类的过程，主要用到的是 `Proxy.newProxyInstance`。接下来，我们从 源码中去找问题。
```java
/*
* Look up or generate the designated proxy class.
* 获取代理对象
*/
Class<?> cl = getProxyClass0(loader, intfs);

private static Class<?> getProxyClass0(ClassLoader loader,
                                           Class<?>... interfaces) {
        if (interfaces.length > 65535) {
            throw new IllegalArgumentException("interface limit exceeded");
        }

        // If the proxy class defined by the given loader implementing
        // the given interfaces exists, this will simply return the cached copy;
        // otherwise, it will create the proxy class via the ProxyClassFactory
        // 如果实现给定接口的给定加载器定义的代理类存在，这将简单地返回缓存副本;
    	// 否则，它将通过ProxyClassFactory创建代理类。 这里提到通过 ProxyClassFactory 创建代理类，
    	// 下面我们去看这个是怎么创建对象的
        return proxyClassCache.get(loader, interfaces);
    }

```
`ProxyClassFactory`是proxy 里面的一个内部类，实现了 `BiFunction`函数式接口
```java
private static final class ProxyClassFactory
        implements BiFunction<ClassLoader, Class<?>[], Class<?>>
    {
        // prefix for all proxy class names
        private static final String proxyClassNamePrefix = "$Proxy";

        // next number to use for generation of unique proxy class names
        private static final AtomicLong nextUniqueNumber = new AtomicLong();

        @Override
        public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {

            Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
            for (Class<?> intf : interfaces) {
                ....
                /*
                 * Verify that the Class object actually represents an
                 * interface.
                 * 这里会做一次接口检查
                 */
                if (!interfaceClass.isInterface()) {
                    throw new IllegalArgumentException(
                        interfaceClass.getName() + " is not an interface");
                }
                ...
            }

            ...

            /*
             * Generate the specified proxy class.
             * 这里会去生成一个代理对象
             */
            byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
                proxyName, interfaces, accessFlags);
            ....
        }
    }
```
把通过JDK代理创建的代理对象输出出来，我们就明白他的 `前因后果`了。
```java
@Test
    public void test_jdk(){
        UserService service = new UserService();
        Class[] clazzs = new Class[]{IUserService.class};
        byte[] bytes = ProxyGenerator.generateProxyClass("UserService", clazzs);
        File path = new File("E:\\huangle\\UserService.class");
        try {
            FileOutputStream outputStream = new FileOutputStream(path);
            outputStream.write(bytes);
            outputStream.flush();
            outputStream.close();
        }catch (Exception e){
            e.printStackTrace();
        }
    }


//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by FernFlower decompiler)
//
public final class UserService extends Proxy implements IUserService {
    private static Method m1;
    private static Method m2;
    private static Method m4;
    private static Method m0;
    private static Method m3;

    public UserService(InvocationHandler var1) throws  {
        super(var1);
    }
}
```
通过上面输出的内容，就可以很明显的看出问题所在。
当通过JDK代理后，生成的类，重点在 `extends Proxy`, 根据 java `单继承、多实现`的原理，所以`代理类`是不能继承 `实现类`去完成业务的扩展的，所以只能使用 实现接口的方式，来完成对业务的扩展 
**总结：为什么jdk只能代理接口**
> 因为jdk代理是通过继承 Proxy 来实现的，根据 java 单继承的特点，所以他不能在去继承 `业务类`类完成对 业务类的扩展，所以只能 通过实现业务类接口的方式来完成扩展。

## 实现
AOP  实现 方法扩展的 主要是通过几个操作实现的。`方法拦截器`、`方法匹配器`、`代理对象`等。
![](https://s3.bmp.ovh/imgs/2023/03/20/155cbb4787448cd0.png#id=b5bHZ&originHeight=709&originWidth=798&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
![](https://s3.bmp.ovh/imgs/2023/03/20/71127238b87f960d.png#id=JyBM1&originHeight=913&originWidth=1654&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
### 匹配器（类&方法）
主要是 `ClassFilter`/`MethodMatcher`
```java
public interface ClassFilter {

    /**
     * Should the pointcut apply to the given interface or target class?
     * @param clazz the candidate target class
     * @return whether the advice should apply to the given target class
     */
    boolean matches(Class<?> clazz);

}

public interface MethodMatcher {

    /**
     * Perform static checking whether the given method matches. If this
     * @return whether or not this method matches statically
     */
    boolean matches(Method method, Class<?> targetClass);

}

```
### 切点（匹配器包装接口）
这个接口主要是把 类 匹配器和方法匹配器，包装起来，统一管理（统称切点）
```java
public interface Pointcut {


    ClassFilter getClassFilter();

    MethodMatcher getMethodMatcher();


}
```
### 拦截器(处理方法)
在 aop 中，这类操作 可以分为 前置通知，后置通知， 环绕通知
```java
package org.aopalliance.aop;

public interface Advice {
}

// 观察者模式？
public interface Advisor {

    Advice getAdvice();

}
// 前置通知
public interface BeforeAdvice extends Advice {
}
// 方法前置通知
public interface MethodBeforeAdvice extends BeforeAdvice{

    void before(Method method, Object[] args, Object target) throws Throwable;

}
```
### jdk & cglib 代理类
```java
/**
 * @author huangle
 * @date 2023/3/15 15:35
 */
public class CglibAopProxy implements AopProxy{

    private final AdvisedSupport advised;

    public CglibAopProxy(AdvisedSupport advised) {
        this.advised = advised;
    }

    @Override
    public Object getProxy() {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(advised.getTargetSource().getTarget().getClass());
        enhancer.setInterfaces(advised.getTargetSource().getTargetClass());
        enhancer.setCallback(new DynamicAdvisedInterceptor(advised));
        return enhancer.create();
    }

    private static class DynamicAdvisedInterceptor implements MethodInterceptor {

        private AdvisedSupport advised;

        public DynamicAdvisedInterceptor(AdvisedSupport advised) {
            this.advised = advised;
        }

        @Override
        public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
            CglibMethodInvocation methodInvocation = new CglibMethodInvocation(advised.getTargetSource().getTarget(), method, args, proxy);
            if (advised.getMethodMatcher().matches(method, advised.getTargetSource().getTarget().getClass())) {
                return advised.getMethodInterceptor().invoke(methodInvocation);
            }
            return methodInvocation.proceed();
        }
    }

    private static class CglibMethodInvocation extends ReflectiveMethodInvocation {

        private MethodProxy methodProxy;

        public CglibMethodInvocation(Object target, Method method, Object[] args, MethodProxy methodProxy) {
            super(target, method, args);
            this.methodProxy = methodProxy;
        }

        @Override
        public Object proceed() throws Throwable {
            return this.methodProxy.invoke(this.target, this.args);
        }

    }
}


public class JdkDynamicAopProxy implements AopProxy, InvocationHandler {

    private final AdvisedSupport advisedSupport;

    public JdkDynamicAopProxy(AdvisedSupport advisedSupport) {
        this.advisedSupport = advisedSupport;
    }

    @Override
    public Object getProxy() {
        return Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), advisedSupport.getTargetSource().getTargetClass(), this);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (advisedSupport.getMethodMatcher().matches(method, advisedSupport.getTargetSource().getTarget().getClass())) {
            MethodInterceptor interceptor = advisedSupport.getMethodInterceptor();
            return interceptor.invoke(new ReflectiveMethodInvocation(advisedSupport.getTargetSource().getTarget(), method, args));
        }
        return method.invoke(advisedSupport.getTargetSource().getTarget(), args);
    }
}
```
### AOP 织入spring生命周期
aop 织入 spring的生命周期，是通过 BeanFactoryPostProcessor 实现的，在beanDefinition 注册后，给beanDefinition 添加代理对象。实现替换
```java
@Override
    protected Object createBean(String beanName, BeanDefinition beanDefinition, Object[] args) {
        Object bean = null;
        try {
            // 检查是否需要代理
            bean = resolveBeforeInstantiation(beanName, beanDefinition);
            if (bean != null) {
                return bean;
            }
            bean = createBeanInstance(beanDefinition, beanName, args);
            // 在设置bean属性钱，允许修改bean的属性（通过BeanPostProcessor）
            applyBeanPostProcessorsBeforeApplyingPropertyValues(beanName, bean, beanDefinition);
            // 注入属性
            applyPropertyValues(beanName, bean, beanDefinition);
            // 提供给外部的扩展包装，执行 Bean 的初始化方法和 BeanPostProcessor 的前置和后置处理方法
            bean = initializeBean(beanName, bean, beanDefinition);
        } catch (Exception e) {
            throw new RuntimeException("bean create error!", e);
        }
        // 注册实现了 DisposableBean 接口的 Bean 对象
        registerDisposableBeanIfNecessary(beanName, bean, beanDefinition);
        // 检查bean是单例还是原型
        if (beanDefinition.isSingleton()) {
            registerSingleton(beanName, bean);
        }
        return bean;
    }
// 检查是否需要代理
protected Object resolveBeforeInstantiation(String beanName, BeanDefinition beanDefinition) throws BeansException {
        Object bean = applyBeanPostProcessorBeforeInstantiation(beanDefinition.getBeanClass(), beanName);
        if (bean != null) {
            bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
        }
        return bean;
    }
// 每一个bean 都会去检查
public Object applyBeanPostProcessorBeforeInstantiation(Class<?> clazz, String beanName) {
        for (BeanPostProcessor processor : getBeanPostProcessors()) {
            if (processor instanceof InstantiationAwareBeanPostProcessor) {
                Object bean = ((InstantiationAwareBeanPostProcessor) processor).postProcessBeforeInstantiation(clazz, beanName);
                if (bean != null) {
                    return bean;
                }
            }
        }
        return null;
    }
```
```java
public class DefaultAdvisorAutoProxyCreator implements InstantiationAwareBeanPostProcessor, BeanFactoryAware {


    private DefaultListableBeanFactory beanFactory;


    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = (DefaultListableBeanFactory) beanFactory;
    }


    @Override
    public Object postProcessBeforeInstantiation(Class<?> clazz, String beanName) {
    	// 框架类，直接跳过 （切点、拦截类、匹配类等）
        if (isInfrastructureClass(clazz)) return null;

        // 获取所有需要处理的切点处理方法
        Collection<AspectJExpressionPointcutAdvisor> pointcutAdvisors = beanFactory.getBeansOfType(AspectJExpressionPointcutAdvisor.class).values();

        for (AspectJExpressionPointcutAdvisor advisor : pointcutAdvisors) {
            ClassFilter classFilter = advisor.getPointcut().getClassFilter();
            if (!classFilter.matches(clazz)) {
                continue;
            }
            AdvisedSupport support = new AdvisedSupport();
            TargetSource targetSource = null;
            try {
                targetSource = new TargetSource(clazz.getDeclaredConstructor().newInstance());
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
            support.setTargetSource(targetSource);
            support.setMethodInterceptor((MethodInterceptor) advisor.getAdvice());
            support.setMethodMatcher(advisor.getPointcut().getMethodMatcher());
            support.setProxyTargetClass(false);

            return new ProxyFactory(support).getProxy();
        }

        return null;
    }


    private boolean isInfrastructureClass(Class<?> beanClass) {
        return Advice.class.isAssignableFrom(beanClass) || Pointcut.class.isAssignableFrom(beanClass) || Advisor.class.isAssignableFrom(beanClass);
    }


    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
}
```
## 测试
定义拦截处理器
```java
public class UserServiceBeforeAdvice implements MethodBeforeAdvice {
    @Override
    public void before(Method method, Object[] args, Object target) throws Throwable {
        System.out.println("拦截方法："+method.getName());
    }
}
```
注册bean
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans>

    <bean id="userService" class="cn.anoxia.springframework.beans.factory.support.UserService"/>

    <bean id="autoProxyCreator" class="cn.anoxia.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator"/>

    <bean id="beforeAdvice" class="cn.anoxia.springframework.beans.factory.support.proxy.UserServiceBeforeAdvice"/>

    <bean id="methodInterceptor" class="cn.anoxia.springframework.aop.framework.adapter.MethodBeforeAdviceInterceptor">
        <property name="advice" ref="beforeAdvice"/>
    </bean>

    <bean id="pointcutAdvisor" class="cn.anoxia.springframework.aop.aspect.AspectJExpressionPointcutAdvisor">
        <property name="expression" value="execution(* cn.anoxia.springframework.beans.factory.support.IUserService.*(..))"/>
        <property name="advice" ref="methodInterceptor"/>
    </bean>
</beans>
```
![](https://s3.bmp.ovh/imgs/2023/03/20/e097a247569e2dd7.png#id=UXNF1&originHeight=130&originWidth=709&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
