### Command命令模式

将一个请求封装为一个对象，从而使你可用不同的请求对客户进行参数化；对请求排队或记录请求日志，以及支持可撤消的操作。

##### 适应性：

1. 在不同的时刻指定 、排列和执行请求；
2. 支持取消操作；
3. 支持修改日志；
4. 用构建在原语操作上的高层操作构造一个系统。

##### 效果：

1. 将调用操作的对象与知道如何实现该操作的对象解耦；
2. Command可像其它对象一样被操纵和扩展；
3. 可将多个命令装配成一个复合命令；
4. 增加新的Command很容易，无需改变已有的类。

##### 结构：

1. 一个命令对象应达到何种智能程序；
2. 支持取消和重做；
3. 避免取消操作过程中的错误积累。

##### Dubbo中的应用：

+ TelnetHanlder

  接口：

  ```java
  @SPI
  public interface TelnetHandler {
  
      String telnet(Channel channel, String message) throws RemotingException;
  
  }
  ```

  实现：

  1. ClearTelnetHandler

  ```java
  @Activate
  @Help(parameter = "[lines]", summary = "Clear screen.", detail = "Clear screen.")
  public class ClearTelnetHandler implements TelnetHandler {
  
      @Override
      public String telnet(Channel channel, String message) {
          int lines = 100;
          if (message.length() > 0) {
              if (!StringUtils.isInteger(message)) {
                  return "Illegal lines " + message + ", must be integer.";
              }
              lines = Integer.parseInt(message);
          }
          StringBuilder buf = new StringBuilder();
          for (int i = 0; i < lines; i++) {
              buf.append("\r\n");
          }
          return buf.toString();
      }
  
  }
  ```

  2. ExitTelnetHandler

  ```java
  @Activate
  @Help(parameter = "", summary = "Exit the telnet.", detail = "Exit the telnet.")
  public class ExitTelnetHandler implements TelnetHandler {
  
      @Override
      public String telnet(Channel channel, String message) {
          channel.close();
          return null;
      }
  
  }
  ```

  3. HelpTelnetHandler

  ```java
  @Activate
  @Help(parameter = "[command]", summary = "Show help.", detail = "Show help.")
  public class HelpTelnetHandler implements TelnetHandler {
  
      private final ExtensionLoader<TelnetHandler> extensionLoader = ExtensionLoader.getExtensionLoader(TelnetHandler.class);
  
      @Override
      public String telnet(Channel channel, String message) {
          if (message.length() > 0) {
              if (!extensionLoader.hasExtension(message)) {
                  return "No such command " + message;
              }
              TelnetHandler handler = extensionLoader.getExtension(message);
              Help help = handler.getClass().getAnnotation(Help.class);
              StringBuilder buf = new StringBuilder();
              buf.append("Command:\r\n    ");
              buf.append(message + " " + help.parameter().replace("\r\n", " ").replace("\n", " "));
              buf.append("\r\nSummary:\r\n    ");
              buf.append(help.summary().replace("\r\n", " ").replace("\n", " "));
              buf.append("\r\nDetail:\r\n    ");
              buf.append(help.detail().replace("\r\n", "    \r\n").replace("\n", "    \n"));
              return buf.toString();
          } else {
              List<List<String>> table = new ArrayList<List<String>>();
              List<TelnetHandler> handlers = extensionLoader.getActivateExtension(channel.getUrl(), "telnet");
              if (CollectionUtils.isNotEmpty(handlers)) {
                  for (TelnetHandler handler : handlers) {
                      Help help = handler.getClass().getAnnotation(Help.class);
                      List<String> row = new ArrayList<String>();
                      String parameter = " " + extensionLoader.getExtensionName(handler) + " " + (help != null ? help.parameter().replace("\r\n", " ").replace("\n", " ") : "");
                      row.add(parameter.length() > 55 ? parameter.substring(0, 55) + "..." : parameter);
                      String summary = help != null ? help.summary().replace("\r\n", " ").replace("\n", " ") : "";
                      row.add(summary.length() > 55 ? summary.substring(0, 55) + "..." : summary);
                      table.add(row);
                  }
              }
              return "Please input \"help [command]\" show detail.\r\n" + TelnetUtils.toList(table);
          }
      }
  
  }
  ```

  4. LogTelnetHandler

  ```java
  @Activate
  @Help(parameter = "level", summary = "Change log level or show log ", detail = "Change log level or show log")
  public class LogTelnetHandler implements TelnetHandler {
  
      public static final String SERVICE_KEY = "telnet.log";
  
      @Override
      public String telnet(Channel channel, String message) {
          long size = 0;
          File file = LoggerFactory.getFile();
          StringBuffer buf = new StringBuffer();
          if (message == null || message.trim().length() == 0) {
              buf.append("EXAMPLE: log error / log 100");
          } else {
              String[] str = message.split(" ");
              if (!StringUtils.isInteger(str[0])) {
                  LoggerFactory.setLevel(Level.valueOf(message.toUpperCase()));
              } else {
                  int showLogLength = Integer.parseInt(str[0]);
  
                  if (file != null && file.exists()) {
                      try {
                          try (FileInputStream fis = new FileInputStream(file)) {
                              try (FileChannel filechannel = fis.getChannel()) {
                                  size = filechannel.size();
                                  ByteBuffer bb;
                                  if (size <= showLogLength) {
                                      bb = ByteBuffer.allocate((int) size);
                                      filechannel.read(bb, 0);
                                  } else {
                                      int pos = (int) (size - showLogLength);
                                      bb = ByteBuffer.allocate(showLogLength);
                                      filechannel.read(bb, pos);
                                  }
                                  bb.flip();
                                  String content = new String(bb.array()).replace("<", "&lt;")
                                          .replace(">", "&gt;").replace("\n", "<br/><br/>");
                                  buf.append("\r\ncontent:" + content);
  
                                  buf.append("\r\nmodified:" + (new SimpleDateFormat("yyyy-MM-dd HH:mm:ss")
                                          .format(new Date(file.lastModified()))));
                                  buf.append("\r\nsize:" + size + "\r\n");
                              }
                          }
                      } catch (Exception e) {
                          buf.append(e.getMessage());
                      }
                  } else {
                      size = 0;
                      buf.append("\r\nMESSAGE: log file not exists or log appender is console .");
                  }
              }
          }
          buf.append("\r\nCURRENT LOG LEVEL:" + LoggerFactory.getLevel())
                  .append("\r\nCURRENT LOG APPENDER:" + (file == null ? "console" : file.getAbsolutePath()));
          return buf.toString();
      }
  
  }
  ```

  5. StatusTelnetHandler

  ```java
  @Activate
  @Help(parameter = "[-l]", summary = "Show status.", detail = "Show status.")
  public class StatusTelnetHandler implements TelnetHandler {
  
      private final ExtensionLoader<StatusChecker> extensionLoader = ExtensionLoader.getExtensionLoader(StatusChecker.class);
  
      @Override
      public String telnet(Channel channel, String message) {
          if (message.equals("-l")) {
              List<StatusChecker> checkers = extensionLoader.getActivateExtension(channel.getUrl(), "status");
              String[] header = new String[]{"resource", "status", "message"};
              List<List<String>> table = new ArrayList<List<String>>();
              Map<String, Status> statuses = new HashMap<String, Status>();
              if (CollectionUtils.isNotEmpty(checkers)) {
                  for (StatusChecker checker : checkers) {
                      String name = extensionLoader.getExtensionName(checker);
                      Status stat;
                      try {
                          stat = checker.check();
                      } catch (Throwable t) {
                          stat = new Status(Status.Level.ERROR, t.getMessage());
                      }
                      statuses.put(name, stat);
                      if (stat.getLevel() != null && stat.getLevel() != Status.Level.UNKNOWN) {
                          List<String> row = new ArrayList<String>();
                          row.add(name);
                          row.add(String.valueOf(stat.getLevel()));
                          row.add(stat.getMessage() == null ? "" : stat.getMessage());
                          table.add(row);
                      }
                  }
              }
              Status stat = StatusUtils.getSummaryStatus(statuses);
              List<String> row = new ArrayList<String>();
              row.add("summary");
              row.add(String.valueOf(stat.getLevel()));
              row.add(stat.getMessage());
              table.add(row);
              return TelnetUtils.toTable(header, table);
          } else if (message.length() > 0) {
              return "Unsupported parameter " + message + " for status.";
          }
          String status = channel.getUrl().getParameter("status");
          Map<String, Status> statuses = new HashMap<String, Status>();
          if (CollectionUtils.isNotEmptyMap(statuses)) {
              String[] ss = COMMA_SPLIT_PATTERN.split(status);
              for (String s : ss) {
                  StatusChecker handler = extensionLoader.getExtension(s);
                  Status stat;
                  try {
                      stat = handler.check();
                  } catch (Throwable t) {
                      stat = new Status(Status.Level.ERROR, t.getMessage());
                  }
                  statuses.put(s, stat);
              }
          }
          Status stat = StatusUtils.getSummaryStatus(statuses);
          return String.valueOf(stat.getLevel());
      }
  
  }
  ```

  