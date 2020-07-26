# Spring使用构造函数初始化bean

## 把所有准备初始化的bean准备好

### 1.bean初始化的时机

Bean初始化是从AnnotationConfigApplicationContext的refresh()函数调用的时候开始。

```java
annotationConfigApplicationContext.refresh();
```

### 2.开始对bean进行初始化

```
AbstractApplicationContext.finishBeanFactoryInitialization中执行到beanFactory.preInstantiateSingletons();
```

3.preInstantiateSingletons()方法默认是使用DefaultListableBeanFactory的，这个方法里面把所有扫描出来的beanName拿出来，然后开始初始化所有非静态、单例、非懒加载的bean，主要就是看getBean()方法

下面代码的重点是

- 拿出扫描出来的所有beanName
- 将非抽象、非单例、非懒加载的bean进行初始化

```java
	//准备实现单例
	@Override
	public void preInstantiateSingletons() throws BeansException {
		if (logger.isTraceEnabled()) {
			logger.trace("Pre-instantiating singletons in " + this);
		}
		// 1.创建beanDefinitionNames的副本beanNames用于后续的遍历，以允许init等方法注册新的bean定义
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		//触发所有非延迟加载单例beans的初始化，主要步骤为调用getBean
		for (String beanName : beanNames) {
			//合并父类的bd，在xml配置中用
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			//如果不是抽象 && 是单例 && 不是懒加载的
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					//判断是不是FactoryBean
					if (bean instanceof FactoryBean) {
						final FactoryBean<?> factory = (FactoryBean<?>) bean;
						boolean isEagerInit;
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
											((SmartFactoryBean<?>) factory)::isEagerInit,
									getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean<?>) factory).isEagerInit());
						}
						if (isEagerInit) {
							getBean(beanName);
						}
					}
				}
                //我们的普通类都是走这里
				else {
					getBean(beanName);
				}
			}
		}
		// Trigger post-initialization callback for all applicable beans...
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
						smartSingleton.afterSingletonsInstantiated();
						return null;
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}		
```

getBean()里面调用dogetBean()方法，这里就是正式 开始初始化

## doGetBean方法（bean真正创建地方）

先创建一个空白的bean对象，

然后

```
	protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			/**
			 * 创建 bean 实例，并将实例包裹在 BeanWrapper 实现类对象中返回。
			 * createBeanInstance中包含三种创建 bean 实例的方式：
			 *   1. 通过工厂方法创建 bean 实例
			 *   2. 通过构造方法自动注入（autowire by constructor）的方式创建 bean 实例
			 *   3. 通过无参构造方法方法创建 bean 实例
			 *
			 * 若 bean 的配置信息中配置了 lookup-method 和 replace-method，则会使用 CGLIB
			 * 增强 bean 实例。关于lookup-method和replace-method后面再说。
			 */
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		final Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}

		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) {
				logger.trace("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			//把bean从DefaultSingletonBeanRegistry.earlySingletonObjects中移除
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		//这是时候还是原生的bean对象
		Object exposedObject = bean;
		try {
			//设置属性，非常重要
			populateBean(beanName, mbd, instanceWrapper);
			//执行后置处理器，aop就是在这里完成的处理!!!
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}
```

### 1.createBeanInstance创建一个空的bean对象

1. 如果bean有factory-method，就用factory-method初始化bean

2. 如果对象已经被创建过（比如这个对象是prototype类型的），工厂里就会保存有创建过的构造函数和参数，可以直接根据缓存的构造函数和参数

   - 无参构造函数instantiateBean创建被BeanWrapper包裹的bean

   - 有参构造函数autowireConstructor

3. 如果是第一次初始化bean，由后置处理器决定要用哪些构造方法

```java
	protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
		// Make sure bean class is actually resolved at this point.
		Class<?> beanClass = resolveBeanClass(mbd, beanName);

		/**
		 * 检测一个类的访问权限spring默认情况下对于非public的类是允许访问的。
		 */
		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}

		//跳过
		Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
		if (instanceSupplier != null) {
			return obtainFromSupplier(instanceSupplier, beanName);
		}

		/**
		 *
		 * 如果工厂方法不为空，则通过工厂方法构建 bean 对象
		 * 这种构建 bean 的方式可以自己写个demo去试试
		 * 源码就不做深入分析了，有兴趣的同学可以和我私下讨论
		 */
		if (mbd.getFactoryMethodName() != null) {
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		// Shortcut when re-creating the same bean...
		/**
		 * 从spring的原始注释可以知道这个是一个Shortcut，什么意思呢？
		 * 当多次构建同一个 bean 时，可以使用这个Shortcut，
		 * 也就是说不在需要次推断应该使用哪种方式构造bean
		 *  比如在多次构建同一个prototype类型的 bean 时，就可以走此处的hortcut
		 * 这里的 resolved 和 mbd.constructorArgumentsResolved 将会在 bean 第一次实例
		 * 化的过程中被设置，后面来证明
		 */
		boolean resolved = false;
		boolean autowireNecessary = false;
		if (args == null) {
			synchronized (mbd.constructorArgumentLock) {
				//如果bd是由factoryMethod创建
				if (mbd.resolvedConstructorOrFactoryMethod != null) {
					resolved = true;
					//如果已经解析了构造方法的参数，则必须要通过一个带参构造方法来实例
					autowireNecessary = mbd.constructorArgumentsResolved;
				}
			}
		}
		//如果是factoryMethod创建的方法
		if (resolved) {
			if (autowireNecessary) {
				//有参
				// 通过构造方法自动装配的方式构造 bean 对象
				return autowireConstructor(beanName, mbd, null, null);
			}
			else {
				//通过默认的无参构造方法进行
				return instantiateBean(beanName, mbd);
			}
		}

		/**
		 * 下面的逻辑是，如果只有默认的无参构造方法，ctors是null 直接使用默认的无参构造方法进行初始化
		 * 如果有有参构造方法，
		 */
		// Candidate constructors for autowiring?
		//由后置处理器决定返回哪些构造方法
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		//getResolvedAutowireMode()获取spring自动装配的模型
		/**
		 * spring 自动装配的模型 != 自动装配的技术
		 * 比如这里，getResolvedAutowireMode()返回0，AUTOWIRE_NO，默认是通过类型自动装配
		 * 但是和byType不是一回事
		 *
		 */
		if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		// Preferred constructors for default construction?
		ctors = mbd.getPreferredConstructors();
		if (ctors != null) {
			return autowireConstructor(beanName, mbd, ctors, null);
		}

		// No special handling: simply use no-arg constructor.
		//使用默认的无参构造方法进行初始化
		return instantiateBean(beanName, mbd);
	}
```

### 2.通过后置处理器决定使用哪些构造函数

AutowiredAnnotationBeanPostProcessor后置处理器通过determineCandidateConstructors方法决定使用哪些构造方法

candidateConstructors --最终要返回的构造函数，如果缓存中有就直接返回
candidates --最终适用的构造函数
requiredConstructor --带有required=true的构造函数
defaultConstructor --默认构造函数

先遍历一遍遍历所有的构造函数

1.如果当前构造函数有注解

​	a.requiredConstructor不为空，说明前面已经存在过一个requred=true的构造函数，抛异常

​	b.当前构造函数是required=true的，判断candidates是否为空，为空的话说明前面已经有过加注解的构造函数，抛异常

​    c.前面过滤条件都满足，将当前requird=true的构造函数赋值给requiredConstructor

​	d.前面没有带注解的构造函数，或者当前构造函数required=true条件满足，将当前构造函数加入candidates

 2.如果当前构造函数没有注解并且参数是0个，将当前的无参构造函数赋值给defaultConstructor

​	a.在构造函数只有一个且有参是，将此唯一有参构造函数加入candidates

​    b.在构造函数有两个的时候，并且存在无参构造函数，将defaultConstructo(上面的无参构造函数)r加入candidates

​	c.在构造器数量大于两个，并且存在无参构造器的情况下，将返回一个空的candidateConstructors集合，也就是没有找到构造器。

```java
	@Override
	@Nullable
	public Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, final String beanName)
			throws BeanCreationException {

		// Let's check for lookup methods here...
		if (!this.lookupMethodsChecked.contains(beanName)) {
			try {
				ReflectionUtils.doWithMethods(beanClass, method -> {
					Lookup lookup = method.getAnnotation(Lookup.class);
					if (lookup != null) {
						Assert.state(this.beanFactory != null, "No BeanFactory available");
						LookupOverride override = new LookupOverride(method, lookup.value());
						try {
							RootBeanDefinition mbd = (RootBeanDefinition)
									this.beanFactory.getMergedBeanDefinition(beanName);
							mbd.getMethodOverrides().addOverride(override);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(beanName,
									"Cannot apply @Lookup to beans without corresponding bean definition");
						}
					}
				});
			}
			catch (IllegalStateException ex) {
				throw new BeanCreationException(beanName, "Lookup method resolution failed", ex);
			}
			this.lookupMethodsChecked.add(beanName);
		}

		// Quick check on the concurrent map first, with minimal locking.
		//从构造方法的缓存当中去拿一个
		Constructor<?>[] candidateConstructors = this.candidateConstructorsCache.get(beanClass);
		//如果缓存中没有
		if (candidateConstructors == null) {
			// Fully synchronized resolution now...
			synchronized (this.candidateConstructorsCache) {
				candidateConstructors = this.candidateConstructorsCache.get(beanClass);
				//双重判断，避免多线程并发问题
				if (candidateConstructors == null) {
					Constructor<?>[] rawCandidates;
					try {
						//获取所有的构造函数
						rawCandidates = beanClass.getDeclaredConstructors();
					}
					catch (Throwable ex) {
						throw new BeanCreationException(beanName,
								"Resolution of declared constructors on bean Class [" + beanClass.getName() +
								"] from ClassLoader [" + beanClass.getClassLoader() + "] failed", ex);
					}
					//最终适用的构造器集合
					List<Constructor<?>> candidates = new ArrayList<>(rawCandidates.length);
					//存放依赖注入的required=true的构造器
					Constructor<?> requiredConstructor = null;
					//存放默认构造器
					Constructor<?> defaultConstructor = null;
					Constructor<?> primaryConstructor = BeanUtils.findPrimaryConstructor(beanClass);
					int nonSyntheticConstructors = 0;
					//循环拿出来的所有构造方法
					for (Constructor<?> candidate : rawCandidates) {
						if (!candidate.isSynthetic()) {
							nonSyntheticConstructors++;
						}
						else if (primaryConstructor != null) {
							continue;
						}
						//查找当前构造器上的注解
						AnnotationAttributes ann = findAutowiredAnnotation(candidate);
						//如果没有注解，再看看父类的注解
						if (ann == null) {
							Class<?> userClass = ClassUtils.getUserClass(beanClass);
							if (userClass != beanClass) {
								try {
									Constructor<?> superCtor =
											userClass.getDeclaredConstructor(candidate.getParameterTypes());
									ann = findAutowiredAnnotation(superCtor);
								}
								catch (NoSuchMethodException ex) {
									// Simply proceed, no equivalent superclass constructor found...
								}
							}
						}
						//若有注解
						if (ann != null) {
							//已经存在一个required=true的构造器了，抛出异常
							if (requiredConstructor != null) {
								throw new BeanCreationException(beanName,
										"Invalid autowire-marked constructor: " + candidate +
										". Found constructor with 'required' Autowired annotation already: " +
										requiredConstructor);
							}
							//判断此注解上的required属性
							boolean required = determineRequiredStatus(ann);
							if (required) {
								if (!candidates.isEmpty()) {
									throw new BeanCreationException(beanName,
											"Invalid autowire-marked constructors: " + candidates +
											". Found constructor with 'required' Autowired annotation: " +
											candidate);
								}
								requiredConstructor = candidate;
							}
							//将当前构造器加入requiredConstructor集合
							candidates.add(candidate);
						}
						//如果该构造函数上没有注解，再判断构造函数上的参数个数是否为0
						//当某个构造函数的参数是0个，也就是无参构造函数，赋值给defaultConstructor
						else if (candidate.getParameterCount() == 0) {
							defaultConstructor = candidate;
						}
					}
					//适用的构造器集合若不为空
					if (!candidates.isEmpty()) {
						// Add default constructor to list of optional constructors, as fallback.
						//若没有required=true的构造器
						if (requiredConstructor == null) {
							if (defaultConstructor != null) {
								//将defaultConstructor集合的构造器加入适用构造器集合
								candidates.add(defaultConstructor);
							}
							else if (candidates.size() == 1 && logger.isInfoEnabled()) {
								logger.info("Inconsistent constructor declaration on bean with name '" + beanName +
										"': single autowire-marked constructor flagged as optional - " +
										"this constructor is effectively required since there is no " +
										"default constructor to fall back to: " + candidates.get(0));
							}
						}
						//将适用构造器集合赋值给将要返回的构造器集合
						candidateConstructors = candidates.toArray(new Constructor<?>[0]);
					}
					//如果构造函数只有一个，并且这个构造函数的参数大于0
					else if (rawCandidates.length == 1 && rawCandidates[0].getParameterCount() > 0) {
						candidateConstructors = new Constructor<?>[] {rawCandidates[0]};
					}
					//当构造函数有两个 &&  提供了主要的构造函数 && 默认构造函数不为空 && 主要构造函数不等于默认构造方法
					else if (nonSyntheticConstructors == 2 && primaryConstructor != null &&
							defaultConstructor != null && !primaryConstructor.equals(defaultConstructor)) {
						candidateConstructors = new Constructor<?>[] {primaryConstructor, defaultConstructor};
					}
					//构造
					else if (nonSyntheticConstructors == 1 && primaryConstructor != null) {
						candidateConstructors = new Constructor<?>[] {primaryConstructor};
					}
					else {
						candidateConstructors = new Constructor<?>[0];
					}
					this.candidateConstructorsCache.put(beanClass, candidateConstructors);
				}
			}
		}
		return (candidateConstructors.length > 0 ? candidateConstructors : null);
	}
```

