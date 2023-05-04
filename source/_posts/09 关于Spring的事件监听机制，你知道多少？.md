## 前言
前一节，我们了解了Spring 提供的 `Aware`接口，我们可以通过这个实现这个接口的一些类获取到我们需要的东西。具体内容见前一节。
Spring 也提供了一种单机的事件机制。可以通过发送、监听，来实现一些异步操作。
使用这种 `类似 MQ` 的事件机制，我们可以通过 这个事件机制来完成一些自己的业务操作。在我们使用spring提供的事件机制时，我们只需要关注自己的事件，和自己的事件处理器。所有的事件都会继承 `ApplicationEvent`，然后再完成对于自己时间的监听器，监听器都 实现 ` ApplicationListener`接口，实现 `onApplicationEvent`方法，来处理监听到的事件。
下面是事件注册、监听的整体过程。主要就是 在spring容器加载的时候，把所有的监听器统一注册到`SimpleApplicationEventMulticaster`事件分发器中，然后统一处理。
![](https://s3.bmp.ovh/imgs/2023/03/14/c7f27c649e6fa840.png#id=ZIfiU&originHeight=530&originWidth=1101&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
![](https://cdn.nlark.com/yuque/0/2023/jpeg/2869098/1678786043791-2d1cd093-6c93-4ecb-97a5-6966f692f972.jpeg)
## 实现
`ApplicationEvent` 事件基类, `EventObject`是java提供的一个类
```java
public class ApplicationEvent extends EventObject {
    /**
     * Constructs a prototypical Event.
     *
     * @param source The object on which the Event initially occurred.
     * @throws IllegalArgumentException if source is null.
     */
    public ApplicationEvent(Object source) {
        super(source);
    }
}
```
`ApplicationContextEvent`系统事件
```java
public class ApplicationContextEvent extends ApplicationEvent {
    /**
     * Constructs a prototypical Event.
     *
     * @param source The object on which the Event initially occurred.
     * @throws IllegalArgumentException if source is null.
     */
    public ApplicationContextEvent(ApplicationContext source) {
        super(source);
    }


    public final ApplicationContext getApplicationContext() {
        return (ApplicationContext) getSource();
    }
}
```
`ApplicationEventPublisher` 事件发布器，统一发布事件。
```java
public interface ApplicationEventPublisher {

    /**
     * 发布事件
     * @param event 事件
     */
    void publishEvent(ApplicationEvent event);

}
```
`ApplicationEventMulticaster`事件管理、分发器。`ApplicationEventMulticaster`统一定义公共行为， `AbstractApplicationEventMulticaster`抽象类处理公共逻辑。`SimpleApplicationEventMulticaster`默认分发器，只需要执行具体的分发逻辑。`supportEvent`方法检查事件是否需要被处理。
```java
public interface ApplicationEventMulticaster {

    /**
     * Add a listener to be notified of all events.
     * @param listener the listener to add
     */
    void addApplicationListener(ApplicationListener listener);

    /**
     * Remove a listener from the notification list.
     * @param listener the listener to remove
     */
    void removeApplicationListener(ApplicationListener listener);

    /**
     * Multicast the given application event to appropriate listeners.
     * @param event the event to multicast
     */
    void multicastEvent(ApplicationEvent event);

}

public abstract class AbstractApplicationEventMulticaster implements ApplicationEventMulticaster, BeanFactoryAware {

    private final Set<ApplicationListener> applicationListeners = new LinkedHashSet<>();

    private BeanFactory beanFactory;

    @Override
    public void addApplicationListener(ApplicationListener listener) {
        this.applicationListeners.add((ApplicationListener) listener);
    }

    @Override
    public void removeApplicationListener(ApplicationListener listener) {
        this.applicationListeners.remove(listener);
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) {
        this.beanFactory = beanFactory;
    }

    protected Collection<ApplicationListener<?>> getApplicationListeners(ApplicationEvent event) throws BeansException {
        LinkedList<ApplicationListener<?>> allListeners = new LinkedList<>();
        for (ApplicationListener<?> listener : applicationListeners) {
            if (supportEvent(listener, event)) {
                allListeners.add(listener);
            }
        }
        return allListeners;
    }

    /**
     * 监听器是否对该事件感兴趣
     */
    protected boolean supportEvent(ApplicationListener listener, ApplicationEvent event) throws BeansException {
        // 检查事件是否需要关注
        Class<? extends ApplicationListener> clazz = listener.getClass();
        Class<?> targetClass = ClassUtils.isCglibProxyClass(clazz) ? clazz.getSuperclass() : clazz;
        Type genericInterface = targetClass.getGenericInterfaces()[0];
        Type typeArgument = ((ParameterizedType) genericInterface).getActualTypeArguments()[0];
        String className = typeArgument.getTypeName();
        Class<?> eventClassName;
        try {
            eventClassName = Class.forName(className);
        } catch (ClassNotFoundException e) {
            throw new BeansException("wrong event class name: " + className);
        }
        // 判定此 eventClassName 对象所表示的类或接口与指定的 event.getClass() 参数所表示的类或接口是否相同，或是否是其超类或超接口。
        // isAssignableFrom是用来判断子类和父类的关系的，或者接口的实现类和接口的关系的，默认所有的类的终极父类都是Object。如果A.isAssignableFrom(B)结果是true，证明B可以转换成为A,也就是A可以由B转换而来。
        return eventClassName.isAssignableFrom(event.getClass());
    }
}

public class SimpleApplicationEventMulticaster extends AbstractApplicationEventMulticaster{
    @Override
    public void multicastEvent(ApplicationEvent event) {
        try {
            // 实现事件处理
            Collection<ApplicationListener<?>> listeners = getApplicationListeners(event);
            for (ApplicationListener listener : listeners) {
                listener.onApplicationEvent(event);
            }
        }catch (Exception e){
            // 执行失败
        }
    }
}
```
`AbstractApplicationContext`容器刷新，完成 事件分发器的注册、事件监听器的注册。
```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader implements ConfigurableApplicationContext {

    public static final String APPLICATION_EVENT_MULTICASTER_BEAN_NAME = "applicationEventMulticaster";

    private ApplicationEventMulticaster applicationEventMulticaster;

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
        // 3. 添加 ApplicationContextAwareProcessor，让继承自 ApplicationContextAware 的 Bean 对象都能感知所属的 ApplicationContext
        beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
        // 4. bean 实例化之前 执行 beanFactoryPostProcessor
        invokeBeanFactoryPostProcessors(beanFactory);
        // 5. beanPostProcessor 完成 bean 实例化后的注册操作
        registerBeanPostProcessors(beanFactory);
        // 6. 提前实例化 单例bean 对象
        beanFactory.preInstantiateSingletons();
        // 初始化事件发布者
        initApplicationEventMulticaster();
        // 注册监听事件
        registerListeners();
        // 完成刷新
        finishRefresh();
    }

    private void finishRefresh() {
        publishEvent(new ContextRefreshedEvent(this));
    }

    private void registerListeners() {
        ConfigurableListableBeanFactory beanFactory = getBeanFactory();
        Collection<ApplicationListener> listeners = beanFactory.getBeansOfType(ApplicationListener.class).values();
        for (ApplicationListener listener : listeners) {
            applicationEventMulticaster.addApplicationListener(listener);
        }
    }

    private void initApplicationEventMulticaster() {
        ConfigurableListableBeanFactory beanFactory = getBeanFactory();
        applicationEventMulticaster = new SimpleApplicationEventMulticaster();
        beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, applicationEventMulticaster);
    }


    @Override
    public void publishEvent(ApplicationEvent event) {
        applicationEventMulticaster.multicastEvent(event);
    }

}
```
## 测试
定义事件、定义事件监听器。
```java
/**
 * @author huangle
 * @date 2023/3/14 16:21
 */
public class ConsumerEvent extends ApplicationContextEvent {

    private String msgId;

    private String msgBody;

    public ConsumerEvent(ApplicationContext source, String msgId, String msgBody) {
        super(source);
        this.msgId = msgId;
        this.msgBody = msgBody;
    }
}


public class ConsumerEventLister implements ApplicationListener<ConsumerEvent> {
    @Override
    public void onApplicationEvent(ConsumerEvent consumerEvent) {
        System.out.println(this.getClass()+"接受到消息:"+consumerEvent.getMsgId()+"-"+consumerEvent.getMsgBody());
    }
}

```
 注入事件监听器
```java
<?xml version="1.0" encoding="UTF-8"?>
<beans>
	<bean id="consumerEventLister" class="cn.anoxia.springframework.beans.factory.support.event.ConsumerEventLister"/>
    <bean id="applicationRefershEventListener" class="cn.anoxia.springframework.beans.factory.support.event.ApplicationRefershEventListener"/>
    <bean id="applicationCloseEventListener" class="cn.anoxia.springframework.beans.factory.support.event.ApplicationCloseEventListener"/>
</beans>
```
发布事件
```java
 @Test
    public void testEvent() throws BeansException{
        ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:spring.xml");
        applicationContext.publishEvent(new ConsumerEvent(applicationContext, UUID.randomUUID().toString(),"消息发送成功！"));
        applicationContext.registerShutdownHook();
    }
```
![](https://s3.bmp.ovh/imgs/2023/03/14/da75094751994adb.png#id=JDGXS&originHeight=155&originWidth=1412&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
