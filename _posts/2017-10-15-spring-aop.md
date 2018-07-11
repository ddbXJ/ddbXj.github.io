---
layout:     post
title:      "重温SpringAOP"
subtitle:   "AOP示例 + 原理分析"
date:       2017-10-15
author:     "lee"
header-img: "img/back.jpeg"
---


最近在工作中跟小伙伴聊到spring事务,提起同一个类中,方法a调用方法b,a没有加`@Transactional`注解,b有加注解,这种情况下,spring声明式事务不会生效.
究其原因,是spring的事务是由AOP实现,导致了这种类中方法间调用不会生效.<br>
同事J让我具体讲下实现细节,我愣是想不起来....<br>
翻了下自己的wiz note,两年前还研究过,竟然就忘了.借这个机会,自己也重新再梳理一下Spring AOP的细节.<br>

---

### 从添加一个简单的前置通知入手，回顾AOP
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


### 调用过程分析
#### 获取aop代理对象
```java
AopService aopService = (AopService) context.getBean("proxyFactoryBean");
```

###### 简单解释
这行大体思想是通过`applicationContext`获取到`AopService`的代理对象`proxyFactoryBean`.  
`proxyFactoryBean`在被初始化时,会被装载进所有有关aop的信息,包括 `通知类` ,和 `切入点` 等,并持有 `target` 的实例,变成该实例的代理对象  
所以获得到的`aopService`实际上是`aopServiceImpl`实例的代理对象.
###### 调用图解
![aop1](/assets/images/aop1.jpg)
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
`AdvisorAdapterRegistry`会把传入的`advice`包装**(wrap)**成`Advisor`,`ProxyFactoryBean`继承了`ProxyCreatorSupport`,`ProxyCreatorSupport`继承了`AdvisedSupport`,`AdvisedSupport`里维护了`List<Advisor> advisors`,所以`addAdvisorOnChainCreation`方法最终会把所有通知器**(interceptors)**都添加到这个列表中.

初始化完通知链后,就会调用`getSingletonInstance()`获取aop代理对象并返回
```java
//class: ProxyFactoryBean
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

//class: ProxyCreatorSupport
protected final synchronized AopProxy createAopProxy() {
	if (!this.active) {
		activate();
	}
	//getAopProxyFactory()拿到默认的代理工厂类是DefaultAopProxyFactory
	return getAopProxyFactory().createAopProxy(this);
}

//class: DefaultAopProxyFactory
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
	if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
		Class targetClass = config.getTargetClass();
		if (targetClass == null) {
			throw new AopConfigException("TargetSource cannot determine target class: " +
					"Either an interface or a target is required for proxy creation.");
		}
		if (targetClass.isInterface()) {
			return new JdkDynamicAopProxy(config);
		}
		return CglibProxyFactory.createCglibProxy(config);
	}
	else {
		return new JdkDynamicAopProxy(config);
	}
}
```
> 从以上方法调用能看到,`getSingletonInstance()`方法里,会从代理工厂创建一个`AopProxy`,由于`ProxyCreatorSupport`继承于`AdvisedSupport`,所以代理工厂会通过`ProxyFacotyBean`创建一个AOP代理类.这个代理类,包含一系列需要进行通知的通知器信息(advisors),也包含被代理的目标对象的构造函数信息,需要被intercept的接口(或类的方法名)信息等.  
对于本例来说,由于是对接口进行代理,所以默认会使用jdk的动态代理,在创建代理类的时候,会调用`Proxy.newProxyInstance(ClassLoader loader,Class<?>[] interfaces,InvocationHandler h)`构造代理类.  
`JdkDynamicAopProxy`实现了`InvocationHandler`,在`invoke()`方法中实现了方法调用和通知的调用.  
另外,如果代理是直接针对类,则一般会使用`Cglib`.

#### 执行方法调用
```java
aopService.sayHello();
```
###### 简单解释
这行的调用,实际会通过代理对象去调用目标对象`(target)`的方法,在调用过程中,所有切面的工作,都会由代理来完成.
###### 调用图解
![aop2](/assets/images/aop2.jpg)
###### 详细分析
AOP联盟`(aopalliance)`规约了一系列AOP相关接口.  
`MethodInvocation`接口继承于`Invocation`,`Invocation`继承于`Joinpoint`接口,`Joinpoint`接口里定义了`proceed()`方法:  
```java
public interface Joinpoint {
    Object proceed() throws Throwable;

    ...
}
```
`MethodInterceptor`接口定义了`invoke()`方法:
```java
public interface MethodInterceptor extends Interceptor {
    Object invoke(MethodInvocation var1) throws Throwable;
}
```
`aopService.sayHello()`方法被调用时,由于`aopService`是一个jdk动态代理生成的代理对象,所以会调用到`句柄(h)`的`invoke()`方法.
`h`是由`JdkDynamicAopProxy`实现,所以来分析一下`JdkDynamicAopProxy`的`invoke()`方法.
```java
//class: JdkDynamicAopProxy

public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
	MethodInvocation invocation;
	Object oldProxy = null;
	boolean setProxyContext = false;

	TargetSource targetSource = this.advised.targetSource;
	Class targetClass = null;
	Object target = null;

	try {
		if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
			// The target does not implement the equals(Object) method itself.
			return equals(args[0]);
		}
		if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
			// The target does not implement the hashCode() method itself.
			return hashCode();
		}
		if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
				method.getDeclaringClass().isAssignableFrom(Advised.class)) {
			// Service invocations on ProxyConfig with the proxy config...
			return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
		}

		Object retVal;

		if (this.advised.exposeProxy) {
			// Make invocation available if necessary.
			oldProxy = AopContext.setCurrentProxy(proxy);
			setProxyContext = true;
		}

		// May be null. Get as late as possible to minimize the time we "own" the target,
		// in case it comes from a pool.
		target = targetSource.getTarget();
		if (target != null) {
			targetClass = target.getClass();
		}

		//获取这个方法的拦截器链(通知器链)
		// Get the interception chain for this method.
		List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

		// Check whether we have any advice. If we don't, we can fallback on direct
		// reflective invocation of the target, and avoid creating a MethodInvocation.
		if (chain.isEmpty()) {
			// We can skip creating a MethodInvocation: just invoke the target directly
			// Note that the final invoker must be an InvokerInterceptor so we know it does
			// nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
			retVal = AopUtils.invokeJoinpointUsingReflection(target, method, args);
		}
		else {
			// We need to create a method invocation...
			invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
			//这行是执行调用链的入口
			// Proceed to the joinpoint through the interceptor chain.
			retVal = invocation.proceed();
		}

		// Massage return value if necessary.
		Class<?> returnType = method.getReturnType();
		if (retVal != null && retVal == target && returnType.isInstance(proxy) &&
				!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
			// Special case: it returned "this" and the return type of the method
			// is type-compatible. Note that we can't help if the target sets
			// a reference to itself in another returned object.
			retVal = proxy;
		} else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
			throw new AopInvocationException("Null return value from advice does not match primitive return type for: " + method);
		}
		return retVal;
	}
	finally {
		if (target != null && !targetSource.isStatic()) {
			// Must have come from TargetSource.
			targetSource.releaseTarget(target);
		}
		if (setProxyContext) {
			// Restore old proxy.
			AopContext.setCurrentProxy(oldProxy);
		}
	}
}

//class: ReflectiveMethodInvocation

public Object proceed() throws Throwable {
	//	We start with an index of -1 and increment early.
	if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
		return invokeJoinpoint();
	}

	Object interceptorOrInterceptionAdvice =
			this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
	if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
		// Evaluate dynamic method matcher here: static part will already have
		// been evaluated and found to match.
		InterceptorAndDynamicMethodMatcher dm =
				(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
		if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
			return dm.interceptor.invoke(this);
		}
		else {
			// Dynamic matching failed.
			// Skip this interceptor and invoke the next in the chain.
			return proceed();
		}
	}
	else {
		// It's an interceptor, so we just invoke it: The pointcut will have
		// been evaluated statically before this object was constructed.
		return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
	}
}

protected Object invokeJoinpoint() throws Throwable {
	return AopUtils.invokeJoinpointUsingReflection(this.target, this.method, this.arguments);
}

//class: AopUtils

public static Object invokeJoinpointUsingReflection(Object target, Method method, Object[] args)
		throws Throwable {

	// Use reflection to invoke the method.
	try {
		ReflectionUtils.makeAccessible(method);
		return method.invoke(target, args);
	}
	catch (InvocationTargetException ex) {
		// Invoked method threw a checked exception.
		// We must rethrow it. The client won't see the interceptor.
		throw ex.getTargetException();
	}
	catch (IllegalArgumentException ex) {
		throw new AopInvocationException("AOP configuration seems to be invalid: tried calling method [" +
				method + "] on target [" + target + "]", ex);
	}
	catch (IllegalAccessException ex) {
		throw new AopInvocationException("Could not access method [" + method + "]", ex);
	}
}
```
在`JdkDynamicAopProxy.invoke()`方法中,可以看到,首先获取到了通知器链`List<Object> chain`,然后`ReflectiveMethodInvocation`被创建出来,它的`proceed()`作为方法调用的入口,在该方法中,会通过一个自增的标志位`currentInterceptorIndex`从拦截器链中获取通知器,调用`MethodInterceptor.invoke(this)`方法,这里当前对象(`MethodInvocation`)会作为参数被传入,方便`invoke()`方法中执行完intercept操作后进行回调,这里也属于一种责任链模式.  
通过看前置通知,后置通知,异常通知的拦截器的实现,也能更清晰地看出这里的链式调用过程
```java
public class MethodBeforeAdviceInterceptor implements MethodInterceptor, Serializable {

	private MethodBeforeAdvice advice;

	...

	public Object invoke(MethodInvocation mi) throws Throwable {
		this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis() );
		return mi.proceed();
	}

}

public class AfterReturningAdviceInterceptor implements MethodInterceptor, AfterAdvice, Serializable {

	private final AfterReturningAdvice advice;

	...

	public Object invoke(MethodInvocation mi) throws Throwable {
		Object retVal = mi.proceed();
		this.advice.afterReturning(retVal, mi.getMethod(), mi.getArguments(), mi.getThis());
		return retVal;
	}

}

public class ThrowsAdviceInterceptor implements MethodInterceptor, AfterAdvice {

	private final Object throwsAdvice;

	...

	public Object invoke(MethodInvocation mi) throws Throwable {
		try {
			return mi.proceed();
		}
		catch (Throwable ex) {
			Method handlerMethod = getExceptionHandler(ex);
			if (handlerMethod != null) {
				invokeHandlerMethod(mi, ex, handlerMethod);
			}
			throw ex;
		}
	}
}
```
当通知器链上的`invoke()`方法都被调用过之后,`ReflectiveMethodInvocation.invokeJoinpoint()`方法会被调用,这时会调用`AopUtils.invokeJoinpointUsingReflection`,通过反射执行真正的方法调用.

---

所以再回过头来看文章最前面的问题:同一个实现类中,`methodA`调用`methodB`,只在`methodB`上声明了事务,这时事务会失效.原因就是`Spring`的事务是基于AOP来实现的,而AOP实际上就是利用了动态代理,在调用发起时,首先是走的代理类,然后代理类去调用`methodA`的时候,最终会调用到真正实现类的`methodA`,这时候,`methodA`里去调用`methodB`,其实就是普通的类间方法调用了,不会走AOP,从而事务也就失效了.


















