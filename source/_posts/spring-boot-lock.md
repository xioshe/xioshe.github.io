---
title: 如何在 Spring Boot 中更简便地加锁
date: 2024-11-22 21:54:17
tags:
- Spring Boot
- 锁
- 并发
categories:
- 技术实践
---

本文将介绍一种锁的封装方式，用于简化项目中的加锁操作。特点是同时支持 JVM 锁和分布式锁，可以根据需要自由选择。我们的口号是：让项目中没有难用的锁～

## 使用效果

针对不同使用场景，提供了声明式风格和编程式风格两种操作方式。

声明式风格使用注解使用，使用简单，直接在方法上添加注解 `@WithLock` 即可。

```java
// 指定锁的 key，支持 SpEL 表达式
@WithLock(key = "'test:' + #key", waitTime = 200, leaseTime = 500, fair = true)
public String testMethod(String key) {
    // do something
    return "success";
}
```

因为基于 Spring AOP，所以也会受到 Spring AOP 的限制，比如只能在 Spring Bean 中使用，不能在私有方法中使用，不能用在 final 方法， 不能在静态方法使用，不能同类方法调用等。

编程式风格使用模板方法，需要先获取 `LockTemplate` 实例，然后调用其 `execute` 方法。

```java
LockInfo lockInfo = LockInfo.builder()
        .key("test:" + key)
        .waitTime(200)
        .leaseTime(500)
        .timeUnit(TimeUnit.MILLISECONDS)
        .fair(true)
        .build();
lockTemplate.execute(lockInfo, () -> {
    // do something
    return "success";
});
```

不难看出，设计上借鉴了 Spring 事务的思路，通过声明式风格提供简便的粗粒度的锁控制，通过编程式风格提供复杂的细粒度的加锁操作。

作为题外话，Spring 充满了这种设计，声明式风格和编程式风格结合，比如 Spring Data 系列中，各种 `Repository` 接口代表了声明式风格，各种 `Template` 代表了编程式风格，通常将编程式作为声明式的补充。

另一个功能是可配置的锁实现方式，默认使用本地锁，同时支持 Redis 分布式锁，可以根据需求自由切换。

## 配置方式

基于 Spring Boot 的自动装配机制，通过 `@EnableLocking` 注解开启锁支持。同时，提供了 `LockProperties` 配置类，用于配置锁的默认参数。

```yaml
app:
  lock:
    enabled: true # 与 @EnableLocking 注解功能一致
    type: local # 锁类型，local/redis
    wait-time: 200 # 等待锁的时间
    lease-time: 0 # 锁的持有时间，JVM 锁不支持这个参数，Redis 锁中 0 表示开启自动续期
    default-fair: true # 是否默认为公平锁，大多数时候 false 就好
    key-store-prefix: ‘lock:‘ # 锁的 key 存储前缀，针对 Redis 锁有效

```

## 实现原理

让我们先看看代码结构。

```plain
├── annotation
│   ├── EnableLocking.java     # 开启锁支持
│   └── WithLock.java          # 方法级锁注解
├── aspect
│   └── LockAspect.java        # AOP 切面，用于支持 @WithLock 注解
├── config
│   ├── LockAutoConfiguration.java    # 自动装配类
│   ├── LockingEnabledCondition.java  # 启用配置类，用于支持 @EnableLocking 注解
│   └── LockProperties.java           # 配置类
├── core
│   ├── LocalLockExecutor.java        # JVM 锁实现类
│   ├── LockExecutor.java             # 加锁操作接口
│   ├── LockInfo.java
│   └── RedisLockExecutor.java        # Redis 锁实现类，使用 Redisson 实现
├── exception
│   └── LockException.java
├── template
│   └── LockTemplate.java             # 编程式模板
└── util
    └── SpELUtil.java                 # SpEL 表达式工具类
```

下面是核心组件。

- `LockExecutor` 接口，实现加锁功能，也是多种所实现的关键
- `LockTemplate` 定义了加锁命令的范式，内部调用 `LockExecutor` 实现加锁
- `LockAspect` Spring AOP 切面，用于支持 `@WithLock` 注解，内部调用 `LockTemplate`

### 实现加锁功能

LockExecutor 接口比较简单。

```java
public interface LockExecutor {
    boolean tryLock(LockInfo lockInfo);
    void unlock(LockInfo lockInfo);
}
```

首先提供本地锁的实现。使用 juc 包中的 ReentrantLock 类，低并发性能比 synchronized 差，但编程更灵活。由于要关联 key 和锁，使用 ConcurrentHashMap 缓存锁对象。

```java
public class LocalLockExecutor implements LockExecutor {
    private final ConcurrentHashMap<String, LockHolder> lockMap = new ConcurrentHashMap<>();

    @Override
    public boolean tryLock(LockInfo lockInfo) {
        // 省略校验代码

        LockHolder holder = lockMap.compute(lockInfo.getKey(), (key, existingHolder) -> {
            if (existingHolder != null) {
                // 更新访问时间，避免锁对象被清理
                existingHolder.updateLastAccessTime();
                return existingHolder;
            }
            // LockHolder 是一个包装类，内部持有 ReentrantLock 对象，还额外记录了 accessTime，用于清理锁对象，避免内存泄露
            return new LockHolder(new ReentrantLock(lockInfo.isFair()), lockInfo.isFair());
        });

        try {
            return holder.getLock().tryLock(lockInfo.getWaitTime(), lockInfo.getTimeUnit());
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return false;
        }
    }

    @Override
    public void unlock(LockInfo lockInfo) {
        // 省略校验代码

        lockMap.computeIfPresent(lockInfo.getKey(), (key, holder) -> {
            if (holder.getLock().isHeldByCurrentThread()) {
                holder.getLock().unlock();
                // 为了复用锁对象，只有在没有等待线程且超过空闲时间时才移除
                if (!holder.getLock().hasQueuedThreads() && holder.isExpired()) {
                    return null; // 返回 null 会让 Map 移除该 entry
                }
            }
            return holder;
        });
    }
}
```

LockHolder 类的实现。

```java
class LockHolder {
    private final ReentrantLock lock;
    private final long lastAccessTime;
    private final boolean fair;
}
```

`lastAccessTime` 用于实现锁对象的超时清理，避免某个线程长时间占用锁对象，或是忘了释放锁的情况。同时因为 unlock 时并非立即从 Map 移除 holder 对象，取决于当时是否有其他线程在排队，通过自动清理机制作为兜底，可以避免内存泄露。

关键代码如下。

```java
private final ScheduledExecutorService scheduler;

private void startCleanupTask() {
    scheduler.scheduleAtFixedRate(() -> {
        try {
            cleanup();
        } catch (Exception e) {
            log.error("Error during lock cleanup", e);
        }
    }, CLEANUP_INTERVAL, CLEANUP_INTERVAL, TimeUnit.MILLISECONDS); // 1 min
}

private void cleanup() {
    lockMap.entrySet().removeIf(entry -> {
        LockHolder holder = entry.getValue();
        return !holder.getLock().isLocked() &&
                !holder.getLock().hasQueuedThreads() &&
                holder.isExpired();
    });
}
```

接下来是 Redis 分布式锁的实现。直接使用 Redisson 提供的分布式锁，避免重复造轮子。

```java
public class RedisLockExecutor implements LockExecutor {
    private final RedissonClient redissonClient;
    private final String lockPrefix;

    @Override
    public boolean tryLock(LockInfo lockInfo) {
        // 省略校验代码

        RLock lock = getLock(lockInfo);
        try {
            return lock.tryLock(
                    lockInfo.getWaitTime(),
                    lockInfo.getLeaseTime(),
                    lockInfo.getTimeUnit()
            );
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return false;
        }
    }

    @Override
    public void unlock(LockInfo lockInfo) {
        // 省略校验代码

        RLock lock = getLock(lockInfo);
        if (lock.isHeldByCurrentThread()) {
            lock.unlock();
        }
    }

    private RLock getLock(LockInfo lockInfo) {
        String key = lockPrefix + lockInfo.getKey();
        return lockInfo.isFair()
                ? redissonClient.getFairLock(key)
                : redissonClient.getLock(key);
    }
}
```

Redis 自带超时机制，不需要手动清理。

### 公平性的细节

这里涉及到一个细节，是否需要保证同一个 Key 前后加锁的一致公平。简单来说，先用 key-1 申请公平锁加锁成功，锁还没释放时，又用 key-1 申请非公平锁。也可以先申请非公平锁，再申请公平锁，顺序不重要。这里的问题是第二次加锁是否成功。这个问题又包含两种情况。

- 前后两次加锁的线程相同，此时涉及到了锁的可重入性质
- 前后两次加锁的线程不同，此时取决于锁的互斥性

对于 LocalLockExecutor 而言，第二次会从缓存中获取到第一次加锁时创建的锁对象，表现和先后申请同一把锁一致，只有在相同线程第二次加锁才能成功，否则会等待。这很明显有问题，锁的公平性变了，第二次需要的是非公平锁，拿到的却是公平锁。

对于 RedisLockExecutor 而言，内部使用 Redisson 实现。这不存在获取到不同公平性锁的情况，但 Redisson 的公平锁和非公平锁使用的 key 相同，导致二者会互斥，也会共用重入性。具体表现为如果前后两次加锁线程相同，则第二次加锁会成功，否则等待。

两个 LockExecutor 实现类的表现一致，但并非意味着没问题，而是错到一起了。理想情况是，重入时也要比较锁的公平性，只有前后公平性相同时才允许重入。

针对这个要求，对代码进行调整。

首先为 LocalLockExecutor 添加公平性检查。在 LockHolder 中添加 `fair` 属性，在 tryLock 时检查，如果前后两次加锁的公平性不同，则加锁失败返回 false。

```java
@Override
public boolean tryLock(LockInfo lockInfo, boolean strictFair) {
    LockHolder holder = lockMap.compute(lockInfo.getKey(), (key, existingHolder) -> {
        if (existingHolder != null) {
            existingHolder.updateLastAccessTime();
            return existingHolder;
        }
        return new LockHolder(new ReentrantLock(lockInfo.isFair()), lockInfo.isFair());
    });
    // 检查公平性设置是否一致，避免同一 key 申请了不同的公平性
    if (strictFair && holder.isFair() != lockInfo.isFair()) {
        log.warn("Lock fairness setting mismatch for key: {}. Existing: {}, Requested: {}",
                lockInfo.getKey(), holder.isFair(), lockInfo.isFair());
        return false;
    }

    try {
        return holder.getLock().tryLock(lockInfo.getWaitTime(), lockInfo.getTimeUnit());
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        return false;
    }
}
```

这里用 strictFair 参数控制是否严格检查公平性，提供一个灵活的选择。当 strictFair 为 false 时，不检查公平性，退化到最开始的表现，是否加锁成功取决于是否同一线程加锁。

对于 RedisLockExecutor，由于只需要解决线程重入的公平性问题，所以不需要考虑分布式的情况，直接用 ConcurrentHashMap 记录锁的公平性，再基于记录在加锁时检查。

```java
private final ConcurrentHashMap<String, Boolean> fairnessCache = new ConcurrentHashMap<>();
@Override
public boolean tryLock(LockInfo lockInfo, boolean strictFair) {

    // 检查并更新公平性设置
    if (hasFairnessMismatch(lockInfo, strictFair)) return false;

    RLock lock = getLock(lockInfo);
    try {
        boolean acquired = lock.tryLock(
                lockInfo.getWaitTime(),
                lockInfo.getLeaseTime(),
                lockInfo.getTimeUnit()
        );
        if (acquired) {
            // 保证公平性缓存与锁一致，也避免 unlock() 提前删除缓存的问题
            fairnessCache.put(lockInfo.getKey(), lockInfo.isFair());
        }
        return acquired;
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        return false;
    }
}

private boolean hasFairnessMismatch(LockInfo lockInfo, boolean strictFair) {
    boolean existingFairness = fairnessCache.compute(lockInfo.getKey(), (k, existing) -> {
        if (existing != null) {
            return existing;
        }
        return lockInfo.isFair();
    });

    // 检查公平性设置是否一致
    if (strictFair && existingFairness != lockInfo.isFair()) {
        log.warn("Lock fairness setting mismatch for key: {}. Existing: {}, Requested: {}",
                lockInfo.getKey(), existingFairness, lockInfo.isFair());
        return true;
    }
    return false;
}
```

公平性检测的逻辑如下。

1. 从 Map 获取 key 对应的公平性，不存在则记录
2. 如果严格检查，不一致时直接加锁失败
3. 加锁成功后，更新记录，保持记录一致

同时，在释放锁时，如果锁没有被任何线程持有，则清除公平性记录。

```java
@Override
public void unlock(LockInfo lockInfo) {
    RLock lock = getLock(lockInfo);
    if (lock.isHeldByCurrentThread()) {
        lock.unlock();
        // 如果锁没有被任何线程持有，则清除缓存
        if (!lock.isLocked()) {
            // 无法实现 hasQueuedThreads() 判断，可能导致缓存被提前删除，在 tryLock() 进行了处理
            fairnessCache.remove(lockInfo.getKey());
        }
    }
}
```

现在，我们就实现了对加锁功能的封装，后续代码基于 LockExecutor 这个抽象进行，不需要再关注底层细节。

### 实现加锁模板

LockTemplate 主要在确定编程式加锁的 API，专注于如何便捷地使用锁。设计上使用模板方法，定义整体加锁和释放锁的流程，让使用者提供业务逻辑。

核心方法如下。

```java
public <T> T execute(LockInfo lockInfo, Supplier<T> supplier) {
    if (!lockExecutor.tryLock(lockInfo)) {
        throw new LockException("Failed to acquire lock: " + lockInfo.getKey());
    }
    try {
        return supplier.get();
    } finally {
        lockExecutor.unlock(lockInfo);
    }
}
```

这里选择 Supplier 作为业务逻辑的抽象，而不是 Callable，因为 Callable 需要抛出异常。在加锁逻辑中，业务的异常应该由业务代码自己处理，加锁时并不需要处理业务逻辑的异常。Supplier 中抛出的异常，应该由 execute 方法的调用者去关注。

为了易用，还需要提供一些重载实现，比如无返回值的业务逻辑，用 Runnable 抽象。

```java
public void execute(String key, Runnable runnable) {
    execute(key, () -> {
        runnable.run();
        return null;
    });
}

public void execute(LockInfo lockInfo, Runnable runnable) {
    execute(lockInfo, () -> {
        runnable.run();
        return null;
    });
}

public <T> T execute(String key, Supplier<T> supplier) {
    return execute(createLockInfo(key), supplier);
}
```

只提供 key 的版本，使用全局默认配置，由 LockProperties 提供。代码比较简单，不再赘述。

### 实现 AOP 切面

AOP 切面只需要关注一个功能，获取注解中的配置，然后调用 LockTemplate 执行加锁。调用 LockExecutor 也可以，但代码会复杂一些。

```java
@Aspect
public class LockAspect {
    private final LockTemplate lockTemplate;

    @Around("@annotation(withLock)")
    public Object around(ProceedingJoinPoint point, WithLock withLock) {
        LockInfo lockInfo = parseLockInfo(point, withLock);

        return lockTemplate.execute(lockInfo, () -> {
            try {
                return point.proceed();
            } catch (Throwable throwable) {
                if (throwable instanceof RuntimeException) {
                    throw (RuntimeException) throwable;
                }
                throw new LockException("Error occurred while executing locked method", throwable);
            }
        });
    }
}
```

重点在于解析注解中的配置，解析 SpEL 表达式复杂一些，但也是些模板代码。

```java
private LockInfo parseLockInfo(ProceedingJoinPoint point, WithLock withLock) {
    try {
        // 解析SpEL表达式获取锁的key
        String key = SpELUtil.parseExpression(withLock.key(), point);

        return LockInfo.builder()
                .key(key)
                .waitTime(getWaitTime(withLock))
                .leaseTime(getLeaseTime(withLock))
                .timeUnit(withLock.timeUnit())
                .fair(withLock.fair())
                .build();
    } catch (Exception e) {
        throw new LockException("Failed to parse lock info", e);
    }
}

public class SpELUtil {
    private static final ExpressionParser PARSER = new SpelExpressionParser();
    private static final DefaultParameterNameDiscoverer NAME_DISCOVERER = new DefaultParameterNameDiscoverer();


    public static String parseExpression(String expression, ProceedingJoinPoint point) {
        MethodSignature signature = (MethodSignature) point.getSignature();
        Method method = signature.getMethod();
        EvaluationContext context = createContext(point, method);
        return PARSER.parseExpression(expression).getValue(context, String.class);
    }

    private static EvaluationContext createContext(ProceedingJoinPoint point, Method method) {
        StandardEvaluationContext context = new StandardEvaluationContext();
        Object[] args = point.getArgs();
        String[] parameterNames = NAME_DISCOVERER.getParameterNames(method);

        for (int i = 0; i < args.length; i++) {
            context.setVariable(parameterNames[i], args[i]);
        }

        return context;
    }
}
```

解析 SpEL 表达式，需要一个 EvaluationContext。表达式中的用到的参数，都从 Context 中获取，所以需要将方法参数设置到 Context 中。获取参数名也是固定的逻辑。

### 自动装配

自动装配时有两个重要功能。

一是支持 @EnableLocking 注解。使用 @Conditional 注解，实现 Condition 接口。

```java
public class LockingEnabledCondition implements Condition {

    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        String propertyValue = context.getEnvironment().getProperty("app.lock.enabled");
        if ("true".equalsIgnoreCase(propertyValue)) {
            return true;
        }

        return context.getBeanFactory().getBeanNamesForAnnotation(EnableLocking.class).length > 0;
    }
}

@AutoConfiguration
@EnableConfigurationProperties(LockProperties.class)
@Conditional(LockingEnabledCondition.class)
public class LockAutoConfiguration {
    // 省略其他代码
}
```

二是可配置的 LockExecutor 实现类。使用 @ConditionalOnProperty 注解根据配置选择不同的实现类。

```java
@Bean
@ConditionalOnProperty(prefix = "app.lock", name = "type", havingValue = "LOCAL", matchIfMissing = true)
public LockExecutor localLockExecutor() {
    return new LocalLockExecutor();
}

@Bean
@ConditionalOnProperty(prefix = "app.lock", name = "type", havingValue = "REDIS")
@ConditionalOnClass(RedissonClient.class)
public LockExecutor redisLockExecutor(RedissonClient redissonClient, LockProperties lockProperties) {
    return new RedisLockExecutor(redissonClient, lockProperties.getKeyStorePrefix());
}
```

RedissonClient 由外部环境提供，不需要在当前配置类中初始化。

当然，为了 Spring Boot 能定位 AutoConfiguration 配置类，需要提供配置文件。不同 Spring Boot 版本有不同的配置文件格式。Spring Boot 3.0 使用的是 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件。在里面添加 LockAutoConfiguration 类的完全限定名即可。

## 可扩展点

基于上述流程，已经实现了一个比较完善的锁框架。功能不多，但足够用。相对于直接在业务逻辑中操作锁，代码简洁了一些，可以更加专注于业务代码的实现，降低编程时的心智负担。

有一些可能会需要的扩展方向，简单提供一下思路，因为我还没有遇到这样的需求，所以就没有实现。

### 锁类型过于单一

理想情况下至少应该支持一下读写锁，在高并发时读写锁的锁粒度更小，读操作性能更好。要实现读写锁，最好是提供一个独立于当前组件的 ReadWriteLockExecutor 接口，在里面定义读写锁的加锁逻辑，然后 ReadWriteLockTemplate 和 @ReadWriteLock 注解。实现时可以提取与排他锁相同的公共逻辑，放到 BaseLockExecutor 之类的抽象类中。不提取也没关系，代码复杂度不会增加太多。

至于其他的锁类型，比如联锁、红锁、分段锁等，实现的逻辑类似，都可以参考现版本实现。

### 锁实现只能二选一

本地锁和分布式锁只能选其一，其实二者可以共存。实现起来也简单，调整一下 LockTemplate 的实现即可，内部维持多个 LockExecutor 实现类，根据不同的锁类型调用不同的实现。同时，提供多个不同类型的注解 @LocalLock 和 @RedisLock，进一步简化使用。

## 总结

本文提供了一种在 Spring Boot 中更简便地加锁的封装思路，关键点是基于 AOP 切面和模板方法，实现了一个功能比较完善的锁框架。

完整代码可以从 [GitHub](https://github.com/xioshe/net-channels/tree/master/net-channels-common/src/main/java/com/github/xioshe/net/channels/common/lock) 获取，也包含了相关单元测试。
