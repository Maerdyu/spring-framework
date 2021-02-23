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
- getEnvironment()获取环境变量, 返回[ConfigurableEnvironment](../../../spring-core/src/main/java/org/springframework/core/env/ConfigurableEnvironment.java)，参考[StandardEnvironment类图](类图/StandardEnvironment类图.md)，代码如下
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
> StandardEnvironment默认空构造，会调用父类[AbstractEnvironment](../../../spring-core/src/main/java/org/springframework/core/env/AbstractEnvironment.java)的空构造,代码如下
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

- resolveRequiredPlaceholders(path)替换占位符,实现在[AbstractEnvironment](../../../spring-core/src/main/java/org/springframework/core/env/AbstractEnvironment.java),是ConfigurableEnvironment的实现类，代码如下
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
> propertyResolver默认使用[PropertySourcesPropertyResolver](../../../spring-core/src/main/java/org/springframework/core/env/PropertySourcesPropertyResolver.java),参考[PropertySourcesPropertyResolver类图](类图/StandardEnvironment类图.md)
> validateRequiredProperties在其父类[AbstractPropertyResolver](../../../)中实现,代码如下
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

