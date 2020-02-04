### Decorator装饰器模式

动态地给一个对象添加一些额外的职责。就增加功能来说，Decorator模式相比生成子类更为灵活。

##### 标志

包装器Wrapper

##### 适应性：

1. 在不影响其他对象的情况下，以动态、透明的方式给单个对象添加职责；
2. 处理那些可以撤消的职责；
3. 当不能采用生成子类的方法进行扩充时。

##### 效果：

1. 比静态继承更灵活；
2. 避免在层次结构高层的类有太多的特征；
3. Decorator与它的Component（被装饰类）不一样，扩展了功能；
4. 有许多小对象。

##### 结构：

1. 接口的一致性；
2. 省略抽象的Decorator类；
3. 保持Component类的简单性；
4. 改变的是对象外壳（策略模式改变的是对象内核）。

##### Dubbo中的应用：

+ Protocol

  接口：

  ```java
  @SPI("dubbo")
  public interface Protocol {
  
      int getDefaultPort();
  
      @Adaptive
      <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;
  
      @Adaptive
      <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;
  
      void destroy();
  
  }
  ```

  实现：

  1. ProtocolListenerWrapper

  ```java
  public class ProtocolListenerWrapper implements Protocol {
  
      private final Protocol protocol;
  
      public ProtocolListenerWrapper(Protocol protocol) {
          if (protocol == null) {
              throw new IllegalArgumentException("protocol == null");
          }
          this.protocol = protocol;
      }
  
      @Override
      public int getDefaultPort() {
          return protocol.getDefaultPort();
      }
  
      @Override
      public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
          if (REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {
              return protocol.export(invoker);
          }
          return new ListenerExporterWrapper<T>(protocol.export(invoker),
                  Collections.unmodifiableList(ExtensionLoader.getExtensionLoader(ExporterListener.class)
                          .getActivateExtension(invoker.getUrl(), EXPORTER_LISTENER_KEY)));
      }
  
      @Override
      public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
          if (REGISTRY_PROTOCOL.equals(url.getProtocol())) {
              return protocol.refer(type, url);
          }
          return new ListenerInvokerWrapper<T>(protocol.refer(type, url),
                  Collections.unmodifiableList(
                          ExtensionLoader.getExtensionLoader(InvokerListener.class)
                                  .getActivateExtension(url, INVOKER_LISTENER_KEY)));
      }
  
      @Override
      public void destroy() {
          protocol.destroy();
      }
  
  }
  ```

  2. ProtocolFilterWrapper

     CallbackRegistrationInvoker

     ```java
     public class ProtocolFilterWrapper implements Protocol {
     
         private final Protocol protocol;
     
         public ProtocolFilterWrapper(Protocol protocol) {
             if (protocol == null) {
                 throw new IllegalArgumentException("protocol == null");
             }
             this.protocol = protocol;
         }
     
         private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {
             Invoker<T> last = invoker;
             List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);
             //....
             return new CallbackRegistrationInvoker<>(last, filters);
         }
     
         @Override
         public int getDefaultPort() {
             return protocol.getDefaultPort();
         }
     
         @Override
         public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
             if (REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {
                 return protocol.export(invoker);
             }
             return protocol.export(buildInvokerChain(invoker, SERVICE_FILTER_KEY, CommonConstants.PROVIDER));
         }
     
         @Override
         public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
             if (REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                 return protocol.refer(type, url);
             }
             return buildInvokerChain(protocol.refer(type, url), REFERENCE_FILTER_KEY, CommonConstants.CONSUMER);
         }
     
         @Override
         public void destroy() {
             protocol.destroy();
         }
     
         
         static class CallbackRegistrationInvoker<T> implements Invoker<T> {
     
             private final Invoker<T> filterInvoker;
             private final List<Filter> filters;
     
             public CallbackRegistrationInvoker(Invoker<T> filterInvoker, List<Filter> filters) {
                 this.filterInvoker = filterInvoker;
                 this.filters = filters;
             }
     
             @Override
             public Result invoke(Invocation invocation) throws RpcException {
                 Result asyncResult = filterInvoker.invoke(invocation);
     
                 asyncResult = asyncResult.whenCompleteWithContext((r, t) -> {
                     for (int i = filters.size() - 1; i >= 0; i--) {
                         Filter filter = filters.get(i);
                         // onResponse callback
                         if (filter instanceof ListenableFilter) {
                             Filter.Listener listener = ((ListenableFilter) filter).listener();
                             if (listener != null) {
                                 if (t == null) {
                                     listener.onResponse(r, filterInvoker, invocation);
                                 } else {
                                     listener.onError(t, filterInvoker, invocation);
                                 }
                             }
                         } else {
                             filter.onResponse(r, filterInvoker, invocation);
                         }
                     }
                 });
                 return asyncResult;
             }
     
             @Override
             public Class<T> getInterface() {
                 return filterInvoker.getInterface();
             }
     
             @Override
             public URL getUrl() {
                 return filterInvoker.getUrl();
             }
     
             @Override
             public boolean isAvailable() {
                 return filterInvoker.isAvailable();
             }
     
             @Override
             public void destroy() {
                 filterInvoker.destroy();
             }
         }
     }
     ```

+ ListenerInvokerWrapper

  接口：

  ```java
  public interface Invoker<T> extends Node {
  
      Class<T> getInterface();
  
      Result invoke(Invocation invocation) throws RpcException;
  
  }
  ```

  实现：

  ```java
  public class ListenerInvokerWrapper<T> implements Invoker<T> {
  
      private static final Logger logger = LoggerFactory.getLogger(ListenerInvokerWrapper.class);
  
      private final Invoker<T> invoker;
  
      private final List<InvokerListener> listeners;
  
      public ListenerInvokerWrapper(Invoker<T> invoker, List<InvokerListener> listeners) {
          if (invoker == null) {
              throw new IllegalArgumentException("invoker == null");
          }
          this.invoker = invoker;
          this.listeners = listeners;
          if (CollectionUtils.isNotEmpty(listeners)) {
              for (InvokerListener listener : listeners) {
                  if (listener != null) {
                      try {
                          listener.referred(invoker);
                      } catch (Throwable t) {
                          logger.error(t.getMessage(), t);
                      }
                  }
              }
          }
      }
  
      @Override
      public Class<T> getInterface() {
          return invoker.getInterface();
      }
  
      @Override
      public URL getUrl() {
          return invoker.getUrl();
      }
  
      @Override
      public boolean isAvailable() {
          return invoker.isAvailable();
      }
  
      @Override
      public Result invoke(Invocation invocation) throws RpcException {
          return invoker.invoke(invocation);
      }
  
      @Override
      public String toString() {
          return getInterface() + " -> " + (getUrl() == null ? " " : getUrl().toString());
      }
  
      @Override
      public void destroy() {
          try {
              invoker.destroy();
          } finally {
              if (CollectionUtils.isNotEmpty(listeners)) {
                  for (InvokerListener listener : listeners) {
                      if (listener != null) {
                          try {
                              listener.destroyed(invoker);
                          } catch (Throwable t) {
                              logger.error(t.getMessage(), t);
                          }
                      }
                  }
              }
          }
      }
  
  }
  ```

+ Logger

  接口：

  ```java
  public interface Logger {
  
      void trace(String msg);
  
      void trace(Throwable e);
  
      void trace(String msg, Throwable e);
  
      void debug(String msg);
  
      void debug(Throwable e);
  
      void debug(String msg, Throwable e);
  
      void info(String msg);
  
      void info(Throwable e);
  
      void info(String msg, Throwable e);
  
      void warn(String msg);
  
      void warn(Throwable e);
  
      void warn(String msg, Throwable e);
  
      void error(String msg);
  
      void error(Throwable e);
  
      void error(String msg, Throwable e);
  
      boolean isTraceEnabled();
  
      boolean isDebugEnabled();
  
      boolean isInfoEnabled();
  
      boolean isWarnEnabled();
  
      boolean isErrorEnabled();
  
  }
  ```

  实现：

  ```java
  public class FailsafeLogger implements Logger {
  
      private Logger logger;
  
      public FailsafeLogger(Logger logger) {
          this.logger = logger;
      }
  
      public Logger getLogger() {
          return logger;
      }
  
      public void setLogger(Logger logger) {
          this.logger = logger;
      }
  
      private String appendContextMessage(String msg) {
          return " [DUBBO] " + msg + ", dubbo version: " + Version.getVersion() + ", current host: " + NetUtils.getLocalHost();
      }
  
      @Override
      public void trace(String msg, Throwable e) {
          try {
              logger.trace(appendContextMessage(msg), e);
          } catch (Throwable t) {
          }
      }
  
      @Override
      public void trace(Throwable e) {
          try {
              logger.trace(e);
          } catch (Throwable t) {
          }
      }
  
      @Override
      public void trace(String msg) {
          try {
              logger.trace(appendContextMessage(msg));
          } catch (Throwable t) {
          }
      }
  
      @Override
      public void debug(String msg, Throwable e) {
          try {
              logger.debug(appendContextMessage(msg), e);
          } catch (Throwable t) {
          }
      }
  
      @Override
      public void debug(Throwable e) {
          try {
              logger.debug(e);
          } catch (Throwable t) {
          }
      }
  
      @Override
      public void debug(String msg) {
          try {
              logger.debug(appendContextMessage(msg));
          } catch (Throwable t) {
          }
      }
  
      @Override
      public void info(String msg, Throwable e) {
          try {
              logger.info(appendContextMessage(msg), e);
          } catch (Throwable t) {
          }
      }
  
      @Override
      public void info(String msg) {
          try {
              logger.info(appendContextMessage(msg));
          } catch (Throwable t) {
          }
      }
  
      @Override
      public void warn(String msg, Throwable e) {
          try {
              logger.warn(appendContextMessage(msg), e);
          } catch (Throwable t) {
          }
      }
  
      @Override
      public void warn(String msg) {
          try {
              logger.warn(appendContextMessage(msg));
          } catch (Throwable t) {
          }
      }
  
      @Override
      public void error(String msg, Throwable e) {
          try {
              logger.error(appendContextMessage(msg), e);
          } catch (Throwable t) {
          }
      }
  
      @Override
      public void error(String msg) {
          try {
              logger.error(appendContextMessage(msg));
          } catch (Throwable t) {
          }
      }
  
      @Override
      public void error(Throwable e) {
          try {
              logger.error(e);
          } catch (Throwable t) {
          }
      }
  
      @Override
      public void info(Throwable e) {
          try {
              logger.info(e);
          } catch (Throwable t) {
          }
      }
  
      @Override
      public void warn(Throwable e) {
          try {
              logger.warn(e);
          } catch (Throwable t) {
          }
      }
  
      @Override
      public boolean isTraceEnabled() {
          try {
              return logger.isTraceEnabled();
          } catch (Throwable t) {
              return false;
          }
      }
  
      @Override
      public boolean isDebugEnabled() {
          try {
              return logger.isDebugEnabled();
          } catch (Throwable t) {
              return false;
          }
      }
  
      @Override
      public boolean isInfoEnabled() {
          try {
              return logger.isInfoEnabled();
          } catch (Throwable t) {
              return false;
          }
      }
  
      @Override
      public boolean isWarnEnabled() {
          try {
              return logger.isWarnEnabled();
          } catch (Throwable t) {
              return false;
          }
      }
  
      @Override
      public boolean isErrorEnabled() {
          try {
              return logger.isErrorEnabled();
          } catch (Throwable t) {
              return false;
          }
      }
  
  }
  ```

  