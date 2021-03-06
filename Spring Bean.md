# Spring Bean

![img](C:%5CUsers%5CAdministrator%5CDesktop%5Cnote%5CSpring%20Bean.assets%5C181453414212066.png)

![img](C:%5CUsers%5CAdministrator%5CDesktop%5Cnote%5CSpring%20Bean.assets%5C181454040628981.png)

## Spring容器的启动流程

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // 刷新前的预处理
        prepareRefresh();

		// 获取BeanFactory；刚创建的默认DefaultListableBeanFactory
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

		// 对BeanFactory进行初始化.
		prepareBeanFactory(beanFactory);

		try {
			// BeanFactory准备工作完成后进行的后置处理工作；
            // 抽象的方法，当前未做处理。子类通过重写这个方法来在BeanFactory创建并预准备完成以后做进一步的设置
			postProcessBeanFactory(beanFactory);
            
              /**************************以上是BeanFactory的创建及预准备工作 ****************/

			 // 5 执行BeanFactoryPostProcessor的方法；
			//BeanFactoryPostProcessor：BeanFactory的后置处理器。在BeanFactory标准初始化之后执行的；
			//他的重要两个接口：BeanFactoryPostProcessor、BeanDefinitionRegistryPostProcessor
			invokeBeanFactoryPostProcessors(beanFactory);

			// 6 注册BeanPostProcessor（Bean的后置处理器）
			registerBeanPostProcessors(beanFactory);

			// 7 initMessageSource();初始化MessageSource组件（做国际化功能；消息绑定，消息解析）；
			initMessageSource();

			//  8 初始化事件派发器
			initApplicationEventMulticaster();

			// 9 子类重写这个方法，在容器刷新的时候可以自定义逻辑；
			onRefresh();

			// 10 给容器中将所有项目里面的ApplicationListener注册进来
			registerListeners();

			// 11.初始化所有剩下的单实例bean；
			finishBeanFactoryInitialization(beanFactory);

			//  12.完成BeanFactory的初始化创建工作；IOC容器就创建完成；
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



## Spring IOC官方文档

springframework.beans包和springframework.context包是IoC功能的基础功能。`BeanFactory`接口提供了管理各种类型对象的配置的功能。`ApplicationContext`是`BeanFactory`的子接口。它添加了如下功能：

* 更容易与AOP特性集成
* i18n的配置
* 事件发布
* 应用层特定的上下问如Web的`WebApplicationContext`



### Spring Bean的循环依赖

循环依赖：

Bean A → Bean B → Bean C → Bean D → Bean E → Bean A

Spring加载Bean时会按照关联关系加载bean，如Bean A → Bean B → Bean C ：

Spring容器会先创建BeanC然后时BeanB再然后是BeanA。但如果Bean之间存在循环依赖，Spring则不知道先加载哪个，会抛出`BeanCurrentlyInCreationExc zfangfseption`异常。

当使用构造方法注入时，也会遇到这种情况，使用其他类型注入则可能不会遇到这种情况。

#### Spring中的三级缓存

spring单例对象的创建步骤：

* createBeanInstance：第一步就是通过构造方法去进行实例化对象，并没有把对象的属性也给注入进去
* 注入实例对象的属性，也就是从这步对spring xml中指定的property进行populate
* 最后一步其实是初始化XML中的init方法，来进行最终完成实例对象的创建。但是AfterPropertiesSet方法会发生循环依赖的步骤集中在第一步和第二步。

```java
// singletonObjects指单例对象的cache （一级缓存）
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);

// singletonFactories指单例对象工厂的cache（三级缓存）
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16);

// earlySingletonObjects指提前曝光的单例对象的cache（二级缓存）
private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);

```

调用缓存的步骤：

* Spring调用getSingleton（String beanName, boolean allowEarlyReference）来获取想要的单例对象
* 第一步会先进行通过singletonObjects这个一级缓存的集合中去获取对象，如果没有获取成功的话并且使用isSingletonCurrentlyInCreation（beanName）去判断对应的单例对象是否正在创建中（也就是说当单例对象没有被初始化完全，走到初始化的第一步或者第二的时候），如果是正在创建中的话，会继续走到下一步
* 从earlySingletonObjects中继续获取这个对象，如果又没有获取到这个单例对象的话，并且通过参数传进来的allowEarlyReference标志，看是不是允许singletonFactories（三级缓存集合）去拿到该实例对象，如果allowEarlyReference为Ture的话，那么继续下一步
* 此时上一步中并没有从earlySingletonObjects二级缓存集合中拿到想要的实例对象，最后只能从三级缓存singletonFactories （单例工厂集合中）去获取实例对象，
* 然后把获取的对象通过Put（beanName, singletonObject）放到earlySingletonObjects（二级缓存中），然后在再从singletonFactories（三级缓存）对象中的集合中把该对象给remove（beanName）出去。

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
 从一级缓存获取
   Object singletonObject = this.singletonObjects.get(beanName);
   if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
      synchronized (this.singletonObjects) {
       从二级缓存获取
         singletonObject = this.earlySingletonObjects.get(beanName);
         if (singletonObject == null && allowEarlyReference) {
          从三级缓存获取
            ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
            if (singletonFactory != null) {
               singletonObject = singletonFactory.getObject();
               this.earlySingletonObjects.put(beanName, singletonObject);
               this.singletonFactories.remove(beanName);
            }
         }
      }
   }
   return (singletonObject != NULL_OBJECT ? singletonObject : null);}

```



#### 解决方案

1. 重新设计，去掉之间的依赖

2. 使用`@Lazy`注解，就是使用延时加载，在需要用到时才被加载。

3. 使用setter注入，Spring官方推荐这种方案。通过这种方式创建Bean的时候，此时的依赖还未被注入，只有需要时才会被注入。

4. 使用`@PostConstruct`：通过属性注入，使用`@Autowired`，并使用`@Postconstruct`标注另一个方法。在方法里设置对其他的依赖。

   ```java
   @Component
   public class CircularDependencyA {
    
       @Autowired
       private CircularDependencyB circB;
    
       @PostConstruct
       public void init() {
           circB.setCircA(this);
       }
    
       public CircularDependencyB getCircB() {
           return circB;
       }
   }
   
   @Component
   public class CircularDependencyB {
    
       private CircularDependencyA circA;
        
       private String message = "Hi!";
    
       public void setCircA(CircularDependencyA circA) {
           this.circA = circA;
       }
        
       public String getMessage() {
           return message;
       }
   }
   ```

   5. 实现`ApplicationContextAware`和`InitializingBean`使用`InitializingBean`接口表明属性设置完成后做一些后置的处理操作。

   ```java
   @Component
   public class CircularDependencyA implements ApplicationContextAware, InitializingBean {
    
       private CircularDependencyB circB;
    
       private ApplicationContext context;
    
       public CircularDependencyB getCircB() {
           return circB;
       }
    
       @Override
       public void afterPropertiesSet() throws Exception {
           circB = context.getBean(CircularDependencyB.class);
       }
    
       @Override
       public void setApplicationContext(final ApplicationContext ctx) throws BeansException {
           context = ctx;
       }
   }
   
   public class CircularDependencyB {
    
       private CircularDependencyA circA;
    
       private String message = "Hi!";
    
       @Autowired
       public void setCircA(CircularDependencyA circA) {
           this.circA = circA;
       }
    
       public String getMessage() {
           return message;
       }
   }
   ```

   

## 参考

[Spring 启动流程refresh()源码解析之一](https://blog.csdn.net/yangliuhbhd/article/details/80790761)

[spring中的循环依赖解决方案](https://www.jianshu.com/p/b65c57f4d45d)

[Circular Dependencies in Spring](https://www.baeldung.com/circular-dependencies-in-spring)

[Spring常问的------真实大厂面试题汇总（含答案）](https://blog.csdn.net/qq_36520235/article/details/88257749)