## Spring

### AOP

**作用：**面向切面编程，是面向对象的升华，切面是将与业务无关的代码（比方说日志、事务、安全等等）进行封装，供业务代码进行调用，减少系统中的重复代码，降低系统的耦合度，增加系统的服务性。

**应用场景：**接口鉴权、打印日志、方法耗时统计、spring的声明式事务。[AOP代码实战篇](https://segmentfault.com/a/1190000007469982#articleHeader4)

#### AOP专业术语

1. **advice通知（增强）：**拦截到连接点之后要执行的代码，通知分为***前置 后置 异常 最终 环绕***。告诉AOP 切面之后做什么，也就是说，我们知道了在哪里进行切面，那么我们也该让spring知道在切点处做什么。

2. **joinpoint连接点：** advice 是在 join point 上执行的, 而 point cut 规定了哪些 join point 可以执行哪些 advice。joinpoint可以是一个执行方法。

**poincut切入点：**通知定义了切面的“什么”和“何时”的话，那么切点就定义了“何处”。来描述对哪些 Join point 使用 advise。首先我们需要告诉AOP在哪里进行切面。比如在某个类的方法前后进行切面。 

~~~java
public interface Pointcut {
                     ClassFilter getClassFilter();
                     // AOP 的作用是代理方法，那么，Spring 怎么知道代理哪些方法呢？必须通过某种方式来匹配方              法的名称来决定是否对该方法进行增强，这就是 MethodMatcher 的作用。
                     MethodMatcher getMethodMatcher(); 
                     Pointcut TRUE = TruePointcut.INSTANCE;
}
~~~

3. **Aspect切面：**通知和切点的结合。
4. **Advisor通知器：**将 Advice 和 PointCut 结合起来。
5. **织入(Weaving)：**将aspect应用到目标对象并且创建代理对象的过程。动态代理织入, 在运行期为目标类添加增强(Advice)生成子类的方式。Spring 采用动态代理织入，而AspectJ采用编译器织入和类装载期织入。

#### AOP实现策略

AOP实现的关键在于AOP框架自动创建的AOP代理，AOP代理主要分为静态代理和动态代理，静态代理的代表为AspectJ，而动态代理则以Spring AOP为代表。AspectJ是静态代理的增强，所谓的静态代理就是AOP框架会在编译阶段生成AOP代理类，因此也称为编译时增强。Spring AOP实现的是动态代理，主要有两种方式，JDK动态代理和CGLIB动态代理，AOP框架不会去修改字节码，而是在内存中临时为方法生成一个AOP对象，这个AOP对象包含了目标对象的全部方法，并且在特定的切点做了增强处理，并回调原对象的方法。

- JDK动态代理，在运行期间动态生成实现对象，在这个实现对象的接口方法中可以添加增强代码，要求被代理的类必须实现一个接口，如果没有则考虑CGLIB。

  ~~~java
  // 动态代码模式
  Proxy.newProxyInstance(targetObject.getClass().getClassLoader(),targetObject.getClass().getInterfaces(), InvocationHandler实现类);
  ~~~

- CGLIB动态代理，运行期间指定一个类的子类对象，并且覆盖其中特定方法，覆盖方法时可以添加增强代码。

  ~~~java
          //通过反射机制给他实例化
          Enhancer enhancer = new Enhancer();
          //注入父类，告诉CGLib，生成的子类需要继承那个类
          enhancer.setSuperclass(clazz);
          //添加监听 this对象必须实现MethodInterceptor类并且实现方法intercept 调用代理类的方法时，会进入  intercept方法!!!!
          enhancer.setCallback(this);
          //1.生成源代码
          //2.编译成.class文件
          //3.加载到JVM中，返回对象
          return enhancer.create();
  
  ~~~

#### AOP源码分析

##### XML的方式

~~~java
<bean id="testAdvisor" class="com.xavior.aopdemo.TestAdvisor"></bean>
    <bean id="testTarget" class="com.xavior.aopdemo.TestTarget"></bean>
    // 重点关注ProxyFactoryBean这个类
    <bean id="testAOP" class="org.springframework.aop.framework.ProxyFactoryBean">
        <property name="targetName">
            <value>testTarget</value>
        </property>
        <property name="interceptorNames">
            <list>
                <value>testAdvisor</value>
            </list>
        </property>
 </bean>
 TestTarget target = (TestTarget) applicationContext.getBean("testAOP");

~~~

AbstractApplicationContext.getBean，从容器或获取 Bean，再调用 doGetBean 方法。进入getObjectForBeanInstance：

~~~java
if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
            return beanInstance;
}
~~~

然后会进入方法doGetObjectFromFactoryBean(factory, beanName)，该方法会直接进入object = factory.getObject() 行，也就是 ProxyFactoryBean 的 getObject 方法，还记得我们说过，Spring 允许我们从写 getObject 方法来实现特定逻辑吗？ 我们看看该方法实现：

~~~java
	@Override
	public Object getObject() throws BeansException {
		initializeAdvisorChain();// 为代理对象配置advisor通知链：里面有获取切点和通知的方法
		if (isSingleton()) {
			return getSingletonInstance();
		}
		else {
			if (this.targetName == null) {
				logger.warn("Using non-singleton proxies with singleton targets is often undesirable. " +
						"Enable prototype proxies by setting the 'targetName' property.");
			}
			return newPrototypeInstance();
		}
	}
~~~

~~~java
	private synchronized Object getSingletonInstance() {
		if (this.singletonInstance == null) {
			this.targetSource = freshTargetSource();
			if (this.autodetectInterfaces && getProxiedInterfaces().length == 0 && !isProxyTargetClass()) {
				// Rely on AOP infrastructure to tell us what interfaces to proxy.
				Class<?> targetClass = getTargetClass();
				if (targetClass == null) {
					throw new FactoryBeanNotInitializedException("Cannot determine target class for proxy");
				}
				setInterfaces(ClassUtils.getAllInterfacesForClass(targetClass, this.proxyClassLoader));
			}
			// Initialize the shared singleton instance.
			super.setFrozen(this.freezeProxy);
      // createAopProxy()这个方法，如果目标类含有接口，则创建JdkDynamicAopProxy，否则
      // 创建ObjenesisCglibAopProxy，最后么，就是获取代理对象。
			this.singletonInstance = getProxy(createAopProxy());
		}
		return this.singletonInstance;
	}
~~~

总结：XML的方式实现AOP，其中真正起作用的还是ProxyFactoryBean，在定制化bean的过程中起到了很大的作用，也提醒了我们，如果想在spring的bean容器实现一些特别的功能，可以实现FactoryBean接口，自定义自己的需要bean。

##### 注解方式

**回顾：**

基于XML：扩展FactoryBean接口（ProxyFactoryBean），对特定的bean进行增强。接口Pointcut里面有个MethodMatcher用于决定对该方法是否进行增强。接口Advice是一个标识，什么都没有定义，但是我们常用的几个接口，比如 BeforeAdvice，AfterAdvice，都是继承自它。接口Advisor(通知器)结合Pointcut接口和Advice，里面最重要的两个方法：getAdvice，getPointcut。接下来，我们可以停下来思考一下，现在有了这个东西，我们怎么实现面向切面编程：
1. 首先我们需要告诉AOP在哪里进行切面。比如在某个类的方法前后进行切面。   
2. 告诉AOP 切面之后做什么，也就是说，我们知道了在哪里进行切面，那么我们也该让spring知道在切点处做什么。 
3. 我们知道，Spring AOP 的底层实现是动态代理（不管是JDK还是Cglib），那么就需要一个代理对象，那么如何生成呢？通过ProxyFactoryBean生成代理对象createAopProxy，里面使用默认的DefaultAopProxyFactory生成AopProxy（继承结构：一个是 JDK 动态代理，一个是 Cglib 代理）。

**步入正题：**

扩展BeanPostProcessor(后置处理器接口)接口，对特定的bean进行增强。

~~~java
       public interface BeanPostProcessor {
           Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
           Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException; 
            // 会在bean初始化后，调用此实现方法，createProxy创建代码对象
           }
~~~

![image-20190807150750973](http://ww1.sinaimg.cn/large/006tNc79gy1g61n4z47ksj324e0qq157.jpg)

AnnotationAwareAspectJAutoProxyCreator（这个类就是根据注解创建代理的默认类）的抽象父类实现BeanPostProcessor中的postProcessAfterInitialization方法。

~~~java
@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		if (bean != null) {
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			if (!this.earlyProxyReferences.contains(cacheKey)) {
        // 重点看这个方法
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}

protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
		if (beanName != null && this.targetSourcedBeans.contains(beanName)) {
			return bean;
		}
		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
			return bean;
		}
		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}

		// Create proxy if we have advice.
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		if (specificInterceptors != DO_NOT_PROXY) {
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
			// 就在此处进行创建代理 进入该方法就能看到：ProxyFactory proxyFactory = new ProxyFactory();return proxyFactory.getProxy(getProxyClassLoader());
      // ProxyFactory和ProxyFactoryBean一样继承了 ProxyCreatorSupport!底层实现是一样的
      Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}
~~~

**补充：**基于xml和基于注解在AbstractBeanFactory 的 doGetBean 方法开始分道扬镳，走向了不同的逻辑。

~~~java
// 基于XML扩展的是FactoryBean 基于注解扩展的BeanPostProcessor接口
           if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
			         return beanInstance;
		       }
~~~

**总结：**

首先，通过分析源码我们知道注解方式和 XML 配置方式的底层实现都是一样的，都是通过继承 ProxyCreatorSupport 来实现的，不同的通过扩展不同的 Spring 提供的接口，XML 扩展的是FactoryBean 接口，而注解方式扩展的是 BenaPostProcessor 接口，通过Spring 的扩展接口，能够对特定的Bean进行增强。而 AOP 正式通过这种方式实现的。这也提醒了我们，我们也可以通过扩展 Spring 的某些接口来增强我们需要的 Bean 的某些功能。当然，篇幅有限，我们这篇文章只是了解了XML 配置方式和注解方式创建代理的区别，关于如何 @Aspect 和 @Around 的底层实现，还有通知器的底层实现，我们还没有分析，但我们隐隐的感觉到，其实万变不离其宗，底层的也是通过扩展 advice 和 pointcut 接口来实现的。 我们将会在后面的文章继续分析 AOP 是如何编织通知的。

##### SpringBoot Aop

- AnnotationAwareAspectJAutoProxyCreator后置处理器注册过程，上文已经介绍过这个类了， 他继承了 AbstractAutoProxyCreator 抽象类，该类实现了后置处理器接口的方法。实现BeanPostProcessor。我们要看下在哪里有注册后置处理器的逻辑-》ioc容器初始化会调用refresh() 方法。

  ~~~java
   // 注册Bean的后处理器，在Bean创建过程中调用
                registerBeanPostProcessors(beanFactory);
                // 将后置处理器添加进工厂
                beanFactory.addBeanPostProcessor(postProcessor);
  ~~~

- 标注 @Aspect 注解的 bean 的后置处理器的处理过程：@Aspec 同时也会使用类似 @Component 的注解，表示这是一个Bean，只要是bean，就会调用getBean方法，只要调用 getBean 方法，都会调用AbstractAutowireCapableBeanFactory的createBean方法

  ~~~java
  // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.返回一个代理
  Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
  // 实例化前方法  进入该方法拿出在之前注册在BeanFactory 的后置处理器，并循环调用他们的 postProcessBeforeInstantiation 方法  我们关注的是AbstractAutoProxyCreator.后置处理器，
  bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
  if (bean != null) {
  // 初始化后方法
   bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
  	}
  ~~~

- 创建代理的过程，在执行完 resolveBeforeInstantiation 返回 null 之后，就执行 doCreateBean 方法

  ~~~java
  Object beanInstance = doCreateBean(beanName, mbdToUse, args);
  					          // 依赖注入工作
  					          populateBean(beanName, mbd, instanceWrapper);
  									  if (exposedObject != null) {
  									  	    // 初始化bean
  													exposedObject = initializeBean(beanName, exposedObject, mbd);
  									  }
  ~~~

  上面的initializeBean逻辑是一个非常重要的，在里面调用了3个重要逻辑：applyBeanPostProcessorsBeforeInitialization、invokeInitMethods、applyBeanPostProcessorsAfterInitialization
  首先调用了前置处理器的 postProcessBeforeInitialization 方法，简单的返回了bean。 然后判断是否实现 InitializingBean 接口从而决定是否调用 afterPropertiesSet 方法。最后到了最重要的 applyBeanPostProcessorsAfterInitialization 方法， 该方法会循环调用每个bean的后置处理器的 postProcessAfterInitialization 方法。

  ~~~java
  // 重写了BeanPostProcessor接口的方法 注意区分，BeanPostProcessor 是初始化，InstantiationAwareBeanPostProcessor 是实例化，实例化先执行，初始化后执行。
  	@Override
  	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
  		if (bean != null) {
  			Object cacheKey = getCacheKey(bean.getClass(), beanName);
  			if (!this.earlyProxyReferences.contains(cacheKey)) {
  				return wrapIfNecessary(bean, beanName, cacheKey);
  			}
  		}
  		return bean;
  	}
  	// 然后直接进入wrapIfNecessary方法，再经过一系列的判断，进入
  	// 生成代理
  Object proxy = createProxy(
  		bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
  ~~~

  小总结来一波：我们从整个流程可以看到，AOP的编织是通过定义@Aspect 和 @Around 等注解创建通知器，和我们在XML中编程几乎一样，只是Spring 向我们屏蔽了底层。最后，Spring拿着目标类和通过器去创建代理。

- 目标对象方法调用的过程，applicationContext.getBean("....")此时返回的就是一个代理对象，在执行该代理对象的方法时，代理类内部回调了 CglibAopProxy 的内部类 DynamicAdvisedInterceptor的intercept 方法。

##### spring扩展接口

- FactroyBean，上文已经讲过，spring使用此接口构建AOP，ProxyFactoryBean就是实现了该接口（XML的方式）。
- BeanPostProcess 在每个bean初始化成前后做操作。spring用注解的方式构建AOP就是用了扩展BeanPostProcess的方式。我们可以自定义一个实现类，实现上述两个方法，项目中所有的Bean在初始化的时候都调用该方法。因此，我们在以后的开发中就可以做一些自定义的事情。
- InstantiationAwareBeanPostProcessor 在Bean实例化前后做一些操作。继承BeanPostProcess，新增三个方法，分别是 postProcessBeforeInstantiation （实例化之前），postProcessAfterInstantiation （实例化之后），postProcessPropertyValues （在处理Bean属性之前），开发者可以在这三个方法中添加自定义逻辑，比如AOP。
- BeanNameAware、ApplicationContextAware 和 BeanFactoryAware 针对bean工厂，可以获取上下文（BeanFactory、ApplicationContext），可以获取当前bean的id。
- InitialingBean 在属性设置完毕后做一些自定义操作。 DisposableBean 在关闭容器前做一些操作。

**总结：**我们了解了 Spring 留给我们的扩展接口，以提高我们使用 Spring 的水平，在以后的业务中，也就可以基于 Spring 做一些除了简单的注入这种基本功能之外的功能。同时，我们也发现，Spring 的扩展性非常的高，符合设计模式中的开闭原则，对修改关闭，对扩展开放，实现这些的基础就是 Spring 的 IOC，IOC 可以说是 Spring 的核心， 在 IOC 的过程中，对预定义的接口做了很多的预留工作。这让其他框架与 Spring 的组合变得更加的简单，我们在以后的开发工作中也可以借鉴 Spring 的思想，让程序更加的优美。

##### springboot事务

- BookShopService#purchase方法上面已经@Transactional，BookShopService实例对象其实已经被Cglib 代理了，那么他肯定会走 DynamicAdvisedInterceptor 的 intercept 方法，该方法最重要的事情就是执行通知器或者拦截器的方法，那么该代理有拦截器吗？显然是有的，debug发现该拦截器是TransactionInterceptor，这个类实现了Advice MethodInterceptor的invoke方法，我们来看看TransactionInterceptor的invoke方法做了啥？

- 第一步获取事务属性，通过注解解析器解析Method 对象是否含有注解@Transactional

- @Transactional 注解解析器 SpringTransactionAnnotationParser如何解析注解的呢，返回了一个 RuleBasedTransactionAttribute 对象。有了事务属性，再获取事务管理器。也就是 determineTransactionManager 方法

- 第二步获取事务管理器，final PlatformTransactionManager tm = determineTransactionManager(txAttr); DataSourceTransactionManager里面方法：getDataSouce 、doCommit、 doRollback.........

  ~~~java
   protected void doCommit(DefaultTransactionStatus status) {
  		DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
  		Connection con = txObject.getConnectionHolder().getConnection();
  		if (status.isDebug()) {
  			logger.debug("Committing JDBC transaction on Connection [" + con + "]");
  		}
  		try {
  			con.commit();
  		}
  		catch (SQLException ex) {
  			throw new TransactionSystemException("Could not commit JDBC transaction", ex);
  		}
  	  }
  	  // 获得事务管理器后，继续往下走。。。底层也是调用 JDBC 的 Connection 的 rollback 方法。
  	  try {
  				// 该方法会执行下一个通知器的拦截方法（如果有的话），最后执行目标方法，
  				retVal = invocation.proceedWithInvocation();
  			}
  			catch (Throwable ex) {
  				// 目标方法发生异常，被try住
  				completeTransactionAfterThrowing(txInfo, ex);
  				throw ex;
  			}
  			finally {
  				cleanupTransactionInfo(txInfo);
  			}
  			// 目标方法成功执行后，事务提交
  			commitTransactionAfterReturning(txInfo);
  		// TransactionInterceptor 的 completeTransactionAfterThrowing 方法（事务如何回滚）
  		// 该事务对象是否和该异常匹配，如果匹配，则回滚，否则提交。
  		if (txInfo.transactionAttribute.rollbackOn(ex)) {
  				try {
  					txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
  				}
  				catch (TransactionSystemException ex2) {
  					。。
  				}
  				。。。
  			}
  			else {
  				// We don't roll back on this exception.
  				// Will still roll back if TransactionStatus.isRollbackOnly() is true.
  				try {
  					txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
  				}
  				catch (TransactionSystemException ex2) {
  					。。
  				}
  				。。
  			}
  ~~~

- TransactionInterceptor 的 commitTransactionAfterReturning 方法（事务如何提交）底层也是调用 JDBC 的 Connection 的 commit 方法。

- 事务运行之前做了哪些工作？事务管理器从何而来？ TransactionAttributeSource 属性何时生成？AnnotationTransactionAttributeSource 构造什么时候调用？在Spring 中，有一个现成的类，ProxyTransactionManagementConfiguration

- mybatis 和 Spring 的事务管理权力之争，判断Spirng 持有的 SqlSession 和 Mybatis 持有的是否是同一个，如果是，则交给Spring，否则，Mybatis 自己处理。可以说很合理。

**总结：**整个事务其实是建立在AOP的基础之上，其核心类就是 TransactionInterceptor，该类就是 invokeWithinTransaction 方法是就事务处理的核心方法，其中封装了我们创建的 DataSourceTransactionManager 对象，该对象就是执行回滚或者提交的执行单位 其实，TransactionInterceptor 和我们平时标注 @Aspect 注解的类的作用相同，就是拦截指定的方法，而在TransactionInterceptor 中是通过是否标有事务注解来决定的。如果一个类中任意方法含有事务注解，那么这个方法就会被代理。而Mybatis 的事务和Spring 的事务协作 则根据他们的SqlSession 是否是同一个SqlSession 来决定的，如果是同一个，则交给Spring，如果不是，Mybatis 则自己处理。

- propagation:事务传播行为：
  PROPAGATION_REQUIRED：如果当前存在事务，那么加入事务，不存在事务，那么新建一个事务。这个是默认值。
  PROPAGATION_REQUIRES_NEW：创建一个新的事务，如果当前事务存在，则把当前事务挂起。
  PROPAGATION_SUPPORTS：如果当前存在事务，则加入事务，如果当前没有事务，则以非事务的方式运行。
  PROPAGATION_NOT_SUPPORTED：以非事务的方式运行，如果当前存在事务，则把当前事务挂起。
  PROPAGATION_NEVER：以非事务的方式运行，如果当前存在事务，则抛出异常。
  PROPAGATION_MANDATORY（强制的）：如果当前存在事务，那么加入事务，如果当前没有事务，则抛出异常。
  PROPAGATION_NESTED：如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于TransactionDefinition.PROPAGATION_REQUIRED。
- isolation:事务隔离级别，默认为数据库的隔离级别。读未提交、读已提交（一般的数据默认值）、可重复读、序列化。
- read-only:只读，默认值为false，代码只读，不修改数据的情况下，设置为true可以帮助数据库引擎优化事务。
- rollback-for：发生哪些异常回滚，默认是针对unchecked exception回滚。也就是默认对RuntimeException()异常或是其子类进行事务回滚；如果事务被try catch了 则不进行回滚 try catch(){throw ...}异常可以选择往外抛就会回滚，检查型异常（Checked Exception）与非检查型异常（Unchecked Exception）：前者编译器通不过，必须try catch，不然编译失败   后者编译器不会去检查，可以不用try catch。
- no-rollback-for:发生哪些异常不回滚
- Timeout: 事务超时，允许一个事务所执行的时间，超过这个时间，事务没有执行完，则自动回滚事务。默认设置为底层事务系统的超时值，如果底层数据库事务系统没有设置超时值，那么就是none，没有超时限制。

### IOC

**概念：**为了解决对象之间的耦合度过高的问题！Spring IOC 负责创建对象，管理对象（通过依赖注入（DI），装配对象，配置对象，并且管理这些对象的整个生命周期。我们可以把IOC容器的工作模式看做是工厂模式的升华，可以把IOC容器看作是一个工厂，这个工厂里要生产的对象都在配置文件中给出定义，然后利用编程语言的的反射编程，根据配置文件中给出的类名生成相应的对象。从实现来看，IOC是把以前在工厂方法里写死的对象生成代码，改变为由配置文件来定义，也就是把工厂和对象生成这两者独立分隔开来，目的就是提高灵活性和可维护性。原理：底层就是java的反射。给定一个字符串能创建一个实例，利用set方法对实例的依赖进行注入。

**IOC概念和DI概念：**调用者不负责被调用者实例的创建，而是由spring容器来创建，控制权交给了外部容器，控制权发生了反转，因此叫做控制反转。spring容器负责被调用者实例的创建，实例创建后又负责将实例注入调用者，因此成为依赖注入（*构造器注入、Setter方法注入、接口注入*）。

**BeanFactory与ApplicationContext：**BeanFactory（IOC的核心接口）是spring容器的顶层接口，提供了容器最基本的功能（getbean、bean的生命周期的管理包括初始化、销毁）。而ApplicationContext添加了更多高级的功能（支持国际化、统一的资源文件读取方式、针对Web应用的WebApplicationContext等等），另外ApplicationContext容器初始化完成后，容器中所有singleton bean 也被实例化了，getbean的时候，速度会更快。

**BeanDefinition：**IOC的基本数据结构。我们知道，每个bean都有自己的信息，各个属性，类名，类型，是否单例，这些都是bena的信息，spring中如何管理bean的信息呢？对，就是 BeanDefinition， Spring通过定义 BeanDefinition 来管理基于Spring的应用中的各种对象以及他们直接的相互依赖关系。BeanDefinition 抽象了我们对 Bean的定义，是让容器起作用的主要数据类型。对 IOC 容器来说，BeanDefinition 就是对依赖反转模式中管理的对象依赖关系的数据抽象。也是容器实现依赖反转功能的核心数据结构。

**beanfactory和factorybean的区别：**前者是一个工厂（容器），是管理bean的一个工厂。 一般情况下，spring通过反射机制利用<bean>的class属性指定实例化bean，在某些情况下，实例化bean过程比较复杂，如果按照传统的方式，则需要在 <bean> 中提供大量的配置信息。 这个时候就可以采用编码的方式，spring提交了一个接口FactoryBean<T>，用户可以通过实现该接口定制实例化 Bean 的逻辑。 它就是一个bean，也是被beanfactory容器所管理，与普通bean不一样的地方在于，根据bean的id获取到的bean并不是factorybean本身，而是FactoryBean的getObject()返回的对象。

**依赖注入的集中方式：**set注入（必须要有set方法）、构造器注入（利用构造方法）、工厂方法注入、注解的方式注入（byName、byType）。

**spring注解：**

> Autowired是自动注入，自动从spring的上下文找到合适的bean来注入，byType
> Resource用来指定名称注入
> Qualifier和Autowired配合使用，指定bean的名称
>  Service，Controller，Repository分别标记类是Service层类，Controller层类，数据存储层的类，spring扫描注解配置时，会标记这些类要生成bean。
> Component是一种泛指，标记类是组件，spring扫描注解配置时，会标记这些类要生成bean。
> @Autowired//默认按type注入
>  @Qualifier("cusInfoService")//一般作为@Autowired()的修饰用 场景是可能存在多个实例
> @Resource(name="cusInfoService")//默认按name注入，可以通过name和type属性进行选择性注入
>  @Autowired(required=true)// 一定要找到匹配的Bean，否则抛异常。 默认值就是true 
>   @Scope("prototype")// 该注解可以使每次使用bean时都实例化一个新的对象 另外还有singleton,prototype,session,request,session,globalSession

**IOC过程：**初始化和注入依赖。[详尽版本](https://javadoop.com/post/spring-ioc)

[简洁版本，推荐](https://yikun.github.io/2015/05/29/Spring-IOC%E6%A0%B8%E5%BF%83%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0/)

![img](http://ww1.sinaimg.cn/large/006tNc79ly1g61mq9ugspj30jp01kjr8.jpg)

![img](http://ww1.sinaimg.cn/large/006tNc79ly1g61mqasix4j30i901k0sl.jpg)

- 资源(Resource)定位
- BeanDefinition 的载入（从xml中读取配置文件，解析成BeanDefinition）和 BeanFactory 的构造
- 向 IOC 容器(BeanFactory)注册 BeanDefinition. map.put(beanname,beandefinition)
- 根据 lazy-init 属性初始化 Bean 实例和依赖注入.依赖注入的前提是容器中的BeanDefinition数据已经建立好
- 补充：lazy-init 默认为false立即加载，设置为true延迟加载，第一次向容器通过getBean索取bean时实例化的。ApplicationContext实现的默认行为就是在启动服务器时将所有singleton bean提前进行实例化(也就是依赖注入)，当真正的请求bean的时候，其实只是从缓存中读取而已。依赖注入发生在容器执行refresh的过程中，即IoC容器初始化的过程中。

**容器初始化后，依赖注入发生的时间：**

- 用户第一次通过getBean方法向IoC容索要Bean时，IoC容器触发依赖注入
- 当用户在Bean定义资源中为<Bean>元素配置了lazy-init属性，即让容器在解析注册Bean定义时进行预实例化，触发依赖注入

**依赖注入其实包括两个主要过程：**生产Bean所包含的Java对象、Bean对象生成之后,把这些Bean对象的依赖关系设置好

[自己实现一个IOC容器：](https://blog.csdn.net/qq_38182963/article/details/78724305)

- 设计组件：BeanFactory 容器，BeanDefinition Bean的基本数据结构，当然还需要加载Bean的资源加载器。大概最后最重要的就是这几个组件。容器用来存放初始化好的Bean，BeanDefinition就是Bean的基本数据结构，比如Bean的名称，Bean的属性 PropertyValue，Bean的方法，是否延迟加载，依赖关系等。资源加载器就简单了，就是一个读取XML配置文件的类，读取每个标签并解析。
- 设计接口：BeanFactory两个方法getBean和registerBeanDefinition。BeanDefinition实体类，字段bean beanClass bean的属性 bean 的类全限定名称。BeanDefinitionReader接口，从XML中读取配置文件， 解析成 BeanDefinition，AbstractBeanDefinitionReader实现BeanDefinitionReader，XmlBeanDefinitionReader继承AbstractBeanDefinitionReader创建一个资源管理器ResourceLoader，readxml，把读取到的内容封装到beanDefinition，最后注册到容器中（hashmap,key是beanname,value是beanDefinition）。

### 常见题目

- 你怎样定义类的作用域?

  当定义一个<bean> 在Spring里，我们还能给这个bean声明一个作用域。它可以通过bean 定义中的scope属性来定义。如，当Spring要在需要的时候每次生产一个新的bean实例，bean的scope属性被指定为prototype。
  另一方面，一个bean每次使用的时候必须返回同一个实例，这个bean的scope 属性 必须设为 singleton。

- 解释Spring支持的几种bean的作用域?

  > singleton : bean在每个Spring ioc 容器中只有一个实例。缺省的Spring bean 的作用域是Singleton。
  >
  > prototype：一个bean的定义可以有多个实例。
  >
  > request：每次http请求都会创建一个bean，该作用域仅在基于web的Spring ApplicationContext情形下有效。
  >
  > session：在一个HTTP Session中，一个bean定义对应一个实例。该作用域仅在基于web的Spring ApplicationContext情形下有效。
  >
  > global-session：在一个全局的HTTP Session中，一个bean定义对应一个实例。该作用域仅在基于web的Spring ApplicationContext情形下有效。
  >
  > 

  

- Spring框架中的单例bean是线程安全的吗?

  不，Spring框架中的单例bean不是线程安全的。大部分的Spring bean并没有可变的状态(比如Service类和DAO类)，所以在某种程度上说Spring的单例bean是线程安全的。开发者自行保证线程安全。最浅显的解决办法就是将多态bean的作用域由“singleton”变更为“prototype”。只有无状态的Bean才可以在多线程环境下共享，Spring使用ThreadLocal解决线程安全问题。

- 解释Spring框架中bean的生命周期?

  Spring容器 从XML 文件中读取bean的定义，并实例化bean。
  Spring根据bean的定义填充所有的属性。
  如果bean实现了BeanNameAware 接口，Spring 传递bean 的ID 到 setBeanName方法。
  如果Bean 实现了 BeanFactoryAware 接口， Spring传递beanfactory 给setBeanFactory 方法。
  如果有任何与bean相关联的BeanPostProcessors，Spring会在postProcesserBeforeInitialization()方法内调用它们。
  如果bean实现IntializingBean了，调用它的afterPropertySet方法，如果bean声明了初始化方法，调用此初始化方法。
  如果有BeanPostProcessors 和bean 关联，这些bean的postProcessAfterInitialization() 方法将被调用。
  如果bean实现了 DisposableBean ，它将调用destroy()方法。

  ~~~java
  before 实例化 testController InstantiationAwareBeanPostProcessor
  after 实例化 testController  InstantiationAwareBeanPostProcessor
  before 初始化 testController BeanPostProcessor
  afterpropertiesset...       InitializingBean
  after 初始化 testController  BeanPostProcessor
  destory...                  DisposableBean
  ~~~

  @PostConstruct：在bean初始化后调用，bean的标签是init-method，可以自定义初始化方法。afterPropertiesSet方法也是可以实现，但是必须要实现InitializingBean，完成该bean的所有赋值后，会进行调用afterPropertiesSet方法。

  @PreDestroy：销毁bean的时候会进行调用。

  @Required注解：用在set方法上，一旦用了这个注解，那么容器在初始化bean的时候必须要进行set，也就是说必须对这个值进行依赖注入。

- Spring 框架中都用到了哪些设计模式？

  > 代理模式—在AOP和remoting中被用的比较多。
  > 单例模式—在spring配置文件中定义的bean默认为单例模式。
  > 模板方法—用来解决代码重复的问题。比如. RestTemplate, JmsTemplate, JpaTemplate。
  > 前端控制器—Spring提供了DispatcherServlet来对请求进行分发。
  > 视图帮助(View Helper )—Spring提供了一系列的JSP标签，高效宏来辅助将分散的代码整合在视图里。
  > 依赖注入—贯穿于BeanFactory / ApplicationContext接口的核心理念。
  > 工厂模式—BeanFactory用来创建对象的实例。

- IoC 和 DI的区别？

  IoC 控制反转，指将对象的创建权，反转到Spring容器 ， DI 依赖注入，指Spring创建对象的过程中，将对象依赖属性通过配置进行注入，让相互协作的几个类保持松散耦合。

  

### springboot

> 添加start依赖：依赖更加简洁，依赖版本互相兼容；
> 自动配置；减少spring配置的数量。利用了spring4对条件化配置的支持。
> groovy编写程序，并且使用springboot cli运行程序；
> actuator；maven中添加spring-boot-actuator;查看管理端点：比方说列出运行所配置的bean，查看自动配置情况，列出应用的线程，展现当前应用的健康状况。        