---
layout:     post
title:      "springboot bean加载顺序问题"
subtitle:   "记一次疑难bug解决经历"
date:       2019-08-20
author:     "caotc"
header-img: "img/home-bg.jpg"
tags:
    - Springboot
    - springboot
    - Override
    - override
    - Bean
    - bean
    - Jackson
    - jackson
    - Resource
    - resource
---


# 一次疑难bug解决经历

## bug情况
项目是一个spring cloud项目,使用了spring cloud stream组件规范来收发消息,平时http请求使用的是fastjson,但是stream message使用的还是默认的jackson.

而BUG是在发送stream message的时候,本地环境时枚举是序列化为name,而在线上却序列化为code,即线上和线下的序列化行为不一致.

## bug排查过程
**首先考虑是否是环境配置的影响.**

**将线上服务器的jar包直接下载下来.使用命令指定环境为开发在本机启动,成功复现出此BUG,说明并非环境配置导致的问题.**

**接着,怀疑是线上和线下的jar包有差异,差异导致了这个bug.**

于是使用比较工具比较线上和线下的jar包,发现了一部分内部包为快照版本,而线上为正式版本.
另外还有部分同为正式版的内部jar包也有差异,但是基本与bug和jackson不存在相关性.

于是在idea中将同版本代码中的快照版本指定为正式版本,然后进行测试.但是idea中的代码却依旧没有复现这个bug.

至于部分有日期差异的正式版,比较字节数大小是相同的,而且基本与bug和jackson不存在相关性,似乎不太有关系.

**也就是说不太可能是线上线下的jar包代码差异导致的bug.**

此时思路陷入了僵局,不知道从何开始入手.

在漫无目的的从头开始断点前,我决定开启最详细的日志,然后从头开始比较一次idea本地项目启动和线上jar包的日志.

然后在**启动线上jar包时使用命令行参数指定debug模式并且将日志的输出级别调至debug,然后仔细对比两者的日志.**

经过仔细对比忽然发现了启动日志中有明显差异的地方.
>2019-08-19 16:59:45.678 INFO  [main] [/] org.springframework.beans.factory.support.DefaultListableBeanFactory Line:828 - Overriding bean definition for bean 'simpleModule' with a different definition: replacing [Root bean: class [null]; scope=; abstract=false; lazyInit=false; autowireMode=3; dependencyCheck=0; autowireCandidate=true; primary=false;factoryBeanName=streamConfiguration; factoryMethodName=simpleModule; initMethodName=null; destroyMethodName=(inferred); defined in class path resource [***/StreamConfiguration.class]] with [Root bean: class [null]; scope=; abstract=false; lazyInit=false; autowireMode=3; dependencyCheck=0; autowireCandidate=true; primary=false; 
 factoryBeanName=feignConfiguration; factoryMethodName=simpleModule; initMethodName=null; destroyMethodName=(inferred); defined in class path resource [***/FeignConfiguration.class]]
 
>2019-08-19 10:12:25.497 INFO  [main] [/] org.springframework.beans.factory.support.DefaultListableBeanFactory Line:828 - Overriding bean definition for bean 'simpleModule' with a different definition: replacing [Root bean: class [null]; scope=; abstract=false; lazyInit=false; autowireMode=3; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=feignConfiguration; factoryMethodName=simpleModule; initMethodName=null; destroyMethodName=(inferred); defined in class path resource [***/FeignConfiguration.class]] with [Root bean: class [null]; scope=; abstract=false; lazyInit=false; autowireMode=3; dependencyCheck=0; autowireCandidate=true; primary=false; 
 factoryBeanName=streamConfiguration; factoryMethodName=simpleModule; initMethodName=null; destroyMethodName=(inferred); defined in class path resource [***/StreamConfiguration.class]]

很显然,这段日志说明了同名的bean定义被覆盖了,且两者的覆盖顺序或者说最后生效的bean是不同的.
而且bean的名称为simpleModule,很像是jackson的SimpleModule对象.

果不其然,**在这两个类中都定义了jackson的SimpleModule对象作为bean,但是一个对于枚举的返回值指定为输出code,一个指定为输出name.**

**在线上和线下环境加载顺序不同导致了bug出现.**

那么需要寻找的就是为什么线上和线下会导致加载顺序不同了,接下来可以开始断点了(**线上jar包断点可以采用远程断点模式,只不过监听的是本地的端口而已**).

从日志可以看出,加载bean的定义的类为org.springframework.beans.factory.support.DefaultListableBeanFactory.
查看类源码

```java
public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory
		implements ConfigurableListableBeanFactory, BeanDefinitionRegistry, Serializable {
//......
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

		BeanDefinition oldBeanDefinition;

		oldBeanDefinition = this.beanDefinitionMap.get(beanName);
		if (oldBeanDefinition != null) {
			if (!isAllowBeanDefinitionOverriding()) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +
						"': There is already [" + oldBeanDefinition + "] bound.");
			}
			else if (oldBeanDefinition.getRole() < beanDefinition.getRole()) {
				// e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
				if (this.logger.isWarnEnabled()) {
					this.logger.warn("Overriding user-defined bean definition for bean '" + beanName +
							"' with a framework-generated bean definition: replacing [" +
							oldBeanDefinition + "] with [" + beanDefinition + "]");
				}
			}
			else if (!beanDefinition.equals(oldBeanDefinition)) {
				if (this.logger.isInfoEnabled()) {
					this.logger.info("Overriding bean definition for bean '" + beanName +
							"' with a different definition: replacing [" + oldBeanDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
			else {
				if (this.logger.isDebugEnabled()) {
					this.logger.debug("Overriding bean definition for bean '" + beanName +
							"' with an equivalent definition: replacing [" + oldBeanDefinition +
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
					List<String> updatedDefinitions = new ArrayList<String>(this.beanDefinitionNames.size() + 1);
					updatedDefinitions.addAll(this.beanDefinitionNames);
					updatedDefinitions.add(beanName);
					this.beanDefinitionNames = updatedDefinitions;
					if (this.manualSingletonNames.contains(beanName)) {
						Set<String> updatedSingletons = new LinkedHashSet<String>(this.manualSingletonNames);
						updatedSingletons.remove(beanName);
						this.manualSingletonNames = updatedSingletons;
					}
				}
			}
			else {
				// Still in startup registration phase
				this.beanDefinitionMap.put(beanName, beanDefinition);
				this.beanDefinitionNames.add(beanName);
				this.manualSingletonNames.remove(beanName);
			}
			this.frozenBeanDefinitionNames = null;
		}

		if (oldBeanDefinition != null || containsSingleton(beanName)) {
			resetBeanDefinition(beanName);
		}
	}
	
//......
	}
```

很显然,这是一个注册bean定义的方法,以最后注册成功的bean定义集合去加载实际生效的bean.

从该方法入手开始断点,可以快速找到加载@Configuration注解定义的配置类中的bean的源码在org.springframework.context.annotation.ConfigurationClassPostProcessor类中.
相关源码如下
```java
public class ConfigurationClassPostProcessor implements BeanDefinitionRegistryPostProcessor,
		PriorityOrdered, ResourceLoaderAware, BeanClassLoaderAware, EnvironmentAware {
//......

	/**
	 * Build and validate a configuration model based on the registry of
	 * {@link Configuration} classes.
	 */
	public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
		List<BeanDefinitionHolder> configCandidates = new ArrayList<BeanDefinitionHolder>();
		String[] candidateNames = registry.getBeanDefinitionNames();

		for (String beanName : candidateNames) {
			BeanDefinition beanDef = registry.getBeanDefinition(beanName);
			if (ConfigurationClassUtils.isFullConfigurationClass(beanDef) ||
					ConfigurationClassUtils.isLiteConfigurationClass(beanDef)) {
				if (logger.isDebugEnabled()) {
					logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
				}
			}
			else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
				configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
			}
		}

		// Return immediately if no @Configuration classes were found
		if (configCandidates.isEmpty()) {
			return;
		}

		// Sort by previously determined @Order value, if applicable
		Collections.sort(configCandidates, new Comparator<BeanDefinitionHolder>() {
			@Override
			public int compare(BeanDefinitionHolder bd1, BeanDefinitionHolder bd2) {
				int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
				int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
				return (i1 < i2) ? -1 : (i1 > i2) ? 1 : 0;
			}
		});

		// Detect any custom bean name generation strategy supplied through the enclosing application context
		SingletonBeanRegistry sbr = null;
		if (registry instanceof SingletonBeanRegistry) {
			sbr = (SingletonBeanRegistry) registry;
			if (!this.localBeanNameGeneratorSet && sbr.containsSingleton(CONFIGURATION_BEAN_NAME_GENERATOR)) {
				BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(CONFIGURATION_BEAN_NAME_GENERATOR);
				this.componentScanBeanNameGenerator = generator;
				this.importBeanNameGenerator = generator;
			}
		}

		// Parse each @Configuration class
		ConfigurationClassParser parser = new ConfigurationClassParser(
				this.metadataReaderFactory, this.problemReporter, this.environment,
				this.resourceLoader, this.componentScanBeanNameGenerator, registry);

		Set<BeanDefinitionHolder> candidates = new LinkedHashSet<BeanDefinitionHolder>(configCandidates);
		Set<ConfigurationClass> alreadyParsed = new HashSet<ConfigurationClass>(configCandidates.size());
		do {
			parser.parse(candidates);
			parser.validate();

			Set<ConfigurationClass> configClasses = new LinkedHashSet<ConfigurationClass>(parser.getConfigurationClasses());
			configClasses.removeAll(alreadyParsed);

			// Read the model and create bean definitions based on its content
			if (this.reader == null) {
				this.reader = new ConfigurationClassBeanDefinitionReader(
						registry, this.sourceExtractor, this.resourceLoader, this.environment,
						this.importBeanNameGenerator, parser.getImportRegistry());
			}
			this.reader.loadBeanDefinitions(configClasses);
			alreadyParsed.addAll(configClasses);

			candidates.clear();
			if (registry.getBeanDefinitionCount() > candidateNames.length) {
				String[] newCandidateNames = registry.getBeanDefinitionNames();
				Set<String> oldCandidateNames = new HashSet<String>(Arrays.asList(candidateNames));
				Set<String> alreadyParsedClasses = new HashSet<String>();
				for (ConfigurationClass configurationClass : alreadyParsed) {
					alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
				}
				for (String candidateName : newCandidateNames) {
					if (!oldCandidateNames.contains(candidateName)) {
						BeanDefinition bd = registry.getBeanDefinition(candidateName);
						if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
								!alreadyParsedClasses.contains(bd.getBeanClassName())) {
							candidates.add(new BeanDefinitionHolder(bd, candidateName));
						}
					}
				}
				candidateNames = newCandidateNames;
			}
		}
		while (!candidates.isEmpty());

		// Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
		if (sbr != null) {
			if (!sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
				sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
			}
		}

		if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
			((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
		}
	}

//......    
		}
```

很显然,我们可以看到**加载@Configuration注解定义的配置类时是放在一个LinkedHashSet中一起处理的**,很可能在这个set中顺序放在前面就会先被加载,顺序放在后面就会后被加载.

断点查看set中的ConfigurationClass顺序,果然结果跟我们的推测相同,**ConfigurationClass的LinkedHashSet顺序决定了加载顺序,导致最后生效的SimpleModule对象不同.**

众所周知LinkedHashSet的顺序为元素添加到集合的顺序.
而configClasses这个变量的数据来源为parser.getConfigurationClasses()方法,所以我们需要继续查看该方法源码.
```java
class ConfigurationClassParser {
    //......   
    	private final Map<ConfigurationClass, ConfigurationClass> configurationClasses =
    			new LinkedHashMap<ConfigurationClass, ConfigurationClass>();
    //......   
    
    
    //......   
    	public Set<ConfigurationClass> getConfigurationClasses() {
    		return this.configurationClasses.keySet();
    	}
    //......   
}
```
显然这也是一个LinkedHashSet还要继续寻找元素添加的来源顺序.
从org.springframework.context.annotation.ConfigurationClassParser类中的parse方法继续追踪下去,具体处理的逻辑在doProcessConfigurationClass方法中,源码如下
```java
class ConfigurationClassParser {
    //......   

/**
	 * Apply processing and build a complete {@link ConfigurationClass} by reading the
	 * annotations, members and methods from the source class. This method can be called
	 * multiple times as relevant sources are discovered.
	 * @param configClass the configuration class being build
	 * @param sourceClass a source class
	 * @return the superclass, or {@code null} if none found or previously processed
	 */
	protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
			throws IOException {

		// Recursively process any member (nested) classes first
		processMemberClasses(configClass, sourceClass);

		// Process any @PropertySource annotations
		for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
				sourceClass.getMetadata(), PropertySources.class,
				org.springframework.context.annotation.PropertySource.class)) {
			if (this.environment instanceof ConfigurableEnvironment) {
				processPropertySource(propertySource);
			}
			else {
				logger.warn("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
						"]. Reason: Environment must implement ConfigurableEnvironment");
			}
		}

		// Process any @ComponentScan annotations
		Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
				sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
		if (!componentScans.isEmpty() &&
				!this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
			for (AnnotationAttributes componentScan : componentScans) {
				// The config class is annotated with @ComponentScan -> perform the scan immediately
				Set<BeanDefinitionHolder> scannedBeanDefinitions =
						this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
				// Check the set of scanned definitions for any further config classes and parse recursively if needed
				for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
					BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
					if (bdCand == null) {
						bdCand = holder.getBeanDefinition();
					}
					if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
						parse(bdCand.getBeanClassName(), holder.getBeanName());
					}
				}
			}
		}

		// Process any @Import annotations
		processImports(configClass, sourceClass, getImports(sourceClass), true);

		// Process any @ImportResource annotations
		if (sourceClass.getMetadata().isAnnotated(ImportResource.class.getName())) {
			AnnotationAttributes importResource =
					AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
			String[] resources = importResource.getStringArray("locations");
			Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
			for (String resource : resources) {
				String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
				configClass.addImportedResource(resolvedResource, readerClass);
			}
		}

		// Process individual @Bean methods
		Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
		for (MethodMetadata methodMetadata : beanMethods) {
			configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
		}

		// Process default methods on interfaces
		processInterfaces(configClass, sourceClass);

		// Process superclass, if any
		if (sourceClass.getMetadata().hasSuperClass()) {
			String superclass = sourceClass.getMetadata().getSuperClassName();
			if (!superclass.startsWith("java") && !this.knownSuperclasses.containsKey(superclass)) {
				this.knownSuperclasses.put(superclass, configClass);
				// Superclass found, return its annotation metadata and recurse
				return sourceClass.getSuperClass();
			}
		}

		// No superclass -> processing is complete
		return null;
	}

    //......   
}
```
其中scannedBeanDefinitions变量依然是个LinkedHashSet并且线上与本地的jar包中bean顺序不同.

继续查看org.springframework.context.annotation.ComponentScanAnnotationParser的parse方法源码如下
```java
class ComponentScanAnnotationParser {
    //......   
    
    public Set<BeanDefinitionHolder> parse(AnnotationAttributes componentScan, final String declaringClass) {
    		Assert.state(this.environment != null, "Environment must not be null");
    		Assert.state(this.resourceLoader != null, "ResourceLoader must not be null");
    
    		ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(this.registry,
    				componentScan.getBoolean("useDefaultFilters"), this.environment, this.resourceLoader);
    
    		Class<? extends BeanNameGenerator> generatorClass = componentScan.getClass("nameGenerator");
    		boolean useInheritedGenerator = (BeanNameGenerator.class == generatorClass);
    		scanner.setBeanNameGenerator(useInheritedGenerator ? this.beanNameGenerator :
    				BeanUtils.instantiateClass(generatorClass));
    
    		ScopedProxyMode scopedProxyMode = componentScan.getEnum("scopedProxy");
    		if (scopedProxyMode != ScopedProxyMode.DEFAULT) {
    			scanner.setScopedProxyMode(scopedProxyMode);
    		}
    		else {
    			Class<? extends ScopeMetadataResolver> resolverClass = componentScan.getClass("scopeResolver");
    			scanner.setScopeMetadataResolver(BeanUtils.instantiateClass(resolverClass));
    		}
    
    		scanner.setResourcePattern(componentScan.getString("resourcePattern"));
    
    		for (AnnotationAttributes filter : componentScan.getAnnotationArray("includeFilters")) {
    			for (TypeFilter typeFilter : typeFiltersFor(filter)) {
    				scanner.addIncludeFilter(typeFilter);
    			}
    		}
    		for (AnnotationAttributes filter : componentScan.getAnnotationArray("excludeFilters")) {
    			for (TypeFilter typeFilter : typeFiltersFor(filter)) {
    				scanner.addExcludeFilter(typeFilter);
    			}
    		}
    
    		boolean lazyInit = componentScan.getBoolean("lazyInit");
    		if (lazyInit) {
    			scanner.getBeanDefinitionDefaults().setLazyInit(true);
    		}
    
    		Set<String> basePackages = new LinkedHashSet<String>();
    		String[] basePackagesArray = componentScan.getStringArray("basePackages");
    		for (String pkg : basePackagesArray) {
    			String[] tokenized = StringUtils.tokenizeToStringArray(this.environment.resolvePlaceholders(pkg),
    					ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
    			basePackages.addAll(Arrays.asList(tokenized));
    		}
    		for (Class<?> clazz : componentScan.getClassArray("basePackageClasses")) {
    			basePackages.add(ClassUtils.getPackageName(clazz));
    		}
    		if (basePackages.isEmpty()) {
    			basePackages.add(ClassUtils.getPackageName(declaringClass));
    		}
    
    		scanner.addExcludeFilter(new AbstractTypeHierarchyTraversingFilter(false, false) {
    			@Override
    			protected boolean matchClassName(String className) {
    				return declaringClass.equals(className);
    			}
    		});
    		return scanner.doScan(StringUtils.toStringArray(basePackages));
    	}
    
    //......   
}
```

很显然,具体逻辑在scanner.doScan方法中,具体查看org.springframework.context.annotation.ClassPathBeanDefinitionScanner的doScan方法如下
```java
public class ClassPathBeanDefinitionScanner extends ClassPathScanningCandidateComponentProvider {
   //......  
   
   /**
   	 * Perform a scan within the specified base packages,
   	 * returning the registered bean definitions.
   	 * <p>This method does <i>not</i> register an annotation config processor
   	 * but rather leaves this up to the caller.
   	 * @param basePackages the packages to check for annotated classes
   	 * @return set of beans registered if any for tooling registration purposes (never {@code null})
   	 */
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
   					BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
   					definitionHolder =
   							AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
   					beanDefinitions.add(definitionHolder);
   					registerBeanDefinition(definitionHolder, this.registry);
   				}
   			}
   		}
   		return beanDefinitions;
   	}
   
   //......   
}
```

显然,顺序来自于findCandidateComponents方法
```java
public class ClassPathScanningCandidateComponentProvider implements EnvironmentCapable, ResourceLoaderAware {
    //...... 
    
	/**
	 * Scan the class path for candidate components.
	 * @param basePackage the package to check for annotated classes
	 * @return a corresponding Set of autodetected bean definitions
	 */
	public Set<BeanDefinition> findCandidateComponents(String basePackage) {
		Set<BeanDefinition> candidates = new LinkedHashSet<BeanDefinition>();
		try {
			String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
					resolveBasePackage(basePackage) + '/' + this.resourcePattern;
			Resource[] resources = this.resourcePatternResolver.getResources(packageSearchPath);
			boolean traceEnabled = logger.isTraceEnabled();
			boolean debugEnabled = logger.isDebugEnabled();
			for (Resource resource : resources) {
				if (traceEnabled) {
					logger.trace("Scanning " + resource);
				}
				if (resource.isReadable()) {
					try {
						MetadataReader metadataReader = this.metadataReaderFactory.getMetadataReader(resource);
						if (isCandidateComponent(metadataReader)) {
							ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
							sbd.setResource(resource);
							sbd.setSource(resource);
							if (isCandidateComponent(sbd)) {
								if (debugEnabled) {
									logger.debug("Identified candidate component class: " + resource);
								}
								candidates.add(sbd);
							}
							else {
								if (debugEnabled) {
									logger.debug("Ignored because not a concrete top-level class: " + resource);
								}
							}
						}
						else {
							if (traceEnabled) {
								logger.trace("Ignored because not matching any filter: " + resource);
							}
						}
					}
					catch (Throwable ex) {
						throw new BeanDefinitionStoreException(
								"Failed to read candidate component class: " + resource, ex);
					}
				}
				else {
					if (traceEnabled) {
						logger.trace("Ignored because not readable: " + resource);
					}
				}
			}
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
		}
		return candidates;
	}
    
    //...... 
}
```
可以看到,顺序的最终来源是Resource数组的顺序,继续查看扫描resources的具体逻辑源码
```java
public class PathMatchingResourcePatternResolver implements ResourcePatternResolver {
    //...... 
    
    	/**
    	 * Find all resources that match the given location pattern via the
    	 * Ant-style PathMatcher. Supports resources in jar files and zip files
    	 * and in the file system.
    	 * @param locationPattern the location pattern to match
    	 * @return the result as Resource array
    	 * @throws IOException in case of I/O errors
    	 * @see #doFindPathMatchingJarResources
    	 * @see #doFindPathMatchingFileResources
    	 * @see org.springframework.util.PathMatcher
    	 */
    	protected Resource[] findPathMatchingResources(String locationPattern) throws IOException {
    		String rootDirPath = determineRootDir(locationPattern);
    		String subPattern = locationPattern.substring(rootDirPath.length());
    		Resource[] rootDirResources = getResources(rootDirPath);
    		Set<Resource> result = new LinkedHashSet<Resource>(16);
    		for (Resource rootDirResource : rootDirResources) {
    			rootDirResource = resolveRootDirResource(rootDirResource);
    			URL rootDirUrl = rootDirResource.getURL();
    			if (equinoxResolveMethod != null) {
    				if (rootDirUrl.getProtocol().startsWith("bundle")) {
    					rootDirUrl = (URL) ReflectionUtils.invokeMethod(equinoxResolveMethod, null, rootDirUrl);
    					rootDirResource = new UrlResource(rootDirUrl);
    				}
    			}
    			if (rootDirUrl.getProtocol().startsWith(ResourceUtils.URL_PROTOCOL_VFS)) {
    				result.addAll(VfsResourceMatchingDelegate.findMatchingResources(rootDirUrl, subPattern, getPathMatcher()));
    			}
    			else if (ResourceUtils.isJarURL(rootDirUrl) || isJarResource(rootDirResource)) {
    				result.addAll(doFindPathMatchingJarResources(rootDirResource, rootDirUrl, subPattern));
    			}
    			else {
    				result.addAll(doFindPathMatchingFileResources(rootDirResource, subPattern));
    			}
    		}
    		if (logger.isDebugEnabled()) {
    			logger.debug("Resolved location pattern [" + locationPattern + "] to resources " + result);
    		}
    		return result.toArray(new Resource[result.size()]);
    	}
    
    //...... 
}
```

经过断点发现,在本地启动时,进入的是
```
result.addAll(doFindPathMatchingFileResources(rootDirResource, subPattern))分支.
```
而在线上jar包时进入的是jar包分支
```
else if (ResourceUtils.isJarURL(rootDirUrl) || isJarResource(rootDirResource)) {
    				result.addAll(doFindPathMatchingJarResources(rootDirResource, rootDirUrl, subPattern));
}
```

终于,在这里出现了运行分支的区别.

首先查看本地启动时的逻辑
```java
public class PathMatchingResourcePatternResolver implements ResourcePatternResolver {
    //...... 
    
    	/**
    	 * Recursively retrieve files that match the given pattern,
    	 * adding them to the given result list.
    	 * @param fullPattern the pattern to match against,
    	 * with prepended root directory path
    	 * @param dir the current directory
    	 * @param result the Set of matching File instances to add to
    	 * @throws IOException if directory contents could not be retrieved
    	 */
    	protected void doRetrieveMatchingFiles(String fullPattern, File dir, Set<File> result) throws IOException {
    		if (logger.isDebugEnabled()) {
    			logger.debug("Searching directory [" + dir.getAbsolutePath() +
    					"] for files matching pattern [" + fullPattern + "]");
    		}
    		File[] dirContents = dir.listFiles();
    		if (dirContents == null) {
    			if (logger.isWarnEnabled()) {
    				logger.warn("Could not retrieve contents of directory [" + dir.getAbsolutePath() + "]");
    			}
    			return;
    		}
    		Arrays.sort(dirContents);
    		for (File content : dirContents) {
    			String currPath = StringUtils.replace(content.getAbsolutePath(), File.separator, "/");
    			if (content.isDirectory() && getPathMatcher().matchStart(fullPattern, currPath + "/")) {
    				if (!content.canRead()) {
    					if (logger.isDebugEnabled()) {
    						logger.debug("Skipping subdirectory [" + dir.getAbsolutePath() +
    								"] because the application is not allowed to read the directory");
    					}
    				}
    				else {
    					doRetrieveMatchingFiles(fullPattern, content, result);
    				}
    			}
    			if (getPathMatcher().match(fullPattern, currPath)) {
    				result.add(content);
    			}
    		}
    	}
    
    //...... 
}
```

很显然,在此时,加载Resource的顺序取决于文件和文件夹的排序顺序,再具体查看File类的比较源码
```java
public class File
    implements Serializable, Comparable<File>
{
    //...... 
    
    private static final FileSystem fs = DefaultFileSystem.getFileSystem();
    
    public int compareTo(File pathname) {
            return fs.compare(this, pathname);
        }
    
    //...... 
}
```

显然在本地是windows系统,所以FileSystem具体实现类为java.io.WinNTFileSystem,比较方法源码如下
```java
class WinNTFileSystem extends FileSystem {
    //...... 
        @Override
        public int compare(File f1, File f2) {
            return f1.getPath().compareToIgnoreCase(f2.getPath());
        }
    //...... 
}
```
所以文件排序顺序为文件的绝对路径的字符串进行无视大小写的比较,继续查看字符串的比较逻辑
```java
    private static class CaseInsensitiveComparator
            implements Comparator<String>, java.io.Serializable {
        // use serialVersionUID from JDK 1.2.2 for interoperability
        private static final long serialVersionUID = 8575799808933029326L;

        public int compare(String s1, String s2) {
            int n1 = s1.length();
            int n2 = s2.length();
            int min = Math.min(n1, n2);
            for (int i = 0; i < min; i++) {
                char c1 = s1.charAt(i);
                char c2 = s2.charAt(i);
                if (c1 != c2) {
                    c1 = Character.toUpperCase(c1);
                    c2 = Character.toUpperCase(c2);
                    if (c1 != c2) {
                        c1 = Character.toLowerCase(c1);
                        c2 = Character.toLowerCase(c2);
                        if (c1 != c2) {
                            // No overflow because of numeric promotion
                            return c1 - c2;
                        }
                    }
                }
            }
            return n1 - n2;
        }

        /** Replaces the de-serialized object. */
        private Object readResolve() { return CASE_INSENSITIVE_ORDER; }
    }
```
可以得知字符串的比较逻辑为首个非相同字符的char值的差,所以字母越小,排序越前.
所以我们可以看到**顺序是和idea中包和类的排序顺序是一样**的.

**所以在本地启动必定是FeignConfiguration先加载,StreamConfiguration后加载,StreamConfiguration的SimpleModule覆盖了FeignConfiguration的SimpleModule.**

接下来看线上jar包的逻辑源码
```java
public class PathMatchingResourcePatternResolver implements ResourcePatternResolver {
    //...... 
    
    /**
    	 * Find all resources in jar files that match the given location pattern
    	 * via the Ant-style PathMatcher.
    	 * @param rootDirResource the root directory as Resource
    	 * @param rootDirURL the pre-resolved root directory URL
    	 * @param subPattern the sub pattern to match (below the root directory)
    	 * @return a mutable Set of matching Resource instances
    	 * @throws IOException in case of I/O errors
    	 * @since 4.3
    	 * @see java.net.JarURLConnection
    	 * @see org.springframework.util.PathMatcher
    	 */
    	@SuppressWarnings("deprecation")
    	protected Set<Resource> doFindPathMatchingJarResources(Resource rootDirResource, URL rootDirURL, String subPattern)
    			throws IOException {
    
    		// Check deprecated variant for potential overriding first...
    		Set<Resource> result = doFindPathMatchingJarResources(rootDirResource, subPattern);
    		if (result != null) {
    			return result;
    		}
    
    		URLConnection con = rootDirURL.openConnection();
    		JarFile jarFile;
    		String jarFileUrl;
    		String rootEntryPath;
    		boolean closeJarFile;
    
    		if (con instanceof JarURLConnection) {
    			// Should usually be the case for traditional JAR files.
    			JarURLConnection jarCon = (JarURLConnection) con;
    			ResourceUtils.useCachesIfNecessary(jarCon);
    			jarFile = jarCon.getJarFile();
    			jarFileUrl = jarCon.getJarFileURL().toExternalForm();
    			JarEntry jarEntry = jarCon.getJarEntry();
    			rootEntryPath = (jarEntry != null ? jarEntry.getName() : "");
    			closeJarFile = !jarCon.getUseCaches();
    		}
    		else {
    			// No JarURLConnection -> need to resort to URL file parsing.
    			// We'll assume URLs of the format "jar:path!/entry", with the protocol
    			// being arbitrary as long as following the entry format.
    			// We'll also handle paths with and without leading "file:" prefix.
    			String urlFile = rootDirURL.getFile();
    			try {
    				int separatorIndex = urlFile.indexOf(ResourceUtils.WAR_URL_SEPARATOR);
    				if (separatorIndex == -1) {
    					separatorIndex = urlFile.indexOf(ResourceUtils.JAR_URL_SEPARATOR);
    				}
    				if (separatorIndex != -1) {
    					jarFileUrl = urlFile.substring(0, separatorIndex);
    					rootEntryPath = urlFile.substring(separatorIndex + 2);  // both separators are 2 chars
    					jarFile = getJarFile(jarFileUrl);
    				}
    				else {
    					jarFile = new JarFile(urlFile);
    					jarFileUrl = urlFile;
    					rootEntryPath = "";
    				}
    				closeJarFile = true;
    			}
    			catch (ZipException ex) {
    				if (logger.isDebugEnabled()) {
    					logger.debug("Skipping invalid jar classpath entry [" + urlFile + "]");
    				}
    				return Collections.emptySet();
    			}
    		}
    
    		try {
    			if (logger.isDebugEnabled()) {
    				logger.debug("Looking for matching resources in jar file [" + jarFileUrl + "]");
    			}
    			if (!"".equals(rootEntryPath) && !rootEntryPath.endsWith("/")) {
    				// Root entry path must end with slash to allow for proper matching.
    				// The Sun JRE does not return a slash here, but BEA JRockit does.
    				rootEntryPath = rootEntryPath + "/";
    			}
    			result = new LinkedHashSet<Resource>(8);
    			for (Enumeration<JarEntry> entries = jarFile.entries(); entries.hasMoreElements();) {
    				JarEntry entry = entries.nextElement();
    				String entryPath = entry.getName();
    				if (entryPath.startsWith(rootEntryPath)) {
    					String relativePath = entryPath.substring(rootEntryPath.length());
    					if (getPathMatcher().match(subPattern, relativePath)) {
    						result.add(rootDirResource.createRelative(relativePath));
    					}
    				}
    			}
    			return result;
    		}
    		finally {
    			if (closeJarFile) {
    				jarFile.close();
    			}
    		}
    	}
    
    //...... 
}
```
可以看出此时加载顺序取决于jarFile.entries()方法,继续看源码
```java
public
class JarFile extends ZipFile {
 //...... 
 
     private class JarEntryIterator implements Enumeration<JarEntry>,
             Iterator<JarEntry>
     {
         final Enumeration<? extends ZipEntry> e = JarFile.super.entries();
 
         public boolean hasNext() {
             return e.hasMoreElements();
         }
 
         public JarEntry next() {
             ZipEntry ze = e.nextElement();
             return new JarFileEntry(ze);
         }
 
         public boolean hasMoreElements() {
             return hasNext();
         }
 
         public JarEntry nextElement() {
             return next();
         }
     }
 
     /**
      * Returns an enumeration of the zip file entries.
      */
     public Enumeration<JarEntry> entries() {
         return new JarEntryIterator();
     }
 
 //...... 
}
public
class ZipFile implements ZipConstants, Closeable {
 //...... 
 
 private class ZipEntryIterator implements Enumeration<ZipEntry>, Iterator<ZipEntry> {
         private int i = 0;
 
         public ZipEntryIterator() {
             ensureOpen();
         }
 
         public boolean hasMoreElements() {
             return hasNext();
         }
 
         public boolean hasNext() {
             synchronized (ZipFile.this) {
                 ensureOpen();
                 return i < total;
             }
         }
 
         public ZipEntry nextElement() {
             return next();
         }
 
         public ZipEntry next() {
             synchronized (ZipFile.this) {
                 ensureOpen();
                 if (i >= total) {
                     throw new NoSuchElementException();
                 }
                 long jzentry = getNextEntry(jzfile, i++);
                 if (jzentry == 0) {
                     String message;
                     if (closeRequested) {
                         message = "ZipFile concurrently closed";
                     } else {
                         message = getZipMessage(ZipFile.this.jzfile);
                     }
                     throw new ZipError("jzentry == 0" +
                                        ",\n jzfile = " + ZipFile.this.jzfile +
                                        ",\n total = " + ZipFile.this.total +
                                        ",\n name = " + ZipFile.this.name +
                                        ",\n i = " + i +
                                        ",\n message = " + message
                         );
                 }
                 ZipEntry ze = getZipEntry(null, jzentry);
                 freeEntry(jzfile, jzentry);
                 return ze;
             }
         }
     }
 
     /**
      * Returns an enumeration of the ZIP file entries.
      * @return an enumeration of the ZIP file entries
      * @throws IllegalStateException if the zip file has been closed
      */
     public Enumeration<? extends ZipEntry> entries() {
         return new ZipEntryIterator();
     }
 
     private static native long getNextEntry(long jzfile, int i);
     
 //...... 
}
```

**其顺序获取的方法是一个native的本地方法**,无法再详细查看.

但是从结果来看,显然有一套自己的排序规则,**与普通情况下的排序并不相同**.

所以这就是为什么本地与线上行为不同,但是代码相同,也不是环境影响的原因.

## bug总结
**在加载bean定义时,本地直接启动和打包后发布启动,走的是不同的资源加载顺序.**

**本地直接启动时,分支为普通的class文件和其文件夹的加载,顺序为其绝对路径的字符串比较,取决于第一个非相同字符的字母排序.**

**打包为jar包发布后启动时,分支为jar包资源加载,加载顺序为jdk本地方法,与普通的class文件加载顺序不同.**

**因此导致了代码完全相同,排除环境影响,但是加载顺序不同,导致最终代码行为不同.**