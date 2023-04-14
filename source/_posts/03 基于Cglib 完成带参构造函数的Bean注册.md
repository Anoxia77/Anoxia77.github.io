## 前言
Spring 提供了多种方式去完成一个 bean 的初始化。前面我们已经实现了通过 `Class.newInstance()`方法来创建一个bean，但是这种方式只能创建无参构造的bean，但是实际项目中，一般的bean都是带参数的，所以我们要实现一个根据带参构造来创建bean的方法。这个时候我们就需要选择一种新的方式来实现 bean 的创建。在上一节有提到 通过 cglib 的方式来创建bean。
![image.png](https://cdn.nlark.com/yuque/0/2023/png/2869098/1675757601167-c432c664-8a09-4607-b054-feda3e153447.png#averageHue=%2395fa33&clientId=u635e9ef9-17cd-4&from=paste&height=317&id=udc7c3177&name=image.png&originHeight=317&originWidth=718&originalType=binary&ratio=1&rotation=0&showTitle=false&size=21841&status=done&style=none&taskId=u5ea93ff3-e399-4419-8755-345e2bbaf7a&title=&width=718)
## 实现
#### 1、添加带参的getBean方法
```java
public interface BeanFactory {

    Object getBean(String name);

    Object getBean(String name, Object... args);
}
```
#### 2、重新实现
这里直接目前直接使用 cglib 的`CglibSubclassingInstantiationStrategy` 动态代理来完成创建。
后面使用**工厂+策略模式 **把这里替换掉，简单了解一下 `工厂模式`和 `策略模式`。
> **工厂模式**：工厂模式（Factory Pattern）是一种创建型设计模式，它提供了一种创建对象的最佳方式。在工厂模式中，我们定义一个接口或抽象类，该接口或抽象类用于创建对象，但不会指定要创建哪个类的实例。
> **策略模式**：策略模式（Strategy Pattern）是一种行为型设计模式，它允许在运行时选择算法的行为。在策略模式中，我们定义一组算法，将每个算法封装在一个独立的类中，并使它们可以互相替换。这使得算法的选择不仅取决于调用方，还可以在运行时动态地更改。

```java
/**
 * @author huangle
 * @date 2023/2/7 9:37
 */
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory{

    private CglibSubclassingInstantiationStrategy instantiationStrategy = new CglibSubclassingInstantiationStrategy();

    @Override
    public Object createBean(String beanName, BeanDefinition beanDefinition,Object[] args) {
        Object instance = null;
        try {
            instance = createBeanInstance(beanDefinition,beanName,args);
        }catch (Exception e){
            throw new RuntimeException("bean create error!",e);
        }
        registerSingleton(beanName,instance);
        return instance;
    }


    private Object createBeanInstance(BeanDefinition beanDefinition,String beanName,Object[] args){
        Constructor<?> userCons = null;
        Class<?> clazz = beanDefinition.getBeanClass();
        Constructor<?>[] declaredConstructors = clazz.getDeclaredConstructors();
        for (Constructor<?> constructor : declaredConstructors) {
            if (args != null && constructor.getParameterTypes().length == args.length){
                userCons = constructor;
                break;
            }
        }
        return instantiationStrategy.instantiate(beanName,beanDefinition,userCons,args);
    }


}

```
#### 3、定义instance策略接口
定义 初始化 策略接口，统一初始化行为。通过 接口 抽象统一创建行为，具体的创建过程由各自 `实现类` 完成。在后期，如果通过 工厂 管理这些不同的实现类的时候，可以通过这个接口拿到所有的实现类，统一做管理
```java
/**
 * 初始化bean策略接口
 */
public interface InstantiationStrategy {

    Object instantiate(String beanName, BeanDefinition beanDefinition, Constructor<?> constructor, Object[] args);

}
```
#### 4、实现
##### JDK 默认的实现
这里主要的方式还是通过class获取到bean的构造函数来实现bean的创建
```java
public class SimpleInstantiationStrategy implements InstantiationStrategy{

    public Object instantiate(String beanName, BeanDefinition beanDefinition, Constructor<?> constructor, Object[] args) {
        Class<?> clazz = beanDefinition.getBeanClass();
        try {
            if (constructor != null){
               return clazz.getDeclaredConstructor(constructor.getParameterTypes()).newInstance(args);
            }else {
               return clazz.getDeclaredConstructor().newInstance();
            }
        }catch (Exception e){
            throw new RuntimeException("JDK create bean error,failed to create ["+clazz.getName()+"]!");
        }
    }
}
```
##### Cglib 的实现
通过cglib来完成 bean 的创建
```java
public class CglibSubclassingInstantiationStrategy implements InstantiationStrategy {
    public Object instantiate(String beanName, BeanDefinition beanDefinition, Constructor<?> constructor, Object[] args) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(beanDefinition.getBeanClass());
        enhancer.setCallback(new NoOp() {
            @Override
            public int hashCode() {
                return super.hashCode();
            }
        });
        if (constructor == null) {
            return enhancer.create();
        }
        return enhancer.create(constructor.getParameterTypes(), args);
    }
}

```
## 测试
```java
public class UserService {

    private String name;

    public UserService(String name) {
        this.name = name;
    }

    public String say(){
        return this.name+",hello";
    }

}


public class DefaultListableBeanFactoryTest {

    @Test
    public void test(){
        DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
        BeanDefinition beanDefinition = new BeanDefinition(UserService.class);
        beanFactory.registerBeanDefinition("userService",beanDefinition);
        UserService userService = (UserService) beanFactory.getBean("userService","Anoxia");
        System.out.println(userService.say());
    }

}
```
![image.png](https://cdn.nlark.com/yuque/0/2023/png/2869098/1675759002107-7e047277-58a1-48ef-8f6b-c60437e2e5d6.png#averageHue=%232b313b&clientId=u635e9ef9-17cd-4&from=paste&height=122&id=uaaf7f2bb&name=image.png&originHeight=122&originWidth=999&originalType=binary&ratio=1&rotation=0&showTitle=false&size=21739&status=done&style=none&taskId=u1246d26e-a5a1-4bea-af5e-14f2eacc90a&title=&width=999)
## 总结
本次就是在创建bean的过程中，为了满足不同的场景，而添加不同 的实现方式，很多时候，眼下的需求可能很简单，实现起来也比较简单，但是随着项目的迭代，慢慢的需求变的越来越复杂，这个时候，如果没有原本预留的设计，可能会变得很难进行下去，写代码不是 一堆一堆 叠上去，而是一块一块组装起来。
