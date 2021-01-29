# Spring注解驱动

## IOC

### 组件注册

#### @Configuration

- 标注配置类，@Configuration注解集成了@component注解

#### @Bean

- 结合@Configuration注解使用，可以向容器中导入第三方组件；类上标注@Configuration注解，方法上可以使用@Bean注解，将方法的返回值注入容器中

#### @ComponentScan

- 可以指定value，value是一个String[]，value传入的是包扫描的路径，包内标注了@Controller等注解的类都会注入容器中

#### @Filter

- 配合@ComponentScan注解使用，指定包扫描规则；等同于在xml文件中配置包的扫描规则
- 在@Component注解中可以设置includeFilters和excludeFilters属性，通过@Filter注解设置type属性，type属性属于**FilterType枚举类**的一个对象，且指定classes属性(通过class对象指定过滤规则)
- ![@Filter](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/@Filter.png)

#### @Scope

- 设置组件作用域，单实例或多实例

#### @Lazy

- 设置组件是否懒加载，如果是懒加载，即使该组件是单实例，也不会在容器启动时创建，只有在使用时才会加载

#### @Conditional

- 按照一定的规则注册组件，只有判断为true的才进行注册
- 可以标注在类上，表示判断条件如果不为真，类中的所有组件都不会加载
- value属性是一个Condition数组，可以自定义判别规则，编写一个类继承于**Condition接口**
- ![Condition接口](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/Condition接口.png)
- 实现Condition接口的类重写matches方法，主要利用上context(上下文环境)进行判别

#### @Import

- value属性中传入class数组，容器中就会自动注册数组中的组件，默认id为组件的全类名，如org.jiang.Person

- value属性传入ImportSelector接口的实现类class对象，实现类中重写selectImports方法，方法中的参数importingClassMetadata可以获取到类的注解信息，最后返回一个String数组，数组中的元素为类的全类名，则可以将返回值中的所有类的组件注册到容器中
- ![ImportSelector](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/ImportSelector.png)

- ![ImportSelector实现类](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/ImportSelector实现类.png)

- value属性可以传入ImportBeanDefinitionRegistrar的实现类手动注册bean，重写registerBeanDefinitions方法，主要使用参数registry(也就是bean注册器)
- ![ImportBeanDefinitionRegistrar实现类](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/ImportBeanDefinitionRegistrar实现类.png)

#### BeanFactory接口

- https://www.cnblogs.com/aspirant/p/9082858.html

- 工厂bean，通过编写一个FactoryBean<Person>的实现类并指定泛型，继而重写getObject、getObjectType、isSingleton方法指定工厂创建的bean类型，接着将该工厂注册到容器当中
- 当获取该bean对象时，原本装配的是工厂类型，而实际该bean就是工厂创建的类型
- ![image-20201028153249858](C:\Users\19939\AppData\Roaming\Typora\typora-user-images\image-20201028153249858.png)

- ![image-20201028153405546](C:\Users\19939\AppData\Roaming\Typora\typora-user-images\image-20201028153405546.png)

- spring框架在整合其他框架时(比如mybatis)常使用到FactoryBean工厂对象


### 生命周期

#### 通过@Bean注解指定生命周期

- 使用@Bean注解向容器中放置对象时，可以在该对象的类上添加初始化方法和销毁方法，当容器中的对象初始化和销毁时就会调用该方法
- ![image-20201028165148356](C:\Users\19939\AppData\Roaming\Typora\typora-user-images\image-20201028165148356.png)

- ![image-20201028165201870](C:\Users\19939\AppData\Roaming\Typora\typora-user-images\image-20201028165201870.png)

- 单实例bean
  - 初始化在对象被创建并赋值之后进行调用；销毁在容器关闭的时候也就是调用context.close()方法时
- 多实例bean
  - 初始化在获取对象时创建并进行赋值初始化；即使在容器关闭时也不会进行销毁

#### InitializeBean接口(<u>***重点***</u>)和DisposableBean接口

- 通过创建InitializingBean接口的实现类定义初始化逻辑，该实现类被调用的节点是创建完Bean对象且设置属性之后
- ![image-20201029114052051](C:\Users\19939\AppData\Roaming\Typora\typora-user-images\image-20201029114052051.png)
- 通过创建DisposableBean接口的实现类定义销毁逻辑，该实现类在BeanFactory销毁一个Bean时被调用
- ![image-20201029114112789](C:\Users\19939\AppData\Roaming\Typora\typora-user-images\image-20201029114112789.png)

- InitializingBean接口经典用法

  项目中需要使用配置文件中属性，需要编写一个工具类来获取这些属性

  ![image-20201029115241787](C:\Users\19939\AppData\Roaming\Typora\typora-user-images\image-20201029115241787.png)

  通过@Value注解获取到配置文件中的值，进而赋值给变量；再通过调用重写的afterPropertiesSet方法将变量值赋值给常量

##### BeanPostProcessor 后置处理器  <!--本质上就是拦截并添加逻辑功能-->

- 创建一个类实现BeanPostProcessor接口
- 重写的postProcessBeforeInitialization方法用于对象初始化之后调用，比如@Bean注解定义的初始化方法、@InitializingBean接口重写的afterPropertiesSet方法之前调用
- 重写的postProcessAfterInitialization方法用于对象初始化之后调用，也就是@Bean注解定义的初始化方法、@InitializingBean接口重写的afterPropertiesSet方法之后调用
- 参数中的bean和beanName可以对容器中对象初始化前后添加逻辑
- ![image-20201029120530449](C:\Users\19939\AppData\Roaming\Typora\typora-user-images\image-20201029120530449.png)

- ![image-20201029120552034](C:\Users\19939\AppData\Roaming\Typora\typora-user-images\image-20201029120552034.png)
- 后置处理器使用方式
  - 获取到IOC容器在其他位置使用
    - 创建一个类实现ApplicationContextAware接口，重写setApplicationContext方法，可以在实现类上添加一个变量，并在重写的方法中将IOC容器赋值给该变量，就可以在其他位置使用IOC容器
    - ![image-20201029122345445](C:\Users\19939\AppData\Roaming\Typora\typora-user-images\image-20201029122345445.png)
    - ![image-20201029122518527](C:\Users\19939\AppData\Roaming\Typora\typora-user-images\image-20201029122518527.png)
    - **本质上就是利用了ApplicationContextAwareProcessor(BeanPostProcessor的实现类)**
      - 在Bean对象初始化之前，先判断bean对象是否实现了以下三个接口，如果没有就直接返回bean对象；如果有，就调用invokeAwareInterfaces方法
      - ![image-20201029123130970](C:\Users\19939\AppData\Roaming\Typora\typora-user-images\image-20201029123130970.png)
      - 在invokeAwareInterfaces方法中判断bean类型，进而调用对应的setXXX方法，将IOC容器传入参数中进行使用
      - ![image-20201029123316949](C:\Users\19939\AppData\Roaming\Typora\typora-user-images\image-20201029123316949.png)
  - @Autowired注解使用了后置处理器

### 属性赋值

#### @Value

- 对属性进行赋值，可以使用spEL表达式---#{}、基本字面量、计算表达式、取出配置文件中的值--- ${}(**在运行环境变量中的值**)

#### @PropertySource

- 读取外部配置文件中的k-v键值对保存到运行环境中
- value属性值为字符串数组，代表需要引入的多个外部文件
- ![PropertiesValue](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/PropertiesValue.png)
- 搭配@Value注解的${}取出配置文件中对应key的value

### 自动装配

#### @AutoWired

- 优先使用类型匹配，也就是xxx.class；如果属性匹配多个，再使用注解下定义的变量名进行匹配
- 可以指定required属性为false，指定后即使没有装配上也不会报错
- 可以搭配@Qualifier注解并指定value的值进行使用，value的值就是需要装配的bean的名称

#### @Qualifier

- 结合@AutoWired进行使用，如果容器中存在多个同一种类型的bean，通过对@Qualifier的value属性赋值，指定需要装配哪个对象
- ![Qualifier](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/Qualifier.png)
- 一般按照@AutoWired下面注入的属性名进行装配，但添加了@Qualifier之后，就会优先按照@Qualifier的value属性进行装配

#### 利用Aware接口的子接口将Spring底层一些组件注入到自定义Bean中

- Aware接口的子接口
- ![image-20201029151823730](C:\Users\19939\AppData\Roaming\Typora\typora-user-images\image-20201029151823730.png)
- 编写一个类继承于Aware接口的子接口(不同的子接口作用不同)，将该bean对象注入到容器当中，通过重写子接口的方法可以获取到Spring的底层组件并操作
- ![image-20201029152136653](C:\Users\19939\AppData\Roaming\Typora\typora-user-images\image-20201029152136653.png)
  - 如果实现ServletContextAware接口，就可以获取到ServletContext(Servlet的上下文环境)
- 当继承于Aware子接口的实现类重写了对应的方法之后，Aware子接口**对应的后置处理器**通过一系列的判断最终就调用实现类中重写的方法

#### @Profile

- 可以根据当前指定环境，动态的切换和激活一系列组件的功能
- 当在配置类上向容器中注入组件时，需要使用@Bean注解，不过可以在@Bean注解的基础上添加@Profile注解，并指定value的值来定义这个组件是什么环境下的组件
- 当在配置文件中指定spring.profile时就会加载指定环境的组件
- ![Profile](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/Profile.png)

## AOP

#### @EnableAspectJAutoProxy

![EnableAspectJAutoProxy](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/EnableAspectJAutoProxy-1605664188852.png)

- @EnableAspectJAutoProxy向容器中导入了AspectJAutoProxyRegistrar组件

- ![AspectJAutoProxyRegistrar](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/AspectJAutoProxyRegistrar.png)
  - AspectJAutoProxyRegistrar类继承于ImportBeanDefinitionRegistrar接口，**根据前面@Import注解学习**，ImportBeanDefinitionRegistrar接口实现类可以通过重写registerBeanDefinitions方法向容器中自定义注册bean

  - registerBeanDefinitions方法首先判断是否需要向容器中注入AspectJAnnotationAutoProxyCreator

    - 当执行到AopConfigUtils调用registerOrEscalateApcAsRequired方法时

    - ![image-20201118101844246](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201118101844246.png)

    - 首先判断容器中是否存在org.springframework.aop.config.internalAutoProxyCreator这种组件

      - 如果没有，则向容器中注入一个beanDefinition，通过debug可知获取到的beanDefinition为

      ```java
      org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator
      ```

      - 再将该beanDefinition注册到容器中

      ```java
      registry.registerBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator", beanDefinition);
      ```

- 第一步总结

  - 向容器中注入了AnnotationAwareAspectJAutoProxyCreator组件

#### AnnotationAwareAspectJAutoProxyCreator

- 继承关系

  - ![image-20201118103252881](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201118103252881.png)

  - 针对 AbstractAutoProxyCreator(同时具有后置处理器和设置BeanFactory的功能)
  - ![image-20201118103457992](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201118103457992.png)
    -  AbstractAutoProxyCreator实现了SmartInstantiationAwareBeanPostProcessor和BeanFactoryAware接口
    - SmartInstantiationAwareBeanPostProcessor实现了BeanPostProcessor接口(后置处理器)，可以对bean创建前后进行处理
    - BeanFactoryAware接口通过重写setBeanFactory方法可以设置BeanFactory
    - ![image-20201118103631103](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201118103631103.png)

- setBeanFactory方法实现
  - setBeanFactory方法的具体实现在AbstractAdvisorAutoProxyCreator类上
  - ![image-20201118105102035](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201118105102035.png)

- 后置处理器方法实现
  - 后置处理器具体方法实现在AbstractAutoProxyCreator类上
  - ![image-20201118110121783](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201118110121783.png)