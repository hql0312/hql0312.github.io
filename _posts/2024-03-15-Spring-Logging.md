---
layout:     post
title:      "Logback在SpringBoot中的启动"
date:       2024-03-15
author:     "hql0312"
tags:
- Spring
---


# 引言
在SpringBoot项目中，只要增加了logback.xml 配置文件，日志就会按配置好的方式，进行日志的输出，那日志的功能是如何配置到项目中的呢？
# 分析
日志启动依赖于spring容器中的监听器机制，`LoggingApplicationListener`类实现了该功能
```java
public void onApplicationEvent(ApplicationEvent event) {
// 应用启动事件
if (event instanceof ApplicationStartingEvent) {
    onApplicationStartingEvent((ApplicationStartingEvent) event);
}
    // 环境变量准备事件
else if (event instanceof ApplicationEnvironmentPreparedEvent) {
    onApplicationEnvironmentPreparedEvent((ApplicationEnvironmentPreparedEvent) event);
}
    // 应用准备好事件
else if (event instanceof ApplicationPreparedEvent) {
    onApplicationPreparedEvent((ApplicationPreparedEvent) event);
}
else if (event instanceof ContextClosedEvent
         && ((ContextClosedEvent) event).getApplicationContext().getParent() == null) {
    onContextClosedEvent();
}
else if (event instanceof ApplicationFailedEvent) {
    onApplicationFailedEvent();
}
}
```

`LoggingApplicationListener`实现了对 `ApplicationStartingEvent`、`ApplicationEnvironmentPreparedEvent`、`ApplicationPreparedEvent`的监听。
`ApplicationStartingEvent`：负责初始化。
`ApplicationEnvironmentPreparedEvent`: 负责监听配置变更的处理，当有配置变更时，会进行重新加载。

## 初始化
```java
private void onApplicationStartingEvent(ApplicationStartingEvent event) {
// 确认当前使用种日志框架,按以下顺序逐个尝试
//systems.put("ch.qos.logback.core.Appender", "org.springframework.boot.logging.logback.LogbackLoggingSystem");
//	systems.put("org.apache.logging.log4j.core.impl.Log4jContextFactory",
//			"org.springframework.boot.logging.log4j2.Log4J2LoggingSystem");
//	systems.put("java.util.logging.LogManager", "org.springframework.boot.logging.java.JavaLoggingSystem");
this.loggingSystem = LoggingSystem.get(event.getSpringApplication().getClassLoader());
// 初始化前的处理
this.loggingSystem.beforeInitialize();
}
```
该方法只是初始化当前的`LoggingSystem`对象，同时会按注释中的顺序进行查看当前要使用哪个`LoggingSystem`的子类，是否在当前环境中有该类，因为现在以logback来分析，所以 关注`LogbackLoggingSystem`即可。
## 环境配置准备
当environment中的数据准备好后，就可以开始进行处理log框架的初始化内容。
```java
private void onApplicationEnvironmentPreparedEvent(ApplicationEnvironmentPreparedEvent event) {
if (this.loggingSystem == null) {
    this.loggingSystem = LoggingSystem.get(event.getSpringApplication().getClassLoader());
}
initialize(event.getEnvironment(), event.getSpringApplication().getClassLoader());
}
```
```java
protected void initialize(ConfigurableEnvironment environment, ClassLoader classLoader) {
    // 应用配置中以logging.开头的配置
    new LoggingSystemProperties(environment).apply();
    // 日志文件
    this.logFile = LogFile.get(environment);
    if (this.logFile != null) {
        this.logFile.applyToSystemProperties();
    }
    this.loggerGroups = new LoggerGroups(DEFAULT_GROUP_LOGGERS);
    initializeEarlyLoggingLevel(environment);
    // 初始化logSystem,这里是重点
    initializeSystem(environment, this.loggingSystem, this.logFile);
    // 初始化logging level,从环境变量中获取，配置文件中同名的logger level会覆盖配置中的
    initializeFinalLoggingLevels(environment, this.loggingSystem);
    registerShutdownHookIfNecessary(environment, this.loggingSystem);
}
```
## 读取配置文件
```java
	@Override
	public void initialize(LoggingInitializationContext initializationContext, String configLocation, LogFile logFile) {
		// 单例的LoggerContext
        LoggerContext loggerContext = getLoggerContext();
		if (isAlreadyInitialized(loggerContext)) {
			return;
		}
        // 进行初始化
		super.initialize(initializationContext, configLocation, logFile);
		loggerContext.getTurboFilterList().remove(FILTER);
		markAsInitialized(loggerContext);
		if (StringUtils.hasText(System.getProperty(CONFIGURATION_FILE_PROPERTY))) {
			getLogger(LogbackLoggingSystem.class.getName()).warn("Ignoring '" + CONFIGURATION_FILE_PROPERTY
					+ "' system property. Please use 'logging.config' instead.");
		}
	}
```
```java
	public void initialize(LoggingInitializationContext initializationContext, String configLocation, LogFile logFile) {
		if (StringUtils.hasLength(configLocation)) {
			initializeWithSpecificConfig(initializationContext, configLocation, logFile);
			return;
		}
        // 初始化
		initializeWithConventions(initializationContext, logFile);
	}
```
```java
	private void initializeWithConventions(LoggingInitializationContext initializationContext, LogFile logFile) {
		// 尝试标准的日志配置文件,如   "logback-test.groovy", "logback-test.xml", "logback.groovy", "logback.xml"
        String config = getSelfInitializationConfig();
		if (config != null && logFile == null) { // 如果存在，则自身进行初始化
			// self initialization has occurred, reinitialize in case of property changes
			reinitialize(initializationContext);
			return;
		}
        // 如果没有找到标准的，则使用spring-开头的标准文件 "logback-test.groovy", "logback-test.xml", "logback.groovy", "logback.xml"
		if (config == null) {
			config = getSpringInitializationConfig();
		}
		if (config != null) {
			loadConfiguration(initializationContext, config, logFile);
			return;
		}
		loadDefaults(initializationContext, logFile);
	}
```
```java
	protected void reinitialize(LoggingInitializationContext initializationContext) {
		getLoggerContext().reset();
		getLoggerContext().getStatusManager().clear();
        // 加载配置文件 
		loadConfiguration(initializationContext, getSelfInitializationConfig(), null);
	}
```
```java
	protected void loadConfiguration(LoggingInitializationContext initializationContext, String location,
			LogFile logFile) {
		super.loadConfiguration(initializationContext, location, logFile);
		LoggerContext loggerContext = getLoggerContext();
		stopAndReset(loggerContext);
		try {
            //从配置文件中进行配置
			configureByResourceUrl(initializationContext, loggerContext, ResourceUtils.getURL(location));
		}
		catch (Exception ex) {
			throw new IllegalStateException("Could not initialize Logback logging from " + location, ex);
		}
		List<Status> statuses = loggerContext.getStatusManager().getCopyOfStatusList();
		StringBuilder errors = new StringBuilder();
		for (Status status : statuses) {
			if (status.getLevel() == Status.ERROR) {
				errors.append((errors.length() > 0) ? String.format("%n") : "");
				errors.append(status.toString());
			}
		}
		if (errors.length() > 0) {
			throw new IllegalStateException(String.format("Logback configuration error detected: %n%s", errors));
		}
	}
```
## 配置日志级别
```java
	private void initializeFinalLoggingLevels(ConfigurableEnvironment environment, LoggingSystem system) {
		// logGroup 是定义一系列的Loggername为一个分组，后续可以对当前这个分组进行配置loggerLevel
        // 这样，在相同组下的所有logger的配置日志级别都会被配置
        bindLoggerGroups(environment);
		if (this.springBootLogging != null) {
			initializeLogLevel(system, this.springBootLogging);
		}
        // 配置级别
        // springboot中的yml的配置，会覆盖logback.xml中的配置
		setLogLevels(system, environment);
	}
```
## 应用准备
```java
private void onApplicationPreparedEvent(ApplicationPreparedEvent event) {
    // 将LogggingSystem注册到Spring容器中
    ConfigurableListableBeanFactory beanFactory = event.getApplicationContext().getBeanFactory();
    if (!beanFactory.containsBean(LOGGING_SYSTEM_BEAN_NAME)) {
        beanFactory.registerSingleton(LOGGING_SYSTEM_BEAN_NAME, this.loggingSystem);
    }
    if (this.logFile != null && !beanFactory.containsBean(LOG_FILE_BEAN_NAME)) {
        beanFactory.registerSingleton(LOG_FILE_BEAN_NAME, this.logFile);
    }
    // 注册LoggerGroups的bean
    if (this.loggerGroups != null && !beanFactory.containsBean(LOGGER_GROUPS_BEAN_NAME)) {
        beanFactory.registerSingleton(LOGGER_GROUPS_BEAN_NAME, this.loggerGroups);
    }
}
```
在上面会注册三个Bean，在Spring容器中可以获取出来，进行调整对应的配置
# demo配置示例
## 示例代码
```java
@RestController
public class LoggerController {

    Logger logger = LoggerFactory.getLogger(LoggerController.class);
    @Autowired
    @Qualifier(LOGGER_GROUPS_BEAN_NAME)
    LoggerGroups loggerGroups;
    @Autowired
    @Qualifier(LOGGING_SYSTEM_BEAN_NAME)
    LoggingSystem loggingSystem;

    /**
     * 获取当前分组
     * @return
     */
    @RequestMapping("/api/logger-groups")
    public LoggerGroups getLoggerGroups() {
        return loggerGroups;
    }

    /**
     * 设置logger级别
     * @param loggerName
     * @param level
     */
    @RequestMapping(value = "/api/set-logger-level")
    public void getLogLevel(@RequestParam("name") String loggerName, @RequestParam("level") LogLevel level) {
        this.loggingSystem.setLogLevel(loggerName, level);
    }

    /**
     * 设置分组级别
     * @param group
     * @param level
     */
    @RequestMapping(value = "/api/set-group-level")
    public void updateLoggerGroup(@RequestParam("group") String group,@RequestParam("level") LogLevel level) {
        this.loggerGroups.get(group).configureLogLevel(level,(k,l) ->
                this.loggingSystem.setLogLevel(k,l)
                );
    }

    @Scheduled(fixedRate = 2000)
    public void run() {
        logger.debug("debug...");
        logger.info("info...");
        logger.warn("warn...");
        logger.error("error...");
    }
}
```
## 配置
```java
logging:
  group:
    demo:
      - org.example.LoggerController
      - org.example.TestController
```
## 测试
### 测试分组
http://localhost:8080/api/set-group-level?group=demo&level=WARN
### 测试指定loggername
localhost:8080/api/set-logger-level?name=org.example.TestController&level=DEBUG
