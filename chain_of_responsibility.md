### Chain Of Responsibility责任链模式

使多个对象都有机会处理请求，从而避免请求的发送者和接收者之间的耦合关系。将这些对象连成一条链，并沿着这条链传递该请求，直到有一个对象处理它为止。

##### 适应性：

1. 有 多个对象可以处理一个请求，哪个对象处理该请求运行时刻自动确定；
2. 你想在不明确指定接收者的情况下，向多个对象中的一个提交一个请求；
3. 可处理一个请求的对象集合诮被动态指定。

##### 效果：

1. 降低耦合度；
2. 增强了给对象指派职责的灵活性；
3. 不保证被接受。

##### 结构：



##### Dubbo中的应用：

+ TelnetHandlerAdapter

  ```java
  public class TelnetHandlerAdapter extends ChannelHandlerAdapter implements TelnetHandler {
  
      private final ExtensionLoader<TelnetHandler> extensionLoader = ExtensionLoader.getExtensionLoader(TelnetHandler.class);
  
      @Override
      public String telnet(Channel channel, String message) throws RemotingException {
          String prompt = channel.getUrl().getParameterAndDecoded(Constants.PROMPT_KEY, Constants.DEFAULT_PROMPT);
          boolean noprompt = message.contains("--no-prompt");
          message = message.replace("--no-prompt", "");
          StringBuilder buf = new StringBuilder();
          message = message.trim();
          String command;
          if (message.length() > 0) {
              int i = message.indexOf(' ');
              if (i > 0) {
                  command = message.substring(0, i).trim();
                  message = message.substring(i + 1).trim();
              } else {
                  command = message;
                  message = "";
              }
          } else {
              command = "";
          }
          if (command.length() > 0) {
              if (extensionLoader.hasExtension(command)) {
                  if (commandEnabled(channel.getUrl(), command)) {
                      try {
                          String result = extensionLoader.getExtension(command).telnet(channel, message);
                          if (result == null) {
                              return null;
                          }
                          buf.append(result);
                      } catch (Throwable t) {
                          buf.append(t.getMessage());
                      }
                  } else {
                      buf.append("Command: ");
                      buf.append(command);
                      buf.append(" disabled");
                  }
              } else {
                  buf.append("Unsupported command: ");
                  buf.append(command);
              }
          }
          if (buf.length() > 0) {
              buf.append("\r\n");
          }
          if (StringUtils.isNotEmpty(prompt) && !noprompt) {
              buf.append(prompt);
          }
          return buf.toString();
      }
  
      private boolean commandEnabled(URL url, String command) {
          String supportCommands = url.getParameter(TELNET);
          if (StringUtils.isEmpty(supportCommands)) {
              return true;
          }
          String[] commands = COMMA_SPLIT_PATTERN.split(supportCommands);
          for (String c : commands) {
              if (command.equals(c)) {
                  return true;
              }
          }
          return false;
      }
  
  }
  ```

  