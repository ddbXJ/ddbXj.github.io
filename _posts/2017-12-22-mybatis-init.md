---
layout:     post
title:      "MyBatis整合Spring:初始化相关问题"
subtitle:   "MyBatis,Spring,MapperScannerConfigurer"
date:       2017-12-22
author:     "lee"
header-img: "img/back.jpeg"
---

主要围绕三个问题对源码进行研究
> 1. 通常生成`dao`层`mapper`的时候,为什么`xml`文件的`namespace`要与`mapper`类的`namespace`保持一致? (相对路径)
1. 单独配置每个`mapper`,与通过声明`MapperScannerConfigurer`有什么不同?
1. 使用`mybatis-spring-1.1.0`版本时,通过配置`MapperScannerConfigurer`会导致项目无法启动,升级为`1.2.2`可以解决.原因是什么?



### 问题1解决过程

##### `mybatis`+`spring`最常见的用法:

`applicationContext.xml`文件:
```xml
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="dataSource"/>
        <property name="configLocation" value="classpath:mybatis-config.xml"/>
    </bean>

    <bean id="dataSource" class="org.apache.tomcat.jdbc.pool.DataSource" destroy-method="close">
        <property name="poolProperties">
            <bean class="org.apache.tomcat.jdbc.pool.PoolProperties">
                <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="${com.xxx.datasource.url}"/>
                <property name="username" value="${com.xxx.datasource.username}"/>
                <property name="password" value="${com.xxx.datasource.password}"/>
                ....
            </bean>
        </property>
    </bean>

    <bean id="xxxMapper" class="org.mybatis.spring.mapper.MapperFactoryBean">
        <property name="sqlSessionFactory" ref="sqlSessionFactory"/>
        <property name="mapperInterface" value="com.xxx.dao.mapper.XxxMapper"/>
    </bean>
```

dao层结构:
```
|——dao
|    |——src
|        |——main
|        |    |——java
|        |    |   |——com.xxx.dao                     //数据访问层
|        |    |       |——dto                         //系统生成mapper使用的参数实体
|        |    |       |——generatedMapper             //系统生成mapper
|        |    |——resources
|        |        |——com.xxx.dao                     //mybatis映射文件,都是xxx.xml格式
|        |            |——generatedMapper
```

##### 分析

项目加载的时候,首先会加载`SqlSessionFactoryBean`,作为连接池  
它通过`dataSource`字段来配置数据源,`configLocation`负责加载`mybatis`的相关配置

`SqlSessionFactoryBean`实现了`spring`的`InitializingBean`接口,所以`SqlSessionFactoryBean`被初始化之后,会调用它实现的`afterPropertiesSet()`方法
```java
  //SqlSessionFactoryBean.class
  public void afterPropertiesSet() throws Exception {
    notNull(dataSource, "Property 'dataSource' is required");
    notNull(sqlSessionFactoryBuilder, "Property 'sqlSessionFactoryBuilder' is required");

    //可以看到,会真正的build sqlSessionFactory
    this.sqlSessionFactory = buildSqlSessionFactory();
  }

  protected SqlSessionFactory buildSqlSessionFactory() throws IOException {

    Configuration configuration;

    //先从配置的mybatis-config.xml文件中,读出mybatis配置信息

    XMLConfigBuilder xmlConfigBuilder = null;
    if (this.configLocation != null) {
      xmlConfigBuilder = new XMLConfigBuilder(this.configLocation.getInputStream(), null, this.configurationProperties);
      configuration = xmlConfigBuilder.getConfiguration();
    } else {
      if (this.logger.isDebugEnabled()) {
        this.logger.debug("Property 'configLocation' not specified, using default MyBatis Configuration");
      }
      configuration = new Configuration();
      configuration.setVariables(this.configurationProperties);
    }

    //接下来的一系列判断,都是根据创建时配置的properties,来做相应的操作
    //如果有配置,则添加到对应的Registry上

    if (hasLength(this.typeAliasesPackage)) {
      String[] typeAliasPackageArray = tokenizeToStringArray(this.typeAliasesPackage,
          ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
      for (String packageToScan : typeAliasPackageArray) {
        configuration.getTypeAliasRegistry().registerAliases(packageToScan);
        if (this.logger.isDebugEnabled()) {
          this.logger.debug("Scanned package: '" + packageToScan + "' for aliases");
        }
      }
    }

    if (!isEmpty(this.typeAliases)) {
      for (Class<?> typeAlias : this.typeAliases) {
        configuration.getTypeAliasRegistry().registerAlias(typeAlias);
        if (this.logger.isDebugEnabled()) {
          this.logger.debug("Registered type alias: '" + typeAlias + "'");
        }
      }
    }

    if (!isEmpty(this.plugins)) {
      for (Interceptor plugin : this.plugins) {
        configuration.addInterceptor(plugin);
        if (this.logger.isDebugEnabled()) {
          this.logger.debug("Registered plugin: '" + plugin + "'");
        }
      }
    }

    if (hasLength(this.typeHandlersPackage)) {
      String[] typeHandlersPackageArray = tokenizeToStringArray(this.typeHandlersPackage,
          ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
      for (String packageToScan : typeHandlersPackageArray) {
        configuration.getTypeHandlerRegistry().register(packageToScan);
        if (this.logger.isDebugEnabled()) {
          this.logger.debug("Scanned package: '" + packageToScan + "' for type handlers");
        }
      }
    }

    if (!isEmpty(this.typeHandlers)) {
      for (TypeHandler<?> typeHandler : this.typeHandlers) {
        configuration.getTypeHandlerRegistry().register(typeHandler);
        if (this.logger.isDebugEnabled()) {
          this.logger.debug("Registered type handler: '" + typeHandler + "'");
        }
      }
    }

    if (xmlConfigBuilder != null) {
      try {

      	//这里,会对加载到的mybatis-config.xml文件,进行解析

        xmlConfigBuilder.parse();

        if (this.logger.isDebugEnabled()) {
          this.logger.debug("Parsed configuration file: '" + this.configLocation + "'");
        }
      } catch (Exception ex) {
        throw new NestedIOException("Failed to parse config resource: " + this.configLocation, ex);
      } finally {
        ErrorContext.instance().reset();
      }
    }

    if (this.transactionFactory == null) {
      this.transactionFactory = new SpringManagedTransactionFactory();
    }

    Environment environment = new Environment(this.environment, this.transactionFactory, this.dataSource);
    configuration.setEnvironment(environment);

    if (this.databaseIdProvider != null) {
      try {
        configuration.setDatabaseId(this.databaseIdProvider.getDatabaseId(this.dataSource));
      } catch (SQLException e) {
        throw new NestedIOException("Failed getting a databaseId", e);
      }
    }

    if (!isEmpty(this.mapperLocations)) {
      for (Resource mapperLocation : this.mapperLocations) {
        if (mapperLocation == null) {
          continue;
        }

        try {
          XMLMapperBuilder xmlMapperBuilder = new XMLMapperBuilder(mapperLocation.getInputStream(),
              configuration, mapperLocation.toString(), configuration.getSqlFragments());
          xmlMapperBuilder.parse();
        } catch (Exception e) {
          throw new NestedIOException("Failed to parse mapping resource: '" + mapperLocation + "'", e);
        } finally {
          ErrorContext.instance().reset();
        }

        if (this.logger.isDebugEnabled()) {
          this.logger.debug("Parsed mapper file: '" + mapperLocation + "'");
        }
      }
    } else {
      if (this.logger.isDebugEnabled()) {
        this.logger.debug("Property 'mapperLocations' was not specified or no matching resources found");
      }
    }

    //加载完所有的配置信息之后,会调用这个来构建 sqlSessionFactory

    return this.sqlSessionFactoryBuilder.build(configuration);
  }
```

对于上述配置方法,由于我们的`mybatis`配置没有指定具体要扫描的`mapper`,而是通过在`spring`的`bean`配置里对每个`mapper`单独指定  
所以在`sqlSessionFactory`被加载完之后,其实还没有建立`mapper`的相关配置加载.
这个时候在加载上例的`xxxMapper`的时候,会初始化`MapperFactoryBean<XxxMapper>`
```java
public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T>
...


public abstract class SqlSessionDaoSupport extends DaoSupport
...

public abstract class DaoSupport implements InitializingBean

```
可以看到`xxxMapper`被初始化完了之后,会走到`DaoSupport`的`afterPropertiesSet()`方法

```java

	//class : DaoSupport
	@Override
	public final void afterPropertiesSet() throws IllegalArgumentException, BeanInitializationException {
		// Let abstract subclasses check their configuration.
		checkDaoConfig();

		// Let concrete implementations initialize themselves.
		try {
			initDao();
		}
		catch (Exception ex) {
			throw new BeanInitializationException("Initialization of DAO failed", ex);
		}
	}

	//class : MapperFactoryBean
	@Override
	protected void checkDaoConfig() {
		super.checkDaoConfig();

		notNull(this.mapperInterface, "Property 'mapperInterface' is required");

		Configuration configuration = getSqlSession().getConfiguration();
		if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
		  try {
		  	//这个方法很重要
		    configuration.addMapper(this.mapperInterface);
		  } catch (Throwable t) {
		    logger.error("Error while adding the mapper '" + this.mapperInterface + "' to configuration.", t);
		    throw new IllegalArgumentException(t);
		  } finally {
		    ErrorContext.instance().reset();
		  }
		}
	}

```

在调用`configuration.addMapper(this.mapperInterface);`的方法时,会走到`MapperRegistry`的`addMapper`的方法
```java

  //MapperRegistry.class
  public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {
      if (hasMapper(type)) {
        throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
      }
      boolean loadCompleted = false;
      try {
        knownMappers.put(type, new MapperProxyFactory<T>(type));
        // It's important that the type is added before the parser is run
        // otherwise the binding may automatically be attempted by the
        // mapper parser. If the type is already known, it won't try.
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
        parser.parse();
        loadCompleted = true;
      } finally {
        if (!loadCompleted) {
          knownMappers.remove(type);
        }
      }
    }
  }

  //MapperAnnotationBuilder.class
  public void parse() {
    String resource = type.toString();
    if (!configuration.isResourceLoaded(resource)) {
      loadXmlResource();
      configuration.addLoadedResource(resource);
      assistant.setCurrentNamespace(type.getName());
      parseCache();
      parseCacheRef();
      Method[] methods = type.getMethods();
      for (Method method : methods) {
        try {
          parseStatement(method);
        } catch (IncompleteElementException e) {
          configuration.addIncompleteMethod(new MethodResolver(this, method));
        }
      }
    }
    parsePendingMethods();
  }

  private void loadXmlResource() {
    // Spring may not know the real resource name so we check a flag
    // to prevent loading again a resource twice
    // this flag is set at XMLMapperBuilder#bindMapperForNamespace
    if (!configuration.isResourceLoaded("namespace:" + type.getName())) {

      //这里保证了xml文件,会根据mapper类的包路径,来找对应路径的resource文件

      String xmlResource = type.getName().replace('.', '/') + ".xml";
      InputStream inputStream = null;
      try {
        inputStream = Resources.getResourceAsStream(type.getClassLoader(), xmlResource);
      } catch (IOException e) {
        // ignore, resource is not required
      }
      if (inputStream != null) {
        XMLMapperBuilder xmlParser = new XMLMapperBuilder(inputStream, assistant.getConfiguration(), xmlResource, configuration.getSqlFragments(), type.getName());
        xmlParser.parse();
      }
    }
  }

```
所以可以看到,是因为`MapperAnnotationBuilder`类的`loadXmlResource()`方法,决定了加载的`xml`文件,会从接口对应的路径的资源文件路径去读.



### 问题2解决过程

随着`mapper`越来越多,每多一个,就添加一个`mapper`配置,很繁琐.
所以`mybatis`也提供了一个扫描`mapper`包的工具:`MapperScannerConfigurer`  
像下面这样配置,就可以不用每个一个一个`mapper`单独配:
```xml
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="basePackage" value="com.xxx.dao.xxxMapper,...."/>
        <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>
    </bean>
``` 
`MapperScannerConfigurer`初始化的时候,会调用到下面的方法( *具体原因见问题3的分析* ),`scanner`负责扫描所有声明了注解的`mapper`类,或者通过上面的配置,配在`basePackage`里的包.
```java

  //MapperScannerConfigurer.class
  public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
    if (this.processPropertyPlaceHolders) {
      processPropertyPlaceHolders();
    }

    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    scanner.setAddToConfig(this.addToConfig);
    scanner.setAnnotationClass(this.annotationClass);
    scanner.setMarkerInterface(this.markerInterface);
    scanner.setSqlSessionFactory(this.sqlSessionFactory);
    scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
    scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
    scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
    scanner.setResourceLoader(this.applicationContext);
    scanner.setBeanNameGenerator(this.nameGenerator);
    scanner.registerFilters();
    scanner.scan(StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
  }

  //ClassPathBeanDefinitionScanner.class
  public int scan(String... basePackages) {
    int beanCountAtScanStart = this.registry.getBeanDefinitionCount();

    doScan(basePackages);

    // Register annotation config processors, if necessary.
    if (this.includeAnnotationConfig) {
      AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
    }

    return this.registry.getBeanDefinitionCount() - beanCountAtScanStart;
  }


  //ClassPathBeanDefinitionScanner.class
  protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Assert.notEmpty(basePackages, "At least one base package must be specified");
    Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<BeanDefinitionHolder>();
    for (String basePackage : basePackages) {
      Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
      for (BeanDefinition candidate : candidates) {
        ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
        candidate.setScope(scopeMetadata.getScopeName());
        String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
        if (candidate instanceof AbstractBeanDefinition) {
          postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
        }
        if (candidate instanceof AnnotatedBeanDefinition) {
          AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
        }
        if (checkCandidate(beanName, candidate)) {
          
          //第一次扫描到这些mapper的时候,会走到这里,然后创建对应的beanDefinition,并注册

          BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
          definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
          beanDefinitions.add(definitionHolder);
          registerBeanDefinition(definitionHolder, this.registry);
        }
      }
    }
    return beanDefinitions;
  }

```
通过以上代码,可以看出,`MapperScannerConfigurer`被初始化之后,所有该扫描的包里的`mapper`类,都已经被注册到`beanDefinitionMap`中,但是没有初始化  
在初始化各个配置的`bean`的时候,如果有要求注入特定`mapper`的,会触发该`mapper`的初始化.
初始化完成之后,变会走到各个`mapper`的`checkDaoConfig()`( *问题1中有讲* ) ,来真正的加载`mapper`的`xml`文件




### 问题3解决过程

经过研究发现,这个问题是`mybatis-spring-1.1.0`的`bug`:
> 使用这个版本的jar包时,如果采用每个mapper单独配置,则配置加载没有问题  
  但如果使用了`MapperScannerConfigurer`,则会因为配置中声明的**占位符**无法被替换,而导致spring无法初始化


##### 先看使用有问题版本的`MapperScannerConfigurer`的时候:
项目启动,会报如下错误.
```
failed to connect to server ${xxxx}, error message is:null
....
```

###### 先明确一些基本概念:

`spring`加载配置文件,从`properties`文件中拿出配置值,并对`xml`文件里的`${xxx}`进行替换,是发生在`PropertyResourceConfigurer`和`PropertyPlaceholderConfigurer`这两个类里.  

`PropertyResourceConfigurer`实现了`BeanFactoryPostProcessor`和`PriorityOrdered`接口,所以在前置处理的时候,它的`postProcessBeanFactory()`方法变会被调用,然后`PropertyPlaceholderConfigurer.processProperties()`会执行替换的逻辑
```java
  //PropertyResourceConfigurer.class
  @Override
  public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
    try {
      Properties mergedProps = mergeProperties();

      // Convert the merged properties, if necessary.
      convertProperties(mergedProps);

      // Let the subclass process the properties.
      processProperties(beanFactory, mergedProps);
    }
    catch (IOException ex) {
      throw new BeanInitializationException("Could not load properties", ex);
    }
  }

  //PropertyPlaceholderConfigurer.class
  @Override
  protected void processProperties(ConfigurableListableBeanFactory beanFactoryToProcess, Properties props)
      throws BeansException {

    StringValueResolver valueResolver = new PlaceholderResolvingStringValueResolver(props);

    this.doProcessProperties(beanFactoryToProcess, valueResolver);
  }
```
所以问题的出现,肯定是由于初始化某些需要替换配置的`bean`的时候,`properties`还没有被加载,从而报错

###### 下面看下具体原因


`MapperScannerConfigurer`由于实现了`BeanDefinitionRegistryPostProcessor`接口  
所以会在`spring`初始化的前置阶段被调用到它实现的`postProcessBeanDefinitionRegistry()`方法  
这个方法在`PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors()`中被调用,并且它的调用顺序是早于上面说的`PropertyResourceConfigurer`中的方法,见下面的代码
```java

//扫描mapper包的工具类
public class MapperScannerConfigurer implements BeanDefinitionRegistryPostProcessor, InitializingBean, ApplicationContextAware, BeanNameAware {....}

//加载配置的工具类
public abstract class PropertyResourceConfigurer extends PropertiesLoaderSupport
    implements BeanFactoryPostProcessor, PriorityOrdered {....}

//前置处理类
class PostProcessorRegistrationDelegate {

  public static void invokeBeanFactoryPostProcessors(
    ...

    if (beanFactory instanceof BeanDefinitionRegistry) {

      ...

      // Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
      boolean reiterate = true;
      while (reiterate) {
        reiterate = false;
        postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
          if (!processedBeans.contains(ppName)) {
            BeanDefinitionRegistryPostProcessor pp = beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class);
            registryPostProcessors.add(pp);
            processedBeans.add(ppName);
            
            //这里,会调用到MapperScannerConfigurer的方法

            pp.postProcessBeanDefinitionRegistry(registry);
            reiterate = true;
          }
        }
      }

      ...

    }

    //然后这里,才会加载配置替换的类

    // First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
    OrderComparator.sort(priorityOrderedPostProcessors);
    invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

```
上面代码中的`postProcessorNames`,会包含一个`MapperScannerConfigurer`:  
> `MapperScannerConfigurer#0`  
(因为没有指定id和name,所以`spring`指定的`beanName`会在类名后面追加`#`加数字)

可见,`MapperScannerConfigurer.postProcessBeanDefinitionRegistry()`的调用会早于`PropertyResourceConfigurer.postProcessBeanFactory()`  
然后,就导致了,`MapperScannerConfigurer.postProcessBeanDefinitionRegistry()`一系列方法执行的时候,配置还没有被加载进来.

看一下`MapperScannerConfigurer.postProcessBeanDefinitionRegistry()`的逻辑:

```java

  //class: MapperScannerConfigurer

  public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry beanDefinitionRegistry) throws BeansException {
    //这里,出问题的方法
    processPropertyPlaceHolders();

    Scanner scanner = new Scanner(beanDefinitionRegistry);
    scanner.setResourceLoader(this.applicationContext);

    scanner.scan(StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
  }

  private void processPropertyPlaceHolders() {
    //会调用到这里
    Map<String, PropertyResourceConfigurer> prcs = applicationContext.getBeansOfType(PropertyResourceConfigurer.class);

    ....
  }

```
导致了错误的一句,就是
```java
applicationContext.getBeansOfType(PropertyResourceConfigurer.class)
```
它会走到`DefaultListableBeanFactory.doGetBeanNamesForType()`方法  
```java

  //class: DefaultListableBeanFactory

  @Override
  public <T> Map<String, T> getBeansOfType(Class<T> type) throws BeansException {
    return getBeansOfType(type, true, true);
  }

  @Override
  public <T> Map<String, T> getBeansOfType(Class<T> type, boolean includeNonSingletons, boolean allowEagerInit)
      throws BeansException {...}

  ...

  private String[] doGetBeanNamesForType(Class<?> type, boolean includeNonSingletons, boolean allowEagerInit) {
    List<String> result = new ArrayList<String>();

    //拿出所有已经在beanDefinitionMap里的

    // Check all bean definitions.
    String[] beanDefinitionNames = getBeanDefinitionNames();
    for (String beanName : beanDefinitionNames) {
      // Only consider bean as eligible if the bean name
      // is not defined as alias for some other bean.
      if (!isAlias(beanName)) {
        try {
          RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
          // Only check bean definition if it is complete.
          if (!mbd.isAbstract() && (allowEagerInit ||
              ((mbd.hasBeanClass() || !mbd.isLazyInit() || this.allowEagerClassLoading)) &&
                  !requiresEagerInitForType(mbd.getFactoryBeanName()))) {
            // In case of FactoryBean, match object created by FactoryBean.
            boolean isFactoryBean = isFactoryBean(beanName, mbd);

            //这里

            boolean matchFound = (allowEagerInit || !isFactoryBean || containsSingleton(beanName)) &&
                (includeNonSingletons || isSingleton(beanName)) && isTypeMatch(beanName, type);
            if (!matchFound && isFactoryBean) {
              // In case of FactoryBean, try to match FactoryBean instance itself next.
              beanName = FACTORY_BEAN_PREFIX + beanName;
              matchFound = (includeNonSingletons || mbd.isSingleton()) && isTypeMatch(beanName, type);
            }
            if (matchFound) {
              result.add(beanName);
            }
          }
        }
        ...
    }

    ...
  }
```
***注意***,这种方法导致`doGetBeanNamesForType()`方法被调用到的时候,`allowEagerInit`参数的值是`true`,这个参数又导致了,会提前实例化已经在`beanDefinitionMap`中的`bean`.

> 注: 会走到`AbstractBeanFactory.isTypeMatch()`方法  
 isTypeMatch -> getTypeForFactoryBean -> doGetBean -> getSingleton -> getObject

所以,这个bug导致了,如果我们配置的`bean`中有需要根据`placeHolder`替换配置值的,会失效.


##### 再看使用每个mapper单独配置的时候,为什么正常:
由于没有配置`MapperScannerConfigurer`,就不会干扰`spring`的前置处理流程  
最先走的是`PropertyResourceConfigurer`,在`bean`被初始化之前,配置已经被初始化了,所以能正常运行


##### 最后看一下修复过后, mybatis-spring-1.2.2版本
```java
  
  //class: MapperScannerConfigurer

  public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
    //这个值默认就是false,默认不会走processPropertyPlaceHolders()这个方法
    if (this.processPropertyPlaceHolders) {
      processPropertyPlaceHolders();
    }

    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    scanner.setAddToConfig(this.addToConfig);
    scanner.setAnnotationClass(this.annotationClass);
    scanner.setMarkerInterface(this.markerInterface);
    scanner.setSqlSessionFactory(this.sqlSessionFactory);
    scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
    scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
    scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
    scanner.setResourceLoader(this.applicationContext);
    scanner.setBeanNameGenerator(this.nameGenerator);
    scanner.registerFilters();
    scanner.scan(StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
  }

```
这种方法,默认不会走到上述讨论的`getBeansOfType()`方法  
虽然在`PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors()`方法里  
也会走到
```java
postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
```
进而走到`doGetBeanNamesForType()`方法  
但是***注意***,这里的最后一个参数,也就是`allowEagerInit`,是`false`,也就是并不会提前初始化`bean`.  
所以这就保证了`MapperScannerConfigurer`提前扫描配置的`mapper`包后,也不会初始化这些`bean`.  
而是在`PropertyResourceConfigurer.convertProperties`方法被调用过后,才会初始化`bean`.


不过,对于修复过的新版本jar包,我们仍然可以重现问题.  
一个很简单的方法:在配置`MapperScannerConfigurer`的时候,添加一个值:
```xml
<property name="processPropertyPlaceHolders" value="true"/>
```
这样配置,就会导致`MapperScannerConfigurer`初始化的时候,就走到`processPropertyPlaceHolders()`方法,然后重现bug

---

通过对上面三个问题的研究,`mybatis`在`spring`中的初始化过程,就相对很清晰了.(有时间的时候,上个图)




















