#### `Bean`的生命周期

------



##### 基于XML配置文件：

1.创建XML文件，定义`Bean`信息

2.通过`ClassPathXMlApplicationContenxt`类的`ClassPathXMlApplicationContenxt("beans.xml")`,XML文件存于`AbstractRefreshableConfigApplicationContenxt`中的`ConfigLocations`内

```java
	/*
	* ClassPathXMlApplicationContenxt
	*/
	public ClassPathXmlApplicationContext(
			String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
			throws BeansException {

		super(parent);
		setConfigLocations(configLocations);
		if (refresh) {
			refresh();
		}
	}

	public void setConfigLocations(@Nullable String... locations) {
		if (locations != null) {
			Assert.noNullElements(locations, "Config locations must not be null");
			this.configLocations = new String[locations.length];
			for (int i = 0; i < locations.length; i++) {
				this.configLocations[i] = resolvePath(locations[i]).trim();
			}
		}
		else {
			this.configLocations = null;
		}
	}
```

3.执行`refresh`,`AbstractApplicationContenxt`是`ApplicationContenxt`的抽象实现类，该方法定义了容器在加载配置文件后的各种处理，这里会调用通用refresh处理类`AbsractRefreshApplicationContenxt`处理刷新容器的任务，在`AbsractRefreshApplicationContenxt`中的`refreshBeanFactory`方法中调用`loadBeandefinitions`方法通过`location`获取`Resources`对象，该对象是一个数组，然后让具体实现加载Bean定义的类通过该resource去加载bean的定义，也就是让`XmlBeandefinitionReader`去通过生成的Resources加载Beandefinition

```java
/*
* AbstractApplicationContenxt
*/
@Override
	public void refresh() throws BeansException, IllegalStateException {
    	...
			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
  		...
  }

	/**
	 * Tell the subclass to refresh the internal bean factory.
	 * @return the fresh BeanFactory instance
	 * @see #refreshBeanFactory()
	 * @see #getBeanFactory()
	 */
	protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		refreshBeanFactory();//实现方法 - AbstractRefreshApplicationContext.refreshBeanFactory() \ GenericApplicationContext.refreshBeanFactory()
		return getBeanFactory();
	}
```

```java
/**
	 * AbstracRefreshableApplicationContenxt
	 * This implementation performs an actual refresh of this context's underlying
	 * bean factory, shutting down the previous bean factory (if any) and
	 * initializing a fresh bean factory for the next phase of the context's lifecycle.
	 */
	@Override
	protected final void refreshBeanFactory() throws BeansException {
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			customizeBeanFactory(beanFactory);
      /*
      * 实现类
      * AbstractXmlApplicationContext.loadBeanDefinitions()
      * AnnotationConfigWebApplicationContext.loadBeandefinitions()
      * GrroyWebApplicationContext.loadBeandefintions()
      * XmlWebApplicationContext.loadBeandefinitions()
      */
			loadBeanDefinitions(beanFactory);
			this.beanFactory = beanFactory;
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
```

`AbstractXmlApplicationContext`执行`loadBeandefinitions`方法,从Xml文件中读取类定义

```java
	/**
	 *  AbstractXmlApplicationContext
	 */
public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
		ResourceLoader resourceLoader = getResourceLoader();
		if (resourceLoader == null) {
			throw new BeanDefinitionStoreException(
					"Cannot load bean definitions from location [" + location + "]: no ResourceLoader available");
		}

		if (resourceLoader instanceof ResourcePatternResolver) {
			// Resource pattern matching available.
			try {
        //生成InputStream流对象
				Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
        //交给具体实现加载Beandefinition的XmlBeandefinitionReader实现
				int count = loadBeanDefinitions(resources);
				if (actualResources != null) {
					Collections.addAll(actualResources, resources);
				}
				if (logger.isTraceEnabled()) {
					logger.trace("Loaded " + count + " bean definitions from location pattern [" + location + "]");
				}
				return count;
			}
			catch (IOException ex) {
				throw new BeanDefinitionStoreException(
						"Could not resolve bean definition resource pattern [" + location + "]", ex);
			}
		}
		else {
			// Can only load single resources by absolute URL.
			Resource resource = resourceLoader.getResource(location);
			int count = loadBeanDefinitions(resource);
			if (actualResources != null) {
				actualResources.add(resource);
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Loaded " + count + " bean definitions from location [" + location + "]");
			}
			return count;
		}
}
```

`XmlBeanDefinitionReader`，在`AbstractBeandefinitionReader`中已生成Resources

```java
	@Override
	public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
		return loadBeanDefinitions(new EncodedResource(resource));
	}

	/**
	 * Load bean definitions from the specified XML file.
	 * @param encodedResource the resource descriptor for the XML file,
	 * allowing to specify an encoding to use for parsing the file
	 * @return the number of bean definitions found
	 * @throws BeanDefinitionStoreException in case of loading or parsing errors
	 */
	public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isTraceEnabled()) {
			logger.trace("Loading XML bean definitions from " + encodedResource);
		}

		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();

		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}
		//从resource对象中获取文件流
		try (InputStream inputStream = encodedResource.getResource().getInputStream()) {
      //包装成InputSource对象
			InputSource inputSource = new InputSource(inputStream);
			if (encodedResource.getEncoding() != null) {
				inputSource.setEncoding(encodedResource.getEncoding());
			}
      //实际的beandefinitions解析方法入口
			return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"IOException parsing XML document from " + encodedResource.getResource(), ex);
		}
		finally {
			currentResources.remove(encodedResource);
			if (currentResources.isEmpty()) {
				this.resourcesCurrentlyBeingLoaded.remove();
			}
		}
	}

protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {

		try {
      //解析xml并封装成Document对象然后返回
			Document doc = doLoadDocument(inputSource, resource);
      //传入Document对象，将解析到的标签元素封装成beandefinitions
			int count = registerBeanDefinitions(doc, resource);
			if (logger.isDebugEnabled()) {
				logger.debug("Loaded " + count + " bean definitions from " + resource);
			}
			return count;
		}
}

```

`DefaultBeandefinitionDocumentReader` - 完成beanDefinition的封装

```java
	@Override
	public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
		this.readerContext = readerContext;
		doRegisterBeanDefinitions(doc.getDocumentElement());
	}
	protected void doRegisterBeanDefinitions(Element root) {
		// Any nested <beans> elements will cause recursion in this method. In
		// order to propagate and preserve <beans> default-* attributes correctly,
		// keep track of the current (parent) delegate, which may be null. Create
		// the new (child) delegate with a reference to the parent for fallback purposes,
		// then ultimately reset this.delegate back to its original (parent) reference.
		// this behavior emulates a stack of delegates without actually necessitating one.
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);

		if (this.delegate.isDefaultNamespace(root)) {
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
				// We cannot use Profiles.of(...) since profile expressions are not supported
				// in XML config. See SPR-12458 for details.
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					if (logger.isDebugEnabled()) {
						logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
								"] not matching: " + getReaderContext().getResource());
					}
					return;
				}
			}
		}
		//前置处理xml元素
		preProcessXml(root);
    //解析root节点以及下面的所有节点 - 解析beandefinition的方法
		parseBeanDefinitions(root, this.delegate);
		//后置处理xml元素
    postProcessXml(root);

		this.delegate = parent;
	}

	protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		if (delegate.isDefaultNamespace(root)) {
      //获取root下所有子节点
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
          //判断是否有唯一的xml:ns标签，若没有则当作自定义标签处理
					if (delegate.isDefaultNamespace(ele)) {
            //解析普通默认标签：import bean alias...
						parseDefaultElement(ele, delegate);
					}
					else {
            //解析自定义标签：<contenxt:compant-scan basepackage="xxx.xxx">等前缀的标签
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			delegate.parseCustomElement(root);
		}
	}
	
	//解析默认标签 import bean alias ...
	private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
    //解析import标签
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			importBeanDefinitionResource(ele);
		
    //解析alias标签
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			processAliasRegistration(ele);
		}
    //解析bean标签
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
			processBeanDefinition(ele, delegate);
		}
    //解析bean标签嵌套的标签
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// recurse
			doRegisterBeanDefinitions(ele);
		}
	}
  //解析bean标签 
  protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
		//标签元素解析封装成beanDefinitionHolder对象
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
      //对解析结果进行装饰，动态添加其他功能
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// Register the final decorated instance.
        //注册beanDefinition对象并且缓存
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// Send registration event.
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}
```

`BeanDefinitionReaderUtils` - 工具类，注册beandefinitons

```java
	public static void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {

		// Register bean definition under primary name.
		String beanName = definitionHolder.getBeanName();
    //注册beandefinition的核心方法 - BeanDefinition注册的目标接口 - DefaultListableBeanFactory
		registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

		// Register aliases for bean name, if any.
    //注册bean的别名alias，如果有别名的话
		String[] aliases = definitionHolder.getAliases();
		if (aliases != null) {
			for (String alias : aliases) {
				registry.registerAlias(beanName, alias);
			}
		}
	}
```

`DefaultListableBeanFactory`

```java
	@Override
	public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {

		Assert.hasText(beanName, "Bean name must not be empty");
		Assert.notNull(beanDefinition, "BeanDefinition must not be null");

		if (beanDefinition instanceof AbstractBeanDefinition) {
			try {
				((AbstractBeanDefinition) beanDefinition).validate();
			}
			catch (BeanDefinitionValidationException ex) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Validation of bean definition failed", ex);
			}
		}

		BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
		if (existingDefinition != null) {
			if (!isAllowBeanDefinitionOverriding()) {
				throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
			}
			else if (existingDefinition.getRole() < beanDefinition.getRole()) {
				// e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
				if (logger.isInfoEnabled()) {
					logger.info("Overriding user-defined bean definition for bean '" + beanName +
							"' with a framework-generated bean definition: replacing [" +
							existingDefinition + "] with [" + beanDefinition + "]");
				}
			}
			else if (!beanDefinition.equals(existingDefinition)) {
				if (logger.isDebugEnabled()) {
					logger.debug("Overriding bean definition for bean '" + beanName +
							"' with a different definition: replacing [" + existingDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
			else {
				if (logger.isTraceEnabled()) {
					logger.trace("Overriding bean definition for bean '" + beanName +
							"' with an equivalent definition: replacing [" + existingDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
			this.beanDefinitionMap.put(beanName, beanDefinition);
		}
		else {
			if (hasBeanCreationStarted()) {
				// Cannot modify startup-time collection elements anymore (for stable iteration)
				synchronized (this.beanDefinitionMap) {
					this.beanDefinitionMap.put(beanName, beanDefinition);
					List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
					updatedDefinitions.addAll(this.beanDefinitionNames);
					updatedDefinitions.add(beanName);
					this.beanDefinitionNames = updatedDefinitions;
					removeManualSingletonName(beanName);
				}
			}
			else {
				// Still in startup registration phase
        //beandefinition放到Map中
				this.beanDefinitionMap.put(beanName, beanDefinition);
        //beanName放到Map中
				this.beanDefinitionNames.add(beanName);
				removeManualSingletonName(beanName);
			}
			this.frozenBeanDefinitionNames = null;
		}

		if (existingDefinition != null || containsSingleton(beanName)) {
			resetBeanDefinition(beanName);
		}
		else if (isConfigurationFrozen()) {
			clearByTypeCache();
		}
	}
```



####  `BeanFactoryPostProcessor ` and `BeanPostProcessor` 

------

`BeanFactoryPostProcessor`顾名思义是针对BeanFactory的增强，而BeanFactory中存储了BeanDefinitions，显而易见的是对BeanDefinition的操作，它是Spring提供的容器扩展机制，它可以让Bean在**实例化之前**修改Bean的定义信息或者说类的元数据，即`BeanDefintion`

`BeanPostProcessor`同样是Spring提供的容器扩展机制，不同的是，它在Bean**实例化后、执行初始化方法前后**修改或替换Bean，`BeanPostProcessor`是实现AOP的关键

无论是何种方式注入Bean，`postProcessor`都可在在复杂情况下修改`BeanDefintions`，以应对需要扩展需求的生产环境中，这一功能正是能证明Spring强大的生态

后置处理器在创建完容器后并且加载完BeanDefinition后，在容器刷新方法中的`invokeBeanFactoryPostProcessor()`方法中调用BeanFactory的各种加强器(后置处理器)

此处对应的Bean的注入、`BeanFactoryPostProcessor`在Bean注入的顺序：

1. 加载配置文件读取BeanDefinitions
2. <u>BeanFactory后置处理 - `BeanFactoryPostProcessor`，即修改BeanDefinitions</u>
3. 实例化
4. Bean
5. 初始化



`Person`类的name通过注入为`who`，自定义类实现BeanFactoryPostProcessor，重写`postProcessBeanFactory`方法，可在实例化该类前修改`name`的值为`xiaoyao`

```xml
<bean id="person" class="com.jiakang.admin.testfile.Person" init-method="initMethod">
    <property name="name" value="who"/>
</bean>
```

```java
@Component
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        GenericApplicationContext context = new GenericApplicationContext();
        new XmlBeanDefinitionReader(context).loadBeanDefinitions("applicationContenxt.xml");
        BeanDefinition beanDefinition = context.getBeanDefinition("person");
        System.out.println("原来的值：" + beanDefinition.getPropertyValues().toString());
        MutablePropertyValues values = beanDefinition.getPropertyValues();
        values.add("name", "xiaoyao");
    }
}
```

`BeanPostProcessor`定义了几个回调逻辑，可以自定义类实现该接口

```java
public class MyBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("... " + beanName);
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
}
```

`beans.xml`,这使自定义的`BeanPostProcessor`以及`Person`类处在同一个容器中，Spring只会处理该自定义的`BeanPostProcessor`实现类所在容器创建的类

```xml
<bean id="person" class="com.jiakang.admin.testfile.Person" init-method="initMethod" name="person">
  <property name="name" value="who"/>
</bean>

<bean id="myBeanPostProcessor" class="com.jiakang.admin.testfile.MyBeanPostProcessor"/>
```



#### `ConfigurationClasspostProcessor` --- 

------

找到`SpringApplication run()`方法中的`refreshContext(context)`

首先，根据程序类型`this.webApplicationType`定义了`context`为`AnnotationConfigServletWebServerApplication`,最后调用本类的`refresh()`方法，而`AnnotationConfigServletWebServerApplication`继承自`AbstractApplicationContenxt`，所以最后会调用`AbstractApplicationContenxt`中的`refresh()`方法，代码：

```java
/*
* SpringApplication
*/

public ConfigurableApplicationContext run(String... args) {
		...
		context = createApplicationContext();
		...
		refreshContext(context);
		...
}

protected ConfigurableApplicationContext createApplicationContext() {
		return this.applicationContextFactory.create(this.webApplicationType);
}

protected void refresh(ConfigurableApplicationContext applicationContext) {
		applicationContext.refresh();
}
```

本方法会实例化和调用所有 `BeanFactoryPostProcessor`，包括其子类 `BeanDefinitionRegistryPostProcessor`

```java
/*
* AbstractApplicationContext
*/
@Override
	public void refresh() throws BeansException, IllegalStateException {
    ...
    // Invoke factory processors registered as beans in the context.
    invokeBeanFactoryPostProcessors(beanFactory);
    ...
  }

/**
* Instantiate and invoke all registered BeanFactoryPostProcessor beans,
* respecting explicit order if given.
* <p>Must be called before singleton instantiation.
*/
	protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
		// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
		// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
		if (!NativeDetector.inNativeImage() && beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
		beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
		beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}
	}
```



#### `ConfigurationClassBeanDefintionReader` --- 注册`@Configuration`配置类到Spring容器

------

该类基于javaConfig方式将`beanDefintion`注入到`DefaultListableBeanFactory`中

#### `BeanDefinition` --- 

#### `DefaultListableBeanFactory` --- 加载`BeanDefinition`到容器

------



#### `AliasRegistry` --- 别名注册器

------

该类定义了从注册表中增加别名、移除别名、判断是否存在别名以及获取指定的别名四个基本管理Bean别名的功能。

```java
public interface AliasRegistry {
/**
 * Given a name, register an alias for it.
 * @param name the canonical name
 * @param alias the alias to be registered
 * @throws IllegalStateException if the alias is already in use
 * and may not be overridden
 */
void registerAlias(String name, String alias);

/**
 * Remove the specified alias from this registry.
 * @param alias the alias to remove
 * @throws IllegalStateException if no such alias was found
 */
void removeAlias(String alias);

/**
 * Determine whether the given name is defined as an alias
 * (as opposed to the name of an actually registered component).
 * @param name the name to check
 * @return whether the given name is an alias
 */
boolean isAlias(String name);

/**
 * Return the aliases for the given name, if defined.
 * @param name the name to check for aliases
 * @return the aliases, or an empty array if none
 */
String[] getAliases(String name);

}
```

SimpleAliasRegistry作为AliasRegistry默认接口实现，它维护了一个ConcurrentHashMap作为注册表：

`private final Map<String, String> aliasMap = new ConcurrentHashMap<>(16);`

当在Spring中存在







