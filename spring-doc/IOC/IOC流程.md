# 容器初始化流程

## 示例

```java
//解析demo1.xml配置，刷新容器(启动时初始化容器)
BeanFactory beanFactory=new ClassPathXmlApplicationContext("demo1.xml");
//获取bean
User user=(User)beanFactory.getBean("user");
```

这段代码包含了对XML解析、容器初始化、Bean的获取

## 过程分析
### ClassPathXmlApplicationContext
- ### [类图](类图/ClassPathXmlApplicationContex类图.md)

    ![ClassPathXmlApplicationContex](resource/ClassPathXmlApplicationContext类图.png)
  
- 构造方法
```java
public ClassPathXmlApplicationContext(
    String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
    throws BeansException {

    super(parent);
    //替换占位符，这里初始化了environment
    setConfigLocations(configLocations);
    if (refresh) {
        //XML解析和容器初始化
        refresh();
    }
}
```
ClassPathXmlApplicationContext构造有两种方式,
- 指定多个config location，保存为String[] configLocations，定义在AbstractRefreshableConfigApplicationContext
- 指定classpath路径和Class，保存为Resource[] configResources，定义在ClassPathXmlApplicationContext

下图为loadBeanDefinitions(AbstractXmlApplicationContext), 可以看到两种配置都加载，具体实现后面再介绍
```java
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
    //ClassPathXmlApplicationContext第二种构造方式保存的配置
    Resource[] configResources = getConfigResources();
    if (configResources != null) {
        reader.loadBeanDefinitions(configResources);
    }
    
    //ClassPathXmlApplicationContext第一种构造方式保存的配置
    String[] configLocations = getConfigLocations();
    if (configLocations != null) {
        reader.loadBeanDefinitions(configLocations);
    }
}
```
 在构造方法中执行了两个操作
> - 对参数中的占位符替换
> - 刷新容器
#### 占位符替换
即setConfigLocations(configLocations),接收一个String类型数组，替换后保存到configLocations中，参数为空时，configLocations置为null，实现如下
```java
public void setConfigLocations(@Nullable String... locations) {
    if (locations != null) {
        Assert.noNullElements(locations, "Config locations must not be null");
        this.configLocations = new String[locations.length];
        for (int i = 0; i < locations.length; i++) {
            //替换每一个location中的占位符
            this.configLocations[i] = resolvePath(locations[i]).trim();
        }
    }
    else {
        this.configLocations = null;
    }
}
```
主要是resolvePath方法，先获取环境变量，再对每一个传入的字符串进行替换。
```java
protected String resolvePath(String path) {
    return getEnvironment().resolveRequiredPlaceholders(path);
}
```
- getEnvironment()获取环境变量, 返回[ConfigurableEnvironment](../../spring-core/src/main/java/org/springframework/core/env/ConfigurableEnvironment.java)，参考[StandardEnvironment类图](类图/StandardEnvironment类图.md)，代码如下
> ```java
> public ConfigurableEnvironment getEnvironment() {
>    if (this.environment == null) {
>        this.environment = createEnvironment();
>    }
>    return this.environment;
> }
> ```
> ```java
> protected ConfigurableEnvironment createEnvironment() {
>    return new StandardEnvironment();
> }
> ```
> StandardEnvironment默认空构造，会调用父类[AbstractEnvironment](../../spring-core/src/main/java/org/springframework/core/env/AbstractEnvironment.java)的空构造,代码如下
> ```java
> public AbstractEnvironment() {
>    customizePropertySources(this.propertySources);
> }
> ```
> customizePropertySources在父类是空实现，具体的在子类中配置，即构造函数中默认调用子类customizePropertySources(this.propertySources),子类的实现如下
> ```java
> protected void customizePropertySources(MutablePropertySources propertySources) {
>   propertySources.addLast(
>     new PropertiesPropertySource(SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME, getSystemProperties()));
>   propertySources.addLast(
>     new SystemEnvironmentPropertySource(SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME, getSystemEnvironment()));
> }
> ```
> 把环境变量和属性存到MutablePropertySources propertySources属性中

- resolveRequiredPlaceholders(path)替换占位符,实现在[AbstractEnvironment](../../spring-core/src/main/java/org/springframework/core/env/AbstractEnvironment.java),是ConfigurableEnvironment的实现类，代码如下
> ```java
> public String resolveRequiredPlaceholders(String text) throws IllegalArgumentException {
>   if (this.strictHelper == null) {
>       this.strictHelper = createPlaceholderHelper(false);
>   }
>   return doResolvePlaceholders(text, this.strictHelper);
> }
> ```
> ```java
> private PropertyPlaceholderHelper createPlaceholderHelper(boolean ignoreUnresolvablePlaceholders) {
>   return new PropertyPlaceholderHelper(this.placeholderPrefix, this.placeholderSuffix,
>           this.valueSeparator, ignoreUnresolvablePlaceholders);
> }
> ```
> placeholderPrefix 为"＄{"， placeholderSuffix为"}"，valueSeparator为":"，ignoreUnresolvablePlaceholders为false
> ```java
> public PropertyPlaceholderHelper(String placeholderPrefix, String placeholderSuffix,
>         @Nullable String valueSeparator, boolean ignoreUnresolvablePlaceholders) {
>
>     Assert.notNull(placeholderPrefix, "'placeholderPrefix' must not be null");
>     Assert.notNull(placeholderSuffix, "'placeholderSuffix' must not be null");
>     this.placeholderPrefix = placeholderPrefix;
>     this.placeholderSuffix = placeholderSuffix;
>     String simplePrefixForSuffix = wellKnownSimplePrefixes.get(this.placeholderSuffix);
>     if (simplePrefixForSuffix != null && this.placeholderPrefix.endsWith(simplePrefixForSuffix)) {
>         this.simplePrefix = simplePrefixForSuffix;
>     }
>     else {
>         this.simplePrefix = this.placeholderPrefix;
>     }
>     this.valueSeparator = valueSeparator;
>     this.ignoreUnresolvablePlaceholders = ignoreUnresolvablePlaceholders;
> }
> ```
> doResolvePlaceholders(text, this.strictHelper)代码如下
> ```java
> private String doResolvePlaceholders(String text, PropertyPlaceholderHelper helper) {
>    return helper.replacePlaceholders(text, this::getPropertyAsRawString);
> }
> ```
> 其中```this::getPropertyAsRawString```等价于```placeholderName -> getPropertyAsRawString(placeholderName)``` 返回PlaceholderResolver
> ```java
> @FunctionalInterface
> public interface PlaceholderResolver {
>     @Nullable
>     String resolvePlaceholder(String placeholderName);
> }
> ```
> helper.replacePlaceholders实现如下
> ```java
> public String replacePlaceholders(String value, PlaceholderResolver placeholderResolver) {
>     Assert.notNull(value, "'value' must not be null");
>     return parseStringValue(value, placeholderResolver, null);
> }
> ```
>
> ```java
> protected String parseStringValue(
>         String value, PlaceholderResolver placeholderResolver, @Nullable Set<String> visitedPlaceholders) {
> 
>     int startIndex = value.indexOf(this.placeholderPrefix);
>     if (startIndex == -1) {
>         return value;
>     }
>
>     StringBuilder result = new StringBuilder(value);
>     while (startIndex != -1) {
>         int endIndex = findPlaceholderEndIndex(result, startIndex);
>         if (endIndex != -1) {
>             String placeholder = result.substring(startIndex + this.placeholderPrefix.length(), endIndex);
>             String originalPlaceholder = placeholder;
>             if (visitedPlaceholders == null) {
>                 visitedPlaceholders = new HashSet<>(4);
>             }
>             if (!visitedPlaceholders.add(originalPlaceholder)) {
>                 throw new IllegalArgumentException(
>                         "Circular placeholder reference '" + originalPlaceholder + "' in property definitions");
>             }
>             // Recursive invocation, parsing placeholders contained in the placeholder key.
>             placeholder = parseStringValue(placeholder, placeholderResolver, visitedPlaceholders);
>             // Now obtain the value for the fully resolved key...
>             String propVal = placeholderResolver.resolvePlaceholder(placeholder);
>             if (propVal == null && this.valueSeparator != null) {
>                 int separatorIndex = placeholder.indexOf(this.valueSeparator);
>                 if (separatorIndex != -1) {
>                     String actualPlaceholder = placeholder.substring(0, separatorIndex);
>                     String defaultValue = placeholder.substring(separatorIndex + this.valueSeparator.length());
>                     propVal = placeholderResolver.resolvePlaceholder(actualPlaceholder);
>                     if (propVal == null) {
>                         propVal = defaultValue;
>                     }
>                 }
>             }
>             if (propVal != null) {
>                 // Recursive invocation, parsing placeholders contained in the
>                 // previously resolved placeholder value.
>                 propVal = parseStringValue(propVal, placeholderResolver, visitedPlaceholders);
>                 result.replace(startIndex, endIndex + this.placeholderSuffix.length(), propVal);
>                 if (logger.isTraceEnabled()) {
>                     logger.trace("Resolved placeholder '" + placeholder + "'");
>                 }
>                 startIndex = result.indexOf(this.placeholderPrefix, startIndex + propVal.length());
>             }
>             else if (this.ignoreUnresolvablePlaceholders) {
>                 // Proceed with unprocessed value.
>                 startIndex = result.indexOf(this.placeholderPrefix, endIndex + this.placeholderSuffix.length());
>             }
>             else {
>                 throw new IllegalArgumentException("Could not resolve placeholder '" +
>                         placeholder + "'" + " in value \"" + value + "\"");
>             }
>             visitedPlaceholders.remove(originalPlaceholder);
>         }
>         else {
>             startIndex = -1;
>         }
>     }
>     return result.toString();
> }
> ```
> 支持嵌套和分隔符，resolvePlaceholder方法获取对应的环境变量，replace完成替换。不细究

#### 刷新容器
refresh 方法,定义在ConfigurableApplicationContext接口中,实现在
```java
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        // 提供子类操作的initPropertySources， 验证环境变量
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        // 配置文件解析，注册BeanDefinition
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        // BeanFactory的初始配置
        prepareBeanFactory(beanFactory);

        try {
          // Allows post-processing of the bean factory in context subclasses.
          // 供子类实现
          postProcessBeanFactory(beanFactory);
  
          // Invoke factory processors registered as beans in the context.
          // BeanFactory后置处理器，通知注册BeanDefinition
          invokeBeanFactoryPostProcessors(beanFactory);
  
          // Register bean processors that intercept bean creation.
          // 注册后置处理器
          registerBeanPostProcessors(beanFactory);
  
          // Initialize message source for this context.
          // 国际化
          initMessageSource();
  
          // Initialize event multicaster for this context.
          // 初始化ApplicationEventMulticaster，默认实现SimpleApplicationEventMulticaster
          initApplicationEventMulticaster();
  
          // Initialize other special beans in specific context subclasses.
          // 子类实现此时普通Bean未初始化
          onRefresh();
  
          // Check for listener beans and register them.
          // 初始化ApplicationListener,并广播earlyApplicationEvents
          registerListeners();
  
          // Instantiate all remaining (non-lazy-init) singletons.
          // 初始化剩余Bean
          finishBeanFactoryInitialization(beanFactory);
  
          // Last step: publish corresponding event.
          // 发送ContextRefreshedEvent事件
          finishRefresh();
        }

        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
            logger.warn("Exception encountered during context initialization - " +
            "cancelling refresh attempt: " + ex);
            }

            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // Reset 'active' flag.
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }

        finally {
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
        }
    }
}
```
## prepareRefresh()
> - prepareRefresh(),准备刷新，提供initPropertySources给子类实现，验证环境对象中的requiredProperties属性是否都存在,将初始化earlyApplicationListeners，
> 并添加到applicationListeners，初始化earlyApplicationEvents。
> ```java
> protected void prepareRefresh() {
>     // Switch to active.
>     this.startupDate = System.currentTimeMillis();
>     this.closed.set(false);
>     this.active.set(true);
>
>     if (logger.isDebugEnabled()) {
>         if (logger.isTraceEnabled()) {
>             logger.trace("Refreshing " + this);
>         }
>         else {
>             logger.debug("Refreshing " + getDisplayName());
>         }
>     }
>
>     // Initialize any placeholder property sources in the context environment.
>     // 可以配置必要的环境变量,添加earlyApplicationListeners
>     initPropertySources();
>
>     // Validate that all properties marked as required are resolvable:
>     // see ConfigurablePropertyResolver#setRequiredProperties
>     // 获取所有环境变量，验证requiredProperties中变量是否存在
>     getEnvironment().validateRequiredProperties();
>
>     // Store pre-refresh ApplicationListeners...
>     // 初始化earlyApplicationListeners
>     if (this.earlyApplicationListeners == null) {
>         this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
>     }
>     else {
>         // Reset local application listeners to pre-refresh state.
>         this.applicationListeners.clear();
>         this.applicationListeners.addAll(this.earlyApplicationListeners);
>     }
>
>     // Allow for the collection of early ApplicationEvents,
>     // to be published once the multicaster is available...
>     // 初始化earlyApplicationEvents
>     this.earlyApplicationEvents = new LinkedHashSet<>();
> }
> ```
> >  initPropertySources中能做那些事(持续补充中...)
> > - 指定需要验证的环境变量，后续会验证环境变量是否存在
> > - 配置自定义的earlyApplicationListeners
> getEnvironment在占位符替换已经介绍了，这里也不用再初始化了，validateRequiredProperties代码如下
> ```java
> public void validateRequiredProperties() throws MissingRequiredPropertiesException {
>   this.propertyResolver.validateRequiredProperties();
> }
> ```
> this.propertyResolver中包含了所有的属性和环境变量
> ```java
> private final ConfigurablePropertyResolver propertyResolver =
>   new PropertySourcesPropertyResolver(this.propertySources);
> ```
> - propertyResolver默认使用[PropertySourcesPropertyResolver](../../spring-core/src/main/java/org/springframework/core/env/PropertySourcesPropertyResolver.java),参考[PropertySourcesPropertyResolver类图](类图/StandardEnvironment类图.md)
> - validateRequiredProperties在其父类[AbstractPropertyResolver](../../spring-core/src/main/java/org/springframework/core/env/AbstractPropertyResolver.java)中实现,代码如下
> - propertySources在AbstractEnvironment初始化的时候保存了环境变量
> ```java
> public void validateRequiredProperties() {
>   MissingRequiredPropertiesException ex = new MissingRequiredPropertiesException();
>   for (String key : this.requiredProperties) {
>       if (this.getProperty(key) == null) {
>           ex.addMissingRequiredProperty(key);
>       }
>   }
>   if (!ex.getMissingRequiredProperties().isEmpty()) {
>       throw ex;
>   }
> }
> ```
> 遍历requiredProperties，判断是否存在该属性或者环境变量值，可以通过子类实现initPropertySources方法,调用getEnvironment().setRequiredProperties配置必须的环境变量

## ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
> 这里面解析XML文件并加载BeanDefinition,过程比较复杂，参考[XML解析.md](XML解析.md)

## prepareBeanFactory(beanFactory);
> 主要是对beanFactory配置和添加BeanPostProcessor，参考[BeanPostProcessor](BeanPostProcessor类介绍.md)
```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
      // Tell the internal bean factory to use the context's class loader etc.
      beanFactory.setBeanClassLoader(getClassLoader());
      beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
      beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

      // Configure the bean factory with context callbacks.
      beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
      beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
      beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
      beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
      beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
      beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
      beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

      // BeanFactory interface not registered as resolvable type in a plain factory.
      // MessageSource registered (and found for autowiring) as a bean.
      beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
      beanFactory.registerResolvableDependency(ResourceLoader.class, this);
      beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
      beanFactory.registerResolvableDependency(ApplicationContext.class, this);

      // Register early post-processor for detecting inner beans as ApplicationListeners.
      beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

      // Detect a LoadTimeWeaver and prepare for weaving, if found.
      if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
          beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
          // Set a temporary ClassLoader for type matching.
          beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
      }

      // Register default environment beans.
      if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
          beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
      }
      if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
          beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
      }
      if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
          beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
      }
  }
```
> - ignoreDependencyInterface
> - registerResolvableDependency
> - registerSingleton 注册Bean

## postProcessBeanFactory(beanFactory);
> 子类实现，可以添加BeanFactoryPostProcessors

## invokeBeanFactoryPostProcessors(beanFactory);
> 执行BeanFactoryPostProcessors中的postProcessBeanDefinitionRegistry方法

## registerBeanPostProcessors

```java
public static void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

		String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

		// Register BeanPostProcessorChecker that logs an info message when
		// a bean is created during BeanPostProcessor instantiation, i.e. when
		// a bean is not eligible for getting processed by all BeanPostProcessors.
		int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
		beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

		// Separate between BeanPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
		List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
		List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
		List<String> orderedPostProcessorNames = new ArrayList<>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<>();
		for (String ppName : postProcessorNames) {
			if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
				priorityOrderedPostProcessors.add(pp);
				if (pp instanceof MergedBeanDefinitionPostProcessor) {
					internalPostProcessors.add(pp);
				}
			}
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// First, register the BeanPostProcessors that implement PriorityOrdered.
		sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

		// Next, register the BeanPostProcessors that implement Ordered.
		List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
		for (String ppName : orderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			orderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		sortPostProcessors(orderedPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, orderedPostProcessors);

		// Now, register all regular BeanPostProcessors.
		List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
		for (String ppName : nonOrderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			nonOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

		// Finally, re-register all internal BeanPostProcessors.
		sortPostProcessors(internalPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, internalPostProcessors);

		// Re-register post-processor for detecting inner beans as ApplicationListeners,
		// moving it to the end of the processor chain (for picking up proxies etc).
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
	}
```

### initMessageSource

```java
protected void initMessageSource() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
			this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
			// Make MessageSource aware of parent MessageSource.
			if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
				HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
				if (hms.getParentMessageSource() == null) {
					// Only set parent context as parent MessageSource if no parent MessageSource
					// registered already.
					hms.setParentMessageSource(getInternalParentMessageSource());
				}
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Using MessageSource [" + this.messageSource + "]");
			}
		}
		else {
			// Use empty MessageSource to be able to accept getMessage calls.
			DelegatingMessageSource dms = new DelegatingMessageSource();
			dms.setParentMessageSource(getInternalParentMessageSource());
			this.messageSource = dms;
			beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
			if (logger.isTraceEnabled()) {
				logger.trace("No '" + MESSAGE_SOURCE_BEAN_NAME + "' bean, using [" + this.messageSource + "]");
			}
		}
	}
```




