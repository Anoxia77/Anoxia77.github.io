---
title: 实现Spring上下文，提供Bean创建的扩展
---



## 前言

上一节，我们已经完成通过 xml 配置文件 实现 Bean 的注入。这一节是重头戏，老实说，我现在都还弄的很明白。这是第二遍看这个过程了，还是有点云里雾里的。但是相对于第一次来说，已经上升了一个高度。
在这一节，会对 spring 创建 bean 的过程进行封装。并且提供 `BeanPostProcessor`,`BeanFactoryPostProcessor`等扩展接口。（这两个东西老强了）
上下文把整个创建过程包装起来，对外暴露一个简单的入口，其他的东西都在内部实现。（这个过程其实可以类比成 `模板方法` 是使用，我们直接使用就可以，并不需要知道内部的实现）
由于上下文已经封装好了整个bean的创建过程，内部实现对外是不开放的，所以提供了 BeanPostProcessor（bean 创建 之前处理，bean 创建之后处理）、BeanFactoryPostProcessor （对beanDifinition 创建后处理）给 开发者 的扩展接口，在创建bean的过程中，添加自己的实现逻辑。
![](https://s3.bmp.ovh/imgs/2023/03/06/e38004593280d01d.png#id=qYU9z&originHeight=810&originWidth=1100&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## 实现
### BeanPostProcessor Bean 创建完成前后扩展接口
它可以在 Bean 实例化、依赖注入、初始化前后等时刻对 Bean 进行一些额外的处理，例如修改 Bean 属性、动态代理等等。
这个类使用了`模板方法`模式。扩展类只需实现该接口，然后spring会在创建的bean的过程中调用这个方法完成 对bean的修改。
spring 在获取所有的 扩展类并完成扩展使用的是`装饰器模式`，每个 扩展类都可以类比成一个`装饰器`,然后通过`责任链`的方式完成调用。
> **模板方法：**模板方法是一种行为设计模式，它定义了一个过程的`骨架`，并允许子类在不改变该过程结构的情况下重新定义某些步骤。
> **装饰器模式：**装饰器模式是一种结构型设计模式，它允许动态地向对象添加新的行为，同时又不会改变其原有的行为。该模式通过一种装饰器类来包装原有的类，从而给对象增加新的功能。
> **责任链模式**：责任链模式是一种行为型设计模式，它将多个处理器（Handler）组成一条链，用于依次处理请求，直到请求被处理完成为止。在责任链模式中，每个处理器都有一个指向下一个处理器的引用，从而形成了一条处理链。

在实际开发过程中，我们也可以尝试使用这些设计模式来优化我们的代码。在这个扩展接口的实现过程，我们可以学到 `模板` +`装饰器` +`责任链`的一种使用场景。很多时候，单独是设计模式并不能很好地解决问题，但是当我们把他们组合到一起的时候，他就会变成一种很`优雅的解决方案`.
```java

/**
 * @author huangle
 * @date 2023/2/13 15:18
 */
public interface BeanPostProcessor {

    /**
     * 在 Bean 对象执行初始化方法之前，执行此方法
     *
     * @param bean
     * @param beanName
     * @return
     * @throws BeansException
     */
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;

    /**
     * 在 Bean 对象执行初始化方法之后，执行此方法
     *
     * @param bean
     * @param beanName
     * @return
     * @throws BeansException
     */
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;

}

```
### BeanFactoryPostProcessor beanDefinition 加载完成后扩展接口
这个扩展设计跟上一个差不多，具体可以看上面的设计
```java
public interface BeanFactoryPostProcessor {

    /**
     * 在所有的 BeanDefinition 加载完成后，实例化 Bean 对象之前，提供修改 BeanDefinition 属性的机制
     * @param beanFactory
     */
    void postProcessorBeanFactory(ConfigurableListableBeanFactory beanFactory);

}
```
### AbstractApplicationContext 统一处理抽象类
`AbstractApplicationContext`完成对逻辑过程的整合，定义统一处理流程，实现过程通过定义抽象方法，交给具体的实现类去完成。
这里其实就是对spring容器 初始化做一个 `模板处理`,只定义基本`骨架`，具体都交给后面的子类是完成。
```java
public interface ConfigurableApplicationContext extends ApplicationContext{

    void refresh() throws BeansException;

}

/**
 * @author huangle
 * @date 2023/2/13 15:15
 */
public abstract class AbstractApplicationContext extends DefaultResourceLoader implements ConfigurableApplicationContext {

    /**
     * 模板方法，控制定义spring上下文的刷新与创建
     * @throws BeansException
     */
    @Override
    public void refresh() throws BeansException {
        // 1. 创建beanFactory，加载beanDefinition
        refreshBeanFactory();
        // 2. 获取 beanFactory
        ConfigurableListableBeanFactory beanFactory = getBeanFactory();
        // 3. bean 实例化之前 执行 beanFactoryPostProcessor
        invokeBeanFactoryPostProcessors(beanFactory);
        // 4. beanPostProcessor 完成 bean 实例化后的注册操作
        registerBeanPostProcessors(beanFactory);
        // 5. 提前实例化 单例bean 对象
        beanFactory.preInstantiateSingletons();
    }

    protected abstract void refreshBeanFactory() throws BeansException;

    protected abstract ConfigurableListableBeanFactory getBeanFactory();

    /**
     * 初始化 BeanFactoryPostProcessor 处理器
     * @param beanFactory
     */
    private void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory){
        Map<String, BeanFactoryPostProcessor> beanFactoryPostProcessorMap = beanFactory.getBeansOfType(BeanFactoryPostProcessor.class);
        for (BeanFactoryPostProcessor factoryPostProcessor : beanFactoryPostProcessorMap.values()) {
            // 处理 beanDifinition 扩展逻辑
            factoryPostProcessor.postProcessorBeanFactory(beanFactory);
        }
    }

    /**
     * 初始化 BeanPostProcessor 处理器
     * @param beanFactory
     */
    private void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory){
        Map<String, BeanPostProcessor> beanPostProcessorMap = beanFactory.getBeansOfType(BeanPostProcessor.class);
        for (BeanPostProcessor postProcessor : beanPostProcessorMap.values()) {
            beanFactory.addBeanPostProcessor(postProcessor);
        }
    }
}
```
### AbstractAutowireCapableBeanFactory#createBean 统一处理bean扩展逻辑
spring流程的核心类，所有的扩展都会在这里汇合，一同作用于spring创建bean的过程。
```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory implements AutowireCapableBeanFactory {

    private CglibSubclassingInstantiationStrategy instantiationStrategy = new CglibSubclassingInstantiationStrategy();

    @Override
    public Object createBean(String beanName, BeanDefinition beanDefinition, Object[] args) {
        Object bean = null;
        try {
            bean = createBeanInstance(beanDefinition, beanName, args);
            // 注入属性
            applyPropertyValues(beanName, bean, beanDefinition);
            // 提供给外部的扩展包装，执行 Bean 的初始化方法和 BeanPostProcessor 的前置和后置处理方法
            bean = initializeBean(beanName, bean, beanDefinition);
        } catch (Exception e) {
            throw new RuntimeException("bean create error!", e);
        }
        registerSingleton(beanName, bean);
        return bean;
    }
    private Object initializeBean(String beanName, Object bean, BeanDefinition beanDefinition) throws BeansException {
        // 1. 执行 BeanPostProcess before 操作
        Object wrappedBean = applyBeanPostProcessorsBeforeInitialization(bean, beanName);

        invokeInitMethods(beanName,wrappedBean,beanDefinition);

        // 2. 执行 BeanPostProcess after 操作
        wrappedBean = applyBeanPostProcessorsAfterInitialization(bean,beanName);

        return wrappedBean;
    }

    private void invokeInitMethods(String beanName, Object wrappedBean, BeanDefinition beanDefinition){

    }


    @Override
    public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName) throws BeansException {
        Object result = existingBean;
        // 责任链模式 去获取到用户的扩展内容，然后在这里统一去处理
        for (BeanPostProcessor beanPostProcessor : getBeanPostProcessors()) {
            Object current = beanPostProcessor.postProcessBeforeInitialization(result, beanName);
            if (current == null) {
                return result;
            }
            result = current;
        }
        return result;
    }


    @Override
    public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName) throws BeansException {
       Object result = existingBean;
        for (BeanPostProcessor beanPostProcessor : getBeanPostProcessors()) {
            Object current = beanPostProcessor.postProcessAfterInitialization(result, beanName);
            if (current == null){
                return result;
            }
            result = current;
        }
        return result;
    }
}
```
## 测试
### 定义 MyBeanFactoryPostProcessor、MyBeanPostProcecssor 扩展内容
```java
/**
 * @author huangle
 * @date 2023/3/3 15:55
 */
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessorBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        BeanDefinition userService = beanFactory.getBeanDefinition("userService");
        PropertyValues propertyValues = userService.getPropertyValues();
        PropertyValue propertyValue = new PropertyValue("nickname", "泰罗");
        propertyValues.addPropertyValue(propertyValue);
    }
}


/**
 * @author huangle
 * @date 2023/3/3 15:03
 */
public class MyBeanPostProcecssor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        if ("userService".equals(beanName)) {
            UserService userService = (UserService) bean;
            userService.setName("海绵宝宝");
        }
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
}
```
### 业务逻辑
```java
public class UserDao {

    public String getUserName(){
        return "Anoxia";
    }

}

public class UserService {

    private String name;

    private String nickname;

    private UserDao userDao;

    public UserService(String name) {
        this.name = name;
    }

    public String say(){
        return userDao.getUserName()+" say hello "+name + "and" + nickname;
    }
}


```
### 配置文件
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans>

    <bean id="userDao" class="cn.anoxia.springframework.beans.factory.support.UserDao"/>

    <bean id="userService" class="cn.anoxia.springframework.beans.factory.support.UserService">
        <property name="name" value="Anoxia"/>
        <property name="nickname" value="迪迦"/>
        <property name="userDao" ref="userDao"/>
    </bean>

    <bean id="myBeanPostProcecssor" class="cn.anoxia.springframework.beans.factory.support.MyBeanPostProcecssor"/>
    <bean id="myFactoryPostProcessor" class="cn.anoxia.springframework.beans.factory.support.MyBeanFactoryPostProcessor"/>

</beans>
```
### 测试类
`testContext` 测试 手动创建工厂、创建处理类，注入。`testContext1`测试自动读取xml文件，自动注入
```java
@Test
    public void testContext() throws BeansException {
        // 1. 初始化bean工厂
        DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
        // 2. 定义读取xml处理器,加载bean
        XmlBeanDefinitionReader definitionReader = new XmlBeanDefinitionReader(beanFactory);
        definitionReader.loadBeanDefinitions("classpath:spring.xml");
        // 3. BeanDefinition 加载完成 & Bean实例化之前，修改 BeanDefinition 的属性值
        MyBeanFactoryPostProcessor factoryPostProcessor = new MyBeanFactoryPostProcessor();
        factoryPostProcessor.postProcessorBeanFactory(beanFactory);
        // 4. Bean实例化之后，修改 Bean 属性信息
        MyBeanPostProcecssor postProcecssor = new MyBeanPostProcecssor();
        beanFactory.addBeanPostProcessor(postProcecssor);

        UserService userService = (UserService) beanFactory.getBean("userService");
        System.out.println(userService.say());
    }

    @Test
    public void testContext1() throws BeansException {
        ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:spring.xml");
        UserService userService = (UserService) applicationContext.getBean("userService");
        System.out.println(userService.say());
    }
```
测试结果
![](https://s3.bmp.ovh/imgs/2023/03/06/c64cd049f7c6f638.png#id=yOjvb&originHeight=78&originWidth=876&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
![](https://s3.bmp.ovh/imgs/2023/03/06/4cf9604d5a5d2e33.png#id=HtKRJ&originHeight=124&originWidth=730&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## 总结
这一节是学习spring过程中比较难的一节，如果学明白了这一节，对spring的扩展基本就理解了。学习的过程很枯燥，有很多时候，遇到一点问题就想放弃，没有一追到底的毅力。其实我感觉嗷，学起来累，学不明白，还是自己会的东西太少了，导致上下连不起了，贯通不了，导致寸步难行，继而放弃。
怎么说呢，写代码这东西真的没有什么捷径，多写，多写，多写。只看不写，每一次有问题又是去copy，永远也不会。
