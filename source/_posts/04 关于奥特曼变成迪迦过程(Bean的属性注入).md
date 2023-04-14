## 前言
前面我们完成了通过JDK和Cglib完成带参数的构造函数来完成bean的初始化过程。
接下来就是 给bean 完成 `属性注入`，一般在正常使用过程中，一个bean会有一些自己的基础属性，同时也会引入其他的 bean 来执行一些方法。所以我们在执行注入的时候，就需要区分 是- bean的 `基本类型`，还是 `引用类型`。
Spring 通过对 `引用对象` 做一层简单的包装，来与基础属性分开。类之间的关系如下
![](https://s3.bmp.ovh/imgs/2023/02/08/70005dce24e3e3a7.png#id=yPDft&originHeight=287&originWidth=889&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

## 实现
### 1、包装bean的属性
所有的属性 使用 `PropertyValue`做统一的处理。`BeanReference`封装 引用类型的 bean 。`PropertyValues`对 `PropertyValue` 做一次汇总，方面后面的管理。
这个把两个类型分离开了，有点 `单一职责` 的意思。 相互不影响，单独管理，共同配置组成一个完整的属性。
> **单一职责**：单一职责原则是指一个类应该只有一个引起它变化的原因，即一个类只负责一项职责。这样做可以提高代码的可读性、可维护性和可扩展性，减少了代码的复杂度和耦合度。

```java
/**
 * @author huangle
 * @date 2023/2/8 10:13
 */
public class PropertyValue {

    private final String name;
    private final Object value;

    public PropertyValue(String name, Object value) {
        this.name = name;
        this.value = value;
    }

    public String getName() {
        return name;
    }

    public Object getValue() {
        return value;
    }
}

/**
 * @author huangle
 * @date 2023/2/8 10:12
 */
public class BeanReference {

    private String beanName;

    public BeanReference(String beanName) {
        this.beanName = beanName;
    }

    public String getBeanName() {
        return beanName;
    }
}

/**
 * @author huangle
 * @date 2023/2/8 10:13
 */
public class PropertyValues {

    private List<PropertyValue> propertyValueList = new ArrayList<PropertyValue>();


    public void addPropertyValue(PropertyValue pv) {
        this.propertyValueList.add(pv);
    }


    public PropertyValue[] getPropertyValues() {
        return this.propertyValueList.toArray(new PropertyValue[0]);
    }

    public PropertyValue getPropertyValue(String propertyName) {
        for (PropertyValue propertyValue : propertyValueList) {
            if (propertyValue.getName().equals(propertyName)) {
                return propertyValue;
            }
        }
        return null;
    }


}
```
### 2、注入属性
注入属性是在完成bean创建之后的。如果我们把 创建bean 的过程看做组装一个`奥特曼`,这个时候就应该到了给 `奥特曼`取名字、上颜色、定技能的时候了。当这个阶段完成，他就变成了 `迪迦`（`你相信光吗`）。
> 注：如果这里出现了循环依赖，我们该怎么解决？Spring 又是怎么解决的。欲知后事如何、请看下集 `迪迦的爱恨纠缠`

```java
@Override
    public Object createBean(String beanName, BeanDefinition beanDefinition, Object[] args) {
        Object instance = null;
        try {
            instance = createBeanInstance(beanDefinition, beanName, args);
            // 注入属性
            applyPropertyValues(beanName, instance, beanDefinition);
        } catch (Exception e) {
            throw new RuntimeException("bean create error!", e);
        }
        registerSingleton(beanName, instance);
        return instance;
    }



protected void applyPropertyValues(String beanName, Object bean, BeanDefinition beanDefinition) {
        try {
            // 获取所有的属性，分别注入
            PropertyValues propertyValues = beanDefinition.getPropertyValues();
            for (PropertyValue propertyValue : propertyValues.getPropertyValues()) {
                String name = propertyValue.getName();
                Object value = propertyValue.getValue();
                if (value instanceof BeanReference) {
                    // 依赖其他的bean
                    value = getBean(((BeanReference) value).getBeanName());
                }
                BeanUtil.setFieldValue(bean, name, value);
            }
        } catch (Exception e) {
            throw new RuntimeException("Error setting property values：" + beanName);
        }
    }
```
### 3、测试
测试主要的思路就是要满足基本类型、和 引用类型都能导入
```java
public class UserService {

    private String name;

    private UserDao userDao;

    public UserService(String name) {
        this.name = name;
    }

    public String say(){
        return userDao.getUserName()+name;
    }
}

public class UserDao {

    public String getUserName(){
        return "Anoxia";
    }

}

@Test
public void test(){
        //1 初始化bean工厂
        DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
        //2 注册bean
        BeanDefinition userDaoDefinition = new BeanDefinition(UserDao.class);
        beanFactory.registerBeanDefinition("userDao",userDaoDefinition);
        //3.设置属性
        PropertyValues propertyValues = new PropertyValues();
        propertyValues.addPropertyValue(new PropertyValue("userDao",new BeanReference("userDao")));
        propertyValues.addPropertyValue(new PropertyValue("name","hello"));
        //4.注入属性
        BeanDefinition beanDefinition = new BeanDefinition(UserService.class,propertyValues);
        beanFactory.registerBeanDefinition("userService",beanDefinition);
        //5.获取使用
        UserService userService = (UserService) beanFactory.getBean("userService");
        System.out.println(userService.say());
    }
```
测试结果
![](https://s3.bmp.ovh/imgs/2023/02/08/a57dc430fe32a01e.png#id=mAy43&originHeight=106&originWidth=694&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## 总结
这一节我们完成了 bean 的属性注入、让一个bean拥有了属于自己的属性。在spring里面，spring对 这个一步预留了一个扩展接口 `BeanFactoryPostProcessor`，我们可以通过这个对bean 的属性进行操作。后面的章节中，也会有对 这个扩展接口的介绍。欲知后事如何、请听下回分解。
