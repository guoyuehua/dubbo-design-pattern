### Adapter适配器模式

将一个类的接口转换成客户希望的另一个接口，使得原本由于接口不兼容而不能一起工作的那些类可以一起工作。

##### 适应性：

1. 你想使用一个已经存在的类，而它的接口不符合你的需求；
2. 你想创建一个可以复用的类，该类可以与其他不相关的类或不可预见的类协同工作；
3. 你想使用一些已经存在的类，但是不可能对每一个都行子类化以匹配它们的接口。

##### 效果：

1. Adapter的匹配程序；
2. 可插入的Adapter。

##### 结构：



##### Dubbo中的应用：

+ 日志

  | 接口          | 实例类              |
  | ------------- | ------------------- |
  | LoggerAdapter | JclLoggerAdapter    |
  |               | JdkLoggerAdapter    |
  |               | Log4j2LoggerAdapter |
  |               | Log4jLoggerAdapter  |
  |               | Slf4jLoggerAdapter  |

  接口：

  ```java
  @SPI
  public interface LoggerAdapter {
  
      /**
       * Get a logger
       *
       * @param key the returned logger will be named after clazz
       * @return logger
       */
      Logger getLogger(Class<?> key);
  
      /**
       * Get a logger
       *
       * @param key the returned logger will be named after key
       * @return logger
       */
      Logger getLogger(String key);
  
      /**
       * Get the current logging level
       *
       * @return current logging level
       */
      Level getLevel();
  
      /**
       * Set the current logging level
       *
       * @param level logging level
       */
      void setLevel(Level level);
  
      /**
       * Get the current logging file
       *
       * @return current logging file
       */
      File getFile();
  
      /**
       * Set the current logging file
       *
       * @param file logging file
       */
      void setFile(File file);
  }
  ```

  实现类：

  1. JclLoggerAdapter

  ```java
  public class JclLoggerAdapter implements LoggerAdapter {
  
      private Level level;
      private File file;
  
      @Override
      public Logger getLogger(String key) {
          return new JclLogger(LogFactory.getLog(key));
      }
  
      @Override
      public Logger getLogger(Class<?> key) {
          return new JclLogger(LogFactory.getLog(key));
      }
  
      @Override
      public Level getLevel() {
          return level;
      }
  
      @Override
      public void setLevel(Level level) {
          this.level = level;
      }
  
      @Override
      public File getFile() {
          return file;
      }
  
      @Override
      public void setFile(File file) {
          this.file = file;
      }
  
  }
  ```

  2. Log4jLoggerAdapter

  ```java
  public class Log4jLoggerAdapter implements LoggerAdapter {
  
      private File file;
  
      @SuppressWarnings("unchecked")
      public Log4jLoggerAdapter() {
          try {
              org.apache.log4j.Logger logger = LogManager.getRootLogger();
              if (logger != null) {
                  Enumeration<Appender> appenders = logger.getAllAppenders();
                  if (appenders != null) {
                      while (appenders.hasMoreElements()) {
                          Appender appender = appenders.nextElement();
                          if (appender instanceof FileAppender) {
                              FileAppender fileAppender = (FileAppender) appender;
                              String filename = fileAppender.getFile();
                              file = new File(filename);
                              break;
                          }
                      }
                  }
              }
          } catch (Throwable t) {
          }
      }
  
      private static org.apache.log4j.Level toLog4jLevel(Level level) {
          if (level == Level.ALL) {
              return org.apache.log4j.Level.ALL;
          }
          if (level == Level.TRACE) {
              return org.apache.log4j.Level.TRACE;
          }
          if (level == Level.DEBUG) {
              return org.apache.log4j.Level.DEBUG;
          }
          if (level == Level.INFO) {
              return org.apache.log4j.Level.INFO;
          }
          if (level == Level.WARN) {
              return org.apache.log4j.Level.WARN;
          }
          if (level == Level.ERROR) {
              return org.apache.log4j.Level.ERROR;
          }
          // if (level == Level.OFF)
          return org.apache.log4j.Level.OFF;
      }
  
      private static Level fromLog4jLevel(org.apache.log4j.Level level) {
          if (level == org.apache.log4j.Level.ALL) {
              return Level.ALL;
          }
          if (level == org.apache.log4j.Level.TRACE) {
              return Level.TRACE;
          }
          if (level == org.apache.log4j.Level.DEBUG) {
              return Level.DEBUG;
          }
          if (level == org.apache.log4j.Level.INFO) {
              return Level.INFO;
          }
          if (level == org.apache.log4j.Level.WARN) {
              return Level.WARN;
          }
          if (level == org.apache.log4j.Level.ERROR) {
              return Level.ERROR;
          }
          // if (level == org.apache.log4j.Level.OFF)
          return Level.OFF;
      }
  
      @Override
      public Logger getLogger(Class<?> key) {
          return new Log4jLogger(LogManager.getLogger(key));
      }
  
      @Override
      public Logger getLogger(String key) {
          return new Log4jLogger(LogManager.getLogger(key));
      }
  
      @Override
      public Level getLevel() {
          return fromLog4jLevel(LogManager.getRootLogger().getLevel());
      }
  
      @Override
      public void setLevel(Level level) {
          LogManager.getRootLogger().setLevel(toLog4jLevel(level));
      }
  
      @Override
      public File getFile() {
          return file;
      }
  
      @Override
      public void setFile(File file) {
  
      }
  
  }
  ```

  3. Slf4jLoggerAdapter

  ```java
  public class Slf4jLoggerAdapter implements LoggerAdapter {
  
      private Level level;
      private File file;
  
      @Override
      public Logger getLogger(String key) {
          return new Slf4jLogger(org.slf4j.LoggerFactory.getLogger(key));
      }
  
      @Override
      public Logger getLogger(Class<?> key) {
          return new Slf4jLogger(org.slf4j.LoggerFactory.getLogger(key));
      }
  
      @Override
      public Level getLevel() {
          return level;
      }
  
      @Override
      public void setLevel(Level level) {
          this.level = level;
      }
  
      @Override
      public File getFile() {
          return file;
      }
  
      @Override
      public void setFile(File file) {
          this.file = file;
      }
  
  }
  ```

+ 配置

  接口

  ```java
  public interface Configuration {
  
      default String getString(String key) {
          return convert(String.class, key, null);
      }
  
      default String getString(String key, String defaultValue) {
          return convert(String.class, key, defaultValue);
      }
  
  
      default Object getProperty(String key) {
          return getProperty(key, null);
      }
  
      default Object getProperty(String key, Object defaultValue) {
          Object value = getInternalProperty(key);
          return value != null ? value : defaultValue;
      }
  
      Object getInternalProperty(String key);
  
      default boolean containsKey(String key) {
          return getProperty(key) != null;
      }
  
      //...
  
  }
  ```

  类：

  ```java
  public class ConfigConfigurationAdapter implements Configuration {
  
      private Map<String, String> metaData;
  
      public ConfigConfigurationAdapter(AbstractConfig config) {
          this.metaData = config.getMetaData();
      }
  
      @Override
      public Object getInternalProperty(String key) {
          return metaData.get(key);
      }
  
  }
  ```

  

