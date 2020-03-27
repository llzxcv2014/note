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

## 参考

[Spring 启动流程refresh()源码解析之一](https://blog.csdn.net/yangliuhbhd/article/details/80790761)