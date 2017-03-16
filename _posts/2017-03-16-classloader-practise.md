---
layout: post
title:  "Java classloader example"
date:   "2016-03-10 17:50:00"
categories: java
keywords: classloader, java, tomcat
---


Storm 上传 Job 的过程分为两步，第一步是上传 Jar 包，第二部是上传 Serialized 后的 Bolt 和 Spout. 任务分配完毕后 Supervisor 会创建 Worker，然后把 jar 包放到 worker 的 classpath 中，放到 classpath 后 AppClassloader 就能访问到 external jar 的 class 文件了，所以 Storm 不需要创建自己的 classloader.

而 Tomcat 的做法就复杂些，当一个 job 提交后，它会加载这个 jar 包里的 Webapp 并执行，因为这个 jar 并不在 classpath 上，所以需要使用自己创立的 classloader 创建。此外，不同的 Webapp 有不同的 classloader 安全性也高一些。所以，Tocmat 是在运行期加载 jar 包，用法要高级一些。

### 例子

下面这个例子首先会把 guava 的 HostAndPort 序列化以后放到文件中，然后手动把 guava.jar 从 pom.xml 中删掉，接下来对 guava 任何类的使用都会导致编译失败。接下来，创建一个 URLClassloader, 其 URL 指向网络上的一个 guava.jar, 然后用这个 classloader 来反序列化 HostAndPort 类型并用反射来调用它的方法，方法调用成功表示 classloader 的理解是正确的。

```java
public class CallMethodFromSerializedObj {
    static String filePath = SystemInfo.getProjectPath() + "/" + "hostAndPort.serialized";
    
    // create and serialize object
    public static void step1() throws Exception {
        DefaultSerializer defaultSerializer = new DefaultSerializer();
        DefaultDeserializer defaultDeserializer = new DefaultDeserializer();
        
        HostAndPort hp = HostAndPort.fromString("localhost:80");
        defaultSerializer.serialize(hp, new FileOutputStream(filePath));
        Object deserized = defaultDeserializer.deserialize(new FileInputStream(filePath));
        HostAndPort hp_cp = (HostAndPort) deserized;
        System.out.println(hp_cp.getHostText());
    }
    
    // remove guava.jar from pom.xml and execute step2, expecting ClassNotFoundException
    public static void step2() throws Exception {
        byte[] result_cp = FileUtils.readFileToByteArray(new File(filePath));
        Object hp_cp = SerializationUtils.deserialize(result_cp);
    }    
    
    public static void step3() throws Exception {
        String jarPath = "http://central.maven.org/maven2/com/google/guava/guava/19.0/guava-19.0.jar";
        ClassLoader guavaCl = new URLClassLoader(new URL[]{new URL(jarPath)});

        DefaultDeserializer defaultDeserializer = new DefaultDeserializer(guavaCl);

        Object deserialized = defaultDeserializer.deserialize(new FileInputStream(CallMethodFromSerializedObj.filePath));

        System.out.println(deserialized.getClass().getName());

        Class hostClz = Class.forName("com.google.common.net.HostAndPort", true, guavaCl);
        Method getHost = hostClz.getMethod("getHostText");

        System.out.println(getHost.invoke(deserialized));
    }
}
```

Step1 是一个测试方法，测试序列化和反序列化是成功的，这里使用的是 SpringFramework 的反序列化工具。

Step2 执行之前需要把 guava.jar 从 pom.xml 中去掉，执行 step2 会抛出异常，如果没抛出的话，检查是否有其他的 jar 包包含 guava.jar

Step3 是在 guava.jar 从 pom.xml 中去掉后，试图从 remote jar 中反序列化出 HostAndPort 并执行其方法

### deserialize with classloader

在使用 spring Deserializer 之前，尝试过 Apache common SerializationUtils 类，它提供一个很简便的序列化方法，但是反序列化方法不接收 
classloader 作为参数，这样，就不能反序列化成功。为了使 SerializationUtils 的 classloader 为 URLClassloader

```java
    public static void step3() throws Exception {
        String jarPath = "http://central.maven.org/maven2/com/google/guava/guava/19.0/guava-19.0.jar";
        ClassLoader guavaCl = new URLClassLoader(new URL[]{new URL(jarPath)});

        Class hostClz_verify = Class.forName("com.google.common.net.HostAndPort", true, guavaCl);
        System.out.println(hostClz_verify.getName());

        // create SerializationUtils, make urlCl its classloader, failed because parent classloader enjoys higher priority
        Class serializeClz = Class.forName("org.apache.commons.lang3.SerializationUtils", true, guavaCl);
        System.out.println("class loader of SerializationUtils is " + serializeClz.getClassLoader().toString());

        Method deserializeMtd = serializeClz.getMethod("deserialize", byte[].class);

        byte[] result_cp = FileUtils.readFileToByteArray(new File(filePath));

        // it should be able to find the
        Object hostAndPort = deserializeMtd.invoke(null, result_cp);

        Class hostClz = Class.forName("com.google.common.net.HostAndPort", true, guavaCl);
        Method getHost = hostClz.getMethod("getHostText");

        getHost.invoke(hostAndPort);

    }
```

这样并不成功，即便生成 SerializationUtils 类时使用了反射且指定了 classloader, 但是这个类还是使用 AppClassloader 加载的，所以再用它调用
deserialize 方法时，caller classloader 还是 AppClassloader, 因为 jar 不在 classpath 所以肯定要失败。

查看 Apache SerializationUtils 的加锁方式

```java
    public static <T> T deserialize(final InputStream inputStream) {
        if (inputStream == null) {
            throw new IllegalArgumentException("The InputStream must not be null");
        }
        ObjectInputStream in = null;
        try {
            // stream closed in the finally
            in = new ObjectInputStream(inputStream);
            @SuppressWarnings("unchecked")
            final T obj = (T) in.readObject();
            return obj;

        } catch (final ClassNotFoundException ex) {
            throw new SerializationException(ex);
        } catch (final IOException ex) {
            throw new SerializationException(ex);
        } finally {
            try {
                if (in != null) {
                    in.close();
                }
            } catch (final IOException ex) { // NOPMD
                // ignore close exception
            }
        }
    }
```

ObjectInputStream 可以封装成 ClassloaderAwareInputStream, 以前在哪个地方看到过，但是忘记了

再看看 DefaultDeserialization 的反序列化方法

```java
public Object deserialize(InputStream inputStream) throws IOException {
    ObjectInputStream objectInputStream = new ConfigurableObjectInputStream(inputStream, this.classLoader);
	try {
	    return objectInputStream.readObject();
    }
	catch (ClassNotFoundException ex) {
	    throw new NestedIOException("Failed to deserialize object type", ex);
    }
}
	
public class ConfigurableObjectInputStream extends ObjectInputStream {
	@Override
	protected Class<?> resolveClass(ObjectStreamClass classDesc) throws IOException, ClassNotFoundException {
		try {
			if (this.classLoader != null) {
				// Use the specified ClassLoader to resolve local classes.
				return ClassUtils.forName(classDesc.getName(), this.classLoader);
			}
			else {
				// Use the default ClassLoader...
				return super.resolveClass(classDesc);
			}
		}
		catch (ClassNotFoundException ex) {
			return resolveFallbackIfPossible(classDesc.getName(), ex);
		}
	}

	public static Class<?> forName(String name, ClassLoader classLoader) throws ClassNotFoundException, LinkageError {
		ClassLoader clToUse = classLoader;
		if (clToUse == null) {
			clToUse = getDefaultClassLoader();
		}
		
		return (clToUse != null ? clToUse.loadClass(name) : Class.forName(name));	    
	}
}
```

从上面上面的类可以看出，ObjectInputStream 中实际上是有 resolveClass 这个方法的，考虑到 Apache SerializationUtils 类的参数也是 InputStream,
所以，Apache SerializationUtils 实际上也是可以添加 classloader 的。只要确保传递的 inputStream 是 classloader aware 的即可

ObjectInputStream 解析过程

```java
    private ObjectStreamClass readClassDesc(boolean unshared)
        throws IOException
    {
        byte tc = bin.peekByte();
        ObjectStreamClass descriptor;
        switch (tc) {
            case TC_NULL:
                descriptor = (ObjectStreamClass) readNull();
                break;
            case TC_REFERENCE:
                descriptor = (ObjectStreamClass) readHandle(unshared);
                break;
            case TC_PROXYCLASSDESC:
                descriptor = readProxyDesc(unshared);
                break;
            case TC_CLASSDESC:
                descriptor = readNonProxyDesc(unshared);
                break;
            default:
                throw new StreamCorruptedException(
                    String.format("invalid type code: %02X", tc));
        }
        if (descriptor != null) {
            validateDescriptor(descriptor);
        }
        return descriptor;
    }
    
    private ObjectStreamClass readNonProxyDesc(boolean unshared)
        throws IOException
    {
        if (bin.readByte() != TC_CLASSDESC) {
            throw new InternalError();
        }

        ObjectStreamClass desc = new ObjectStreamClass();
        int descHandle = handles.assign(unshared ? unsharedMarker : desc);
        passHandle = NULL_HANDLE;

        ObjectStreamClass readDesc = null;
        try {
            readDesc = readClassDescriptor();
        } catch (ClassNotFoundException ex) {
            throw (IOException) new InvalidClassException(
                "failed to read class descriptor").initCause(ex);
        }

        Class<?> cl = null;
        ClassNotFoundException resolveEx = null;
        bin.setBlockDataMode(true);
        final boolean checksRequired = isCustomSubclass();
        try {
            if ((cl = resolveClass(readDesc)) == null) {                     // !!! resolve class here
                resolveEx = new ClassNotFoundException("null class");
            } else if (checksRequired) {
                ReflectUtil.checkPackageAccess(cl);
            }
        } catch (ClassNotFoundException ex) {
            resolveEx = ex;
        }
        skipCustomData();

        desc.initNonProxy(readDesc, cl, resolveEx, readClassDesc(false));

        handles.finish(descHandle);
        passHandle = descHandle;
        return desc;
    }
```

找到类名后，会根据类名 resolve class loader, 在 spring 中，resolve class loader 重写成上面的代码了。

```java
ClassUtils.forName(classDesc.getName(), this.classLoader);

ClassLoader clToUse = classLoader;
    if (clToUse == null) {
	    clToUse = getDefaultClassLoader();
    }
	
    try {
	    return (clToUse != null ? clToUse.loadClass(name) : Class.forName(name));
    }
```


### 意义

实现了这个, 就可以写一个简单的 Tomcat 了