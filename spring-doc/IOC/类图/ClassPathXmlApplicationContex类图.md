# ClassPathXmlApplicationContext

- ## 类图如下

![ClassPathXmlApplicationContex](../resource/ClassPathXmlApplicationContext类图.png)

- ## 接口介绍
具体的方法作用，在初始化过程中介绍。
  
  - ### [BeanFactory](../../../spring-beans/src/main/java/org/springframework/beans/factory/BeanFactory.java)
  > 定义多种getBean方法，指定了FactoryBean的前缀&, 其默认实现是[DefaultListableBeanFactory](../../../spring-beans/src/main/java/org/springframework/beans/factory/support/DefaultListableBeanFactory.java)

  - ### [ResourceLoader](../../../spring-core/src/main/java/org/springframework/core/io/ResourceLoader.java)
  > 定义getResource方法，默认实现是[DefaultResourceLoader](../../../spring-core/src/main/java/org/springframework/core/io/DefaultResourceLoader.java)

  - ### [MessageSource](../../../spring-context/src/main/java/org/springframework/context/MessageSource.java)
  > 定义getMessage方法，用于国际化
    
  - ### [ListableBeanFactory](../../../spring-beans/src/main/java/org/springframework/beans/factory/ListableBeanFactory.java)
  > BeanFactory的拓展，包含对BeanDefinition的操作和获取批量Bean操作
    
  - ### [HierarchicalBeanFactory](../../../spring-beans/src/main/java/org/springframework/beans/factory/HierarchicalBeanFactory.java)
  > 提供父工厂访问getParentBeanFactory
  
  - ### [ResourcePatternResolver](../../../)
  > 继承[ResourceLoader](../../../spring-core/src/main/java/org/springframework/core/io/ResourceLoader.java)接口，获取指定路径下的Resource
  > 使用”classpath*:“,可以获取依赖Jar中的配置
  
  - ### [ApplicationEventPublisher](../../../spring-context/src/main/java/org/springframework/context/ApplicationEventPublisher.java)
  > 定义publishEvent方法，发布事件
  
  - ### [EnvironmentCapable](../../../spring-core/src/main/java/org/springframework/core/env/EnvironmentCapable.java)
  > 定义getEnvironment方法，获取环境变量

  - ### [Lifecycle](../../../spring-context/src/main/java/org/springframework/context/Lifecycle.java)
  > 定义start，stop方法

  - ### [AutowireCapableBeanFactory](../../../spring-beans/src/main/java/org/springframework/beans/factory/config/AutowireCapableBeanFactory.java)
  > 继承BeanFactory,主要是用于Bean的装配，定义了Bean的创建、销毁、配置以及BeanPostProcessors的配置方法

  - ### [ApplicationContext](../../../spring-context/src/main/java/org/springframework/context/ApplicationContext.java)
  > 继承多个接口，实现对BeanFactory的拓展，包括国际化、beanDefinition，资源解析，时间发布，环境变量获取等。
  > 本身定义了ID,ApplicationName等的获取，提供[AutowireCapableBeanFactory](../../../spring-beans/src/main/java/org/springframework/beans/factory/config/AutowireCapableBeanFactory.java)的获取

  - ### [ConfigurableApplicationContext](../../../spring-context/src/main/java/org/springframework/context/ConfigurableApplicationContext.java)
  > 继承ApplicationContext，定义addBeanFactoryPostProcessor， addApplicationListener， refresh方法

  - ### [InitializingBean](../../../spring-beans/src/main/java/org/springframework/beans/factory/InitializingBean.java)
  > 定义afterPropertiesSet方法

## 类介绍
  - #### [DefaultResourceLoader](../../../spring-core/src/main/java/org/springframework/core/io/DefaultResourceLoader.java)
  > ResourceLoader的默认实现，是getResource方法，根据传入的不同前缀解析成不同的Resource
```java
public Resource getResource(String location) {
      Assert.notNull(location, "Location must not be null");

      for (ProtocolResolver protocolResolver : getProtocolResolvers()) {
          Resource resource = protocolResolver.resolve(location, this);
          if (resource != null) {
              return resource;
          }
      }

      if (location.startsWith("/")) {
          return getResourceByPath(location);
      }
      else if (location.startsWith(CLASSPATH_URL_PREFIX)) {
          return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
      }
      else {
          try {
              // Try to parse the location as a URL...
              URL url = new URL(location);
              return (ResourceUtils.isFileURL(url) ? new FileUrlResource(url) : new UrlResource(url));
          }
          catch (MalformedURLException ex) {
              // No URL -> resolve as resource path.
              return getResourceByPath(location);
          }
      }
  }
```

  - #### [AbstractApplicationContext](../../../spring-context/src/main/java/org/springframework/context/support/AbstractApplicationContext.java)
  > refresh方法的具体实现，子类包含xml，注解的各种ApplicationContext

  - #### [AbstractRefreshableApplicationContext](../../../spring-context/src/main/java/org/springframework/context/support/AbstractRefreshableApplicationContext.java)
  > 实现refreshBeanFactory方法，定义loadBeanDefinitions抽象方法
```java
protected final void refreshBeanFactory() throws BeansException {
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }
    try {
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        beanFactory.setSerializationId(getId());
        customizeBeanFactory(beanFactory);
        loadBeanDefinitions(beanFactory);
        this.beanFactory = beanFactory;
    }
    catch (IOException ex) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
    }
}
```

  - #### [AbstractRefreshableConfigApplicationContext](../../../spring-context/src/main/java/org/springframework/context/support/AbstractRefreshableConfigApplicationContext.java)
  > 实现占位符替换
```java
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

  - #### [AbstractXmlApplicationContext](../../../spring-context/src/main/java/org/springframework/context/support/AbstractXmlApplicationContext.java)
  > loadBeanDefinitions的实现
```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
    // Create a new XmlBeanDefinitionReader for the given BeanFactory.
    XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

    // Configure the bean definition reader with this context's
    // resource loading environment.
    beanDefinitionReader.setEnvironment(this.getEnvironment());
    beanDefinitionReader.setResourceLoader(this);
    beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

    // Allow a subclass to provide custom initialization of the reader,
    // then proceed with actually loading the bean definitions.
    initBeanDefinitionReader(beanDefinitionReader);
    loadBeanDefinitions(beanDefinitionReader);
}
```