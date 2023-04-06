---
title: 重学Spring-01 BeanFactory到底是什么
---
spring 作为管理bean 的创建与使用，从表面意思来说，可以把spring简单的认为是一个容器，就跟ArrayList、HashMap 一样的容器。<br />在项目中，我们一般会通过 `@Autowired`和 `@Resource`来引用spring中的bean

- autowired 由spring 提供，注解使用的是 通过 className 从spring容器中来找到这个bean
- resource 是 java 提供的，就是说就算不算spring的环境，他也是可以用的，他的主要能力是通过 className 去找bean，如果className 找不到，还会通过 class 类 去找这个 bean，通过class 去找bean是 `@Autowired` 注解所不具备的能力。
### 1、BeanFactory
spring bean 工厂，用来管理所有的bean，spring通过把所有的bean对象拆解，组合，最后面生成一个bean对象存放在spring容器中。<br />![](https://s3.bmp.ovh/imgs/2023/02/07/4713bfb9091b69f0.jpg)
### 2、BeanDefinition
beanDefinition 记录着 bean的基本信息，比如 className，prototype，参数，方法等一些信息，在spring容器中所有的bean都可以被拆分到beandefinition中。spring通过beanDefinition把bean创建出来。（beanDefinition就相当于没有组装的积木包，最后面组装成功后，变成一个完整的积木）<br />BeanFactory、BeanDefinition、Bean 三者的包含关系<br />![](https://s3.bmp.ovh/imgs/2023/02/07/bb74b48a0fe7a83c.jpg)
### 3、设计与实现
上面我们知道了 BeanFactory、BeanDefinition、Bean 之间的联系，和含义，下面我们根据他这个意思去实现他。
#### BeanFactory
首先是beanfactory，我们知道个作为一个工厂，是用来存贮bean对象的。所以他提供的方法有 get和 注册bean
```java
/**
 * @author huangle
 * @date 2023/2/6 15:41
 */
public class BeanFactory {

    private Map<String,BeanDefinition> beanDefinitionMap = new HashMap<String, BeanDefinition>();

    public BeanDefinition getBean(String className){
        return beanDefinitionMap.get(className);
    }

    public void registerBean(String className,BeanDefinition beanDefinition){
        beanDefinitionMap.put(className,beanDefinition);
    }

}
```
#### BeanDefinition
BeanDefinition 就是一个相当于 组装一个bean需要的工具包，关于这个bean所以的信息都会在这个里面有保存。相当于在加一层外壳
```java
/**
 * @author huangle
 * @date 2023/2/6 15:41
 */
public class BeanDefinition {

    private Object bean;

    public BeanDefinition(Object bean) {
        this.bean = bean;
    }

    public Object getBean() {
        return bean;
    }
}
```
### 4、测试
```java
public class BeanFactoryTest {


    @Test
    public void test(){
        BeanFactory beanFactory = new BeanFactory();
        BeanDefinition definition = new BeanDefinition(new User("anoxia"));
        beanFactory.registerBean("user",definition);
        System.out.println(beanFactory.getBean("user").getBean().toString());
    }

}
```
测试结果<br />![](https://s3.bmp.ovh/imgs/2023/02/08/1cf6b024e6fa65ee.png)
