---
layout:     post
title:      "重温SpringAOP"
subtitle:   "AOP示例 + 原理分析"
date:       2017-10-10
author:     "lee"
header-img: ""
---


最近在工作中跟小伙伴聊到spring事务,提起同一个类中,方法a调用方法b,a没有加`@Transactional`注解,b有加注解,这种情况下,spring声明式事务不会生效.
究其原因,是spring的事务是由AOP实现,导致了这种类中方法间调用不会生效.<br>
同事J让我具体讲下实现细节,我愣是想不起来....<br>
翻了下自己的wiz note,两年前还研究过,竟然就忘了.借这个机会,自己也重新再梳理一下Spring AOP的细节.<br>

---

#### 从添加一个简单的前置通知入手，回顾AOP
##### 几个概念
前置通知：`MethodBeforeAdvice`<br>
代理对象：`ProxyFactoryBean`,包含下面三个关键成员
> 代理接口集：`proxyInterfaces`<br>
拦截器名称：`interceptorNames`<br>
被代理的对象：`target`<br>

##### 示例
1. 定义一个接口
```java
public interface AopService {
    void sayHello();
}
```

1. 创建接口的实现类
```java
public class AopServiceImpl implements AopService {
    public void sayHello() {
        System.out.println("hello!");
    }
}
```

1. 创建一个通知类
```java
//实现了MethodBeforeAdvice接口，即前置通知
//在这个类的before函数里，可以进行各种切面的操作，比如添加事务，日志处理等
public class MyMethodBeforeAdvice implements MethodBeforeAdvice {
    public void before(Method method, Object[] objects, Object o) throws Throwable {
        System.out.println("log here, methodName : " + method.getName());
    }
}
```

1. 通过xml配置方式配置Bean
	```xml
	<!-- aop.xml-->
	<!-- 在这个文件里要配置三个东西-->
	<!-- 1.被代理的对象-->
	<!-- 2.通知（例如前置通知MethodBeforeAdvice）-->
	<!-- 3.代理对象（ProxyFactoryBean）-->

	<!-- 要被代理的对象,也就是接口的实现类 -->
	<bean id="aopServiceImpl" class="aop.AopServiceImpl" />

	<!-- 通知（前置通知） -->
	<bean id="myMethodBeforeAdvice" class="aop.MyMethodBeforeAdvice" />
	    
	<!--代理对象-->
	<bean id="proxyFactoryBean" class="org.springframework.aop.framework.ProxyFactoryBean" >
	    <!--代理的接口-->
	    <property name="proxyInterfaces">
	        <list>
	            <value>aop.AopService</value>
	        </list>
	    </property>
	    <!--通知器(拦截器)名字-->
	    <property name="interceptorNames">
	        <value>myMethodBeforeAdvice</value>
	    </property>
	    <!--目标对象(需要被增强\被代理的对象)-->
	    <property name="target" ref="aopServiceImpl" />
	</bean>
	``` 

1. 测试代码
```java
public class AopTest {
    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("/aop.xml");
        AopService aopService = (AopService) context.getBean("proxyFactoryBean");
        aopService.sayHello();
    }
}
//Output:
log here, methodName : sayHello
hello!
```


#### 调用过程分析
##### 获取aop代理对象
```java
AopService aopService = (AopService) context.getBean("proxyFactoryBean");
```

###### 简单解释
这行大体思想是通过`applicationContext`获取到`AopService`的代理对象`proxyFactoryBean`.  
`proxyFactoryBean`在被初始化时,会被装载进所有有关aop的信息,包括 `通知类` ,和 `切入点` 等,并持有 `target` 的实例,变成该实例的代理对象  
所以获得到的`aopService`实际上是`aopServiceImpl`实例的代理对象.
###### 调用图解
![aop1](/img/aop1.jpg)
###### 详细分析
在初始化上下文的时候,配置在`aop.xml`里的`bean`会被装配进`bean工厂` ,由于`ProxyFactoryBean`实现了`FactoryBean`接口,所以在调用`applicationContext.getBean("proxyFactoryBean")`的时候,会调用到`ProxyFactoryBean.getObject()`方法
```java
public Object getObject() throws BeansException {
	initializeAdvisorChain();
	if (isSingleton()) {
		return getSingletonInstance();
	}
	else {
		if (this.targetName == null) {
			logger.warn("Using non-singleton proxies with singleton targets is often undesirable. " +
					"Enable prototype proxies by setting the 'targetName' property.");
		}
		return newPrototypeInstance();
	}
}
```
> `getObject()`方法会创建一个AOP代理实例,这个方法默认会返回一个单例  

`getObject()`里,首先调用`initializeAdvisorChain()`来初始化通知器链,所有`advice`都已经配置在`interceptorNames`里,所以会先根据`name`从`beanFactory`里拿到相应的`advice`,然后调用`addAdvisorOnChainCreation(advice,name)`把对应的`advice`包装成通知器(`advisor`)并添加到链中  
```java
//class: ProxyFactoryBean

private AdvisorAdapterRegistry advisorAdapterRegistry = GlobalAdvisorAdapterRegistry.getInstance();

...

private synchronized void initializeAdvisorChain() throws AopConfigException, BeansException {
	//advisorChainInitialized是通知链是否初始化过的标志位
	if (this.advisorChainInitialized) {
		return;
	}
	//配置在interceptorNames里的通知器(拦截器)
	if (!ObjectUtils.isEmpty(this.interceptorNames)) {
		if (this.beanFactory == null) {
			throw new IllegalStateException("No BeanFactory available anymore (probably due to serialization) " +
					"- cannot resolve interceptor names " + Arrays.asList(this.interceptorNames));
		}

		// Globals can't be last unless we specified a targetSource using the property...
		if (this.interceptorNames[this.interceptorNames.length - 1].endsWith(GLOBAL_SUFFIX) &&
				this.targetName == null && this.targetSource == EMPTY_TARGET_SOURCE) {
			throw new AopConfigException("Target required after globals");
		}

		// Materialize interceptor chain from bean names.
		for (String name : this.interceptorNames) {
			if (logger.isTraceEnabled()) {
				logger.trace("Configuring advisor or advice '" + name + "'");
			}

			if (name.endsWith(GLOBAL_SUFFIX)) {
				if (!(this.beanFactory instanceof ListableBeanFactory)) {
					throw new AopConfigException(
							"Can only use global advisors or interceptors with a ListableBeanFactory");
				}
				addGlobalAdvisor((ListableBeanFactory) this.beanFactory,
						name.substring(0, name.length() - GLOBAL_SUFFIX.length()));
			}

			else {
				// If we get here, we need to add a named interceptor.
				// We must check if it's a singleton or prototype.
				Object advice;
				if (this.singleton || this.beanFactory.isSingleton(name)) {
					//此例会走到这个分支,从bean工厂获取advice
					// Add the real Advisor/Advice to the chain.
					advice = this.beanFactory.getBean(name);
				}
				else {
					// It's a prototype Advice or Advisor: replace with a prototype.
					// Avoid unnecessary creation of prototype bean just for advisor chain initialization.
					advice = new PrototypePlaceholderAdvisor(name);
				}
				//把advice添加到Advisor里
				addAdvisorOnChainCreation(advice, name);
			}
		}
	}

	this.advisorChainInitialized = true;
}

...

private void addAdvisorOnChainCreation(Object next, String name) {
	// We need to convert to an Advisor if necessary so that our source reference
	// matches what we find from superclass interceptors.
	Advisor advisor = namedBeanToAdvisor(next);
	if (logger.isTraceEnabled()) {
		logger.trace("Adding advisor with name '" + name + "'");
	}
	addAdvisor(advisor);
}

...

private Advisor namedBeanToAdvisor(Object next) {
	try {
		return this.advisorAdapterRegistry.wrap(next);
	}
	catch (UnknownAdviceTypeException ex) {
		// We expected this to be an Advisor or Advice,
		// but it wasn't. This is a configuration error.
		throw new AopConfigException("Unknown advisor type " + next.getClass() +
				"; Can only include Advisor or Advice type beans in interceptorNames chain except for last entry," +
				"which may also be target or TargetSource", ex);
	}
}
```
> `ProxyFactoryBean`有一个`AdvisorAdapterRegistry`的成员变量,这个通知器的`Registry`默认持有三个通知器的适配器:`MethodBeforeAdviceAdapter`, `AfterReturningAdviceAdapter`, `ThrowsAdviceAdapter`.通过名字也可以看出,这三个适配器分别是为了适配**前置通知**,**后置通知**,**异常通知**  
`AdvisorAdapterRegistry`会把传入的`advice`包装**(wrap)**成`Advisor`,`ProxyFactoryBean`继承了`ProxyCreatorSupport`,`ProxyCreatorSupport`继承了`AdvisedSupport`,`AdvisedSupport`里维护了`List<Advisor> advisors`,所以`addAdvisorOnChainCreation`方法最终会把所有通知器**(interceptors)**都添加到通知器的列表中.

初始化完通知链后,就会调用`getSingletonInstance()`获取aop代理对象并返回
```java
private synchronized Object getSingletonInstance() {
	if (this.singletonInstance == null) {
		this.targetSource = freshTargetSource();
		if (this.autodetectInterfaces && getProxiedInterfaces().length == 0 && !isProxyTargetClass()) {
			// Rely on AOP infrastructure to tell us what interfaces to proxy.
			Class targetClass = getTargetClass();
			if (targetClass == null) {
				throw new FactoryBeanNotInitializedException("Cannot determine target class for proxy");
			}
			setInterfaces(ClassUtils.getAllInterfacesForClass(targetClass, this.proxyClassLoader));
		}
		// Initialize the shared singleton instance.
		super.setFrozen(this.freezeProxy);
		this.singletonInstance = getProxy(createAopProxy());
	}
	return this.singletonInstance;
}
```

由于本例是对接口进行代理,所以默认会使用jdk的动态代理,即 `Proxy`+`InvocationHandler`,如果代理是直接针对类,则一般会使用`Cglib`

##### 调用方法时,进行aop切入
`aopService.sayHello();`
> 这行的调用,实际会通过代理对象去调用实际的目标对象`(target)`,在调用过程中,所有切面的工作,都会由代理来完成 <br><br>
> 大致的调用图解如下:
![aop2](/img/aop2.jpg)


未完...























