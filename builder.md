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

+ TypeBuilder

  接口：

  ```java
  @SPI
  public interface TypeBuilder {
  
      /**
       * Whether the build accept the type or class passed in.
       */
      boolean accept(Type type, Class<?> clazz);
  
      /**
       * Build type definition with the type or class.
       */
      TypeDefinition build(Type type, Class<?> clazz, Map<Class<?>, TypeDefinition> typeCache);
  
  }
  ```

  实现：

  1. ArrayTypeBuilder

  ```java
  public class ArrayTypeBuilder implements TypeBuilder {
  
      @Override
      public boolean accept(Type type, Class<?> clazz) {
          if (clazz == null) {
              return false;
          }
          return clazz.isArray();
      }
  
      @Override
      public TypeDefinition build(Type type, Class<?> clazz, Map<Class<?>, TypeDefinition> typeCache) {
          // Process the component type of an array.
          Class<?> componentType = clazz.getComponentType();
          TypeDefinitionBuilder.build(componentType, componentType, typeCache);
  
          final String canonicalName = clazz.getCanonicalName();
          return new TypeDefinition(canonicalName);
      }
  
  }
  ```

  2. CollectionTypeBuilder

  ```java
  public class CollectionTypeBuilder implements TypeBuilder {
  
      @Override
      public boolean accept(Type type, Class<?> clazz) {
          if (clazz == null) {
              return false;
          }
          return Collection.class.isAssignableFrom(clazz);
      }
  
      @Override
      public TypeDefinition build(Type type, Class<?> clazz, Map<Class<?>, TypeDefinition> typeCache) {
          if (!(type instanceof ParameterizedType)) {
              return new TypeDefinition(clazz.getName());
          }
  
          ParameterizedType parameterizedType = (ParameterizedType) type;
          Type[] actualTypeArgs = parameterizedType.getActualTypeArguments();
          if (actualTypeArgs == null || actualTypeArgs.length != 1) {
              throw new IllegalArgumentException(MessageFormat.format(
                      "[ServiceDefinitionBuilder] Collection type [{0}] with unexpected amount of arguments [{1}]."
                              + Arrays.toString(actualTypeArgs),
                      type, actualTypeArgs));
          }
  
          Type actualType = actualTypeArgs[0];
          if (actualType instanceof ParameterizedType) {
              // Nested collection or map.
              Class<?> rawType = (Class<?>) ((ParameterizedType) actualType).getRawType();
              TypeDefinitionBuilder.build(actualType, rawType, typeCache);
          } else if (actualType instanceof Class<?>) {
              Class<?> actualClass = (Class<?>) actualType;
              TypeDefinitionBuilder.build(null, actualClass, typeCache);
          }
  
          return new TypeDefinition(type.toString());
      }
  
  }
  ```

  3. EnumTypeBuilder

  ```java
  public class EnumTypeBuilder implements TypeBuilder {
  
      @Override
      public boolean accept(Type type, Class<?> clazz) {
          if (clazz == null) {
              return false;
          }
          return clazz.isEnum();
      }
  
      @Override
      public TypeDefinition build(Type type, Class<?> clazz, Map<Class<?>, TypeDefinition> typeCache) {
          TypeDefinition td = new TypeDefinition(clazz.getCanonicalName());
  
          try {
              Method methodValues = clazz.getDeclaredMethod("values");
              Object[] values = (Object[]) methodValues.invoke(clazz, new Object[0]);
              int length = values.length;
              for (int i = 0; i < length; i++) {
                  Object value = values[i];
                  td.getEnums().add(value.toString());
              }
          } catch (Throwable t) {
              td.setId("-1");
          }
  
          typeCache.put(clazz, td);
          return td;
      }
  
  }
  ```

  4. MapTypeBuilder

  ```java
  public class MapTypeBuilder implements TypeBuilder {
  
      @Override
      public boolean accept(Type type, Class<?> clazz) {
          if (clazz == null) {
              return false;
          }
          return Map.class.isAssignableFrom(clazz);
      }
  
      @Override
      public TypeDefinition build(Type type, Class<?> clazz, Map<Class<?>, TypeDefinition> typeCache) {
          if (!(type instanceof ParameterizedType)) {
              return new TypeDefinition(clazz.getName());
          }
  
          ParameterizedType parameterizedType = (ParameterizedType) type;
          Type[] actualTypeArgs = parameterizedType.getActualTypeArguments();
          if (actualTypeArgs == null || actualTypeArgs.length != 2) {
              throw new IllegalArgumentException(MessageFormat.format(
                      "[ServiceDefinitionBuilder] Map type [{0}] with unexpected amount of arguments [{1}]."
                              + Arrays.toString(actualTypeArgs), type, actualTypeArgs));
          }
  
          for (Type actualType : actualTypeArgs) {
              if (actualType instanceof ParameterizedType) {
                  // Nested collection or map.
                  Class<?> rawType = (Class<?>) ((ParameterizedType) actualType).getRawType();
                  TypeDefinitionBuilder.build(actualType, rawType, typeCache);
              } else if (actualType instanceof Class<?>) {
                  Class<?> actualClass = (Class<?>) actualType;
                  TypeDefinitionBuilder.build(null, actualClass, typeCache);
              }
          }
  
          return new TypeDefinition(type.toString());
      }
  }
  ```

  

+ 配置对象

  | 接口            | 实现                  |
  | --------------- | --------------------- |
  | AbstractBuilder | ConfigCenterBuilder   |
  |                 | ApplicationBuilder    |
  |                 | ConsumerBuilder       |
  |                 | MetadataReportBuilder |
  |                 | MethodBuilder         |
  |                 | ModuleBuilder         |
  |                 | MonitorBuilder        |
  |                 | ProtocolBuilder       |
  |                 | ProviderBuilder       |
  |                 | ReferenceBuilder      |
  |                 | RegistryBuilder       |
  |                 | ServiceBuilder        |

  抽象类

  ```java
  public abstract class AbstractBuilder<T extends AbstractConfig, B extends AbstractBuilder> {
      /**
       * The config id
       */
      protected String id;
      protected String prefix;
  
      protected B id(String id) {
          this.id = id;
          return getThis();
      }
  
      protected B prefix(String prefix) {
          this.prefix = prefix;
          return getThis();
      }
  
      protected abstract B getThis();
  
      protected static Map<String, String> appendParameter(Map<String, String> parameters, String key, String value) {
          if (parameters == null) {
              parameters = new HashMap<>();
          }
          parameters.put(key, value);
          return parameters;
      }
  
      protected static Map<String, String> appendParameters(Map<String, String> parameters, Map<String, String> appendParameters) {
          if (parameters == null) {
              parameters = new HashMap<>();
          }
          parameters.putAll(appendParameters);
          return parameters;
      }
  
      protected void build(T instance) {
          if (!StringUtils.isEmpty(id)) {
              instance.setId(id);
          }
          if (!StringUtils.isEmpty(prefix)) {
              instance.setPrefix(prefix);
          }
      }
  }
  ```

  实现类

  ```java
  public class ConfigCenterBuilder extends AbstractBuilder<ConfigCenterConfig, ConfigCenterBuilder> {
  
      private String protocol;
      private String address;
      private String cluster;
      private String namespace = "dubbo";
      private String group = "dubbo";
      private String username;
      private String password;
      private Long timeout = 3000L;
      private Boolean highestPriority = true;
      private Boolean check = true;
  
      private String appName;
      private String configFile = "dubbo.properties";
      private String appConfigFile;
  
      private Map<String, String> parameters;
  
      public ConfigCenterBuilder protocol(String protocol) {
          this.protocol = protocol;
          return getThis();
      }
  
  	public ConfigCenterBuilder address(String address) {
          this.address = address;
          return getThis();
      }
  
      public ConfigCenterBuilder cluster(String cluster) {
          this.cluster = cluster;
          return getThis();
      }
  
      public ConfigCenterBuilder namespace(String namespace) {
          this.namespace = namespace;
          return getThis();
      }
  
      public ConfigCenterBuilder group(String group) {
          this.group = group;
          return getThis();
      }
  
      public ConfigCenterBuilder username(String username) {
          this.username = username;
          return getThis();
      }
  
      public ConfigCenterBuilder password(String password) {
          this.password = password;
          return getThis();
      }
  
      public ConfigCenterBuilder timeout(Long timeout) {
          this.timeout = timeout;
          return getThis();
      }
  
      public ConfigCenterBuilder highestPriority(Boolean highestPriority) {
          this.highestPriority = highestPriority;
          return getThis();
      }
  
      public ConfigCenterBuilder check(Boolean check) {
          this.check = check;
          return getThis();
      }
  
      public ConfigCenterBuilder configFile(String configFile) {
          this.configFile = configFile;
          return getThis();
      }
  
      public ConfigCenterBuilder appConfigFile(String appConfigFile) {
          this.appConfigFile = appConfigFile;
          return getThis();
      }
  
      public ConfigCenterBuilder appendParameters(Map<String, String> appendParameters) {
          this.parameters = appendParameters(this.parameters, appendParameters);
          return getThis();
      }
  
      public ConfigCenterBuilder appendParameter(String key, String value) {
          this.parameters = appendParameter(this.parameters, key, value);
          return getThis();
      }
  
      public ConfigCenterConfig build() {
          ConfigCenterConfig configCenter = new ConfigCenterConfig();
          super.build(configCenter);
  
          configCenter.setProtocol(protocol);
          configCenter.setAddress(address);
          configCenter.setCluster(cluster);
          configCenter.setNamespace(namespace);
          configCenter.setGroup(group);
          configCenter.setUsername(username);
          configCenter.setPassword(password);
          configCenter.setTimeout(timeout);
          configCenter.setHighestPriority(highestPriority);
          configCenter.setCheck(check);
          configCenter.setConfigFile(configFile);
          configCenter.setAppConfigFile(appConfigFile);
          configCenter.setParameters(parameters);
  
          return configCenter;
      }
  
      @Override
      protected ConfigCenterBuilder getThis() {
          return this;
      }
  }
  ```

