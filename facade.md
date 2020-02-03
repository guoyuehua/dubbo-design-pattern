### Facade外观模式

为子系统中的一组接口提供一个一致的界面，Facade模式定义了一个高层接口，这个接口使得这一子系统更加容易使用。

##### 适应性：

1. 当你要为一个复杂子系统提供一个简单接口时；
2. 客户程序与抽象类的实现部分之间存在着很大的依赖性；
3. 当你需要构建一个层次结构的子系统时，使用Facade模式定义子系统中每层的入口点。

##### 效果：

1. 它对客户屏蔽子系统组件，因而减少了客户处理的对象的数目并使得子系统使用起来更加方便；
2. 它实现了子系统与客户之间的松耦合关系；
3. 如果应用需要，它并不限制它们使用子系统类。

##### 结构：

1. 降低客户-子系统之间的耦合度；
2. 公共子系统类与私有子系统类。

##### Dubbo中的应用：

- Compiler

  接口：

  ```java
  @SPI("javassist")
  public interface Compiler {
  
      /**
       * Compile java source code.
       *
       * @param code        Java source code
       * @param classLoader classloader
       * @return Compiled class
       */
      Class<?> compile(String code, ClassLoader classLoader);
  
  }
  ```

  实现：

  1. AdaptiveCompiler
  2. JavassistCompiler
  3. JdkCompiler



  