### Observer观察者模式

定义对象间的一种一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新。

##### 适应性：

1. 当一个抽象模型有两个方面，其中一个方面依赖于另一个方面。将这二者封装在独立的对象中以使它们可以各自独立地改变和复用；
2. 当对一个对象的改变需要同时改变其它对象，而不知道具体有多少对象有待改变；
3. 当一个对象必须通知其它对象，而它又不能假定其它对象是谁。换言之，你不希望这些对象是紧密耦合的。

##### 效果：

1. 目标和观察者间的抽象耦合；
2. 支持广播通信；
3. 意外的更新。

##### 结构：

1. 创建目标到其观察者之间的映射；
2. 观察多个目标；
3. 谁触发更新；
4. 对已删除目标的悬挂引用；
5. 在发出通知前确保目标的状态自身是一致的；
6. 避免特定于观察者的更新协议——推/拉模型；
7. 显式地指定感兴趣的改变；
8. 封装复杂的更新语义。

##### Dubbo中的应用：

+ CacheListener

  Subject类：

  ```java
  public interface DataListener {
  
      void dataChanged(String path, Object value, EventType eventType);
  }
  
  public class CacheListener implements DataListener {
      private static final int MIN_PATH_DEPTH = 5;
  
      private Map<String, Set<ConfigurationListener>> keyListeners = new ConcurrentHashMap<>();
      private CountDownLatch initializedLatch;
      private String rootPath;
  
      public CacheListener(String rootPath, CountDownLatch initializedLatch) {
          this.rootPath = rootPath;
          this.initializedLatch = initializedLatch;
      }
  
      public void addListener(String key, ConfigurationListener configurationListener) {
          Set<ConfigurationListener> listeners = this.keyListeners.computeIfAbsent(key, k -> new CopyOnWriteArraySet<>());
          listeners.add(configurationListener);
      }
  
      public void removeListener(String key, ConfigurationListener configurationListener) {
          Set<ConfigurationListener> listeners = this.keyListeners.get(key);
          if (listeners != null) {
              listeners.remove(configurationListener);
          }
      }
  
      /**
       * This is used to convert a configuration nodePath into a key
       * TODO doc
       *
       * @param path
       * @return key (nodePath less the config root path)
       */
      private String pathToKey(String path) {
          if (StringUtils.isEmpty(path)) {
              return path;
          }
          String groupKey = path.replace(rootPath + PATH_SEPARATOR, "").replaceAll(PATH_SEPARATOR, DOT_SEPARATOR);
          return groupKey.substring(groupKey.indexOf(DOT_SEPARATOR) + 1);
      }
  
  
      @Override
      public void dataChanged(String path, Object value, EventType eventType) {
          if (eventType == null) {
              return;
          }
  
          if (eventType == EventType.INITIALIZED) {
              initializedLatch.countDown();
              return;
          }
  
          if (path == null || (value == null && eventType != EventType.NodeDeleted)) {
              return;
          }
  
          // TODO We only care the changes happened on a specific path level, for example
          //  /dubbo/config/dubbo/configurators, other config changes not in this level will be ignored,
          if (path.split("/").length >= MIN_PATH_DEPTH) {
              String key = pathToKey(path);
              ConfigChangeType changeType;
              switch (eventType) {
                  case NodeCreated:
                      changeType = ConfigChangeType.ADDED;
                      break;
                  case NodeDeleted:
                      changeType = ConfigChangeType.DELETED;
                      break;
                  case NodeDataChanged:
                      changeType = ConfigChangeType.MODIFIED;
                      break;
                  default:
                      return;
              }
  
              ConfigChangeEvent configChangeEvent = new ConfigChangeEvent(key, (String) value, changeType);
              Set<ConfigurationListener> listeners = keyListeners.get(path);
              if (CollectionUtils.isNotEmpty(listeners)) {
                  listeners.forEach(listener -> listener.process(configChangeEvent));
              }
          }
      }
  }
  
  ```

  Observer类：

  1. ConsumerConfigurationListener

  ​	ReferenceConfigurationListener

  ​	ServiceConfigurationListener

  ​	ProviderConfigurationListener

  ```java
  public interface ConfigurationListener {
  
      void process(ConfigChangeEvent event);
  }
  
  
  public abstract class AbstractConfiguratorListener implements ConfigurationListener {
      private static final Logger logger = LoggerFactory.getLogger(AbstractConfiguratorListener.class);
  
      protected List<Configurator> configurators = Collections.emptyList();
  
  
      protected final void initWith(String key) {
          DynamicConfiguration dynamicConfiguration = DynamicConfiguration.getDynamicConfiguration();
          dynamicConfiguration.addListener(key, this);
          String rawConfig = dynamicConfiguration.getRule(key, DynamicConfiguration.DEFAULT_GROUP);
          if (!StringUtils.isEmpty(rawConfig)) {
              genConfiguratorsFromRawRule(rawConfig);
          }
      }
  
      @Override
      public void process(ConfigChangeEvent event) {
          if (logger.isInfoEnabled()) {
              logger.info("Notification of overriding rule, change type is: " + event.getChangeType() +
                      ", raw config content is:\n " + event.getValue());
          }
  
          if (event.getChangeType().equals(ConfigChangeType.DELETED)) {
              configurators.clear();
          } else {
              if (!genConfiguratorsFromRawRule(event.getValue())) {
                  return;
              }
          }
  
          notifyOverrides();
      }
  
      private boolean genConfiguratorsFromRawRule(String rawConfig) {
          boolean parseSuccess = true;
          try {
              // parseConfigurators will recognize app/service config automatically.
              configurators = Configurator.toConfigurators(ConfigParser.parseConfigurators(rawConfig))
                      .orElse(configurators);
          } catch (Exception e) {
              logger.error("Failed to parse raw dynamic config and it will not take effect, the raw config is: " +
                      rawConfig, e);
              parseSuccess = false;
          }
          return parseSuccess;
      }
  
      protected abstract void notifyOverrides();
  
      public List<Configurator> getConfigurators() {
          return configurators;
      }
  
      public void setConfigurators(List<Configurator> configurators) {
          this.configurators = configurators;
      }
  }
  
  ```

  2. Router

     抽象类：AbstractRouter

     实现类：

       TagRouter（其它Router未实现ConfigurationListener接口）

     ```java
     public class TagRouter extends AbstractRouter implements ConfigurationListener {
         public static final String NAME = "TAG_ROUTER";
         private static final int TAG_ROUTER_DEFAULT_PRIORITY = 100;
         private static final Logger logger = LoggerFactory.getLogger(TagRouter.class);
         private static final String RULE_SUFFIX = ".tag-router";
     
         private TagRouterRule tagRouterRule;
         private String application;
     
         public TagRouter(DynamicConfiguration configuration, URL url) {
             super(configuration, url);
             this.priority = TAG_ROUTER_DEFAULT_PRIORITY;
         }
     
         @Override
         public synchronized void process(ConfigChangeEvent event) {
             if (logger.isDebugEnabled()) {
                 logger.debug("Notification of tag rule, change type is: " + event.getChangeType() + ", raw rule is:\n " +
                         event.getValue());
             }
     
             try {
                 if (event.getChangeType().equals(ConfigChangeType.DELETED)) {
                     this.tagRouterRule = null;
                 } else {
                     this.tagRouterRule = TagRuleParser.parse(event.getValue());
                 }
             } catch (Exception e) {
                 logger.error("Failed to parse the raw tag router rule and it will not take effect, please check if the " +
                         "rule matches with the template, the raw rule is:\n ", e);
             }
         }
         //...
     }
     ```

     抽象类：ListenableRouter

     实现类：

       AppRouter

       ServiceRouter

     ```java
     public abstract class ListenableRouter extends AbstractRouter implements ConfigurationListener {
         //...
         public ListenableRouter(DynamicConfiguration configuration, URL url, String ruleKey) {
             super(configuration, url);
             this.force = false;
             this.init(ruleKey);
         }
     
         @Override
         public synchronized void process(ConfigChangeEvent event) {
             if (logger.isInfoEnabled()) {
                 logger.info("Notification of condition rule, change type is: " + event.getChangeType() +
                         ", raw rule is:\n " + event.getValue());
             }
     
             if (event.getChangeType().equals(ConfigChangeType.DELETED)) {
                 routerRule = null;
                 conditionRouters = Collections.emptyList();
             } else {
                 try {
                     routerRule = ConditionRuleParser.parse(event.getValue());
                     generateConditions(routerRule);
                 } catch (Exception e) {
                     logger.error("Failed to parse the raw condition rule and it will not take effect, please check " +
                             "if the condition rule matches with the template, the raw rule is:\n " + event.getValue(), e);
                 }
             }
         }
     
        //...
     }
     ```

     

     