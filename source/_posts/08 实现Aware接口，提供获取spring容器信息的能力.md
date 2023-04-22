### 前言
spring 对bean的创建过程做了很完整的封装。但是提供了非常多的扩展接口，供我们使用。这一节主要是实现spring提供的获取 beanFactory，classLoader  等属性的能力。
在我们开发过程中我们经常会使用到 `ApplicationContextAware`接口，来获取到 spring的上下文。来完成对bean的获取，当拿到了`BeanFactory`以后，我们能做的东西就多起来了，我们可以通过的spring工厂获取到我们需要的类，等等。
![](https://s3.bmp.ovh/imgs/2023/03/09/cf2b383e1c737084.png#id=UPX80&originHeight=226&originWidth=849&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
### 设计&实现
spring 提供`Aware`接口机制，给外部的类提供获取spring内部信息的能力。目前spring常用的`Aware`接口有
![](https://s3.bmp.ovh/imgs/2023/03/09/f0409d22df34ba4e.png#id=szRhU&originHeight=152&originWidth=931&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
#### Aware 感知接口
`Aware`接口，只做标记。类似于`Serializable`序列化接口，仅标记这个类可以序列化。Aware 仅表示实现类具有在获取springbean创建过程中的一些内部属性的能力。
```java
/**
 * 只做标记
 * spring容器感知接口
 */
public interface Aware {

}

```
#### 提供具体能力的接口
`ApplicationContextAware` 提供获取 applicationContext 的能力
```java
public interface ApplicationContextAware extends Aware {

    void setApplicationContext(ApplicationContext applicationContext);

}
```
`BeanClassLoaderAware`提供获取 classLoader 的能力
```java
public interface BeanClassLoaderAware extends Aware{

    void setBeanClassLoader(ClassLoader classLoader);

}
```
`BeanFactoryAware` 提供获取 BeanFactory 的能力
```java
public interface BeanFactoryAware extends Aware{

    void setBeanFactory(BeanFactory beanFactory) throws BeansException;

}
```
 `BeanNameAware` 提供获取 beanName 的能力
```java
public interface BeanNameAware extends Aware{

    void setBeanName(String beanName);

}
```
他们都在创建bean完成后，在添加bean的扩展属性时，给这个bean加上特定的能力
```java
@Override
    protected Object createBean(String beanName, BeanDefinition beanDefinition, Object[] args) {
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
        // 注册实现了 DisposableBean 接口的 Bean 对象
        registerDisposableBeanIfNecessary(beanName, bean, beanDefinition);
        registerSingleton(beanName, bean);
        return bean;
    }


private Object initializeBean(String beanName, Object bean, BeanDefinition beanDefinition) throws BeansException {

        if (bean instanceof Aware) {
            if (bean instanceof BeanFactoryAware) {
                ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
            }
            if (bean instanceof ApplicationContextAware) {
                ((ApplicationContextAware) bean).setApplicationContext(this);
            }
            if (bean instanceof BeanClassLoaderAware) {
                ((BeanClassLoaderAware) bean).setBeanClassLoader(getClassLoader());
            }
            if (bean instanceof BeanNameAware) {
                ((BeanNameAware) bean).setBeanName(beanName);
            }
        }
		.....
	}
```
### 测试
实现 需要添加特定能力的 Aware接口，实现他们的方法
```java
public class UserService implements InitializingBean, DisposableBean, ApplicationContextAware, BeanClassLoaderAware, BeanNameAware {

    private ApplicationContext applicationContext;
    private ClassLoader classLoader;

    @Override
    public void setBeanClassLoader(ClassLoader classLoader) {
        this.classLoader = classLoader;
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }

}


@Test
    public void testContext1() throws BeansException {
        ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:spring.xml");
        applicationContext.registerShutdownHook();
        UserService userService = (UserService) applicationContext.getBean("userService");
        System.out.println(userService.say());
        System.out.println(userService.getApplicationContext());
        System.out.println(userService.getClassLoader());
        System.out.println(userService.getBeanName());
    }
```
![](https://s3.bmp.ovh/imgs/2023/03/09/13e85198c901bb2b.png#id=DBxTL&originHeight=295&originWidth=1244&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
### 

