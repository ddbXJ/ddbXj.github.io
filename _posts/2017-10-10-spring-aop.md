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


#### 从添加一个简单的前置通知入手，回顾AOP
##### 几个概念
前置通知：`MethodBeforeAdvice`<br>
代理对象：`ProxyFactoryBean`,包含下面三个关键成员
> 代理接口集：`proxyInterfaces`<br>
拦截器名称：`interceptorNames`<br>
被代理的对象：`target`<br>

##### demo
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
	<!-- beans.xml-->
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


#### 简单分析调用过程
* `AopService aopService = (AopService) context.getBean("proxyFactoryBean");`
> 这行大体思想是通过`applicationContext`获取到`AopService`的代理对象`proxyFactoryBean`
`proxyFactoryBean`在被初始化时,会被装载进所有有关aop的信息,并持有target的实例,变成该实例的代理对象
所以获得到的`aopService`实际上是`aopServiceImpl`实例的代理对象

* `aopService.sayHello();`
> 这行的调用,实际会通过代理对象去调用实际的目标对象`(target)`,在调用过程中,所有切面的工作,都会由代理来完成


未完...

