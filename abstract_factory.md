### Abstract Factory 抽象工厂模式



提供一个创建一系列相关或相互依赖对象的接口，而无需指定它们具体的类。

##### 适应性：

1. 一个系统要独立于它的产品的创建、组合和表示时；
2. 一个系统要由多个产品系统中的一个来配置时；
3. 当你要强调一系统相关的产品对象的设计以便进行联合使用时；
4. 当你提供一个产品类库，而只想显示它们的接口而不是实现时。

##### 效果：

1. 它分离了具体的类；
2. 它使得易于交换产品系统；
3. 它有利于产品的一致性；
4. 难以支持新种类的产品。

##### 结构：

![结构图](images/AbstractFactory.png)

##### Dubbo中的应用：

+ 日志

```java
/**
 * Logger factory
 */
public class LoggerFactory {

    private static final ConcurrentMap<String, FailsafeLogger> LOGGERS = new ConcurrentHashMap<>();
    private static volatile LoggerAdapter LOGGER_ADAPTER;

    // search common-used logging frameworks
    static {
        String logger = System.getProperty("dubbo.application.logger", "");
        switch (logger) {
            case "slf4j":
                setLoggerAdapter(new Slf4jLoggerAdapter());
                break;
            case "jcl":
                setLoggerAdapter(new JclLoggerAdapter());
                break;
            case "log4j":
                setLoggerAdapter(new Log4jLoggerAdapter());
                break;
            case "jdk":
                setLoggerAdapter(new JdkLoggerAdapter());
                break;
            case "log4j2":
                setLoggerAdapter(new Log4j2LoggerAdapter());
                break;
            default:
                List<Class<? extends LoggerAdapter>> candidates = Arrays.asList(
                        Log4jLoggerAdapter.class,
                        Slf4jLoggerAdapter.class,
                        Log4j2LoggerAdapter.class,
                        JclLoggerAdapter.class,
                        JdkLoggerAdapter.class
                );
                for (Class<? extends LoggerAdapter> clazz : candidates) {
                    try {
                        setLoggerAdapter(clazz.newInstance());
                        break;
                    } catch (Throwable ignored) {
                    }
                }
        }
    }

    private LoggerFactory() {
    }

    public static void setLoggerAdapter(String loggerAdapter) {
        if (loggerAdapter != null && loggerAdapter.length() > 0) {
            setLoggerAdapter(ExtensionLoader.getExtensionLoader(LoggerAdapter.class).getExtension(loggerAdapter));
        }
    }

    /**
     * Set logger provider
     *
     * @param loggerAdapter logger provider
     */
    public static void setLoggerAdapter(LoggerAdapter loggerAdapter) {
        if (loggerAdapter != null) {
            Logger logger = loggerAdapter.getLogger(LoggerFactory.class.getName());
            logger.info("using logger: " + loggerAdapter.getClass().getName());
            LoggerFactory.LOGGER_ADAPTER = loggerAdapter;
            for (Map.Entry<String, FailsafeLogger> entry : LOGGERS.entrySet()) {
                entry.getValue().setLogger(LOGGER_ADAPTER.getLogger(entry.getKey()));
            }
        }
    }

    /**
     * Get logger provider
     *
     * @param key the returned logger will be named after clazz
     * @return logger
     */
    public static Logger getLogger(Class<?> key) {
        return LOGGERS.computeIfAbsent(key.getName(), name -> new FailsafeLogger(LOGGER_ADAPTER.getLogger(name)));
    }

    /**
     * Get logger provider
     *
     * @param key the returned logger will be named after key
     * @return logger provider
     */
    public static Logger getLogger(String key) {
        return LOGGERS.computeIfAbsent(key, k -> new FailsafeLogger(LOGGER_ADAPTER.getLogger(k)));
    }

    /**
     * Get logging level
     *
     * @return logging level
     */
    public static Level getLevel() {
        return LOGGER_ADAPTER.getLevel();
    }

    /**
     * Set the current logging level
     *
     * @param level logging level
     */
    public static void setLevel(Level level) {
        LOGGER_ADAPTER.setLevel(level);
    }

    /**
     * Get the current logging file
     *
     * @return current logging file
     */
    public static File getFile() {
        return LOGGER_ADAPTER.getFile();
    }

}
```

+ 序列化

抽象类

```java
abstract public class AbstractSerializerFactory {
    /**
     * Returns the serializer for a class.
     *
     * @param cl the class of the object that needs to be serialized.
     * @return a serializer object for the serialization.
     */
    abstract public Serializer getSerializer(Class cl)
            throws HessianProtocolException;

    /**
     * Returns the deserializer for a class.
     *
     * @param cl the class of the object that needs to be deserialized.
     * @return a deserializer object for the serialization.
     */
    abstract public Deserializer getDeserializer(Class cl)
            throws HessianProtocolException;
}
```
工厂类

```java
public class SerializerFactory extends AbstractSerializerFactory {
    //...
    
    // Additional factories
    protected ArrayList _factories = new ArrayList();
    /**
     * Adds a factory.
     */
    public void addFactory(AbstractSerializerFactory factory) {
        _factories.add(factory);
    }

    //...
    
    /**
     * Returns the serializer for a class.
     *
     * @param cl the class of the object that needs to be serialized.
     * @return a serializer object for the serialization.
     */
    @Override
    public Serializer getSerializer(Class cl)
            throws HessianProtocolException {
        Serializer serializer;

        serializer = (Serializer) _staticSerializerMap.get(cl);
        if (serializer != null) {
            return serializer;
        }

        if (_cachedSerializerMap != null) {
            serializer = (Serializer) _cachedSerializerMap.get(cl);
            if (serializer != null) {
                return serializer;
            }
        }

        for (int i = 0;
             serializer == null && _factories != null && i < _factories.size();
             i++) {
            AbstractSerializerFactory factory;
            factory = (AbstractSerializerFactory) _factories.get(i);
            serializer = factory.getSerializer(cl);
        }

        if (serializer != null) {

        } else if (isZoneId(cl)) //must before "else if (JavaSerializer.getWriteReplace(cl) != null)"
            serializer = ZoneIdSerializer.getInstance();
        else if (isEnumSet(cl))
            serializer = EnumSetSerializer.getInstance();
        else if (JavaSerializer.getWriteReplace(cl) != null)
            serializer = new JavaSerializer(cl, _loader);

        else if (HessianRemoteObject.class.isAssignableFrom(cl))
            serializer = new RemoteSerializer();

//    else if (BurlapRemoteObject.class.isAssignableFrom(cl))
//      serializer = new RemoteSerializer();

        else if (Map.class.isAssignableFrom(cl)) {
            if (_mapSerializer == null)
                _mapSerializer = new MapSerializer();

            serializer = _mapSerializer;
        } else if (Collection.class.isAssignableFrom(cl)) {
            if (_collectionSerializer == null) {
                _collectionSerializer = new CollectionSerializer();
            }

            serializer = _collectionSerializer;
        } else if (cl.isArray()) {
            serializer = new ArraySerializer();
        } else if (Throwable.class.isAssignableFrom(cl)) {
            serializer = new ThrowableSerializer(cl, getClassLoader());
        } else if (InputStream.class.isAssignableFrom(cl)) {
            serializer = new InputStreamSerializer();
        } else if (Iterator.class.isAssignableFrom(cl)) {
            serializer = IteratorSerializer.create();
        } else if (Enumeration.class.isAssignableFrom(cl)) {
            serializer = EnumerationSerializer.create();
        } else if (Calendar.class.isAssignableFrom(cl)) {
            serializer = CalendarSerializer.create();
        } else if (Locale.class.isAssignableFrom(cl)) {
            serializer = LocaleSerializer.create();
        } else if (Enum.class.isAssignableFrom(cl)) {
            serializer = new EnumSerializer(cl);
        }

        if (serializer == null) {
            serializer = getDefaultSerializer(cl);
        }

        if (_cachedSerializerMap == null) {
            _cachedSerializerMap = new ConcurrentHashMap(8);
        }

        _cachedSerializerMap.put(cl, serializer);

        return serializer;
    }

    /**
     * Returns the default serializer for a class that isn't matched
     * directly.  Application can override this method to produce
     * bean-style serialization instead of field serialization.
     *
     * @param cl the class of the object that needs to be serialized.
     * @return a serializer object for the serialization.
     */
    protected Serializer getDefaultSerializer(Class cl) {
        if (_defaultSerializer != null)
            return _defaultSerializer;

        if (!Serializable.class.isAssignableFrom(cl)
                && !_isAllowNonSerializable) {
            throw new IllegalStateException("Serialized class " + cl.getName() + " must implement java.io.Serializable");
        }

        return new JavaSerializer(cl, _loader);
    }

    /**
     * Returns the deserializer for a class.
     *
     * @param cl the class of the object that needs to be deserialized.
     * @return a deserializer object for the serialization.
     */
    @Override
    public Deserializer getDeserializer(Class cl)
            throws HessianProtocolException {
        Deserializer deserializer;

        deserializer = (Deserializer) _staticDeserializerMap.get(cl);
        if (deserializer != null)
            return deserializer;

        if (_cachedDeserializerMap != null) {
            deserializer = (Deserializer) _cachedDeserializerMap.get(cl);
            if (deserializer != null)
                return deserializer;
        }


        for (int i = 0;
             deserializer == null && _factories != null && i < _factories.size();
             i++) {
            AbstractSerializerFactory factory;
            factory = (AbstractSerializerFactory) _factories.get(i);

            deserializer = factory.getDeserializer(cl);
        }

        if (deserializer != null) {
        } else if (Collection.class.isAssignableFrom(cl))
            deserializer = new CollectionDeserializer(cl);

        else if (Map.class.isAssignableFrom(cl))
            deserializer = new MapDeserializer(cl);

        else if (cl.isInterface())
            deserializer = new ObjectDeserializer(cl);

        else if (cl.isArray())
            deserializer = new ArrayDeserializer(cl.getComponentType());

        else if (Enumeration.class.isAssignableFrom(cl))
            deserializer = EnumerationDeserializer.create();

        else if (Enum.class.isAssignableFrom(cl))
            deserializer = new EnumDeserializer(cl);

        else if (Class.class.equals(cl))
            deserializer = new ClassDeserializer(_loader);

        else
            deserializer = getDefaultDeserializer(cl);

        if (_cachedDeserializerMap == null)
            _cachedDeserializerMap = new ConcurrentHashMap(8);

        _cachedDeserializerMap.put(cl, deserializer);

        return deserializer;
    }

    /**
     * Returns the default serializer for a class that isn't matched
     * directly.  Application can override this method to produce
     * bean-style serialization instead of field serialization.
     *
     * @param cl the class of the object that needs to be serialized.
     * @return a serializer object for the serialization.
     */
    protected Deserializer getDefaultDeserializer(Class cl) {
        return new JavaDeserializer(cl);
    }

    /**
     * Reads the object as a list.
     */
    public Object readList(AbstractHessianInput in, int length, String type)
            throws HessianProtocolException, IOException {
        Deserializer deserializer = getDeserializer(type);

        if (deserializer != null)
            return deserializer.readList(in, length);
        else
            return new CollectionDeserializer(ArrayList.class).readList(in, length);
    }
    
    /**
     * Reads the object as a map.
     */
    public Object readMap(AbstractHessianInput in, String type, Class<?> expectKeyType, Class<?> expectValueType)
            throws HessianProtocolException, IOException {
        Deserializer deserializer = getDeserializer(type);

        if (deserializer != null)
            return deserializer.readMap(in);
        else if (_hashMapDeserializer != null)
            return _hashMapDeserializer.readMap(in, expectKeyType, expectValueType);
        else {
            _hashMapDeserializer = new MapDeserializer(HashMap.class);

            return _hashMapDeserializer.readMap(in, expectKeyType, expectValueType);
        }
    }

   //...

    
}
```

+ 实现类
1. BeanSerializerFactory


```java
public class BeanSerializerFactory extends SerializerFactory {
    /**
     * Returns the default serializer for a class that isn't matched
     * directly.  Application can override this method to produce
     * bean-style serialization instead of field serialization.
     *
     * @param cl the class of the object that needs to be serialized.
     * @return a serializer object for the serialization.
     */
    @Override
    protected Serializer getDefaultSerializer(Class cl) {
        return new BeanSerializer(cl, getClassLoader());
    }

    /**
     * Returns the default deserializer for a class that isn't matched
     * directly.  Application can override this method to produce
     * bean-style serialization instead of field serialization.
     *
     * @param cl the class of the object that needs to be serialized.
     * @return a serializer object for the serialization.
     */
    @Override
    protected Deserializer getDefaultDeserializer(Class cl) {
        return new BeanDeserializer(cl);
    }
}
```

2. Hessian2SerializerFactory

```java
public class Hessian2SerializerFactory extends SerializerFactory {

    public static final SerializerFactory SERIALIZER_FACTORY = new Hessian2SerializerFactory();

    private Hessian2SerializerFactory() {
    }

    @Override
    public ClassLoader getClassLoader() {
        return Thread.currentThread().getContextClassLoader();
    }

}
```

3. ExtSerializerFactory
```java
public class ExtSerializerFactory extends AbstractSerializerFactory {
    private HashMap _serializerMap = new HashMap();
    private HashMap _deserializerMap = new HashMap();

    /**
     * Adds a serializer.
     *
     * @param cl         the class of the serializer
     * @param serializer the serializer
     */
    public void addSerializer(Class cl, Serializer serializer) {
        _serializerMap.put(cl, serializer);
    }

    /**
     * Adds a deserializer.
     *
     * @param cl           the class of the deserializer
     * @param deserializer the deserializer
     */
    public void addDeserializer(Class cl, Deserializer deserializer) {
        _deserializerMap.put(cl, deserializer);
    }

    /**
     * Returns the serializer for a class.
     *
     * @param cl the class of the object that needs to be serialized.
     * @return a serializer object for the serialization.
     */
    @Override
    public Serializer getSerializer(Class cl)
            throws HessianProtocolException {
        return (Serializer) _serializerMap.get(cl);
    }

    /**
     * Returns the deserializer for a class.
     *
     * @param cl the class of the object that needs to be deserialized.
     * @return a deserializer object for the serialization.
     */
    @Override
    public Deserializer getDeserializer(Class cl)
            throws HessianProtocolException {
        return (Deserializer) _deserializerMap.get(cl);
    }
}
```