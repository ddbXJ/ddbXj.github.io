---
layout:     post
title:      "dubbo扩展点加载机制分析"
subtitle:   "dubbo extension loader"
date:       2017-11-26
author:     "lee"
header-img: ""
---

看`Dubbo`的源码,特别是debug的时候,会发现`ExtensionLoader`的出现频率非常高.刚开始的时候不怎么理解,但通过一遍遍的调试,结合官方的文档,才逐步发现`ExtensionLoader`的精妙.

从dubbo项目的启动谈起.dubbo的Main入口如下:
```
public class Main {

    public static final String CONTAINER_KEY = "dubbo.container";

    public static final String SHUTDOWN_HOOK_KEY = "dubbo.shutdown.hook";
    
    private static final Logger logger = LoggerFactory.getLogger(Main.class);

	//1. 最先加载
    private static final ExtensionLoader<Container> loader = ExtensionLoader.getExtensionLoader(Container.class);
    
    private static volatile boolean running = true;

    public static void main(String[] args) {
        try {
            if (args == null || args.length == 0) {
            	//2. 获取默认的container拓展名:spring
                String config = ConfigUtils.getProperty(CONTAINER_KEY, loader.getDefaultExtensionName());
                args = Constants.COMMA_SPLIT_PATTERN.split(config);
            }
            
            final List<Container> containers = new ArrayList<Container>();
            for (int i = 0; i < args.length; i ++) {
                containers.add(loader.getExtension(args[i]));
            }
            logger.info("Use container type(" + Arrays.toString(args) + ") to run dubbo serivce.");
            
            if ("true".equals(System.getProperty(SHUTDOWN_HOOK_KEY))) {
	            Runtime.getRuntime().addShutdownHook(new Thread() {
	                public void run() {
	                    for (Container container : containers) {
	                        try {
	                            container.stop();
	                            logger.info("Dubbo " + container.getClass().getSimpleName() + " stopped!");
	                        } catch (Throwable t) {
	                            logger.error(t.getMessage(), t);
	                        }
	                        synchronized (Main.class) {
	                            running = false;
	                            Main.class.notify();
	                        }
	                    }
	                }
	            });
            }
            
            for (Container container : containers) {
            	//3. 这里实际上是调用的SpringContainer.start()
                container.start();
                logger.info("Dubbo " + container.getClass().getSimpleName() + " started!");
            }
            System.out.println(new SimpleDateFormat("[yyyy-MM-dd HH:mm:ss]").format(new Date()) + " Dubbo service server started!");
        } catch (RuntimeException e) {
            e.printStackTrace();
            logger.error(e.getMessage(), e);
            System.exit(1);
        }
        synchronized (Main.class) {
            while (running) {
                try {
                    Main.class.wait();
                } catch (Throwable e) {
                }
            }
        }
    }
    
}
```
本次分析,就从上面注释的三个地方入手,来研究dubbo的扩展点加载机制

---

## 代码分析

### ExtensionLoader.getExtensionLoader(Container.class)

首先值得关注的是
`private static final ExtensionLoader<Container> loader = ExtensionLoader.getExtensionLoader(Container.class);`

`ExtensionLoader<T>`没有公共的构造函数,提供了 `public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type)`来获取对应class的`ExtensionLoader`(扩展点加载类)


```
//对外暴露的获取ExtensionLoader的方法
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    if (type == null) 
        throw new IllegalArgumentException("Extension type == null");
    if(!type.isInterface()) { 
        throw new IllegalArgumentException("Extension type(" + type + ") is not interface!");
    }
    if(!withExtensionAnnotation(type)) { //从这里能看到,type必须有SPI注解
        throw new IllegalArgumentException("Extension type(" + type + 
                ") is not extension, because WITHOUT @" + SPI.class.getSimpleName() + " Annotation!");
    }
    
    //ExtensionLoader的静态成员变量,也就是全局缓存: 
    //	private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS
    //所有加载过的ExtensionLoader,会缓存在这个map中
    ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);

    if (loader == null) {
    	//对于没有加载过的ExtensionLoader,先走私有构造函数,然后放入map中缓存
        EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
        loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    }
    return loader;
}

//私有构造函数
private ExtensionLoader(Class<?> type) {
    this.type = type;
    objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
}

```

下面这个图先给出`ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension()`的调用图解,后面代码具体分析
![extensionFactory](/img/extensionFactory.jpg)


先看`ExtensionLoader.getExtensionLoader(ExtensionFactory.class)`
`ExtensionFactory`本身被`SPI`标注,是生成扩展类的工厂: 指定`class type`和`name`,可以获取相应的Extension(T)
```
@SPI
public interface ExtensionFactory {

    <T> T getExtension(Class<T> type, String name);

}
```

对于`ExtensionLoader<ExtensionFactory>`实例,它的`type`是`ExtensionFactory`,`objectFactory`是`null`
对所有`type`不是`ExtensionFactory`的类,初始化他们的`objectFactory`变量时,都会调用`ExtensionLoader<ExtensionFactory>`实例的`getAdaptiveExtension()`方法
所以,在这里,初始化`ExtensionLoader(Container)`的时候,会先初始化`ExtensionLoader<ExtensionFactory>` : 它的`type`为`ExtensionFactory.class`,`objectFactory`为`null`


初始化完,回到`ExtensionLoader(Container)`的构造函数,会调用`ExtensionLoader<ExtensionFactory>`实例的`getAdaptiveExtension()`方法
一系列的方法调用如下:

```
//这个方法就是去获取自适应的扩展类
//默认从缓存中拿,第一次获取的时候,缓存中没有,则会初始化一次
public T getAdaptiveExtension() {
    Object instance = cachedAdaptiveInstance.get();
    if (instance == null) {
        if(createAdaptiveInstanceError == null) {
            synchronized (cachedAdaptiveInstance) {
                instance = cachedAdaptiveInstance.get();
                if (instance == null) {
                    try {
                    	//见下面
                        instance = createAdaptiveExtension();
                        cachedAdaptiveInstance.set(instance);
                    } catch (Throwable t) {
                        createAdaptiveInstanceError = t;
                        throw new IllegalStateException("fail to create adaptive instance: " + t.toString(), t);
                    }
                }
            }
        }
        else {
            throw new IllegalStateException("fail to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
        }
    }

    return (T) instance;
}

//初始化AdaptiveExtension
private T createAdaptiveExtension() {
    try {
        return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    } catch (Exception e) {
        throw new IllegalStateException("Can not create adaptive extenstion " + type + ", cause: " + e.getMessage(), e);
    }
} 

private Class<?> getAdaptiveExtensionClass() {
	//获取到所有扩展类,这个方法很重要,见下面
    getExtensionClasses();
    if (cachedAdaptiveClass != null) {
    	//如果当前SPI有直接注解了@Adaptive的实现类,则经过getExtensionClasses()方法之后,对应的自适应实现类应该已经被缓存
        return cachedAdaptiveClass;
    }
    //如果没有缓存,则需要通过生成java代码,来获取一个自适应的实现类
    //这个很重要,在后面会具体讲解
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}

private Map<String, Class<?>> getExtensionClasses() {
    Map<String, Class<?>> classes = cachedClasses.get();
    if (classes == null) {
        synchronized (cachedClasses) {
            classes = cachedClasses.get();
            if (classes == null) {
            	//如果扩展类还没有被缓存,则加载,见下面
                classes = loadExtensionClasses();
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}

private Map<String, Class<?>> loadExtensionClasses() {
    final SPI defaultAnnotation = type.getAnnotation(SPI.class);
    if(defaultAnnotation != null) {
        String value = defaultAnnotation.value();
        if(value != null && (value = value.trim()).length() > 0) {
            String[] names = NAME_SEPARATOR.split(value);
            if(names.length > 1) {
                throw new IllegalStateException("more than 1 default extension name on extension " + type.getName()
                        + ": " + Arrays.toString(names));
            }
            //这里能看到,cachedDefaultName会存下SPI里的默认值,例如:container的SPI默认标注是'spring'
            if(names.length == 1) cachedDefaultName = names[0];
        }
    }
    
    Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();

    //DUBBO_INTERNAL_DIRECTORY : "META-INF/dubbo/internal/"
    loadFile(extensionClasses, DUBBO_INTERNAL_DIRECTORY);

    //DUBBO_DIRECTORY : "META-INF/dubbo/"
    loadFile(extensionClasses, DUBBO_DIRECTORY);

    //SERVICES_DIRECTORY : "META-INF/services/"
    loadFile(extensionClasses, SERVICES_DIRECTORY);
    return extensionClasses;
}        

//这个方法超级重要,它真正加载了所有扩展类
private void loadFile(Map<String, Class<?>> extensionClasses, String dir) {
    String fileName = dir + type.getName();
    try {
        Enumeration<java.net.URL> urls;
        ClassLoader classLoader = findClassLoader();
        if (classLoader != null) {
            urls = classLoader.getResources(fileName);
        } else {
            urls = ClassLoader.getSystemResources(fileName);
        }
        if (urls != null) {
            while (urls.hasMoreElements()) {
                java.net.URL url = urls.nextElement();
                try {
                    BufferedReader reader = new BufferedReader(new InputStreamReader(url.openStream(), "utf-8"));
                    try {
                        String line = null;
                        while ((line = reader.readLine()) != null) {
                            final int ci = line.indexOf('#');
                            if (ci >= 0) line = line.substring(0, ci);
                            line = line.trim();
                            if (line.length() > 0) {
                                try {
                                    String name = null;
                                    int i = line.indexOf('=');
                                    if (i > 0) {
                                        name = line.substring(0, i).trim();
                                        line = line.substring(i + 1).trim();
                                    }
                                    if (line.length() > 0) {
                                        Class<?> clazz = Class.forName(line, true, classLoader);
                                        if (! type.isAssignableFrom(clazz)) {
                                            throw new IllegalStateException("Error when load extension class(interface: " +
                                                    type + ", class line: " + clazz.getName() + "), class " 
                                                    + clazz.getName() + "is not subtype of interface.");
                                        }
                                        if (clazz.isAnnotationPresent(Adaptive.class)) {
                                            if(cachedAdaptiveClass == null) {
                                            	//这里能看到,如果一个class上面标注了@Adaptive,则它是对应SPI的默认自适应实现,会缓存到cachedAdaptiveClass
                                                cachedAdaptiveClass = clazz;
                                            } else if (! cachedAdaptiveClass.equals(clazz)) {
                                            	//并且,默认自适应实现,只能有一个
                                                throw new IllegalStateException("More than 1 adaptive class found: "
                                                        + cachedAdaptiveClass.getClass().getName()
                                                        + ", " + clazz.getClass().getName());
                                            }
                                        } else {
                                            try {
                                            	//这里,对于配置的wrapper类,会缓存到cachedWrapperClasses
                                                clazz.getConstructor(type);
                                                Set<Class<?>> wrappers = cachedWrapperClasses;
                                                if (wrappers == null) {
                                                    cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
                                                    wrappers = cachedWrapperClasses;
                                                }
                                                wrappers.add(clazz);
                                            } catch (NoSuchMethodException e) {
                                            	//既不是自适应的实现类,也不是装饰类,则会走到这个分支
                                                clazz.getConstructor();
                                                if (name == null || name.length() == 0) {
                                                    name = findAnnotationName(clazz);
                                                    if (name == null || name.length() == 0) {
                                                        if (clazz.getSimpleName().length() > type.getSimpleName().length()
                                                                && clazz.getSimpleName().endsWith(type.getSimpleName())) {
                                                            name = clazz.getSimpleName().substring(0, clazz.getSimpleName().length() - type.getSimpleName().length()).toLowerCase();
                                                        } else {
                                                            throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + url);
                                                        }
                                                    }
                                                }
                                                String[] names = NAME_SEPARATOR.split(name);
                                                if (names != null && names.length > 0) {
                                                    Activate activate = clazz.getAnnotation(Activate.class);
                                                    if (activate != null) {
                                                    	//如果当前实现类注解了@Activate,则缓存到cachedActivates
                                                        cachedActivates.put(names[0], activate);
                                                    }
                                                    for (String n : names) {
                                                        if (! cachedNames.containsKey(clazz)) {
                                                            cachedNames.put(clazz, n);
                                                        }
                                                        Class<?> c = extensionClasses.get(n);
                                                        if (c == null) {
                                                            extensionClasses.put(n, clazz);
                                                        } else if (c != clazz) {
                                                            throw new IllegalStateException("Duplicate extension " + type.getName() + " name " + n + " on " + c.getName() + " and " + clazz.getName());
                                                        }
                                                    }
                                                }
                                            }
                                        }
                                    }
                                } catch (Throwable t) {
                                    IllegalStateException e = new IllegalStateException("Failed to load extension class(interface: " + type + ", class line: " + line + ") in " + url + ", cause: " + t.getMessage(), t);
                                    exceptions.put(line, e);
                                }
                            }
                        } // end of while read lines
                    } finally {
                        reader.close();
                    }
                } catch (Throwable t) {
                    logger.error("Exception when load extension class(interface: " +
                                        type + ", class file: " + url + ") in " + url, t);
                }
            } // end of while urls
        }
    } catch (Throwable t) {
        logger.error("Exception when load extension class(interface: " +
                type + ", description file: " + fileName + ").", t);
    }
}
```

通过上面的一系列方法调用可以看到,在第一次调用`ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension()`的时候,会走到
`loadExtensionClasses`这个方法,这个方法里,会调用`loadFile`,分别会去扫描jar包下的三个路径的的文件,然后进行加载:
```
META-INF/dubbo/internal/com.alibaba.dubbo.common.extension.ExtensionFactory

META-INF/dubbo/com.alibaba.dubbo.common.extension.ExtensionFactory

META-INF/services/com.alibaba.dubbo.common.extension.ExtensionFactory

```

在`dubbo-common`包里的对应路径文件里,存了`adaptive`和`spi`的两个扩展点配置
```
adaptive=com.alibaba.dubbo.common.extension.factory.AdaptiveExtensionFactory
spi=com.alibaba.dubbo.common.extension.factory.SpiExtensionFactory
```
`dubbo-config`包里的对应路径文件里,存了`spring`的扩展点配置
```
spring=com.alibaba.dubbo.config.spring.extension.SpringExtensionFactory
```

`cachedNames`会存`spring`和`spi`
`cachedClasses`是一个`Holder`,可以持有一个实例,会持有一个`name->class`的`map`:
> `sping -> SpringExtensionFactory`
  `spi -> SpiExtensionFactory`

`cachedAdaptiveClass`会存`AdaptiveExtensionFactory`
在`createAdaptiveExtension`方法里,会实例化`AdaptiveExtensionFactory`,下面来看下这个类

```
@Adaptive
public class AdaptiveExtensionFactory implements ExtensionFactory {
    
    //私有成员变量,是持有扩展点生成工厂的List
    private final List<ExtensionFactory> factories;
    
    public AdaptiveExtensionFactory() {
    	//先获取ExtensionFactory对应的扩展点加载类
        ExtensionLoader<ExtensionFactory> loader = ExtensionLoader.getExtensionLoader(ExtensionFactory.class);

        List<ExtensionFactory> list = new ArrayList<ExtensionFactory>();

        //getSupportedExtensions()会获取到前面加载过的`ExtensionClasses`,会从`cachedClasses`里获取,拿出`spi`和`spring`这两个字符串
        for (String name : loader.getSupportedExtensions()) {
        	//然后把`spi`和`spring`对应的ExtensionFactory获取
            list.add(loader.getExtension(name));
        }

        //所以factories持有了SpiExtensionFactory和SpringExtensionFactory
        factories = Collections.unmodifiableList(list);
    }

    public <T> T getExtension(Class<T> type, String name) {
    	//所以每个扩展点加载的时候,调用loader.getExtension(name),都会调用spi和spring对应的getExtension方法
        for (ExtensionFactory factory : factories) {
            T extension = factory.getExtension(type, name);
            if (extension != null) {
                return extension;
            }
        }
        return null;
    }

}



//class:ExtensionLoader
public Set<String> getSupportedExtensions() {
	//1. getExtensionClasses拿出的是string->class的Map
    Map<String, Class<?>> clazzes = getExtensionClasses();

    //2. clazzes.keySet()会拿出string对应的集合,在这里也就是"spi" + "spring"
    return Collections.unmodifiableSet(new TreeSet<String>(clazzes.keySet()));
}

//class:ExtensionLoader
public T getExtension(String name) {
	if (name == null || name.length() == 0)
	    throw new IllegalArgumentException("Extension name == null");
	if ("true".equals(name)) {
	    return getDefaultExtension();
	}
	Holder<Object> holder = cachedInstances.get(name);
	if (holder == null) {
	    cachedInstances.putIfAbsent(name, new Holder<Object>());
	    holder = cachedInstances.get(name);
	}
	Object instance = holder.get();
	if (instance == null) {
	    synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
            	//对于上面的loader.getExtension(name),第一次都会走到这里,name分别是'spi'和'spring'
                instance = createExtension(name);
                holder.set(instance);
            }
        }
	}
	return (T) instance;
}

//class:ExtensionLoader
private T createExtension(String name) {
	//从缓存的cachedClasses中获取
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
    	//尝试从全局的实例缓存中拿
        T instance = (T) EXTENSION_INSTANCES.get(clazz);

        if (instance == null) {
        	//拿不到,则实例化,再缓存起来
            EXTENSION_INSTANCES.putIfAbsent(clazz, (T) clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        injectExtension(instance);
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (wrapperClasses != null && wrapperClasses.size() > 0) {
            for (Class<?> wrapperClass : wrapperClasses) {
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
                type + ")  could not be instantiated: " + t.getMessage(), t);
    }
}
```
所以,到这里为止,`ExtensionLoader<Container>`的`objectFactory`就初始化完毕了,它持有了`AdaptiveExtensionFactory`的实例,然而这个实例的成员变量`factories`又持有了`SpiExtensionFactory`和`SpringExtensionFactory`的实例.



### loader.getDefaultExtensionName()


在`String config = ConfigUtils.getProperty(CONTAINER_KEY, loader.getDefaultExtensionName());`里
会走到ExtensionLoader<Container>的`loadExtensionClasses`方法,
该方法里,会加载所有`META-INF/dubbo/com.alibaba.dubbo.container.Container`里配置过的,如`spring`,`log4j`,`logback`三种容器
而由于`Container`的`SPI`注解里填了默认值`spring,所以`会把`spring`存到`cachedDefaultName`中
```
@SPI("spring")
public interface Container {
    
    /**
     * start.
     */
    void start();
    
    /**
     * stop.
     */
    void stop();

}
```


### container.start();

在这个方法被调用时,对于`SpringContainer`,便会加载所有配置过的bean到`ApplicationContext`

对于一个向外暴露接口的服务提供者,我们通常会声明成下面这样:
```
    <bean id="xxxService" class="com.alibaba.dubbo.config.spring.ServiceBean">
        <property name="interface" value="com.xxx.api.XXXService"/>
        <property name="ref" ref="xXXServiceImpl"/>
        <property name="application" ref="dubboApplicationConfig"/>
        <property name="registry" ref="dubboRegistryConfig"/>
        <property name="protocol" ref="dubboProtocolConfig"/>
        <property name="version" value="${dubbo.reference.version}"/>
        <property name="timeout" value="${dubbo.export.timeout}"/>
        <property name="retries" value="0"/>
    </bean>
```
它作为`ServiceBean`,在spring的初始化时,又会触发一系列ExtensionLoader的加载

```
ServiceBean<T> extends ServiceConfig<T>

public class ServiceConfig<T> extends AbstractServiceConfig {

	...

    private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
    
    private static final ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();

    ...
}
```
`ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();`
这句话加载了`Protocol`的`ExtensionLoader`,然后在调用`getAdaptiveExtension()`方法时,由于`Protocol`没有实现类有声明`Adaptive`
所以在`getAdaptiveExtension` -> `createAdaptiveExtension` -> `getAdaptiveExtensionClass` -> `getExtensionClasses` -> `loadExtensionClasses` -> `loadFile`
这条链路里,`cachedDefaultName`会被填充为`dubbo`(因为Protocol的SPI注解里给了默认值),`cachedClasses`会缓存所有`Protocol`的实现类
但是由于没有类有声明`@Adaptive`,所以会走到`createAdaptiveExtensionClass`,动态生成一个实现类,生成后的代码如下:

```
package com.alibaba.dubbo.rpc;

import com.alibaba.dubbo.common.extension.ExtensionLoader;

public class Protocol$Adpative implements com.alibaba.dubbo.rpc.Protocol {

    public void destroy() {
        throw new UnsupportedOperationException(
            "method public abstract void com.alibaba.dubbo.rpc.Protocol.destroy() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive " 
                + "method!");
    }

    public int getDefaultPort() {
        throw new UnsupportedOperationException(
            "method public abstract int com.alibaba.dubbo.rpc.Protocol.getDefaultPort() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive" 
                + " method!");
    }

    public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.RpcException {
        if (arg0 == null) { throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null"); }
        if (arg0.getUrl() == null) { throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null"); }
        com.alibaba.dubbo.common.URL url = arg0.getUrl();
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null) {
            throw new IllegalStateException(
                "Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        }
        com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(
            com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.export(arg0);
    }

    public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0, com.alibaba.dubbo.common.URL arg1) throws com.alibaba.dubbo.rpc.RpcException {
        if (arg1 == null) { throw new IllegalArgumentException("url == null"); }
        com.alibaba.dubbo.common.URL url = arg1;
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null) {
            throw new IllegalStateException(
                "Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        }
        com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(
            com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.refer(arg0, arg1);
    }
}


//参照官方的生成规则
package <扩展点接口所在包>;  
  
public class <扩展点接口名>$Adpative implements <扩展点接口> {  
    public <有@Adaptive注解的接口方法>(<方法参数>) {  
        if(是否有URL类型方法参数?) 使用该URL参数  
        else if(是否有方法类型上有URL属性) 使用该URL属性  
        # <else 在加载扩展点生成自适应扩展点类时抛异常，即加载扩展点失败！>  
  
        if(获取的URL == null) {  
            throw new IllegalArgumentException("url == null");  
        }  
  
        根据@Adaptive注解上声明的Key的顺序，从URL获致Value，作为实际扩展点名。  
        如URL没有Value，则使用缺省扩展点实现。如没有扩展点， throw new IllegalStateException("Fail to get extension");  
  
        在扩展点实现调用该方法，并返回结果。  
    }  
  
    public <有@Adaptive注解的接口方法>(<方法参数>) {  
        throw new UnsupportedOperationException("is not adaptive method!");  
    }  
}  
```
这里可以看出,自动生成的代码,会从dubbo规约的`URL`上去拿指定的`key`对应的`value`,然后把`value`作为`getExtension`时候的`name`,所以就实现了,配置了什么,实际运行的时候,就加载的是什么.

所以看到这里,对官方文档里所谓的`扩展点自适应`便有了更清晰的认识:
一个接口的实现者可能有多个，并不直接去注入一个具体的实现者，
而是注入一个动态生成的实现者，这个动态生成的实现者的逻辑是确定的，能够根据不同的参数来使用不同的实现者实现相应的方法。

dubbo在整体架构上定义了一系列的spi接口,并给出了所有默认的实现和可选的各种方案,开发者可以根据情况自选,也可以自己开发.灵活性很高


---

## 关于dubbo的缓存

dubbo框架里大量使用了缓存,可以通过`ExtensionLoader`大致看一下:


```
//看一下ExtensionLoader的静态成员变量,这是全局的缓存:

	//所有标注了SPI注解的接口对应的`ExtensionLoader`
    private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<Class<?>, ExtensionLoader<?>>();

    //所有被加载过的扩展类的实例缓存
    private static final ConcurrentMap<Class<?>, Object> EXTENSION_INSTANCES = new ConcurrentHashMap<Class<?>, Object>();


//看一下ExtensionLoader的非静态成员变量,这是针对每个ExtensionLoader实例的缓存:

	//SPI类型
    private final Class<?> type;

    //拓展类工厂: 指定type和name,可以获取相应的Extension(T)
    private final ExtensionFactory objectFactory;

    //拓展类(extension)的名字(name)标识
    private final ConcurrentMap<Class<?>, String> cachedNames = new ConcurrentHashMap<Class<?>, String>();
    
    //缓存的extension实例
    private final Holder<Map<String, Class<?>>> cachedClasses = new Holder<Map<String,Class<?>>>();

    //可自动激活的实现类,name->Activate,其中Activate包含了这个自动激活的类的各种信息,比如时序
    private final Map<String, Activate> cachedActivates = new ConcurrentHashMap<String, Activate>();

    //自适应实现类的class,如果一个接口有实现类直接注解了@Adaptive,则就是那个类;如果是自适应的,就是动态创建的类
    private volatile Class<?> cachedAdaptiveClass = null;

    //name->holder的缓存map,holder持有的是extension实例
    private final ConcurrentMap<String, Holder<Object>> cachedInstances = new ConcurrentHashMap<String, Holder<Object>>();

    //接口的spi里注解的默认值
    private String cachedDefaultName;

    //被缓存的自适应实例Holder,Holder可以获取对应的实例
    private final Holder<Object> cachedAdaptiveInstance = new Holder<Object>();
    private volatile Throwable createAdaptiveInstanceError;

    //缓存的wrapper类
    private Set<Class<?>> cachedWrapperClasses;
    
    private Map<String, IllegalStateException> exceptions = new ConcurrentHashMap<String, IllegalStateException>();

```

---

## 应用

在我们业务中,比较常用的,是基于常用的一些操作来写`dubbo filter`.

dubbo的扩展点加载机制,类似于SPI机制,都是基于固定的路径来约定,一般自己添加的扩展类,都放在自己jar包下的`META-INF/dubbo/接口全限定名`里,然后命名需要符合规范
例如,我们自己添加一个`dubbo filter`,实现了dubbo的`Filter`接口,然后需要声明在自己项目的固定目录下,
```
META-INF/dubbo/com.alibaba.dubbo.rpc.Filter : 
xxx=com.abc.XxxFilter
```
基于对源码的整体认识,这里也不难理解.每个自己写的`Filter`,会在实现类上注解`@Activate`,在加载的时候,会被加载到`ExtensionLoader<Filter>`实例中,`cachedNames`会缓存该实现类上注解的`name`,`cachedActivates`会缓存`name`到`@Activate`的关系,`cachedClasses`会缓存这个实现类的实例.
然后,`ProtocolFilterWrapper`这个类,会对`Filter`做一个包装,通过`buildInvokerChain`这个方法,构造了一个过滤的链.并在`Protocol`类的`export`和`import`方法里,都会包装进去.所以,就实现了对`provider`和`consumer`的`filter`.






