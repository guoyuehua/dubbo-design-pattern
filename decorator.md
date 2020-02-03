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

  