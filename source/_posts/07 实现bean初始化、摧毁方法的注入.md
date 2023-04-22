---
title: 07 实现 bean 初始化、摧毁方法的配置与处理
---



### 前言

spring支持我们自定义 bean 的初始化方法和摧毁方法。配置方式可以通过 xml 的 `init-method` 和 `destory-method`配置，或者实现 `InitializingBean`、`DisposableBean`接口，来完成自定义的初始化和bean的销毁。
在项目开发过程中，相信最多看到的是 `@PostConstruct` 注解标识的方法来进行bean的初始化。
> @PostConstruct 是 Spring Framework 提供的注解，可以用于在 Bean 实例化之后执行初始化操作

### 实现
#### 通过xml配置定义初始化、摧毁方法
BeanDefinition 里面添加 initMethodName、和 destoryMethodName 属性，来记录通过配置注入的初始化和摧毁方法名称。然后在解析 xml 文件的 `cn.anoxia.springframework.beans.factory.xml.XmlBeanDefinitionReader#doLoadBeanDefinitions`方法中，完成 属性的注入。
```java
protected void doLoadBeanDefinitions(InputStream inputStream) throws Exception {
        Document doc = XmlUtil.readXML(inputStream);
        Element root = doc.getDocumentElement();
        NodeList childNodes = root.getChildNodes();
        for (int i = 0; i < childNodes.getLength(); i++) {

            ....
            String initMethod = bean.getAttribute("init-method");
            String destroyMethod = bean.getAttribute("destroy-method");

            Class<?> clazz = Class.forName(calssName);
            String beanName = StrUtil.isNotEmpty(id) ? id : name;
            if (StrUtil.isEmpty(beanName)){
                beanName = StrUtil.lowerFirst(clazz.getSimpleName());
            }

            // 定义bean
            BeanDefinition beanDefinition = new BeanDefinition(clazz);
            // 设置初始化、摧毁方法
            beanDefinition.setInitMethodName(initMethod);
            beanDefinition.setDestoryMethodName(destroyMethod);
            ....

        }
    }
```
#### 通过实现接口
实现 InitializingBean，DisposableBean 并实现里面的方法，来自定义bean的初始化和摧毁方法
```java
public interface InitializingBean {

    void afterPropertiesSet() throws Exception;

}

public interface DisposableBean {

    void destroy() throws Exception;

}
```
在 创建bean的过程中，完成方法的注入，区分xml配置与接口实现。
#### initMethod 方法的注入与执行
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

        // 1. 执行 BeanPostProcess before 操作
        Object wrappedBean = applyBeanPostProcessorsBeforeInitialization(bean, beanName);

        try {
            // 执行bean初始化方法
            invokeInitMethods(beanName,wrappedBean,beanDefinition);
        }catch (Exception e){
            throw new BeansException("Invocation of init method of bean[" + beanName + "] failed", e);
        }

        // 2. 执行 BeanPostProcess after 操作
        wrappedBean = applyBeanPostProcessorsAfterInitialization(bean,beanName);

        return wrappedBean;
    }


private void invokeInitMethods(String beanName, Object bean, BeanDefinition beanDefinition) throws Exception {
    // 如果是通过接口实现，直接使用接口提供的方法    
    if (bean instanceof InitializingBean) {
            ((InitializingBean) bean).afterPropertiesSet();
        }
    // 通过xml配置，获取方法执行
        String initMethodName = beanDefinition.getInitMethodName();
        if (StrUtil.isNotEmpty(initMethodName)) {
            Method method = beanDefinition.getBeanClass().getMethod(initMethodName);
            // getMethod 已经做了非空判断
            if (null == method) {
                throw new BeansException("Could not find an init method named '" + initMethodName + "' on bean with name '" + beanName + "'");
            }
            method.invoke(bean);
        }
    }
```
#### destroyMethod 方法的注入与执行
提供一个适配器、来完成xml和接口的适配处理。处理逻辑基本与 init方法相似
```java
/**
 * bean 摧毁适配器
 * @author huangle
 * @date 2023/3/7 10:26
 */
public class DisposableBeanAdapter implements DisposableBean {

    /**
     * bean名字
     */
    private final String beanName;

    /**
     * bean
     */
    private final Object bean;

    /**
     * 销毁方法名称
     */
    private String destroyMethodName;

    public DisposableBeanAdapter(String beanName, Object bean, BeanDefinition beanDefinition) {
        this.beanName = beanName;
        this.bean = bean;
        this.destroyMethodName = beanDefinition.getDestoryMethodName();
    }

    @Override
    public void destroy() throws Exception {

        // 1. 实现 DisposableBean 接口，完成摧毁扩展
        if (bean instanceof DisposableBean) {
            ((DisposableBean) bean).destroy();
        }
        // 2. 通过xml配置 配置 destroy 方法 实现
        if (StrUtil.isNotEmpty(destroyMethodName) && !(bean instanceof DisposableBean && "destory".equals(destroyMethodName))) {
            Method destroyMethod = bean.getClass().getMethod(destroyMethodName);
            if (null == destroyMethod) {
                throw new BeansException("Couldn't find a destroy method named '" + destroyMethodName + "' on bean with name '" + beanName + "'");
            }
            destroyMethod.invoke(bean);
        }

    }
}

```
### 测试
userDao 通过xml配置初始化和摧毁方法，userService 通过继承接口来实现方法。
```java
<?xml version="1.0" encoding="UTF-8"?>
<beans>

    <bean id="userDao" class="cn.anoxia.springframework.beans.factory.support.UserDao" init-method="initMethod" destroy-method="destroyMethod"/>

    <bean id="userService" class="cn.anoxia.springframework.beans.factory.support.UserService">
        <property name="name" value="Anoxia"/>
        <property name="nickname" value="迪迦"/>
        <property name="userDao" ref="userDao"/>
    </bean>

<!--    <bean id="myBeanPostProcecssor" class="cn.anoxia.springframework.beans.factory.support.MyBeanPostProcecssor"/>-->
<!--    <bean id="myFactoryPostProcessor" class="cn.anoxia.springframework.beans.factory.support.MyBeanFactoryPostProcessor"/>-->

</beans>

public class UserService implements InitializingBean, DisposableBean{

    @Override
    public void destroy() throws Exception {
        System.out.println("userService执行：destroy 方法");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("userService执行：init 方法");
    }
    
}

```
测试结果
![image.png](https://cdn.nlark.com/yuque/0/2023/png/2869098/1678267751188-e61202ee-15f2-4914-87f9-662fe8e06a97.png#averageHue=%23282d36&clientId=u49361cf8-64f9-4&from=paste&height=223&id=u8985ba71&name=image.png&originHeight=223&originWidth=929&originalType=binary&ratio=1&rotation=0&showTitle=false&size=31775&status=done&style=none&taskId=u1ddcfc6b-7217-421f-b0c4-331c2f20482&title=&width=929)
