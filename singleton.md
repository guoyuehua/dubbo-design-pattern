### Singleton单例模式

保证一个类仅有一个实例，并提供一个访问它的全局访问点。

##### 适应性：

1. 当类只能有一个实例而且客户可以从一个众所周知的访问点访问它时；
2. 当这个唯一实例应该通过子类化可扩展的，并且客户应该无需更改代码就能使用一个扩展的实例时。

##### 效果：

1. 对唯一实例的受控访问；
2. 缩小名空间；
3. 允许对操作和表示的精细化；
4. 允许可变数目的实例；
5. 比类操作更灵活。

##### 结构：



##### Dubbo中的应用：

+ ExtensionLoader

  ```java
  public class ExtensionLoader<T> {
  
      private static final Logger logger = LoggerFactory.getLogger(ExtensionLoader.class);
  
      //...
      
      private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<>();
      
      private final ConcurrentMap<String, Holder<Object>> cachedInstances = new ConcurrentHashMap<>();
      
      public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
          if (type == null) {
              throw new IllegalArgumentException("Extension type == null");
          }
          if (!type.isInterface()) {
              throw new IllegalArgumentException("Extension type (" + type + ") is not an interface!");
          }
          if (!withExtensionAnnotation(type)) {
              throw new IllegalArgumentException("Extension type (" + type +
                      ") is not an extension, because it is NOT annotated with @" + SPI.class.getSimpleName() + "!");
          }
  
          ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
          if (loader == null) {
              EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
              loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
          }
          return loader;
      }
      
      
      private Holder<Object> getOrCreateHolder(String name) {
          Holder<Object> holder = cachedInstances.get(name);
          if (holder == null) {
              cachedInstances.putIfAbsent(name, new Holder<>());
              holder = cachedInstances.get(name);
          }
          return holder;
      }
      
      
      public T getExtension(String name) {
          if (StringUtils.isEmpty(name)) {
              throw new IllegalArgumentException("Extension name == null");
          }
          if ("true".equals(name)) {
              return getDefaultExtension();
          }
          final Holder<Object> holder = getOrCreateHolder(name);
          Object instance = holder.get();
          if (instance == null) {
              synchronized (holder) {
                  instance = holder.get();
                  if (instance == null) {
                      instance = createExtension(name);
                      holder.set(instance);
                  }
              }
          }
          return (T) instance;
      }
      
      public T getAdaptiveExtension() {
          Object instance = cachedAdaptiveInstance.get();
          if (instance == null) {
              if (createAdaptiveInstanceError == null) {
                  synchronized (cachedAdaptiveInstance) {
                      instance = cachedAdaptiveInstance.get();
                      if (instance == null) {
                          try {
                              instance = createAdaptiveExtension();
                              cachedAdaptiveInstance.set(instance);
                          } catch (Throwable t) {
                              createAdaptiveInstanceError = t;
                              throw new IllegalStateException("Failed to create adaptive instance: " + t.toString(), t);
                          }
                      }
                  }
              } else {
                  throw new IllegalStateException("Failed to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
              }
          }
  
          return (T) instance;
      }
      
      //...
      
  }
  ```

  