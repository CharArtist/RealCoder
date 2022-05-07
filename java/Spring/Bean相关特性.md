# Bean 相关特性

## bean 生命周期

生命周期即 Spring 中 bean 从创建到销毁的过程，可以简单概括为以下四个步骤：

- 实例化 Instantiation
- 属性赋值 Populate
- 初始化 Initialization
- 销毁 Destruction

上面的这几个过程在 AbstractApplicationContext 的 refresh() 方法的 finishBeanFactoryInitialization(beanFactory) 中可以体现，这里只给出主流程的代码：

```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        instanceWrapper = (BeanWrapper)this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        // 实例化阶段
        instanceWrapper = this.createBeanInstance(beanName, mbd, args);
    }
    ...
    Object exposedObject = bean;

    try {
        // 属性赋值阶段
        this.populateBean(beanName, mbd, instanceWrapper);
        // 初始化阶段
        exposedObject = this.initializeBean(beanName, exposedObject, mbd);
    } catch (Throwable var18) {
        ...
    }
    ...
}
// bean 的销毁是在容器关闭时调用的，可以看 ConfigurableApplicationContext 的 close() 方法
```

## bean 生命周期的扩展点

![image-20220507180133620](../../image/image-20220507180133620.png)

扩展点提供了 Bean 生命周期的定制化功能，可以看一个 demo：

```java
// 要定制的 Bean
@Component
public class BeanAwareTest implements BeanNameAware, InitializingBean, DisposableBean {

    String value = "value-initial";

    @Override
    public void setBeanName(String s) {
        System.out.println("bean name : "+ s);
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("value before : " + value);
        value = "new-value";
        System.out.println("value now : " + value);
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("destroy exec, bye...");
    }
}

// 自定义 BeanPostProcessor
@Configuration
public class PostProcessProcessor implements BeanPostProcessor {

    // 前置处理，可以不覆写，BeanPostProcessor 中用 @Nullable 注解了
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        if(beanName.equals("beanAwareTest")) {
            System.out.println("postProcessBeforeInitialization : " + bean);
        }
        return bean;
    }

    // 后置处理，可以不覆写，BeanPostProcessor 中用 @Nullable 注解了
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if(beanName.equals("beanAwareTest")) {
            System.out.println("postProcessAfterInitialization : " + bean );
        }
        return bean;
    }
}

// 主应用类
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        ConfigurableApplicationContext context = SpringApplication.run(App.class, args);
        DefaultListableBeanFactory beanFactory = (DefaultListableBeanFactory) context.getBeanFactory();
        if (beanFactory.containsBean("beanAwareTest")) {
            beanFactory.destroySingleton("beanAwareTest"); // 销毁定制的 bean
        }
    }
}

----------------------------------------
bean name : beanAwareTest
postProcessBeforeInitialization : realcoder.configuration.BeanAwareTest@4052274f
value before : value-initial
value now : new-value
postProcessAfterInitialization : realcoder.configuration.BeanAwareTest@4052274f
destroy exec, bye...
```

## 解决循环依赖

