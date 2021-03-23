# Bean获取



## 1.源码过程

>```java
>public Object getBean(String name) throws BeansException {
>   return doGetBean(name, null, null, false);
>}
>```

> ```java
> protected <T> T doGetBean(
>       String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
>       throws BeansException {
> 
>    String beanName = transformedBeanName(name);
>    Object bean;
> 
>    // Eagerly check singleton cache for manually registered singletons.
>    Object sharedInstance = getSingleton(beanName);
>    if (sharedInstance != null && args == null) {
>       if (logger.isTraceEnabled()) {
>          if (isSingletonCurrentlyInCreation(beanName)) {
>             logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
>                   "' that is not fully initialized yet - a consequence of a circular reference");
>          }
>          else {
>             logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
>          }
>       }
>       bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
>    }
> 
>    else {
>       // Fail if we're already creating this bean instance:
>       // We're assumably within a circular reference.
>       if (isPrototypeCurrentlyInCreation(beanName)) {
>          throw new BeanCurrentlyInCreationException(beanName);
>       }
> 
>       // Check if bean definition exists in this factory.
>       BeanFactory parentBeanFactory = getParentBeanFactory();
>       if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
>          // Not found -> check parent.
>          String nameToLookup = originalBeanName(name);
>          if (parentBeanFactory instanceof AbstractBeanFactory) {
>             return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
>                   nameToLookup, requiredType, args, typeCheckOnly);
>          }
>          else if (args != null) {
>             // Delegation to parent with explicit args.
>             return (T) parentBeanFactory.getBean(nameToLookup, args);
>          }
>          else if (requiredType != null) {
>             // No args -> delegate to standard getBean method.
>             return parentBeanFactory.getBean(nameToLookup, requiredType);
>          }
>          else {
>             return (T) parentBeanFactory.getBean(nameToLookup);
>          }
>       }
> 
>       if (!typeCheckOnly) {
>          markBeanAsCreated(beanName);
>       }
> 
>       try {
>          RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
>          checkMergedBeanDefinition(mbd, beanName, args);
> 
>          // Guarantee initialization of beans that the current bean depends on.
>          String[] dependsOn = mbd.getDependsOn();
>          if (dependsOn != null) {
>             for (String dep : dependsOn) {
>                if (isDependent(beanName, dep)) {
>                   throw new BeanCreationException(mbd.getResourceDescription(), beanName,
>                         "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
>                }
>                registerDependentBean(dep, beanName);
>                try {
>                   getBean(dep);
>                }
>                catch (NoSuchBeanDefinitionException ex) {
>                   throw new BeanCreationException(mbd.getResourceDescription(), beanName,
>                         "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
>                }
>             }
>          }
> 
>          // Create bean instance.
>          if (mbd.isSingleton()) {
>             sharedInstance = getSingleton(beanName, () -> {
>                try {
>                   return createBean(beanName, mbd, args);
>                }
>                catch (BeansException ex) {
>                   // Explicitly remove instance from singleton cache: It might have been put there
>                   // eagerly by the creation process, to allow for circular reference resolution.
>                   // Also remove any beans that received a temporary reference to the bean.
>                   destroySingleton(beanName);
>                   throw ex;
>                }
>             });
>             bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
>          }
> 
>          else if (mbd.isPrototype()) {
>             // It's a prototype -> create a new instance.
>             Object prototypeInstance = null;
>             try {
>                beforePrototypeCreation(beanName);
>                prototypeInstance = createBean(beanName, mbd, args);
>             }
>             finally {
>                afterPrototypeCreation(beanName);
>             }
>             bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
>          }
> 
>          else {
>             String scopeName = mbd.getScope();
>             if (!StringUtils.hasLength(scopeName)) {
>                throw new IllegalStateException("No scope name defined for bean ´" + beanName + "'");
>             }
>             Scope scope = this.scopes.get(scopeName);
>             if (scope == null) {
>                throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
>             }
>             try {
>                Object scopedInstance = scope.get(beanName, () -> {
>                   beforePrototypeCreation(beanName);
>                   try {
>                      return createBean(beanName, mbd, args);
>                   }
>                   finally {
>                      afterPrototypeCreation(beanName);
>                   }
>                });
>                bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
>             }
>             catch (IllegalStateException ex) {
>                throw new BeanCreationException(beanName,
>                      "Scope '" + scopeName + "' is not active for the current thread; consider " +
>                      "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
>                      ex);
>             }
>          }
>       }
>       catch (BeansException ex) {
>          cleanupAfterBeanCreationFailure(beanName);
>          throw ex;
>       }
>    }
> ```

> transformedBeanName
>
> 获取Bean的名称，包括FactoryBean,会去掉前缀&

> getSingleton
>
> 从缓存中获取 ，包括singletonObjects， earlySingletonObjects， singletonFactories

> getObjectForBeanInstance
>
> 处理FactoryBean，如果是工厂Bean的Ref或者不是工厂bean，都直接返回，如果是工厂Bean，直接调用getObject方法，然后根据shouldPostProcess和是否单例执行beforeSingletonCreation, postProcessObjectFromFactoryBean,afterSingletonCreation方法。

>如果有父BeanFactory,递归从父Beanfactory获取。

> 合并父类的BeanDefinition,然后检查验证。

> 如果有依赖的，先实例化依赖的Bean

> 根据scope去创建，这里分singleton，prototype和其他，都是直接调用createBean方法，最后判断是否需要类型转换。

> 创建Bean的方法代码如下
>
> ```java
> protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
>       throws BeanCreationException {
> 
>    if (logger.isTraceEnabled()) {
>       logger.trace("Creating instance of bean '" + beanName + "'");
>    }
>    RootBeanDefinition mbdToUse = mbd;
> 
>    // Make sure bean class is actually resolved at this point, and
>    // clone the bean definition in case of a dynamically resolved Class
>    // which cannot be stored in the shared merged bean definition.
>    Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
>    if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
>       mbdToUse = new RootBeanDefinition(mbd);
>       mbdToUse.setBeanClass(resolvedClass);
>    }
> 
>    // Prepare method overrides.
>    try {
>       mbdToUse.prepareMethodOverrides();
>    }
>    catch (BeanDefinitionValidationException ex) {
>       throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
>             beanName, "Validation of method overrides failed", ex);
>    }
> 
>    try {
>       // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
>       Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
>       if (bean != null) {
>          return bean;
>       }
>    }
>    catch (Throwable ex) {
>       throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
>             "BeanPostProcessor before instantiation of bean failed", ex);
>    }
> 
>    try {
>       Object beanInstance = doCreateBean(beanName, mbdToUse, args);
>       if (logger.isTraceEnabled()) {
>          logger.trace("Finished creating instance of bean '" + beanName + "'");
>       }
>       return beanInstance;
>    }
>    catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
>       // A previously detected exception with proper bean creation context already,
>       // or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
>       throw ex;
>    }
>    catch (Throwable ex) {
>       throw new BeanCreationException(
>             mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
>    }
> }
> ```

> 获取Bean的Class类型，
>
> resolveBeforeInstantiation 最后可以改变RootBeanDefinition的地方，同时也是aop的地方，aop在此处返回代理，然后直接返回。
>
> doCreateBean为创建Bean的具体方法，代码如下。

> ```java
> protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
>       throws BeanCreationException {
> 
>    // Instantiate the bean.
>    BeanWrapper instanceWrapper = null;
>    if (mbd.isSingleton()) {
>       instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
>    }
>    if (instanceWrapper == null) {
>       instanceWrapper = createBeanInstance(beanName, mbd, args);
>    }
>    Object bean = instanceWrapper.getWrappedInstance();
>    Class<?> beanType = instanceWrapper.getWrappedClass();
>    if (beanType != NullBean.class) {
>       mbd.resolvedTargetType = beanType;
>    }
> 
>    // Allow post-processors to modify the merged bean definition.
>    synchronized (mbd.postProcessingLock) {
>       if (!mbd.postProcessed) {
>          try {
>             applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
>          }
>          catch (Throwable ex) {
>             throw new BeanCreationException(mbd.getResourceDescription(), beanName,
>                   "Post-processing of merged bean definition failed", ex);
>          }
>          mbd.postProcessed = true;
>       }
>    }
> 
>    // Eagerly cache singletons to be able to resolve circular references
>    // even when triggered by lifecycle interfaces like BeanFactoryAware.
>    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
>          isSingletonCurrentlyInCreation(beanName));
>    if (earlySingletonExposure) {
>       if (logger.isTraceEnabled()) {
>          logger.trace("Eagerly caching bean '" + beanName +
>                "' to allow for resolving potential circular references");
>       }
>       addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
>    }
> 
>    // Initialize the bean instance.
>    Object exposedObject = bean;
>    try {
>       populateBean(beanName, mbd, instanceWrapper);
>       exposedObject = initializeBean(beanName, exposedObject, mbd);
>    }
>    catch (Throwable ex) {
>       if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
>          throw (BeanCreationException) ex;
>       }
>       else {
>          throw new BeanCreationException(
>                mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
>       }
>    }
> 
>    if (earlySingletonExposure) {
>       Object earlySingletonReference = getSingleton(beanName, false);
>       if (earlySingletonReference != null) {
>          if (exposedObject == bean) {
>             exposedObject = earlySingletonReference;
>          }
>          else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
>             String[] dependentBeans = getDependentBeans(beanName);
>             Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
>             for (String dependentBean : dependentBeans) {
>                if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
>                   actualDependentBeans.add(dependentBean);
>                }
>             }
>             if (!actualDependentBeans.isEmpty()) {
>                throw new BeanCurrentlyInCreationException(beanName,
>                      "Bean with name '" + beanName + "' has been injected into other beans [" +
>                      StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
>                      "] in its raw version as part of a circular reference, but has eventually been " +
>                      "wrapped. This means that said other beans do not use the final version of the " +
>                      "bean. This is often the result of over-eager type matching - consider using " +
>                      "'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
>             }
>          }
>       }
>    }
> 
>    // Register bean as disposable.
>    try {
>       registerDisposableBeanIfNecessary(beanName, bean, mbd);
>    }
>    catch (BeanDefinitionValidationException ex) {
>       throw new BeanCreationException(
>             mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
>    }
> 
>    return exposedObject;
> }
> ```

> 这里面包含了createBeanInstance方法对Bean对象的构造，然后提前暴露到earlySingletonObjects中，再populateBean方法属性填充,最后如果会调用初始化方法initializeBean，这个方法会在初始化方法之前执行invokeAwareMethods -> applyBeanPostProcessorsBeforeInitialization -> invokeInitMethods -> applyBeanPostProcessorsAfterInitialization。在invokeInitMethods中会先执行afterPropertiesSet方法，再调用配置文件配置的init-method方式。
>
> ApplicationContextAwar这个也是BeanPostProcessor中处理的，ApplicationContextAwareProcessor实现了BeanPostProcessor接口，在prepareBeanFactory中注册的。

> ```java
> protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
>    // Make sure bean class is actually resolved at this point.
>    Class<?> beanClass = resolveBeanClass(mbd, beanName);
> 
>    if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
>       throw new BeanCreationException(mbd.getResourceDescription(), beanName,
>             "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
>    }
> 
>    Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
>    if (instanceSupplier != null) {
>       return obtainFromSupplier(instanceSupplier, beanName);
>    }
> 
>    if (mbd.getFactoryMethodName() != null) {
>       return instantiateUsingFactoryMethod(beanName, mbd, args);
>    }
> 
>    // Shortcut when re-creating the same bean...
>    boolean resolved = false;
>    boolean autowireNecessary = false;
>    if (args == null) {
>       synchronized (mbd.constructorArgumentLock) {
>          if (mbd.resolvedConstructorOrFactoryMethod != null) {
>             resolved = true;
>             autowireNecessary = mbd.constructorArgumentsResolved;
>          }
>       }
>    }
>    if (resolved) {
>       if (autowireNecessary) {
>          return autowireConstructor(beanName, mbd, null, null);
>       }
>       else {
>          return instantiateBean(beanName, mbd);
>       }
>    }
> 
>    // Candidate constructors for autowiring?
>    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
>    if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
>          mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
>       return autowireConstructor(beanName, mbd, ctors, args);
>    }
> 
>    // Preferred constructors for default construction?
>    ctors = mbd.getPreferredConstructors();
>    if (ctors != null) {
>       return autowireConstructor(beanName, mbd, ctors, null);
>    }
> 
>    // No special handling: simply use no-arg constructor.
>    return instantiateBean(beanName, mbd);
> }
> ```

> ```java
> protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
>    if (bw == null) {
>       if (mbd.hasPropertyValues()) {
>          throw new BeanCreationException(
>                mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
>       }
>       else {
>          // Skip property population phase for null instance.
>          return;
>       }
>    }
> 
>    // Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
>    // state of the bean before properties are set. This can be used, for example,
>    // to support styles of field injection.
>    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
>       for (BeanPostProcessor bp : getBeanPostProcessors()) {
>          if (bp instanceof InstantiationAwareBeanPostProcessor) {
>             InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
>             if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
>                return;
>             }
>          }
>       }
>    }
> 
>    PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);
> 
>    int resolvedAutowireMode = mbd.getResolvedAutowireMode();
>    if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
>       MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
>       // Add property values based on autowire by name if applicable.
>       if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
>          autowireByName(beanName, mbd, bw, newPvs);
>       }
>       // Add property values based on autowire by type if applicable.
>       if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
>          autowireByType(beanName, mbd, bw, newPvs);
>       }
>       pvs = newPvs;
>    }
> 
>    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
>    boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);
> 
>    PropertyDescriptor[] filteredPds = null;
>    if (hasInstAwareBpps) {
>       if (pvs == null) {
>          pvs = mbd.getPropertyValues();
>       }
>       for (BeanPostProcessor bp : getBeanPostProcessors()) {
>          if (bp instanceof InstantiationAwareBeanPostProcessor) {
>             InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
>             PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
>             if (pvsToUse == null) {
>                if (filteredPds == null) {
>                   filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
>                }
>                pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
>                if (pvsToUse == null) {
>                   return;
>                }
>             }
>             pvs = pvsToUse;
>          }
>       }
>    }
>    if (needsDepCheck) {
>       if (filteredPds == null) {
>          filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
>       }
>       checkDependencies(beanName, mbd, filteredPds, pvs);
>    }
> 
>    if (pvs != null) {
>       applyPropertyValues(beanName, mbd, bw, pvs);
>    }
> }
> ```