### Factory Method 工厂方法模式

定义一个用于创建对象的接口，让子类决定实例化哪一个类。Factory Method使一个类的实例化延迟到其子类。

##### 适应性：

1. 当一个类不知道它所必须创建的对象的类的时候；
2. 当一个类希望由它的子类来指定它所创建的对象的时候；
3. 当类将创建对象的职责委托给多个帮助子类中的某一个，并且你希望将哪一个帮助子类是代理者这一信息局部化的时候。

##### 效果：
1. 工厂方法不再将与特定应用有关的类绑定到你的代码中；
2. 为子类提供hook；
3. 连接平行的类层次。

##### 结构：



##### Duboo中的应用：
+ 注册中心：

| 抽象类                  | 实现类                   | 实例对象          |
| ----------------------- | ------------------------ | ----------------- |
| AbstractRegistryFactory | DubboRegistryFactory     | DubboRegistry     |
|                         | ZookeeperRegistryFactory | ZookeeperRegistry |
|                         | RedisRegistryFactory     | RedisRegistry     |
|                         | ConsulRegistryFactory    | ConsulRegistry    |
|                         | EtcdRegistryFactory      | EtcdRegistry      |
|                         | MulticastRegistry        | MulticastRegistry |
|                         | MultipleRegistryFactory  | MultipleRegistry  |
| AbstractRegistry        | FailbackRegistry         |                   |

抽象类
```java
@SPI("dubbo")
public interface RegistryFactory {
    @Adaptive({"protocol"})
    Registry getRegistry(URL url);

}
public abstract class AbstractRegistryFactory implements RegistryFactory {
	//.....
    
    @Override
    public Registry getRegistry(URL url) {
		//...
        //create registry by spi/ioc
        registry = createRegistry(url);
        if (registry == null) {
            throw new IllegalStateException("Can not create registry " + url);
        }
        REGISTRIES.put(key, registry);
        return registry;

    }

    protected abstract Registry createRegistry(URL url);

}
```

实现类

1. DubboRegistryFactory

```java
public class DubboRegistryFactory extends AbstractRegistryFactory {

    @Override
    public Registry createRegistry(URL url) {
        //...
        DubboRegistry registry = new DubboRegistry(registryInvoker, registryService);
        //...
        return registry;
    }
}

```

2. ZookeeperRegistryFactory
```java
public class ZookeeperRegistryFactory extends AbstractRegistryFactory {

	//...

    @Override
    public Registry createRegistry(URL url) {
        return new ZookeeperRegistry(url, zookeeperTransporter);
    }

}
```

3. RedisRegistryFactory

```java
public class RedisRegistryFactory extends AbstractRegistryFactory {

    @Override
    protected Registry createRegistry(URL url) {
        return new RedisRegistry(url);
    }

}
```

+ 缓存

| 抽象类               | 实现类                  | 实例对象             |
| -------------------- | ----------------------- | -------------------- |
| AbstractCacheFactory | JCacheFactory           | JCache               |
|                      | LruCacheFactory         | LruCache             |
|                      | ThreadLocalCacheFactory | ThreadLocalCache     |
|                      | ExpiringCacheFactory    | ExpiringCacheFactory |
| Cache                | JCache                  |                      |
|                      | LruCache                |                      |
|                      | ThreadLocalCache        |                      |
|                      | ExpiringCacheFactory    |                      |

抽象类

```java
@SPI("lru")
public interface CacheFactory {

    @Adaptive("cache")
    Cache getCache(URL url, Invocation invocation);

}

/**
 * AbstractCacheFactory is a default implementation of {@link CacheFactory}. It abstract out the key formation from URL along with
 * invocation method. It initially check if the value for key already present in own local in-memory store then it won't check underlying storage cache {@link Cache}.
 * Internally it used {@link ConcurrentHashMap} to store do level-1 caching.
 *
 * @see CacheFactory
 * @see org.apache.dubbo.cache.support.jcache.JCacheFactory
 * @see org.apache.dubbo.cache.support.lru.LruCacheFactory
 * @see org.apache.dubbo.cache.support.threadlocal.ThreadLocalCacheFactory
 * @see org.apache.dubbo.cache.support.expiring.ExpiringCacheFactory
 */
public abstract class AbstractCacheFactory implements CacheFactory {

    /**
     * This is used to store factory level-1 cached data.
     */
    private final ConcurrentMap<String, Cache> caches = new ConcurrentHashMap<String, Cache>();

    @Override
    public Cache getCache(URL url, Invocation invocation) {
        url = url.addParameter(METHOD_KEY, invocation.getMethodName());
        String key = url.toFullString();
        Cache cache = caches.get(key);
        if (cache == null) {
            caches.put(key, createCache(url));
            cache = caches.get(key);
        }
        return cache;
    }

    /**
     * Takes url as an method argument and return new instance of cache store implemented by AbstractCacheFactory subclass.
     * @param url url of the method
     * @return Create and return new instance of cache store used as storage for caching return values.
     */
    protected abstract Cache createCache(URL url);

}
```

实现类

1. JCacheFactory

```java
public class JCacheFactory extends AbstractCacheFactory {

    /**
     * Takes url as an method argument and return new instance of cache store implemented by JCache.
     * @param url url of the method
     * @return JCache instance of cache
     */
    @Override
    protected Cache createCache(URL url) {
        return new JCache(url);
    }

}
```

2. LruCacheFactory

```java
public class LruCacheFactory extends AbstractCacheFactory {

    /**
     * Takes url as an method argument and return new instance of cache store implemented by LruCache.
     * @param url url of the method
     * @return ThreadLocalCache instance of cache
     */
    @Override
    protected Cache createCache(URL url) {
        return new LruCache(url);
    }

}
```

3. ThreadLocalCacheFactory

```java
public class ThreadLocalCacheFactory extends AbstractCacheFactory {

    /**
     * Takes url as an method argument and return new instance of cache store implemented by ThreadLocalCache.
     * @param url url of the method
     * @return ThreadLocalCache instance of cache
     */
    @Override
    protected Cache createCache(URL url) {
        return new ThreadLocalCache(url);
    }

}
```

4. ExpiringCacheFactory

```java
public class ExpiringCacheFactory extends AbstractCacheFactory {

    /**
     * Takes url as an method argument and return new instance of cache store implemented by JCache.
     * @param url url of the method
     * @return ExpiringCache instance of cache
     */
    @Override
    protected Cache createCache(URL url) {
        return new ExpiringCache(url);
    }
}
```

cache接口：

```java
public interface Cache {
    /**
     * API to store value against a key
     * @param key  Unique identifier for the object being store.
     * @param value Value getting store
     */
    void put(Object key, Object value);

    /**
     * API to return stored value using a key.
     * @param key Unique identifier for cache lookup
     * @return Return stored object against key
     */
    Object get(Object key);

}
```

+ 线程池

接口

```java
@SPI("fixed")
public interface ThreadPool {

    /**
     * Thread pool
     *
     * @param url URL contains thread parameter
     * @return thread pool
     */
    @Adaptive({THREADPOOL_KEY})
    Executor getExecutor(URL url);

}
```

实现类

1. CachedThreadPool

```java
public class CachedThreadPool implements ThreadPool {

    @Override
    public Executor getExecutor(URL url) {
        String name = url.getParameter(THREAD_NAME_KEY, DEFAULT_THREAD_NAME);
        int cores = url.getParameter(CORE_THREADS_KEY, DEFAULT_CORE_THREADS);
        int threads = url.getParameter(THREADS_KEY, Integer.MAX_VALUE);
        int queues = url.getParameter(QUEUES_KEY, DEFAULT_QUEUES);
        int alive = url.getParameter(ALIVE_KEY, DEFAULT_ALIVE);
        return new ThreadPoolExecutor(cores, threads, alive, TimeUnit.MILLISECONDS,
                queues == 0 ? new SynchronousQueue<Runnable>() :
                        (queues < 0 ? new LinkedBlockingQueue<Runnable>()
                                : new LinkedBlockingQueue<Runnable>(queues)),
                new NamedInternalThreadFactory(name, true), new AbortPolicyWithReport(name, url));
    }
}
```

2. EagerThreadPool

```java
public class EagerThreadPool implements ThreadPool {

    @Override
    public Executor getExecutor(URL url) {
        String name = url.getParameter(THREAD_NAME_KEY, DEFAULT_THREAD_NAME);
        int cores = url.getParameter(CORE_THREADS_KEY, DEFAULT_CORE_THREADS);
        int threads = url.getParameter(THREADS_KEY, Integer.MAX_VALUE);
        int queues = url.getParameter(QUEUES_KEY, DEFAULT_QUEUES);
        int alive = url.getParameter(ALIVE_KEY, DEFAULT_ALIVE);

        // init queue and executor
        TaskQueue<Runnable> taskQueue = new TaskQueue<Runnable>(queues <= 0 ? 1 : queues);
        EagerThreadPoolExecutor executor = new EagerThreadPoolExecutor(cores,
                threads,
                alive,
                TimeUnit.MILLISECONDS,
                taskQueue,
                new NamedInternalThreadFactory(name, true),
                new AbortPolicyWithReport(name, url));
        taskQueue.setExecutor(executor);
        return executor;
    }
}
```

3. FixedThreadPool

```java
public class FixedThreadPool implements ThreadPool {

    @Override
    public Executor getExecutor(URL url) {
        String name = url.getParameter(THREAD_NAME_KEY, DEFAULT_THREAD_NAME);
        int threads = url.getParameter(THREADS_KEY, DEFAULT_THREADS);
        int queues = url.getParameter(QUEUES_KEY, DEFAULT_QUEUES);
        return new ThreadPoolExecutor(threads, threads, 0, TimeUnit.MILLISECONDS,
                queues == 0 ? new SynchronousQueue<Runnable>() :
                        (queues < 0 ? new LinkedBlockingQueue<Runnable>()
                                : new LinkedBlockingQueue<Runnable>(queues)),
                new NamedInternalThreadFactory(name, true), new AbortPolicyWithReport(name, url));
    }

}
```

4. LimitedThreadPool

```java
public class LimitedThreadPool implements ThreadPool {

    @Override
    public Executor getExecutor(URL url) {
        String name = url.getParameter(THREAD_NAME_KEY, DEFAULT_THREAD_NAME);
        int cores = url.getParameter(CORE_THREADS_KEY, DEFAULT_CORE_THREADS);
        int threads = url.getParameter(THREADS_KEY, DEFAULT_THREADS);
        int queues = url.getParameter(QUEUES_KEY, DEFAULT_QUEUES);
        return new ThreadPoolExecutor(cores, threads, Long.MAX_VALUE, TimeUnit.MILLISECONDS,
                queues == 0 ? new SynchronousQueue<Runnable>() :
                        (queues < 0 ? new LinkedBlockingQueue<Runnable>()
                                : new LinkedBlockingQueue<Runnable>(queues)),
                new NamedInternalThreadFactory(name, true), new AbortPolicyWithReport(name, url));
    }

}
```

