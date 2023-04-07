---
title: 重学Spring02-关于Spring bean 的注册与获取那档子事
---



## 前言

上一节我们简单了解了一下 `BeanFactory`是一个什么。简单来说，他就是一个 容器，跟普通的容器没有什么区别。
这一节，我们从bean的基本创建过程出发，看一下 spring 的怎么把我们的bean管理起来的。
![](https://cdn.nlark.com/yuque/0/2023/jpeg/2869098/1675672108728-0cd30627-56e3-46f9-821e-095ece3c1eb2.jpeg)
先上一个bean定义、注册、获取的类的关系图，根据关系图，来确定一个bean的基本流程。
![Bean的创建与获取.png](https://cdn.nlark.com/yuque/0/2023/png/2869098/1675737115183-f7e66755-ab03-4a08-aee6-d961f5a145e7.png#averageHue=%235f5c5f&clientId=u5870bb51-cb2d-4&from=ui&id=ua7d34fed&name=Bean%E7%9A%84%E5%88%9B%E5%BB%BA%E4%B8%8E%E8%8E%B7%E5%8F%96.png&originHeight=626&originWidth=1131&originalType=binary&ratio=1&rotation=0&showTitle=false&size=87818&status=done&style=none&taskId=ua2c74d57-e17e-4877-9ad8-eed8ba0c9ec&title=)
## 实现
### bean 的定义
前面我们知道，bean的基本信息被spring拆解到 BeanDefinition 里面，spring通过这个里面的`零件`来组装一个完整的bean出来。
创建一个bean的方式有很多种, 我们通常使用以下几种方式去初始化一个bean

- 直接 new 一个
- 通过class的 `Class.newInstance()` 创建
- 通过 动态代理 来创建一个bean （动态代理有 JDK动态代理 和 CGLIB动态代理）

我们这里暂时使用的 是 第二种方式，通过 `Class.newInstance()`来完成bean的创建。
```java
public class BeanDefinition {

    private Class<?> beanClass;

    public BeanDefinition(Class<?> beanClass) {
        this.beanClass = beanClass;
    }

    public Class<?> getBeanClass() {
        return beanClass;
    }
}
```
### bean 的注册
> 编码方式主要依托于：接口定义、类实现接口、抽象类实现接口、继承类、继承抽象类，而这些操作方式可以很好的隔离开每个类的基础功能、通用功能和业务功能，当类的职责清晰后，设计也会变得容易扩展和迭代。

#### 1、BeanFactory 定义
所有的bean都被注册，放在beanFactory 中，spring对beanFactory的设计做了很多的处理。`BeanFactory`只提供获取方法，注册，初始化等操作都交给后面的来完成。
```java
public interface BeanFactory {

    Object getBean(String name);

}
```
#### 2、抽象类定义模板方法
`AbstractBeanFactory`实现 使用 `模板方法`设计模式来 **实现**`BeanFactory#getBean`方法，在里面定义模板。然后抽象出模板的步骤，没一个步骤交接具体的实现类来完成。在这里做统一的闸口，方便管理。
通过 **继承** `DefaultSingletonBeanRegistry`来获得 获取单例bean的能力，这样既可以使用获取单例bean的方法，又不需要自己我处理。
```java
/**
 * @author huangle
 * @date 2023/2/6 18:01
 */
public abstract class AbstractBeanFactory extends DefaultSingletonBeanRegistry implements BeanFactory {


    public Object getBean(String name) {
        Object singleton = getSingleton(name);
        if (singleton != null){
            return singleton;
        }
        BeanDefinition beanDefinition = getBeanDefinition(name);
        return createBean(name,beanDefinition);
    }

    public abstract BeanDefinition getBeanDefinition(String beanName);

    public abstract Object createBean(String beanName,BeanDefinition beanDefinition);


}
```
后面的 `getBeanDefinition` 和 `createBean`分别由不同的子类来实现，因为两者共性不多，如果放在一起实现，会导致整个类变得很乱。而且使用 抽象类去继承类，这样的话，可以不用全部实现，只需要选择自己需要的实现即可。
#### 3、单例bean注册接口定义与实现
单例bean注册、与实现接口，提供单例bean的注册与获取能力
```java
public interface SingletonBeanRegistry {

    Object getSingleton(String name);

}

public abstract class DefaultSingletonBeanRegistry implements SingletonBeanRegistry{

    Map<String,Object> singletonBeanMap = new HashMap<String, Object>();

    public Object getSingleton(String name) {
        return singletonBeanMap.get(name);
    }

    protected void registerSingleton(String name,Object bean){
        singletonBeanMap.put(name, bean);
    }
}
```
#### 4、实例化bean 类
实现`createBean`方法的具体逻辑
```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory{

    @Override
    public Object createBean(String beanName, BeanDefinition beanDefinition) {
        Object instance = null;
        try {
            instance = beanDefinition.getBeanClass().newInstance();
        }catch (Exception e){
            throw new RuntimeException("bean create error!");
        }
        // 实例化完成后，注册到单例bean缓存中
        registerSingleton(beanName,instance);
        return instance;
    }

}

```
#### 5、核心类，对外暴露使用
完成对 `BeanDefinition` 获取与注册。
```java
public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory implements BeanDefinitionRegistry{

    Map<String,BeanDefinition> beanDefinitionMap = new HashMap<String, BeanDefinition>();

    public BeanDefinition getBeanDefinition(String beanName) {
        return beanDefinitionMap.get(beanName);
    }

    public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition) {
        beanDefinitionMap.put(beanName,beanDefinition);
    }
}
```
### 测试
定义一个service类，注册到 bean工厂中，然后从bean工厂中获取 bean
```java
/**
 * @author huangle
 * @date 2023/2/7 10:25
 */
public class UserService {

    public String say(){
        return "hello";
    }

}


@Test
    public void test(){
        DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
        BeanDefinition beanDefinition = new BeanDefinition(UserService.class);
        beanFactory.registerBeanDefinition("userService",beanDefinition);
        UserService userService = (UserService) beanFactory.getBean("userService");
        UserService userService1 = (UserService) beanFactory.getBean("userService");
        System.out.println(userService1.equals(userService));
        System.out.println(userService1.say());
        System.out.println(userService.say());
    }
```
![image.png](https://cdn.nlark.com/yuque/0/2023/png/2869098/1675748600396-0c933182-b7e8-4ac7-89cd-d0a7c77153ce.png#averageHue=%23292e36&clientId=u5870bb51-cb2d-4&from=paste&height=147&id=u7a46fbb7&name=image.png&originHeight=147&originWidth=685&originalType=binary&ratio=1&rotation=0&showTitle=false&size=9675&status=done&style=none&taskId=ue3527c7c-039f-4ae4-9fe9-55a184c1de6&title=&width=685)
