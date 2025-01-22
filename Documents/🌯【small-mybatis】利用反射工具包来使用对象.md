# 目标
我们在实现数据源池化时，对于属性信息的获取，采用的是硬编码的方式。
![[【small-mybatis】利用反射工具包来使用对象 硬编码获取.png]]
- 也就是 `props.getProperty("driver")`、`props.getProperty("url")` 等属性，都是通过手动编码的方式获取的。
- 那其实像 `driver`、`url`、`username`、`password` 不都是标准的固定字段吗，这样获取有什么不对的。如果按照我们现在的理解来说，并没有什么不对，但其实除了这些字段以外，可能还有时候会配置一些扩展字段，那么怎么获取呢，总不能每次都是硬编码。
- 所以如果你有阅读 `Mybatis` 的源码，会发现这里使用了 `Mybatis` 自己实现的元对象反射工具类，可以完成一个对象的属性的反射填充。这块的工具类叫做 `MetaObject` 并提供相应的；元对象、对象包装器、对象工厂、对象包装工厂以及 `Reflector` 反射器的使用。那么本章节我们就来实现一下反射工具包的内容，因为随着我们后续的开发，也会有很多地方都需要使用反射器优雅的处理我们的属性信息。**这也能为你添加一些关于反射的强大的使用！**
# 设计
如果说我们需要对一个对象的所提供的属性进行统一的设置和获取值的操作，那么就需要把当前这个被处理的对象进行解耦，提取出它所有的属性和方法，并按照不同的类型进行反射处理，从而包装成一个工具包。
![[【small-mybatis】利用反射工具包来使用对象 核心架构.png|800]]
- 其实整个设计过程都以围绕如何拆解对象并提供反射操作为主，那么对于一个对象来说，它所包括的有对象的构造函数、对象的属性、对象的方法。而对象的方法因为都是获取和设置值的操作，所以基本都是get、set处理，所以需要把这些方法在对象拆解的过程中需要摘取出来进行保存。
- 当真正的开始操作时，则会依赖于已经实例化的对象，对其进行属性处理。而这些处理过程实际都是在使用 JDK 所提供的反射进行操作的，而反射过程中的方法名称、入参类型都已经被我们拆解和处理了，最终使用的时候直接调用即可。
# 实现
## 工程结构
![[【small-mybatis】利用反射工具包来使用对象 核心类图.png|900]]
- 以 Reflector 反射器类处理对象类中的 get/set 属性，包装为可调用的 Invoker 反射类，这样在对 get/set 方法反射调用的时候，使用方法名称获取对应的 Invoker 即可 `getGetInvoker(String propertyName)`。
- 有了反射器的处理，之后就是对原对象的包装了，由 SystemMetaObject 提供创建 MetaObject 元对象的方法，将我们需要处理的对象进行拆解和 ObjectWrapper 对象包装处理。因为一个对象的类型还需要进行一条细节的处理，以及属性信息的拆解，例如：`班级[0].学生.成绩` 这样一个类中的关联类的属性，则需要进行递归的方式拆解处理后，才能设置和获取属性值。
- 最终在 Mybatis 其他的地方就可以，有需要属性值设定时，就可以使用到反射工具包进行处理了。这里首当其冲的我们会把数据源池化中关于 Properties 属性的处理使用反射工具类进行改造。_参考本章节中对应的源码类_
## 反射调用者 Invoker
### 定义接口
```java
public interface Invoker {
    Object invoke(Object target, Object[] args) throws Exception;

    Class<?> getType();
}
```
### MethodInvoker
```java
public class MethodInvoker implements Invoker {

    private Class<?> type;
    private Method method;

    @Override
    public Object invoke(Object target, Object[] args) throws Exception {
        return method.invoke(target, args);
    }

}
```
- 提供方法反射调用处理，构造函数会传入对应的方法类型。
### GetFieldInvoker
```java
public class GetFieldInvoker implements Invoker {

    private Field field;

    @Override
    public Object invoke(Object target, Object[] args) throws Exception {
        return field.get(target);
    }

}
```
- getter 方法的调用者处理，因为get是有返回值的，所以直接对 Field 字段操作完后直接返回结果。
### SetFieldInvoker
```java
public class SetFieldInvoker implements Invoker {

    private Field field;

    @Override
    public Object invoke(Object target, Object[] args) throws Exception {
        field.set(target, args[0]);
        return null;
    }

}
```
- setter 方法的调用者处理，因为set只是设置值，所以这里就只返回一个 null 就可以了。
## 利用 Reflector 解耦对象
Reflector 反射器专门用于解耦对象信息的，只有把一个对象信息所含带的属性、方法以及关联的类都以此解析出来，才能满足后续对属性值的设置和获取。
```java
public class Reflector {

    private static boolean classCacheEnabled = true;

    private static final String[] EMPTY_STRING_ARRAY = new String[0];
    // 线程安全的缓存
    private static final Map<Class<?>, Reflector> REFLECTOR_MAP = new ConcurrentHashMap<>();

    private Class<?> type;
    // get 属性列表
    private String[] readablePropertyNames = EMPTY_STRING_ARRAY;
    // set 属性列表
    private String[] writeablePropertyNames = EMPTY_STRING_ARRAY;
    // set 方法列表
    private Map<String, Invoker> setMethods = new HashMap<>();
    // get 方法列表
    private Map<String, Invoker> getMethods = new HashMap<>();
    // set 类型列表
    private Map<String, Class<?>> setTypes = new HashMap<>();
    // get 类型列表
    private Map<String, Class<?>> getTypes = new HashMap<>();
    // 构造函数
    private Constructor<?> defaultConstructor;

    private Map<String, String> caseInsensitivePropertyMap = new HashMap<>();

    public Reflector(Class<?> clazz) {
        this.type = clazz;
        // 加入构造函数
        addDefaultConstructor(clazz);
        // 加入 getter
        addGetMethods(clazz);
        // 加入 setter
        addSetMethods(clazz);
        // 加入字段
        addFields(clazz);
        readablePropertyNames = getMethods.keySet().toArray(new String[getMethods.keySet().size()]);
        writeablePropertyNames = setMethods.keySet().toArray(new String[setMethods.keySet().size()]);
        for (String propName : readablePropertyNames) {
            caseInsensitivePropertyMap.put(propName.toUpperCase(Locale.ENGLISH), propName);
        }
        for (String propName : writeablePropertyNames) {
            caseInsensitivePropertyMap.put(propName.toUpperCase(Locale.ENGLISH), propName);
        }
    }
    
    // ... 省略处理方法
}
```
- `Reflector` 反射器类中提供了各类属性、方法、类型以及构造函数的保存操作，当调用反射器时会通过构造函数的处理，逐步从对象类中拆解出这些属性信息，便于后续反射使用。
- 读者在对这部分源码学习时，可以参考对应的类和这里的处理方法，这些方法都是一些对反射的操作，获取出基本的类型、方法信息，并进行整理存放。
## MetaClass 元类
```java
public class MetaClass {

    private Reflector reflector;

    private MetaClass(Class<?> type) {
        this.reflector = Reflector.forClass(type);
    }

    public static MetaClass forClass(Class<?> type) {
        return new MetaClass(type);
    }

    public String[] getGetterNames() {
        return reflector.getGetablePropertyNames();
    }

    public String[] getSetterNames() {
        return reflector.getSetablePropertyNames();
    }

    public Invoker getGetInvoker(String name) {
        return reflector.getGetInvoker(name);
    }

    public Invoker getSetInvoker(String name) {
        return reflector.getSetInvoker(name);
    }

		// ... 方法包装
}
```
- MetaClass 元类相当于是对我们需要处理对象的包装，解耦一个原对象，包装出一个元类。而这些元类、对象包装器以及对象工厂等，再组合出一个元对象。相当于说这些元类和元对象都是对我们需要操作的原对象解耦后的封装。有了这样的操作，就可以让我们处理每一个属性或者方法了。
## 对象包装器
```java
public interface ObjectWrapper {

    // get
    Object get(PropertyTokenizer prop);

    // set
    void set(PropertyTokenizer prop, Object value);

    // 查找属性
    String findProperty(String name, boolean useCamelCaseMapping);

    // 取得getter的名字列表
    String[] getGetterNames();

    // 取得setter的名字列表
    String[] getSetterNames();

    //取得setter的类型
    Class<?> getSetterType(String name);

    // 取得getter的类型
    Class<?> getGetterType(String name);

    // ... 省略

}
```
- 后续所有实现了对象包装器接口的实现类，都需要提供这些方法实现，基本有了这些方法，也就能非常容易的处理一个对象的反射操作了。
- 无论你是设置属性、获取属性、拿到对应的字段列表还是类型都是可以满足的。
## MetaObject 元对象
在有了反射器、元类、对象包装器以后，在使用对象工厂和包装工厂，就可以组合出一个完整的元对象操作类了。因为所有的不同方式的使用，包括：包装器策略、包装工程、统一的方法处理，这些都需要一个统一的处理方，也就是我们的元对象进行管理。
```java
public class MetaObject {
    // 原对象
    private Object originalObject;
    // 对象包装器
    private ObjectWrapper objectWrapper;
    // 对象工厂
    private ObjectFactory objectFactory;
    // 对象包装工厂
    private ObjectWrapperFactory objectWrapperFactory;

    private MetaObject(Object object, ObjectFactory objectFactory, ObjectWrapperFactory objectWrapperFactory) {
        this.originalObject = object;
        this.objectFactory = objectFactory;
        this.objectWrapperFactory = objectWrapperFactory;

        if (object instanceof ObjectWrapper) {
            // 如果对象本身已经是ObjectWrapper型，则直接赋给objectWrapper
            this.objectWrapper = (ObjectWrapper) object;
        } else if (objectWrapperFactory.hasWrapperFor(object)) {
            // 如果有包装器,调用ObjectWrapperFactory.getWrapperFor
            this.objectWrapper = objectWrapperFactory.getWrapperFor(this, object);
        } else if (object instanceof Map) {
            // 如果是Map型，返回MapWrapper
            this.objectWrapper = new MapWrapper(this, (Map) object);
        } else if (object instanceof Collection) {
            // 如果是Collection型，返回CollectionWrapper
            this.objectWrapper = new CollectionWrapper(this, (Collection) object);
        } else {
            // 除此以外，返回BeanWrapper
            this.objectWrapper = new BeanWrapper(this, object);
        }
    }

    public static MetaObject forObject(Object object, ObjectFactory objectFactory, ObjectWrapperFactory objectWrapperFactory) {
        if (object == null) {
            // 处理一下null,将null包装起来
            return SystemMetaObject.NULL_META_OBJECT;
        } else {
            return new MetaObject(object, objectFactory, objectWrapperFactory);
        }
    }
    
    // 取得值
    // 如 班级[0].学生.成绩
    public Object getValue(String name) {
        PropertyTokenizer prop = new PropertyTokenizer(name);
        if (prop.hasNext()) {
            MetaObject metaValue = metaObjectForProperty(prop.getIndexedName());
            if (metaValue == SystemMetaObject.NULL_META_OBJECT) {
                // 如果上层就是null了，那就结束，返回null
                return null;
            } else {
                // 否则继续看下一层，递归调用getValue
                return metaValue.getValue(prop.getChildren());
            }
        } else {
            return objectWrapper.get(prop);
        }
    }

    // 设置值
    // 如 班级[0].学生.成绩
    public void setValue(String name, Object value) {
        PropertyTokenizer prop = new PropertyTokenizer(name);
        if (prop.hasNext()) {
            MetaObject metaValue = metaObjectForProperty(prop.getIndexedName());
            if (metaValue == SystemMetaObject.NULL_META_OBJECT) {
                if (value == null && prop.getChildren() != null) {
                    // don't instantiate child path if value is null
                    // 如果上层就是 null 了，还得看有没有儿子，没有那就结束
                    return;
                } else {
                    // 否则还得 new 一个，委派给 ObjectWrapper.instantiatePropertyValue
                    metaValue = objectWrapper.instantiatePropertyValue(name, prop, objectFactory);
                }
            }
            // 递归调用setValue
            metaValue.setValue(prop.getChildren(), value);
        } else {
            // 到了最后一层了，所以委派给 ObjectWrapper.set
            objectWrapper.set(prop, value);
        }
    }
    
    // ... 省略
}    
```
## 使用元对象
```java
public class UnpooledDataSourceFactory implements DataSourceFactory {

    protected DataSource dataSource;

    public UnpooledDataSourceFactory() {
        this.dataSource = new UnpooledDataSource();
    }

    @Override
    public void setProperties(Properties props) {
        MetaObject metaObject = SystemMetaObject.forObject(dataSource);
        for (Object key : props.keySet()) {
            String propertyName = (String) key;
            if (metaObject.hasSetter(propertyName)) {
                String value = (String) props.get(propertyName);
                Object convertedValue = convertValue(metaObject, propertyName, value);
                metaObject.setValue(propertyName, convertedValue);
            }
        }
    }

    @Override
    public DataSource getDataSource() {
        return dataSource;
    }
    
}
```
- 在之前我们对于数据源中属性信息的获取都是采用的硬编码，那么这回在 setProperties 方法中则可以使用 SystemMetaObject.forObject(dataSource) 获取 DataSource 的元对象了，也就是通过反射就能把我们需要的属性值设置进去。
- 这样在数据源 UnpooledDataSource、PooledDataSource 中就可以拿到对应的属性值信息了，而不是我们那种在2个数据源的实现中硬编码操作。
