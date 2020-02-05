### Strategy策略模式

定义一系列的算法，把它们一个个封装起来，并且使用它们可相互替换。本模式使得算法可独立于使用它的客户而变化。

##### 适应性：

1. 许多相关的类仅仅是行为有异，“策略”提供了一种用多个行为中的一个行为来配置一个类的方法 ；
2. 需要使用一个算法的不同变体；
3. 算法使用客户不应该知道的数据，以避免暴露复杂的、与算法相关的数据结构；
4. 一个类定义了多种行为，并且这些行为在这个类的操作中以多个条件语句的形式出现，将相关的条件分支移入它们各自的Strategy类中以代替这些条件语句。

##### 效果：

1. 相关算法系列；
2. 一个替代继承的方法；
3. 消除了一些条件语句；
4. 实现的选择；
5. 客户必须了解不同的策略；
6. Strategy和Context之间的通信开销；
7. 增加了对象的数目。

##### 结构：



##### Dubbo中的应用：

+ LoadBalance

  接口：

  ```java
  @SPI(RandomLoadBalance.NAME)
  public interface LoadBalance {
  
      @Adaptive("loadbalance")
      <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException;
  
  }
  ```

  实现：

  1. RandomLoadBalance
  2. ConsistentHashLoadBalance
  3. LeastActiveLoadBalance
  4. RoundRobinLoadBalance

+ Cluster

  接口：

  ```java
  @SPI(FailoverCluster.NAME)
  public interface Cluster {
  
      @Adaptive
      <T> Invoker<T> join(Directory<T> directory) throws RpcException;
  
  }
  ```

  实现：

  1. AvailableCluster
  2. BroadcastCluster
  3. FailbackCluster
  4. FailfastCluster
  5. FailoverCluster
  6. FailsafeCluster
  7. ForkingCluster
  8. MergeableCluster
  9. RegistryAwareCluster