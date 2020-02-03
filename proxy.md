### Proxy代理模式

为其它对象提供一种代理以控制对象对这个对象的访问。

##### 适应性：

1. 远程代理；
2. 虚代理；
3. 保护代理；
4. 智能指引。

##### 效果：

1. 远程代理可以隐藏一个对象存在于不同地址空间的事实；
2. 虚代理可以进行最优化，如根据要求创建对象；
3. 保护代理和智能代理都允许在访问一个对象时有一些附加的内务处理。

##### 结构：



##### Dubbo中的应用：

+ ExchangeServer

  接口：

  ```java
  public interface Endpoint {
  
      URL getUrl();
  
      ChannelHandler getChannelHandler();
  
      InetSocketAddress getLocalAddress();
  
      void send(Object message) throws RemotingException;
  
      void send(Object message, boolean sent) throws RemotingException;
  
      void close();
  
      void close(int timeout);
  
      void startClose();
  
      boolean isClosed();
  }
  
  public interface Server extends Endpoint, Resetable, IdleSensible {
  
      boolean isBound();
  
      Collection<Channel> getChannels();
  
      Channel getChannel(InetSocketAddress remoteAddress);
  
      @Deprecated
      void reset(org.apache.dubbo.common.Parameters parameters);
  }
  
  public interface ExchangeServer extends Server {
  
      Collection<ExchangeChannel> getExchangeChannels();
  
      ExchangeChannel getExchangeChannel(InetSocketAddress remoteAddress);
  
  }
  ```

  实现：

  ```java
  public class ExchangeServerDelegate implements ExchangeServer {
  
      private transient ExchangeServer server;
  
      public ExchangeServerDelegate() {
      }
  
      public ExchangeServerDelegate(ExchangeServer server) {
          setServer(server);
      }
  
      public ExchangeServer getServer() {
          return server;
      }
  
      public void setServer(ExchangeServer server) {
          this.server = server;
      }
  
      @Override
      public boolean isBound() {
          return server.isBound();
      }
  
      @Override
      public void reset(URL url) {
          server.reset(url);
      }
  
      @Override
      @Deprecated
      public void reset(org.apache.dubbo.common.Parameters parameters) {
          reset(getUrl().addParameters(parameters.getParameters()));
      }
  
      @Override
      public Collection<Channel> getChannels() {
          return server.getChannels();
      }
  
      @Override
      public Channel getChannel(InetSocketAddress remoteAddress) {
          return server.getChannel(remoteAddress);
      }
  
      @Override
      public URL getUrl() {
          return server.getUrl();
      }
  
      @Override
      public ChannelHandler getChannelHandler() {
          return server.getChannelHandler();
      }
  
      @Override
      public InetSocketAddress getLocalAddress() {
          return server.getLocalAddress();
      }
  
      @Override
      public void send(Object message) throws RemotingException {
          server.send(message);
      }
  
      @Override
      public void send(Object message, boolean sent) throws RemotingException {
          server.send(message, sent);
      }
  
      @Override
      public void close() {
          server.close();
      }
  
      @Override
      public boolean isClosed() {
          return server.isClosed();
      }
  
      @Override
      public Collection<ExchangeChannel> getExchangeChannels() {
          return server.getExchangeChannels();
      }
  
      @Override
      public ExchangeChannel getExchangeChannel(InetSocketAddress remoteAddress) {
          return server.getExchangeChannel(remoteAddress);
      }
  
      @Override
      public void close(int timeout) {
          server.close(timeout);
      }
  
      @Override
      public void startClose() {
          server.startClose();
      }
  
  }
  ```

+ ChannelHandlerDispatcher

  