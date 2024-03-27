# Spring



## Spring  如何解决循环依赖



**循环依赖**



所谓的循环依赖，就是两个或则两个以上的`bean`互相依赖对方，最终形成`闭环`。比如“A对象依赖B对象，而B对象也依赖A对象”，或者“A对象依赖B对象，B对象依赖C对象，C对象依赖A对象”；类似以下代码：

```java
public class A {
    private B b;
}

public class B {
    private A a;
}
```



**Spring如何解决循环依赖**



| 缓存                  | 说明                                                         |
| --------------------- | ------------------------------------------------------------ |
| singletonObjects      | 第一级缓存，存放可用的`成品Bean`。                           |
| earlySingletonObjects | 第二级缓存，存放`半成品的Bean`，`半成品的Bean`是已创建对象，但是未注入属性和初始化。用以解决循环依赖。 |
| singletonFactories    | 第三级缓存，存的是`Bean工厂对象`，用来生成`半成品的Bean`并放入到二级缓存中。用以解决循环依赖。 |

Spring解决循环依赖的核心思想在于提前曝光：

1. 通过构建函数创建A对象（A对象是半成品，还没注入属性和初始化 bean 对象）。
2. Spring 在创建 bean 的时候并不是等它完全完成，而是在创建过程中将创建中的 bean 的 **ObjectFactory** 提前曝光（即加入到 `singletonFactories` 缓存中（`三级缓存`））
3. A对象需要注入B对象，发现缓存里还没有B对象
4. 通过构建函数创建B对象（B对象是半成品，还没注入属性和初始化 bean 对象）。
5. B对象需要注入A对象，在被注入时通过`ObjectFactory.getObject`方式取到半成品对象A，并将生成的对象放入到第`二级缓存`
6. B对象继续注入其他属性和初始化，之后将`完成品B对象`放入`完成品缓存(一级缓存)`。
7. A对象继续注入属性，从`完成品缓存`中取到`完成品B对象`并注入。
8. A对象继续注入其他属性和初始化，之后将`完成品A对象`放入`完成品缓存（一级缓存）`。





## 		SpringAOP

1. AnnotationAwareAspectJAutoProxyCreator#postProcessAfterInitialization

   ```java
   /**
   * 在初始化之后执行
   * @param bean     bean
   * @param beanName beanName
   * @return 处理的bean
   */
   @Override
   public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
      if (bean != null) {
         Object cacheKey = getCacheKey(bean.getClass(), beanName);//构建缓存的Key
         if (this.earlyProxyReferences.remove(cacheKey) != bean) {
            return wrapIfNecessary(bean, beanName, cacheKey);//如果适合被代理, 则需要封装指定的bean
         }
      }
      return bean;
   }
   ```

   

2. 为目标 bean 查找合适的通知器

3. 获取所有对应的bean的增强器后，便可以进行代理的创建，Spring使用了JDKProxy和CglibProxy两种方式的代理。(下面以JDKProxy为例)

4. 我们都知道JDK的动态代理的关键是创建自定义InvocationHandler，而InvocationHandler中包含了需要覆盖的函数getProxy，并且这个函数也一定会有一个invoke函数，并且JdkDynamicAopProxy会把AOP的核心逻辑写在其中

   ```java
   **
   * JDK动态代理执行器
   */
   @Override
   @Nullable
   public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
   	//获取当前方法的拦截器链
         List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
   
         if (chain.isEmpty()) {//没有任何拦截器链
            //如果没有发现任何拦截器那么直接调用切点方法
            Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
            retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
         } else {//有拦截器链
            //将拦截器封装在ReflectiveMethodInvocation
            MethodInvocation invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
            //执行拦截器链
            retVal = invocation.proceed();
         }
         ...
   }
   ```

   上面的函数中最主要的工作就是创建了一个拦截器链，并使用ReflectiveMethodInvocation类进行了链的封装，而在ReflectiveMethodInvocation类的proceed方法中实现了拦截器的逐一调用。









## Spring 的延迟依赖查找



### ObjectFactory



ObjectFactory（或 ObjectProvider） 可关联某一类型的 Bean，仅提供一个 getObject() 方法用于返回目标 Bean 对象，ObjectFactory 对象被依赖注入或依赖查找时并未实时查找到关联类型的目标 Bean 对象，在调用 getObject() 方法才会依赖查找到目标 Bean 对象。

根据 ObjectFactory 的特性，可以说它提供的是**延迟依赖查找**。通过这一特性在 Spring 处理循环依赖（字段注入）的过程中就使用到了 ObjectFactory，在某个 Bean 还没有完全初始化好的时候，会先缓存一个 ObjectFactory 对象（调用其 getObject() 方法可返回当前正在初始化的 Bean 对象），如果初始化的过程中依赖的对象又依赖于当前 Bean，会先通过缓存的 ObjectFactory 对象获取到当前正在初始化的 Bean，这样一来就解决了循环依赖的问题。

注意这里是**延迟依赖查找**而不是延迟初始化，ObjectFactory 无法决定是否延迟初始化，而需要通过配置 Bean 的 lazy 属性来决定这个 Bean 对象是否需要延迟初始化，非延迟初始化的 Bean 在 Spring 应用上下文刷新过程中就会初始化。

提示：如果是 ObjectFactory（或 ObjectProvider）类型的 Bean，在被依赖注入或依赖查找时返回的是 DefaultListableBeanFactory#DependencyObjectProvider 私有内部类，实现了 `ObjectProvider<T>` 接口，关联的类型为 Object。





### ObjectProvider





##  FactoryBean



FactoryBean 关联一个 Bean 对象，提供了一个 getObject() 方法用于返回这个目标 Bean 对象，FactoryBean 对象在被依赖注入或依赖查找时，实际得到的 Bean 就是通过 getObject() 方法获取到的目标类型的 Bean 对象。如果想要获取 FactoryBean 本身这个对象，在 beanName 前面添加 `&` 即可获取。

我们可以通过 FactoryBean 帮助实现复杂的初始化逻辑，例如在 Spring 继集成 MyBatis 的项目中，Mapper 接口没有实现类是如何被注入的？其实 Mapper 接口就是一个 FactoryBean 对象，当你注入该接口时，实际的到的就是其 getObject() 方法返回的一个代理对象，关于数据库的操作都是通过该代理对象来完成。



## ObjectFactory、FactoryBean 和 BeanFactory 的区别



根据其名称可以知道其字面意思分别是：对象工厂，工厂 Bean

ObjectFactory、FactoryBean 和 BeanFactory 均提供依赖查找的能力。

- ObjectFactory 提供的是延迟依赖查找，想要获取某一类型的 Bean，需要调用其 getObject() 方法才能依赖查找到目标 Bean 对象。ObjectFactory 就是一个对象工厂，想要获取该类型的对象，需要调用其 getObject() 方法生产一个对象。
- FactoryBean 不提供延迟性，在被依赖注入或依赖查找时，得到的就是通过 getObject() 方法拿到的实际对象。FactoryBean 关联着某个 Bean，可以说在 Spring 中它就是某个 Bean 对象，无需我们主动去调用 getObject() 方法，如果想要获取 FactoryBean 本身这个对象，在 beanName 前面添加 `&` 即可获取。
- BeanFactory 则是 Spring 底层 IoC 容器，里面保存了所有的单例 Bean，ObjectFactory 和 FactoryBean 自身不具备依赖查找的能力，能力由 BeanFactory 输出。





## Spring 中几种初始化方法的执行顺序



有以下初始化方式：

- Aware 接口：实现了 Spring 提供的相关 XxxAware 接口，例如 BeanNameAware、ApplicationContextAware，其 setXxx 方法会被回调，可以注入相关对象
- @PostConstruct 注解：该注解是 JSR-250 的标准注解，Spring 会调用该注解标注的方法
- InitializingBean 接口：实现了该接口，Spring 会调用其 afterPropertiesSet() 方法
- 自定义初始化方法：通过 init-method 指定的方法会被调用



在 Spring 初始 Bean 的过程中上面的初始化方式的执行顺序如下：

1. `Aware` 接口的回调
2. JSR-250 `@PostConstruct` 标注的方法的调用&后置处理器，before
3. `InitializingBean#afterPropertiesSet` 方法的回调
4. `init-method` 初始化方法的调用
5. 后置处理器，after





## 依赖注入和依赖查找的来源是否相同



否

**依赖查找来源**

| 来源                  | 配置元数据                                   |
| --------------------- | -------------------------------------------- |
| Spring BeanDefinition | <bean id="user" class="org.geekbang...User"> |
|                       | @Bean public User user(){...}                |
|                       | BeanDefinitionBuilder                        |
| 单例对象              | API 实现                                     |



**依赖注入来源**

| 来源                                            | 配置元数据                                   |
| ----------------------------------------------- | -------------------------------------------- |
| Spring BeanDefinition                           | <bean id="user" class="org.geekbang...User"> |
|                                                 | @Bean public User user(){...}                |
|                                                 | BeanDefinitionBuilder                        |
| 单例对象                                        | API 实现                                     |
| 非 Spring 容器管理对象（Resolvable Dependency） |                                              |



## BeanPostProcessor 与 BeanFactoryPostProcessor 的区别



BeanPostProcessor 提供 Spring Bean 初始化前和初始化后的生命周期回调，允许对关心的 Bean 进行扩展，甚至是替换，其相关子类也提供 Spring Bean 生命周期中其他阶段的回调。

BeanFactoryPostProcessor 提供 Spring BeanFactory（底层 IoC 容器）的生命周期的回调，用于扩展 BeanFactory（实际为 ConfigurableListableBeanFactory），BeanFactoryPostProcessor 必须由 Spring ApplicationContext 执行，BeanFactory 无法与其直接交互。



## 简述 Spring 事件机制原理



主要有以下几个角色：

- Spring 事件 - org.springframework.context.ApplicationEvent，实现了 java.util.EventListener 接口
- Spring 事件监听器 - org.springframework.context.ApplicationListener，实现了 java.util.EventObject 类
- Spring 事件发布器 - org.springframework.context.ApplicationEventPublisher
- Spring 事件广播器 - org.springframework.context.event.ApplicationEventMulticaster



Spring 内建事件

- ContextRefreshedEvent：Spring 应用上下文就绪事件
- ContextStartedEvent：Spring 应用上下文启动事件
- ContextStoppedEvent：Spring 应用上下文停止事件
- ContextClosedEvent：Spring 应用上下文关闭事件

Spring 应用上下文就是一个 ApplicationEventPublisher 事件发布器，其内部有一个 ApplicationEventMulticaster 事件广播器（被观察者），里面保存了所有的 ApplicationListener 事件监听器（观察者）。

Spring 应用上下文发布一个事件后会通过 ApplicationEventMulticaster 事件广播器进行广播，能够处理该事件类型的 ApplicationListener 事件监听器则进行处理。



## ~~@Bean 的处理流程是怎样的~~

Spring 应用上下文生命周期，在 BeanDefinition（@Component 注解、XML 配置）的加载完后，会执行所有 BeanDefinitionRegistryPostProcessor 类型的处理器，Spring 内部有一个 ConfigurationClassPostProcessor 处理器，它会对所有的**配置类**进行处理，解析其内部的注解（@PropertySource、@ComponentScan、@Import、@ImportResource、@Bean），其中 @Bean 注解标注的方法会生成对应的 BeanDefinition 对象并注册。详细步骤可查看后续文章：[《死磕 Spring 之 IoC 篇 - @Bean 等注解的实现原理》](https://link.segmentfault.com/?enc=FNy7jP0p5IqJzMmfBH68tA%3D%3D.D%2BSzdyyISPcWmt%2FUXYcLvFCac0V87T7D0ie4HOZnebbKCteB8uhhaSAeREFgMbPjzHBnnGn6Dzl71PDp7n1kmQ%3D%3D)



## @EventListener 的工作原理？



@EventListener 用于标注在方法上面，该方法则可以用来处理 Spring 的相关事件。

Spring 内部有一个处理器 EventListenerMethodProcessor，它实现了 SmartInitializingSingleton 接口，在所有的 Bean（不是抽象、单例模式、不是懒加载方式）初始化后，Spring 会再次遍历所有初始化好的单例 Bean 对象时会执行该处理器对该 Bean 进行处理。在 EventListenerMethodProcessor 中会对标注了 @EventListener 注解的方法进行解析，如果符合条件则**生成**一个 ApplicationListener 事件监听器并注册。



## Spring事务传播性



### 1.1 什么是事务的传播性



事务的传播性一般在事务嵌套的时候使用，比如在事务 A 里面调用了另外一个使用事务的方法，那么这俩个事务是各自作为独立的事务执行提交，还是内层的事务合并到外层的事务一块提交那，这就是事务传播性要确定的问题。

示例

```java
public class BoA {
	public void test(){
		boB.sayHello();
	}
}
```



### **1.2 REQUIRED**



Spring默认的事务传播机制，如果外层有事务则当前事务加入到外层事务，一块提交一回滚，如果外层没有事务则当前开启一个新事务。

BoA 和 boB 都是进行过事务增强后的 bo, 那么在执行 test 的时候会开启一个事务（或者 test 调用方已经存在事务则加入该事务），执行到 sayHello () 时候由于传播性是 PROPAGATION_REQUIRED，所以 sayHello 方法加入到 test 的事务，那么 sayHello 和 test 就会同时提交，同时回滚。值得注意的是如果 test 里面调用 sayHello 时候加了 trycatch 没有把异常跑出去，而 sayHello 方法却抛出了异常，那么整个事务也会回滚，这时候调用 test 的外层会受到”Transaction rolled back because it has been marked as rollback-only 的异常，而把 sayHello 真正的异常吃掉了。

平时我们都是在 bo 里面调用 [数据库] 操作，在 rpc 和 screen 调用 bo, 所以 bo 层不应该 catch 掉异常，而应该抛出来，在 rpc 和 screen 层 catch 异常。



### **1.3 REQUIRES_NEW**



该传播机制是每次新开启一个事务，同时把外层的事务挂起，当前新事务执行完毕后在恢复上层事务的执行。

以上面代码为例，首先进入 test 方法前会开启一个事务，然后调用 sayHello 时候会把 test 的事务挂起，从新开启一个新事务执行 sayHello，执行完毕后恢复 test 的事务。如果 sayHello 抛出来异常则 sayHello 的事务会回滚，那么 test 方法是否回滚那？这个要看情况，如果 test 在调用 sayHello 时候使用了 trycatch 并且异常没有在 catch 中 throw 出来，那么 test 方法不会回滚，这时候 sayHello 是提交和回滚对 test 没有影响。 如果 test 中没有加 trycatch 那么，test 也会回滚。



### **1.4  SUPPORTS**



该传播机制如果外层有事务则加入该事务，如果不存在也不会创建新事务，直接使用非事务方式执行。

下面看下如果 test 隔离级别是 PROPAGATION_REQUIRED，sayHello 传播性是 PROPAGATION_SUPPORTS 的情况。这时候外层 test 会开启一个事务（或者 test 调用方已经存在事务则加入该事务），然后 sayHello 执行时候会加入到 test 的事务和 1.2 类似，同时提交同时回滚。



### **1.5 NOT_SUPPORTED**



该传播机制不支持事务，如果外层存在事务则挂起外层事务 ，然后执行当前逻辑，执行完毕后，恢复外层事务。

同样这里看下如果 test 使用 PROPAGATION_REQUIRED，sayHello 传播性是 PROPAGATION_NOT_SUPPORTED 的情况，首先 test 会开启一个事务（或者 test 调用方已经存在事务则加入该事务），然后 sayHello 执行时候会挂起该事务然后在非事务内做自己的事情，做完后在恢复 test 的事务。 无论 sayHello 是否抛出异常，sayHello 的事务都不会回滚，因为它不在事务范围内，那 test? 这个就和 1.3 一样了，如果 test catch 住 sayHello 的异常没有 throw 出去，那么 test 就不回滚，否者回滚。



### **1.6  NEVER**



该传播机制不支持事务，如果外层存在事务则直接抛出异常。 IllegalTransactionStateException (“Existing transaction found for transaction marked with propagation ‘never’”)



### **1.7 MANDATORY**



该传播机制是说配置了该传播性的方法只能在已经存在事务的方法中被调用，如果在不存在事务的方法中被调用，会抛出异常。 IllegalTransactionStateException (“No existing transaction found for transaction marked with propagation ‘mandatory’”);



### **~~1.8 NESTED~~ **



该传播机制特点是可以保存状态保存点，当事务回滚后会回滚到某一个保存点上，从而避免所有嵌套事务都回滚。

上面代码 test 和 sayHello 都设置为 PROPAGATION_NESTED，如果 sayHello 抛出异常，两者还是都回滚了，因为 sayHello 虽然回滚到了 savePoint 而 savepoint 里面确实包含了 test 的操作，但是 savepoint 后还是会吧异常 throw 给 test，这导致了 test 的回滚。

总结，只有传播性为 PROPAGATION_REQUIRED||PROPAGATION_REQUIRES_NEW||PROPAGATION_NESTED 时候才可能开启一个新事务。



## 对 IoC 的理解



**Inversion of Control（IoC）**是面向对象中的一种编程思想或原则。可以先回到传统方式，当我依赖一个对象，我需要主动去创建它并进行属性赋值，然后我才能去使用这个对象。对于 IoC 这种方式来说，它使得对象或者组件的创建更为透明，你不需要过多地关注细节，如创建对象、属性赋值，这些工作交都由 IoC 容器来完成，已达到解耦的目的。

IoC 控制反转，简单来理解其实就是把获取依赖对象的方式，交由 IoC 容器来实现，由“主动拉取”变为“被动获取”。

实际上，IoC 是为了屏蔽构造细节。例如 new 出来的对象的生命周期中的所有细节对于使用端都是知道的，如果在没有 IoC 容器的前提下，IoC 是没有存在的必要，不过在复杂的系统中，我们的应用更应该关注的是对象的运用，而非它的构造和初始化等细节。



## IoC 和 DI 的区别



DI 依赖注入不完全等同于 IoC，更应该说 DI 依赖注入是 IoC 的一种实现方式或策略。

**依赖查找**和**依赖注入**都是IOC的实现策略。**依赖查找**就是在应用程序里面主动调用 IoC 容器提供的接口去获取对应的 Bean 对象，而**依赖注入**是在 IoC 容器启动或者初始化的时候，通过构造器、字段、setter 方法或者接口等方式注入依赖。**依赖查找**相比于**依赖注入**对于开发者而言更加繁琐，具有一定的代码入侵性，需要借助 IoC 容器提供的接口，所以我们总是强调后者。**依赖注入**在 IoC 容器中的实现也是调用相关的接口获取 Bean 对象，只不过这些工作都是在 IoC 容器启动时由容器帮你实现了，在应用程序中我们通常很少主动去调用接口获取 Bean 对象。





## 什么是 Spring IoC 容器



Spring 框架是一个 IoC 容器的实现，DI 依赖注入是它的实现的一个原则，提供依赖查找和依赖注入两种依赖处理，管理着 Bean 的生命周期。Spring 还提供了 AOP 抽象、事件抽象、事件监听机制、SPI 机制、强大的第三方整合、易测试性等其他特性。



## BeanFactory 和 ApplicationContext 谁才是 Spring IoC 容器



BeanFactory 是 Spring 底层 IoC 容器，ApplicationContext 是 BeanFactory 的子接口，是 BeanFactory 的一个超集，提供 IoC 容器以外更多的功能。ApplicationContext 除了扮演 IoC 容器角色，还提供了这些企业特性：面向切面（AOP）、配置元信息、资源管理、事件机制、国际化、注解、Environment 抽象等。我们一般称 ApplicationContext 是 Spring 应用上下文，BeanFactory 为 Spring 底层 IoC 容器。



## Spring Bean 的生命周期



**生命周期：**

1. Spring Bean 元信息配置阶段，可以通过面向资源（XML或Properties）、面向注解、面向API进行配置

2. Spring Bean 元信息解析阶段，对上一步的配置信息进行解析，解析成BeanDefinition对象，该对象包含定义Bean的所有信息，用于实例化一个Spring Bean

3. Spring Bean 注册阶段，将BeanDefinition配置元信息保存至BeanDefiniitonRegistry的ConcurrentHashMap集合中

4. Spring BeanDefinition合并阶段，定义的Bean可能存在层次性关系，则需要将它们进行合并，存在相同配置则覆盖父属性，最终生成一个RootBeanDefinition对象

5. **Spring Bean 的实例化阶段**，首先的通过类加载器加载出一个 Class 对象，通过这个 Class 对象的构造器创建一个实例对象，构造器注入在此处会完成。在实例化阶段 Spring 提供了实例化前后两个扩展点（InstantiationAwareBeanPostProcessor 的 postProcessBeforeInstantiation、postProcessAfterInstantiation 方法）

6. **Spring Bean 属性赋值阶段**，在 Spring 实例化后，需要对其相关属性进行赋值，注入依赖的对象。首先获取该对象所有属性与属性值的映射，可能已定义，也可能需要注入，在这里都会进行赋值（反射机制）。提示一下，依赖注入的实现通过 CommonAnnotationBeanPostProcessor（@Resource、@PostConstruct、@PreDestroy）和 AutowiredAnnotationBeanPostProcessor（@Autowired、@Value）两个处理器实现的。

7. Spring Bean Aware 接口回调阶段，如果 Spring Bean 是 Spring 提供的 Aware 接口类型（例如 BeanNameAware、ApplicationContextAware），这里会进行接口的回调，注入相关对象（例如 beanName、ApplicationContext）

8. **Spring Bean 初始化阶段**，这里会调用 Spring Bean 配置的初始化方法，执行顺序：@PostConstruct 标注方法、实现 InitializingBean 接口的 afterPropertiesSet() 方法、自定义初始化方法。在初始化阶段 Spring 提供了初始化前后两个扩展点（BeanPostProcessor 的 postProcessBeforeInitialization、postProcessAfterInitialization 方法）

9. Spring Bean 初始化完成阶段，在所有的 Bean（不是抽象、单例模式、不是懒加载方式）初始化后，Spring 会再次遍历所有初始化好的单例 Bean 对象，如果是 SmartInitializingSingleton 类型则调用其 afterSingletonsInstantiated() 方法，这里也属于 Spring 提供的一个扩展点

10. **Spring Bean 销毁阶段**，当 Spring 应用上下文关闭或者你主动销毁某个 Bean 时则进入 Spring Bean 的销毁阶段，执行顺序：@PreDestroy 注解的销毁动作、实现了 DisposableBean 接口的 Bean 的回调、destroy-method 自定义的销毁方法。这里也有一个销毁前阶段，也属于 Spring 提供的一个扩展点，@PreDestroy 就是基于这个实现的

11. **Spring 垃圾收集（GC）**

    

总结：

1. 上面 `1`、`2`、`3` 属于 BeanDefinition 配置元信息阶段，算是 Spring Bean 的前身，想要生成一个 Bean 对象，需要将这个 Bean 的所有信息都定义好；
2. 其中 `4`、`5` 属于实例化阶段，想要生成一个 Java Bean 对象，那么肯定需要根据 Bean 的元信息先实例化一个对象；
3. 接下来的 `6` 属于属性赋值阶段，实例化后的对象还是一个空对象，我们需要根据 Bean 的元信息对该对象的所有属性进行赋值；
4. 后面的 `7`、`8` 、`9` 属于初始化阶段，在 Java Bean 对象生成后，可能需要对这个对象进行相关初始化工作才予以使用；
5. 最后面的 `10`、`11` 属于销毁阶段，当 Spring 应用上下文关闭或者主动销毁某个 Bean 时，可能需要对这个对象进行相关销毁工作，最后等待 JVM 进行回收。



## BeanDefinition 是什么



BeanDefinition 是 Spring Bean 的“前身”，其内部包含了初始化一个 Bean 的所有元信息，在 Spring 初始化一个 Bean 的过程中需要根据该对象生成一个 Bean 对象并进行一系列的初始化工作。



## Spirng 内建的Bean作用域有哪些



| 来源        | 说明                                                    |
| ----------- | ------------------------------------------------------- |
| singleton   | 默认Spring Bean 作用域，一个BeanFactory有且仅有一个实例 |
| prototype   | 原型作用域，每次依赖查找和依赖注入生成新Bean对象        |
| request     | 将 Spring Bean 存储在 ServletRequest 上下文中           |
| session     | 将 Spring Bean 存储在 HttpSession 中                    |
| application | 将 Spring Bean 存储在 ServletContext 中                 |





##  ~~Spring 应用上下文的生命周期~~



Spring 应用上下文就是 ApplicationContext，生命周期主要体现在 org.springframework.context.support.AbstractApplicationContext#refresh() 方法中，大致如下：



1. **Spring 应用上下文启动准备阶段**，设置相关属性，例如启动时间、状态标识、Environment 对象
2. **BeanFactory 初始化阶段**，初始化一个 BeanFactory 对象，加载出 BeanDefinition 们；设置相关组件，例如 ClassLoader 类加载器、表达式语言处理器、属性编辑器，并添加几个 BeanPostProcessor 处理器
3. **BeanFactory 后置处理阶段**，主要是执行 BeanFactoryPostProcessor 和 BeanDefinitionRegistryPostProcessor 的处理，对 BeanFactory 和 BeanDefinitionRegistry 进行后置处理，这里属于 Spring 应用上下文的一个扩展点
4. **BeanFactory 注册 BeanPostProcessor 阶段**，主要初始化 BeanPostProcessor 类型的 Bean（依赖查找），在 Spring Bean 生命周期的许多节点都能见到该类型的处理器
5. **初始化内建 Bean**，初始化当前 Spring 应用上下文的 MessageSource 对象（国际化文案相关）、ApplicationEventMulticaster 事件广播器对象、ThemeSource 对象
6. **Spring 事件监听器注册阶段**，主要获取到所有的 ApplicationListener 事件监听器进行注册，并广播早期事件
7. **BeanFactory 初始化完成阶段**，主要是**初始化**所有还未初始化的 Bean（不是抽象、单例模式、不是懒加载方式）
8. **Spring 应用上下文刷新完成阶段**，清除当前 Spring 应用上下文中的缓存，例如通过 ASM（Java 字节码操作和分析框架）扫描出来的元数据，并发布上下文刷新事件
9. **Spring 应用上下文启动阶段**，需要主动调用 AbstractApplicationContext#start() 方法，会调用所有 Lifecycle 的 start() 方法，最后会发布上下文启动事件
10. **Spring 应用上下文停止阶段**，需要主动调用 AbstractApplicationContext#stop() 方法，会调用所有 Lifecycle 的 stop() 方法，最后会发布上下文停止事件
11. **Spring 应用上下文关闭阶段**，发布当前 Spring 应用上下文关闭事件，销毁所有的单例 Bean，关闭底层 BeanFactory 容器；注意这里会有一个钩子函数（Spring 向 JVM 注册的一个关闭当前 Spring 应用上下文的线程），当 JVM “关闭” 时，会触发这个线程的运行

总结：

- 上面的 `1`、`2`、`3`、`4`、`5`、`6`、`7`、`8` 都属于 Sping 应用上下文的刷新阶段，完成了 Spring 应用上下文一系列的初始化工作；
- `9` 属于 Spring 应用上下文启动阶段，和 Lifecycle 生命周期对象相关，会调用这些对象的 start() 方法，最后发布上下文启动事件；
- `10` 属于 Spring 应用上下文停止阶段，和 Lifecycle 生命周期对象相关，会调用这些对象的 stop() 方法，最后发布上下文停止事件；
- `11` 属于 Spring 应用上下文关闭阶段，发布上下文关闭事件，销毁所有的单例 Bean，关闭底层 BeanFactory 容器。

