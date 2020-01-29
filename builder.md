### Builder 生成器模式

将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以创建不同的表示。

##### 适应性：

1. 当创建复杂对象的算法应该独立于该对象的组成部分以及它们的装配方式时；
2. 当构造过程必须允许被构造的对象有不同的表示时。
##### 效果：

1. 它使你可以改变一个产品的内部表示；
2. 它将构造代码和表示代码分开；
3. 它使你可对构建过程进行更精细的控制。


##### 结构：



##### Dubbo中的应用：

| 接口      | 实现              |
| --------- | ----------------- |
| Directory | RegistryDirectory |
|           | StaticDirectory   |

抽象类
```java
public interface Directory<T> extends Node {

    Class<T> getInterface();

    List<Invoker<T>> list(Invocation invocation) throws RpcException;
}

public abstract class AbstractDirectory<T> implements Directory<T> {

    //...
    @Override
    public List<Invoker<T>> list(Invocation invocation) throws RpcException {
        if (destroyed) {
            throw new RpcException("Directory already destroyed .url: " + getUrl());
        }

        return doList(invocation);
    }
    
	protected abstract List<Invoker<T>> doList(Invocation invocation) throws RpcException;
}

```
实现类

1. RegistryDirectory

```java
public class RegistryDirectory<T> extends AbstractDirectory<T> implements NotifyListener {
	@Override
    public List<Invoker<T>> doList(Invocation invocation) {
        //...

        if (multiGroup) {
            return this.invokers == null ? Collections.emptyList() : this.invokers;
        }

        List<Invoker<T>> invokers = null;
        try {
            // Get invokers from cache, only runtime routers will be executed.
            invokers = routerChain.route(getConsumerUrl(), invocation);
        } catch (Throwable t) {
            logger.error("Failed to execute router: " + getUrl() + ", cause: " + t.getMessage(), t);
        }

        return invokers == null ? Collections.emptyList() : invokers;
    }

    @Override
    public Class<T> getInterface() {
        return serviceType;
    }
}
```

2. StaticDirectory
```java
public class StaticDirectory<T> extends AbstractDirectory<T> {
    
    @Override
    public Class<T> getInterface() {
        return invokers.get(0).getInterface();
    }
    
    //...

    @Override
    protected List<Invoker<T>> doList(Invocation invocation) throws RpcException {
        List<Invoker<T>> finalInvokers = invokers;
        if (routerChain != null) {
            try {
                finalInvokers = routerChain.route(getConsumerUrl(), invocation);
            } catch (Throwable t) {
                logger.error("Failed to execute router: " + getUrl() + ", cause: " + t.getMessage(), t);
            }
        }
        return finalInvokers == null ? Collections.emptyList() : finalInvokers;
    }

}
```