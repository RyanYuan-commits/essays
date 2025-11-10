---
type: Java
sub-type: Framework
topic: Dubbo
---
Dubbo 为了支持多种 [[常见的序列化算法|序列化算法]], 单独抽象了一层 Serialize 层, 其在整个 Dubbo 架构中属于最底层, 对应的模块是 dubbo-serialization 模块:

![[Dubbo Serialization Module.png]]

## 1	核心接口: Serialization

```java
/**
 * Serialization strategy interface that specifies a serializer. (SPI, Singleton, ThreadSafe)
 *
 * The default extension is hessian2 and the default serialization implementation of the dubbo protocol.
 * <pre>
 *     e.g. <dubbo:protocol serialization="xxx" />
 * </pre>
 */
@SPI(scope = ExtensionScope.FRAMEWORK)
public interface Serialization {

    /**
     * Get content type unique id, recommended that custom implementations use values different with
     * any value of {@link Constants} and don't greater than ExchangeCodec.SERIALIZATION_MASK (31)
     * because dubbo protocol use 5 bits to record serialization ID in header.
     *
     * @return content type id
     */
    byte getContentTypeId();

    /**
     * Get content type
     * 每一种序列化算法都对应一个ContentType, 该方法用于获取ContentType
     *
     * @return content type
     */
    String getContentType();

    /**
     * Get a serialization implementation instance
	 * 创建一个 ObjectOutput 对象, 这个对象用于实现序列化功能, 将 Java 对象转化为字节序列
	 *
     * @param url URL address for the remote service
     * @param output the underlying output stream
     * @return serializer
     * @throws IOException
     */
    @Adaptive
    ObjectOutput serialize(URL url, OutputStream output) throws IOException;

    /**
     * Get a deserialization implementation instance
     * 创建一个 ObjectInput 对象, 负责实现反序列化功能, 将字节序列转化为 Java 对象
     *
     * @param url URL address for the remote service
     * @param input the underlying input stream
     * @return deserializer
     * @throws IOException
     */
    @Adaptive
    ObjectInput deserialize(URL url, InputStream input) throws IOException;
}
```

Serialization 接口是 Dubbo 序列化层最核心的接口, 被 @SPI 修饰, 是一个拓展接口, 默认的实现是 Hessian2Serialization, Dubbo 提供了多个 Serialization 接口的实现, 用于接入各种各样的序列化算法, 比如 Hessian2Serialization, AvroSerialization, FastJsonSerialization 等等.

## 2	DataOutput & ObjectOutput

DataOutput 接口中定义了序列化 Java 各种数据结构的相应方法: 

![[Dubbo DataOutput Interface.png]]

ObjectOutput 接口继承了 DataOutput 接口, 并在其基础上拓展了序列化对象的功能, 其中的 writeThrowable, writeEvent, writeAttachment 都是是使用 writeObject 方法来实现的. 

![[Dubbo ObjectOutput Interface.png]]

## 3	Hessian2 Implementation

### 3.1	Hessian2Serialization

```java
public class Hessian2Serialization implements Serialization {

    private static final Logger logger = LoggerFactory.getLogger(Hessian2Serialization.class);

    static {
        Class<?> aClass = null;
        try {
            aClass = com.alibaba.com.caucho.hessian.io.Hessian2Output.class;
        } catch (Throwable ignored) { }
    
        if (aClass == null) {
            logger.info(
                    "Failed to load com.alibaba.com.caucho.hessian.io.Hessian2Output, hessian2 serialization will be disabled.");
            throw new IllegalStateException("The hessian2 is not in classpath.");
        }
    }

    @Override
    public byte getContentTypeId() {
        return HESSIAN2_SERIALIZATION_ID;
    }

    @Override
    public String getContentType() {
        return "x-application/hessian2";
    }

    @Override
    public ObjectOutput serialize(URL url, OutputStream out) throws IOException {
        Hessian2FactoryManager hessian2FactoryManager = Optional.ofNullable(url)
                .map(URL::getOrDefaultFrameworkModel)
                .orElseGet(FrameworkModel::defaultModel)
                .getBeanFactory()
                .getBean(Hessian2FactoryManager.class);
        return new Hessian2ObjectOutput(out, hessian2FactoryManager);
    }

    @Override
    public ObjectInput deserialize(URL url, InputStream is) throws IOException {
        Hessian2FactoryManager hessian2FactoryManager = Optional.ofNullable(url)
                .map(URL::getOrDefaultFrameworkModel)
                .orElseGet(FrameworkModel::defaultModel)
                .getBeanFactory()
                .getBean(Hessian2FactoryManager.class);
        return new Hessian2ObjectInput(is, hessian2FactoryManager);
    }
}
```

### 3.2	Hessian2ObjectOutput

Hessian2ObjectOutput 内部封装了一个 Hessian2Output, 这个对象与线程绑定, Hessian2ObjectOutput 中的序列化方法都是委托给其完成的.

```java
/**
 * Hessian2 object output implementation
 */
public class Hessian2ObjectOutput implements ObjectOutput, Cleanable {

    private final Hessian2Output mH2o;

    public Hessian2ObjectOutput(OutputStream os, Hessian2FactoryManager hessian2FactoryManager) {
        mH2o = new Hessian2Output(os);
        mH2o.setSerializerFactory(hessian2FactoryManager.getSerializerFactory(
                Thread.currentThread().getContextClassLoader()));
    }

    @Override
    public void writeBool(boolean v) throws IOException {
        mH2o.writeBoolean(v);
    }

    @Override
    public void writeByte(byte v) throws IOException {
        mH2o.writeInt(v);
    }

    @Override
    public void writeShort(short v) throws IOException {
        mH2o.writeInt(v);
    }

    @Override
    public void writeInt(int v) throws IOException {
        mH2o.writeInt(v);
    }

    @Override
    public void writeLong(long v) throws IOException {
        mH2o.writeLong(v);
    }

    @Override
    public void writeFloat(float v) throws IOException {
        mH2o.writeDouble(v);
    }

    @Override
    public void writeDouble(double v) throws IOException {
        mH2o.writeDouble(v);
    }

    @Override
    public void writeBytes(byte[] b) throws IOException {
        mH2o.writeBytes(b);
    }

    @Override
    public void writeBytes(byte[] b, int off, int len) throws IOException {
        mH2o.writeBytes(b, off, len);
    }

    @Override
    public void writeUTF(String v) throws IOException {
        mH2o.writeString(v);
    }

    @Override
    public void writeObject(Object obj) throws IOException {
        mH2o.writeObject(obj);
    }

    @Override
    public void flushBuffer() throws IOException {
        mH2o.flushBuffer();
    }

    public OutputStream getOutputStream() throws IOException {
        return mH2o.getBytesOutputStream();
    }

    @Override
    public void cleanup() {
        if (mH2o != null) {
            mH2o.reset();
        }
    }
}
```