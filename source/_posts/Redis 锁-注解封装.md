---
title: Redis锁-注解封装
---
### 前言
简单封装redis的锁，在使用上提升体验度，对于一些简单的业务，可以直接使用注解封装锁，来控制并发。
### 设计
一般来说，使用redis分布式锁，就是对一个key进行setnx，或者检查key是否已经存在，如果存在表示已经加锁。
所以我们首先梳理一下 key 的结构，key 通常都是 使用 `业务描述` + `唯一业务字段` 来唯一性确定一个key，例如这样 key = `redisLock:orderShop:1000000001`
把key 简单的分为三段

-  reidsLock 表示这是一个锁，
- orderShop 表示业务场景，
- 100000001 表示具体的某个订单。

对于系统来说 第一个一般是相同的，因为设计可以方便后面找这个key。所以我们主要需要关注的就是后面两个值，业务场景一般对于一种场景来说都是一样是。所以这个我们可以通过一个值传进去。后面的具体业务字段就不一定了，在同一个业务中，可以需要不同的业务字段来做为锁，所以这里需要在进行变化一下。需要允许多个 业务字段来做为锁。
一般来说，我们锁的都是方法，或者说是方法里面的某一些执行逻辑，对于没有事务的业务来说，我们可以通过这种方法简单的控制。（如果有事务的话，可能就要考虑锁粒度的问题，因为大锁可能会导致事务过长）
### 实现
前面我们提到，锁的话主要都是作用在方法层面的，所以我们可以给方法加一个注解。对于业务参数来说，他就是方法的参数，那么我们可以考虑给方法参数也加一个注解来注入值。
使用的Maven包
```java
<dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-aspects</artifactId>
</dependency>
<dependency>
      <groupId>org.redisson</groupId>
      <artifactId>redisson-spring-boot-starter</artifactId>
      <version>3.17.7</version>
</dependency>
```
#### 方法注解
注解主要的参数就是一些常用的参数，可以根据自己的需要加添加或删除
```java

/**
 * redis 锁注解
 * @author huangle
 */
@Target(ElementType.METHOD)
@Documented
@Retention(RetentionPolicy.RUNTIME)
public @interface RedisLock {

    /**
     * 锁前缀
     */
    String prefix() default "redis:Lock";

    /**
     * 加锁时间
     */
    long lockTime() default 30;

    boolean tryLock() default true;

    /**
     * 加锁不成功的等待时间 -1 不等待
     */
    long waitTime() default -1;

    /**
     * 锁时间类型
     */
    TimeUnit timeUnit() default TimeUnit.MILLISECONDS;



}

```
#### 字段注解
这个注解主要的作用就是帮助我们注入可变的业务参数，我们直接添加到方法参数上面，来获取到他的值
```java
/**
 * 锁字段注解
 * 用于生成 key
 * @author huangle
 */
@Target(ElementType.PARAMETER)
@Documented
@Retention(RetentionPolicy.RUNTIME)
public @interface LockField {

    /**
     * 锁标识字段，没有则取当前参数值，有则按顺序反射取参数对象字段值
     * @return
     */
    String[] values() default {};

}

```
#### 注解切面
注解切面，使用aop的思想，来对注解进行拦截，然后加锁，使用的是 redssion 来进行的加锁操作
```java
/**
 * @author huangle
 * @date 2022/12/8 9:54
 */
@Slf4j
@Aspect
@Component
public class RedisLockAspect {

    @Resource
    private RedissonClient redissonClient;

    /**
     * key 格式 REDIS_LOCK_PREFIX + SEPARATOR + prefix + SEPARATOR + field/fieldName
     */
    private static final String REDIS_LOCK_PREFIX = "anoxiaLock";

    private static final String SEPARATOR = ":";

    @Pointcut("@annotation(cn.anoxia.redis.common.annotation.RedisLock)")
    public void doAspect() {
    }


    @Around("doAspect()")
    public Object doLockAround(ProceedingJoinPoint point) throws Throwable {
        // 获取方法
        MethodSignature methodSignature = (MethodSignature) point.getSignature();
        Method method = methodSignature.getMethod();
        // 获取注解
        RedisLock redisLock = method.getAnnotation(RedisLock.class);
        // 获取前缀
        String prefix = redisLock.prefix();
        // 获取字段
        Parameter[] parameters = method.getParameters();
        // 构建key
        StringBuilder lockKeyStr = new StringBuilder(prefix);
        Object[] args = point.getArgs();
        int index = -1;
        LockField lockField;
        // 遍历所有字段
        for (Parameter parameter : parameters) {
            Object arg = args[++index];
            // 获取注解
            lockField = parameter.getAnnotation(LockField.class);
            if (lockField == null) {
                continue;
            }
            String[] values = lockField.values();
            if (values == null || values.length == 0) {
                lockKeyStr.append(SEPARATOR).append(arg);
            } else {
                List<Object> filedValues = ReflectionUtil.getFiledValues(parameter.getType(), arg, values);
                for (Object filedValue : filedValues) {
                    lockKeyStr.append(SEPARATOR).append(filedValue);
                }
            }
        }
        // 获取锁
        String key = REDIS_LOCK_PREFIX + SEPARATOR + lockKeyStr.toString();
        RLock lock = redissonClient.getLock(key);
        // 加锁成功标志位，用于释放锁
        boolean lockFlag = false;
        // 尝试加锁
        try {
            long lockTime = redisLock.lockTime();
            TimeUnit timeUnit = redisLock.timeUnit();
            long waitTime = redisLock.waitTime();
            boolean tryLock = redisLock.tryLock();
            try {
                if (tryLock) {
                    if (waitTime > 0) {
                        // 尝试加锁，没有成功就等待，waitTime 等待时间，如果超时就直接放弃
                        lockFlag = lock.tryLock(waitTime, lockTime, timeUnit);
                    } else {
                        // 直接不等待，如果加锁失败就返回
                        lockFlag = lock.tryLock(-1,lockTime,timeUnit);
                    }
                } else {
                    lock.lock(lockTime, timeUnit);
                    lockFlag = true;
                }
            } catch (Exception e) {
                log.error("加锁失败！，错误信息：{}", e);
                throw new RuntimeException("加锁失败！");
            }
            if (!lockFlag){
                throw new RuntimeException("加锁失败！");
            }
            // 加锁结束
            // 保证业务逻辑已经完成，在释放锁
            Object proceed = point.proceed();
            return proceed;
        } finally {
            if (lockFlag) {
                log.info("释放锁,key:{}", key);
                lock.unlock();
            }
        }
    }

}

```
使用的到的工具类，这个可以使用其他的也可以，只要可以实现就行
```java
/**
 * @author huangle
 * @date 2022/12/8 10:26
 */
public class ReflectionUtil {

    public static List<Object> getFiledValues(Class<?> type, Object target, String[] fieldNames) throws IllegalAccessException {
        List<Field> fields = getFields(type, fieldNames);
        List<Object> valueList = new ArrayList();
        Iterator fieldIterator = fields.iterator();

        while(fieldIterator.hasNext()) {
            Field field = (Field)fieldIterator.next();
            if (!field.isAccessible()) {
                field.setAccessible(true);
            }

            Object value = field.get(target);
            valueList.add(value);
        }

        return valueList;
    }


    public static List<Field> getFields(Class<?> claszz, String[] fieldNames) {
        if (fieldNames != null && fieldNames.length != 0) {
            List<String> needFieldList = Arrays.asList(fieldNames);
            List<Field> matchFieldList = new ArrayList();
            List<Field> fields = getAllField(claszz);
            Iterator fieldIterator = fields.iterator();

            while(fieldIterator.hasNext()) {
                Field field = (Field)fieldIterator.next();
                if (needFieldList.contains(field.getName())) {
                    matchFieldList.add(field);
                }
            }

            return matchFieldList;
        } else {
            return Collections.EMPTY_LIST;
        }
    }

    public static List<Field> getAllField(Class<?> claszz) {
        if (claszz == null) {
            return Collections.EMPTY_LIST;
        } else {
            List<Field> list = new ArrayList();

            do {
                Field[] array = claszz.getDeclaredFields();
                list.addAll(Arrays.asList(array));
                claszz = claszz.getSuperclass();
            } while(claszz != null && claszz != Object.class);

            return list;
        }
    }

}

```
### 测试
把锁注解，直接添加到方法上，锁参数注解，添加到方法参数里面，来获取到值
```java
/**
 * @author huangle
 * @date 2022/12/8 14:35
 */
@RestController
@RequestMapping("/user")
public class UserController {


    @RedisLock(prefix = "user",lockTime = 1,timeUnit = TimeUnit.MINUTES)
    @GetMapping("/v1/getUser/{userId}")
    public Map<String,Object> getUser(@PathVariable @LockField Long userId){
        Map<String,Object> user = new HashMap<>();
        user.put("name","anoxia");
        user.put("userId",userId);
        return user;
    }

}
```
