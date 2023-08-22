---
title: Jackson自定义序列化器--实现全局名称替换
---

![](https://c.wallhere.com/images/2e/1f/20e966ba29ae977be591d318c28d-1494607.jpg!s1)

## 前言
有这样一个需求，系统中存在两套用户身份信息（后面用A(先在用的)，B（新的））来区分这两套身份系统。
> 需求的要求是，如果 B 里面存在`昵称`，就要把 A 中的`昵称`给替换掉。

A 中的昵称在系统中使用的比较散乱，所以如果需要改的话，需要入侵到业务里面，去把所有用到的地方改掉。
简单找了一下系统中用到的地方，发现基本上很多地方都有用，如果需要改的话，耗时会比较大，而且产品说这种信息可能不止一套，后面可能还有其他的 身份信息，还需要替换。所以直接入侵到业务代码里面的话，对代码质量的影响比较大，而且需要耗费大量的时间。
## 问题
> 1、历史遗留问题、业务没有统一，使用都是零散的
> 2、需要考虑到以后的扩展问题
> 3、对现有代码质量的影响问题

## 设计
考虑到不能直接入侵到代码里面去修改业务逻辑，首先想到的是切面。但是切面加到哪里是一个问题，怎么加又是一个问题。
由于我们需要替换掉 A 里面的 昵称，所以我们只需要关注最后面返回的时候，A 里面的 昵称 能不能被换成 B 里面的昵称。所以最后面选择的就是 在返回之前，对返回值做统一处理。在进行序列化的时候，完成替换。
## 实现
实现自己的序列化方式，在序列化的时候，统一处理。
我们可以通过 继承 `com.fasterxml.jackson.databind.JsonSerializer` 类，实现 `serialize`方法，来实现自己的序列化方式，然后通过 `ApplicationContextAware` 接口，在spring容器启动的时候，把我们自己的处理类注入到spring的容器中。
```java

/**
 * @author huangle
 * @date 2023/3/2 14:26
 */
@Slf4j
@Component
public class NameSerializer extends JsonSerializer<String> implements ApplicationContextAware {

    private static volatile INameSerializerService nameSerializerService = (name, adapter) -> name;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        try {
            INameSerializerService serializerService = applicationContext.getBean(INameSerializerService.class);
            log.info("获取到INameSerializerService 的实现类！");
            NameSerializer.nameSerializerService = serializerService;
        }catch (NoSuchBeanDefinitionException e) {
            log.info("使用INameSerializerService默认实现");
        }
    }

    @Override
    public void serialize(String s, JsonGenerator json, SerializerProvider serializerProvider) throws IOException {
        if (StringUtils.isEmpty(s)) {
            json.writeNull();
            return;
        }
        try {
            json.writeString(nameSerializerService.replaceName(s, (NameSerializerAdapter) json.getOutputContext().getCurrentValue()));
        }catch (Exception e) {
            log.info("出现异常，把原来的写进去！");
            json.writeString(s);
        }
    }
}

```
定义自己的实现逻辑,定义接口，后面扩展方便，这里有一个默认实现的操作
`private static volatile INameSerializerService nameSerializerService = (name, adapter) -> name;` 如果 没有类去实现这个接口，那么就使用这里的默认实现，默认实现就是返回他原来的值。
> 这里可以作为一个解决 没有实现类的 思路，来解决 NoSuchBeanDefinitionException 异常

```java

public interface INameSerializerService {
    /**
     * 替换名称
     * @param name 需要替换的值
     * @param adapter 原始对象
     * @return
     */
    String replaceName(String name, NameSerializerAdapter adapter);

}
```
`NameSerializerAdapter` 适配器接口，通过定义一个接口，来让实现类实现自己的逻辑，适配器统一返回
> 这里的思路就是 适配器做统一管理，这样的话需要替换的类就可以实现这个接口，来做一个统一，对于在不同返回体意义相同，但是 字段命名不一样的，还能做统一处理。

```java

public interface NameSerializerAdapter {

    String getName();

}
```
具体的处理逻辑，这个根据自己实际情况来就可以
```java
@Slf4j
@Service
public class NameSerializerService implements INameSerializerService{


    @Override
    public String replaceName(String name, NameSerializerAdapter adapter) {
        log.info("处理需要替换的值：{},替换的对象：{}",name,adapter);
        return adapter.getName();
    }
}
```
使用
```java
@Data
public class UserRes implements NameSerializerAdapter {

    /**
     * 名字
     */
    @JsonSerialize(using = NameSerializer.class)
    private String name;
    /**
     * 编号
     */
    private Long userNo;

}
```
## 总结
上面的实现方案主要体现了一下的一些知识点
1、使用 `ApplicationContextAware`完成bean的注入。
2、使用 接口的默认实现  来解决 `NoSuchBeanDefinitionException`问题
3、使用适配器模式，对业务字段，做统一处理。
