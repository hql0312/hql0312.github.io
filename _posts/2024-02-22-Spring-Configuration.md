---
layout:     post
title:      "Spring @Configuration proxyBeanMethods详解"
subtitle:   "Spring源码分析"
date:       2024-02-22
author:     "hql0312"
tags:
    - Spring
---

# @Configuration 注解
spring框架中，`@Configuration`用于声明一个Java类为配置类，通过标注该注解的类来声明`@Bean`注解，用于定义Spring 容器中的Bean，定义如下：

```java
public @interface Configuration {
	@AliasFor(annotation = Component.class)
	String value() default "";

	boolean proxyBeanMethods() default true;

}
```

该注解有两个属性：

1. `value` 指定声明的名称
2. `proxyBeanMethods` 该属性标识了是否需要通过代理来增强Bean的生命周期行为，如是否通过调用声明为Bean的方法，来返回共享对象实例，而不是每次调用都创建一个新的实例。该属性默认为true,标识声明为bean的方法，默认是会进行代理的，并且spring 会为其进行缓存结果，多次调用返回同一个实例。
# 示例

```java
@Configuration
public class TestConfig {

    @Bean
    public TestBean1 testBean1() {
        System.out.println("create testBean1 start");
        return new TestBean1();
    }

    @Bean
    public TestBean2 testBean2() {
        System.out.println("create testBean2 start");
        return new TestBean2(testBean1());
    }
}
```

以上声明了两个Bean，同时未配置@Configuration的`proxyBeanMethods` 属性，来看以下的输出

```java
public static void main(String[] args) {
        //SpringApplication.run(Main.class, args);
        AnnotationConfigApplicationContext annotationConfigApplicationContext = new AnnotationConfigApplicationContext(TestConfig.class);
        System.out.println("get proxy bean...");
        TestBean2 testBean2 = annotationConfigApplicationContext.getBean(TestBean2.class);
}
----
create testBean1 start
create testBean2 start

```

从上面的输出可以看出，只输出一次的 create testBean1 start的文本，说明在 testBean2 中调用` testBean1()`方法时，没有再执行内部的逻辑。
接下来再验证下增加属性的配置 `@Configuration(proxyBeanMethods = false)`

```
create testBean1 start
create testBean2 start
create testBean1 start
```

可以从输出中看到，create testBean1 start 有输出两次，说明`testBean1()`的结果并未被缓存，而是每次调用都会执行内部的逻辑。

# 源码分析

处理该逻辑的类为`ConfigurationClassPostProcessor`。

```java
public class ConfigurationClassPostProcessor implements BeanDefinitionRegistryPostProcessor,
PriorityOrdered, ResourceLoaderAware, ApplicationStartupAware, BeanClassLoaderAware, EnvironmentAware {
}
```

该类实现了 `BeanDefinitionRegistryPostProcessor`接口，该接口扩展了 `BeanFactoryPostProcessor`，从其注释中，可知，该接口提供了进行修改Bean定义的一个扩展钩子，只有一个方法：

```java
public interface BeanFactoryPostProcessor {
    // 该方法是在加载完所有bean的定义后进行触发的
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
```

来查看下具体的实现：

```java
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		int factoryId = System.identityHashCode(beanFactory);
		if (this.factoriesPostProcessed.contains(factoryId)) {
			throw new IllegalStateException(
					"postProcessBeanFactory already called on this post-processor against " + beanFactory);
		}
		this.factoriesPostProcessed.add(factoryId);
		if (!this.registriesPostProcessed.contains(factoryId)) {
			//1.  处理ConfigClass的Bean定义
			processConfigBeanDefinitions((BeanDefinitionRegistry) beanFactory);
		}
        // 2. 增强ConfigClass
		enhanceConfigurationClasses(beanFactory);
		beanFactory.addBeanPostProcessor(new ImportAwareBeanPostProcessor(beanFactory));
	}
```

该方法主要处理了两个内容：

1. 处理基于Configuration模型的注册类
2. 增强Configuration模型的注册机制

## processConfigBeanDefinitions
在第一步中，主要会调用`ConfigurationClassUtils.checkConfigurationClassCandidate()`方法来检查是否是一个ConfigClass的Bean定义类，如果是，则会判断proxyBeanMethods的值。

```java
public static boolean checkConfigurationClassCandidate(
			BeanDefinition beanDef, MetadataReaderFactory metadataReaderFactory) {

		String className = beanDef.getBeanClassName();
		if (className == null || beanDef.getFactoryMethodName() != null) {
			return false;
		}

		AnnotationMetadata metadata;
		if (beanDef instanceof AnnotatedBeanDefinition &&
				className.equals(((AnnotatedBeanDefinition) beanDef).getMetadata().getClassName())) {
			// Can reuse the pre-parsed metadata from the given BeanDefinition...
			metadata = ((AnnotatedBeanDefinition) beanDef).getMetadata();
		}
		else if (beanDef instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) beanDef).hasBeanClass()) {
			// Check already loaded Class if present...
			// since we possibly can't even load the class file for this Class.
			Class<?> beanClass = ((AbstractBeanDefinition) beanDef).getBeanClass();
			if (BeanFactoryPostProcessor.class.isAssignableFrom(beanClass) ||
					BeanPostProcessor.class.isAssignableFrom(beanClass) ||
					AopInfrastructureBean.class.isAssignableFrom(beanClass) ||
					EventListenerFactory.class.isAssignableFrom(beanClass)) {
				return false;
			}
			metadata = AnnotationMetadata.introspect(beanClass);
		}
		else {
			try {
				MetadataReader metadataReader = metadataReaderFactory.getMetadataReader(className);
				metadata = metadataReader.getAnnotationMetadata();
			}
			catch (IOException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Could not find class file for introspecting configuration annotations: " +
							className, ex);
				}
				return false;
			}
		}
        // 如果配置了proxyBeanMethod为true，则设置CONFIGURATION_CLASS_ATTRIBUTE为full
        // 否则为lite
		Map<String, Object> config = metadata.getAnnotationAttributes(Configuration.class.getName());
		if (config != null && !Boolean.FALSE.equals(config.get("proxyBeanMethods"))) {
			beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_FULL);
		}
		else if (config != null || isConfigurationCandidate(metadata)) {
			beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_LITE);
		}
		else {
			return false;
		}

		// It's a full or lite configuration candidate... Let's determine the order value, if any.
		Integer order = getOrder(metadata);
		if (order != null) {
			beanDef.setAttribute(ORDER_ATTRIBUTE, order);
		}

		return true;
	}
```

以上逻辑会判断是否配置proxyBeanMethod为true,如果是则设置CONFIGURATION_CLASS_ATTRIBUTE为full，则需要增强，否则为lite。

## enhanceConfigurationClasses

该方法进行增强配置逻辑：

```java
public void enhanceConfigurationClasses(ConfigurableListableBeanFactory beanFactory) {
		StartupStep enhanceConfigClasses = this.applicationStartup.start("spring.context.config-classes.enhance");
		Map<String, AbstractBeanDefinition> configBeanDefs = new LinkedHashMap<>();
		// 遍历所有的bean
        for (String beanName : beanFactory.getBeanDefinitionNames()) {
			BeanDefinition beanDef = beanFactory.getBeanDefinition(beanName);
			Object configClassAttr = beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE);
			AnnotationMetadata annotationMetadata = null;
			MethodMetadata methodMetadata = null;
			if (beanDef instanceof AnnotatedBeanDefinition) {
				AnnotatedBeanDefinition annotatedBeanDefinition = (AnnotatedBeanDefinition) beanDef;
				annotationMetadata = annotatedBeanDefinition.getMetadata();
				methodMetadata = annotatedBeanDefinition.getFactoryMethodMetadata();
			}
			if ((configClassAttr != null || methodMetadata != null) && beanDef instanceof AbstractBeanDefinition) {
				// Configuration class (full or lite) or a configuration-derived @Bean method
				// -> eagerly resolve bean class at this point, unless it's a 'lite' configuration
				// or component class without @Bean methods.
				AbstractBeanDefinition abd = (AbstractBeanDefinition) beanDef;
				if (!abd.hasBeanClass()) {
					boolean liteConfigurationCandidateWithoutBeanMethods =
							(ConfigurationClassUtils.CONFIGURATION_CLASS_LITE.equals(configClassAttr) &&
								annotationMetadata != null && !ConfigurationClassUtils.hasBeanMethods(annotationMetadata));
					if (!liteConfigurationCandidateWithoutBeanMethods) {
						try {
							abd.resolveBeanClass(this.beanClassLoader);
						}
						catch (Throwable ex) {
							throw new IllegalStateException(
									"Cannot load configuration class: " + beanDef.getBeanClassName(), ex);
						}
					}
				}
			}
            // 如果是CONFIGURATION_CLASS_ATTRIBUTE为full,也就是proxyBeanMethod为true
			if (ConfigurationClassUtils.CONFIGURATION_CLASS_FULL.equals(configClassAttr)) {
				if (!(beanDef instanceof AbstractBeanDefinition)) {
					throw new BeanDefinitionStoreException("Cannot enhance @Configuration bean definition '" +
							beanName + "' since it is not stored in an AbstractBeanDefinition subclass");
				}
				else if (logger.isInfoEnabled() && beanFactory.containsSingleton(beanName)) {
					logger.info("Cannot enhance @Configuration bean definition '" + beanName +
							"' since its singleton instance has been created too early. The typical cause " +
							"is a non-static @Bean method with a BeanDefinitionRegistryPostProcessor " +
							"return type: Consider declaring such methods as 'static'.");
				}
				configBeanDefs.put(beanName, (AbstractBeanDefinition) beanDef);
			}
		}
		if (configBeanDefs.isEmpty() || NativeDetector.inNativeImage()) {
			// nothing to enhance -> return immediately
			enhanceConfigClasses.end();
			return;
		}

		ConfigurationClassEnhancer enhancer = new ConfigurationClassEnhancer();
		for (Map.Entry<String, AbstractBeanDefinition> entry : configBeanDefs.entrySet()) {
			AbstractBeanDefinition beanDef = entry.getValue();
			// If a @Configuration class gets proxied, always proxy the target class
			beanDef.setAttribute(AutoProxyUtils.PRESERVE_TARGET_CLASS_ATTRIBUTE, Boolean.TRUE);
			// Set enhanced subclass of the user-specified bean class
			Class<?> configClass = beanDef.getBeanClass();
			Class<?> enhancedClass = enhancer.enhance(configClass, this.beanClassLoader);
			if (configClass != enhancedClass) {
				if (logger.isTraceEnabled()) {
					logger.trace(String.format("Replacing bean definition '%s' existing class '%s' with " +
							"enhanced class '%s'", entry.getKey(), configClass.getName(), enhancedClass.getName()));
				}
				beanDef.setBeanClass(enhancedClass);
			}
		}
		enhanceConfigClasses.tag("classCount", () -> String.valueOf(configBeanDefs.keySet().size())).end();
	}
```

如果配置了proxyBeanMethod为true，则会调用ConfigurationClassEnhancer进行增强ConfigClass类。

```java
	public Class<?> enhance(Class<?> configClass, @Nullable ClassLoader classLoader) {
		if (EnhancedConfiguration.class.isAssignableFrom(configClass)) {
			if (logger.isDebugEnabled()) {
				logger.debug(String.format("Ignoring request to enhance %s as it has " +
						"already been enhanced. This usually indicates that more than one " +
						"ConfigurationClassPostProcessor has been registered (e.g. via " +
						"<context:annotation-config>). This is harmless, but you may " +
						"want check your configuration and remove one CCPP if possible",
						configClass.getName()));
			}
			return configClass;
		}
        // 通过CGLIB进行增强
		Class<?> enhancedClass = createClass(newEnhancer(configClass, classLoader));
		if (logger.isTraceEnabled()) {
			logger.trace(String.format("Successfully enhanced %s; enhanced class name is: %s",
					configClass.getName(), enhancedClass.getName()));
		}
		return enhancedClass;
	}

	private Class<?> createClass(Enhancer enhancer) {
        // 增强类
		Class<?> subclass = enhancer.createClass();
        // 注册回调
		Enhancer.registerStaticCallbacks(subclass, CALLBACKS);
		return subclass;
	}
```

增强类的逻辑中，主要是拦截方法的调用，注册回调 CALLBACKS

```java
	private static final Callback[] CALLBACKS = new Callback[] {
			new BeanMethodInterceptor(),
			new BeanFactoryAwareMethodInterceptor(),
			NoOp.INSTANCE
	};
```

主要关注`BeanMethodInterceptor`的逻辑：

```java
		public Object intercept(Object enhancedConfigInstance, Method beanMethod, Object[] beanMethodArgs,
					MethodProxy cglibMethodProxy) throws Throwable {

			ConfigurableBeanFactory beanFactory = getBeanFactory(enhancedConfigInstance);
			String beanName = BeanAnnotationHelper.determineBeanNameFor(beanMethod);

			// Determine whether this bean is a scoped-proxy
			if (BeanAnnotationHelper.isScopedProxy(beanMethod)) {
				String scopedBeanName = ScopedProxyCreator.getTargetBeanName(beanName);
				if (beanFactory.isCurrentlyInCreation(scopedBeanName)) {
					beanName = scopedBeanName;
				}
			}

			// To handle the case of an inter-bean method reference, we must explicitly check the
			// container for already cached instances.

			// First, check to see if the requested bean is a FactoryBean. If so, create a subclass
			// proxy that intercepts calls to getObject() and returns any cached bean instance.
			// This ensures that the semantics of calling a FactoryBean from within @Bean methods
			// is the same as that of referring to a FactoryBean within XML. See SPR-6602.
			if (factoryContainsBean(beanFactory, BeanFactory.FACTORY_BEAN_PREFIX + beanName) &&
					factoryContainsBean(beanFactory, beanName)) {
				Object factoryBean = beanFactory.getBean(BeanFactory.FACTORY_BEAN_PREFIX + beanName);
				if (factoryBean instanceof ScopedProxyFactoryBean) {
					// Scoped proxy factory beans are a special case and should not be further proxied
				}
				else {
					// It is a candidate FactoryBean - go ahead with enhancement
					return enhanceFactoryBean(factoryBean, beanMethod.getReturnType(), beanFactory, beanName);
				}
			}
            // 如果是调用bean的创建方法，如上面的示例中的testBean1() 方法，则调用代理类的父类的实际方法
			if (isCurrentlyInvokedFactoryMethod(beanMethod)) {
				// The factory is calling the bean method in order to instantiate and register the bean
				// (i.e. via a getBean() call) -> invoke the super implementation of the method to actually
				// create the bean instance.
				if (logger.isInfoEnabled() &&
						BeanFactoryPostProcessor.class.isAssignableFrom(beanMethod.getReturnType())) {
					logger.info(String.format("@Bean method %s.%s is non-static and returns an object " +
									"assignable to Spring's BeanFactoryPostProcessor interface. This will " +
									"result in a failure to process annotations such as @Autowired, " +
									"@Resource and @PostConstruct within the method's declaring " +
									"@Configuration class. Add the 'static' modifier to this method to avoid " +
									"these container lifecycle issues; see @Bean javadoc for complete details.",
							beanMethod.getDeclaringClass().getSimpleName(), beanMethod.getName()));
				}
				return cglibMethodProxy.invokeSuper(enhancedConfigInstance, beanMethodArgs);
			}
            // 如果是A方法调用B方法，则进行bean的解析，可以从BeanFactory获取
			return resolveBeanReference(beanMethod, beanMethodArgs, beanFactory, beanName);
		}
```

1. 如果是调用的ConfigClass的标注了`@Bean`的方法时，如直接调用testBean1方法时，则直接调用父类的testBean1方法进行真实的调用
2. 如果是在其他方法中调用的testBean1方法，如果上面的示例中的testBean2中调用testBean1方法，则会进行拦截，查看是否有对应的Bean的对象的，通过getBean的逻辑。

```java
private Object resolveBeanReference(Method beanMethod, Object[] beanMethodArgs,
				ConfigurableBeanFactory beanFactory, String beanName) {

			// The user (i.e. not the factory) is requesting this bean through a call to
			// the bean method, direct or indirect. The bean may have already been marked
			// as 'in creation' in certain autowiring scenarios; if so, temporarily set
			// the in-creation status to false in order to avoid an exception.
			boolean alreadyInCreation = beanFactory.isCurrentlyInCreation(beanName);
			try {
				if (alreadyInCreation) {
					beanFactory.setCurrentlyInCreation(beanName, false);
				}
				boolean useArgs = !ObjectUtils.isEmpty(beanMethodArgs);
				if (useArgs && beanFactory.isSingleton(beanName)) {
					// Stubbed null arguments just for reference purposes,
					// expecting them to be autowired for regular singleton references?
					// A safe assumption since @Bean singleton arguments cannot be optional...
					for (Object arg : beanMethodArgs) {
						if (arg == null) {
							useArgs = false;
							break;
						}
					}
				}
                // 获取Bean对象
				Object beanInstance = (useArgs ? beanFactory.getBean(beanName, beanMethodArgs) :
						beanFactory.getBean(beanName));
				if (!ClassUtils.isAssignableValue(beanMethod.getReturnType(), beanInstance)) {
					// Detect package-protected NullBean instance through equals(null) check
					if (beanInstance.equals(null)) {
						if (logger.isDebugEnabled()) {
							logger.debug(String.format("@Bean method %s.%s called as bean reference " +
									"for type [%s] returned null bean; resolving to null value.",
									beanMethod.getDeclaringClass().getSimpleName(), beanMethod.getName(),
									beanMethod.getReturnType().getName()));
						}
						beanInstance = null;
					}
					else {
						String msg = String.format("@Bean method %s.%s called as bean reference " +
								"for type [%s] but overridden by non-compatible bean instance of type [%s].",
								beanMethod.getDeclaringClass().getSimpleName(), beanMethod.getName(),
								beanMethod.getReturnType().getName(), beanInstance.getClass().getName());
						try {
							BeanDefinition beanDefinition = beanFactory.getMergedBeanDefinition(beanName);
							msg += " Overriding bean of same name declared in: " + beanDefinition.getResourceDescription();
						}
						catch (NoSuchBeanDefinitionException ex) {
							// Ignore - simply no detailed message then.
						}
						throw new IllegalStateException(msg);
					}
				}
				Method currentlyInvoked = SimpleInstantiationStrategy.getCurrentlyInvokedFactoryMethod();
				if (currentlyInvoked != null) {
					String outerBeanName = BeanAnnotationHelper.determineBeanNameFor(currentlyInvoked);
					beanFactory.registerDependentBean(beanName, outerBeanName);
				}
				return beanInstance;
			}
			finally {
				if (alreadyInCreation) {
					beanFactory.setCurrentlyInCreation(beanName, true);
				}
			}
		}
```

至此，proxyBeanMethods的大致逻辑分析结束。
