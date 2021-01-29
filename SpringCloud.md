SpringCloud

## 1 概述

### 1.1 架构图

![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16d1624a59de8e36)

- 网关gateway、配置中心config、断路器breaker、服务注册registery、链路追踪、服务发现、一次性令牌，全局锁定，领导选举，分布式会话，集群状态

### 1.2 版本关系

![image-20201124112523103](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201124112523103.png)

### 1.3 编码问题

- 项目的yaml文件中如果增加了注释，一定要把file encoding更改为utf-8，默认为系统编码gbk

  ![image-20201124123955456](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201124123955456.png)

## 2 Eureka(服务注册与发现)

### 2.1 概述

- 服务治理
  
- 在传统的RPC远程调用框架中，管理每个服务与服务之间依赖关系比较复杂，管理比较复杂，需要使用服务治理管理服务之间依赖关系，可以实现服务调用、负载均衡、容错和服务注册和发现
  
- 服务注册发现

  - 服务注册和发现中存在一个注册中心，当服务器启动时就会把当前自己服务器的信息比如服务地址通讯地址等以别名方式注册到注册中心上，而另一方(作为消费者)以改别名去注册中心上获取到实际的服务通信地址，然后再实现本地RPC调用

- Euerka架构和Dubbo架构

  ![image-20201124114002432](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201124114002432.png)

### 2.2 单机Euerka

#### 2.2.1 服务端

- 依赖

  - 注意添加的依赖为spring-cloud-starter-netflix-eureka-server，表明为客户端

  ```xml
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
  </dependency>
  ```

- 开启Euerka注册中心功能

  ```java
  @EnableEurekaServer
  ```

- 配置文件

  - eureka.instance.hostname指定主机地址的作用为客户端的注册地址(不再通过ip指定)

  - 服务名称spring.application.name的作用是当创建集群时，集群中的所有服务器可以使用同一个服务名用来做负载均衡

    ![image-20201124142849723](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201124142849723.png)

  ```yaml
  server:
    port: 8001 #指定运行端口
  spring:
    application:
      name: eureka-server #指定服务名称
  eureka:
    instance:
      hostname: localhost #指定主机地址
    client:
      fetch-registry: false #指定是否要从注册中心获取服务（注册中心不需要开启）
      register-with-eureka: false #指定是否要注册到注册中心（注册中心不需要开启）
    server:
      enable-self-preservation: false #关闭保护模式
  ```

  - enable-self-preservation当关闭保护模式后，访问服务器主页时就会有对应的提示

    ![image-20201124133059008](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201124133059008.png)

#### 2.2.2 客户端

- 依赖

  - 注意添加的依赖为spring-cloud-starter-netflix-eureka-client，表明为服务端
  - 注意一定要引入spring-boot-starter-web，不然就会**报客户端注册失败，错误码为204**

  ```xml
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>
  
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  ```

- 表明为Eureka客户端

  ```
  @EnableDiscoveryClient
  ```

- 配置文件

  - defaultZone不会提示，需要手动填写，注意yaml格式值前面需要添加空格
  - localhost代表的是**服务端的eureka.instance.hostname**

  ```yaml
  server:
    port: 8101 #运行端口号
  spring:
    application:
      name: eureka-client #服务名称
  eureka:
    client:
      register-with-eureka: true #注册到Eureka的注册中心
      fetch-registry: true #获取注册实例列表
      service-url:
        defaultZone: http://localhost:8001/eureka/ #配置注册中心地址
  ```

### 2.3 集群版

#### 2.3.1 服务端

- 原有的eureka-server服务器添加两个配置文件，分别为application-replica1.yml和application-replica2.yml用来配置两个注册中心

- application-replica1.yml

  - 相对于单机版，首先eureka.client增加了serviceUrl.defaultZone，并且对应的值为另一个注册中心，显然这里用的同样不是ip，而是eureka.instance.hostname
  - 修改了fetch-registry和register-with-eureka为true，因为现在服务端同样需要互相注册

  ```yaml
  server:
    port: 8002
  spring:
    application:
      name: eureka-server
  eureka:
    instance:
      hostname: replica1
    client:
      serviceUrl:
        defaultZone: http://replica2:8003/eureka/ #注册到另一个Eureka注册中心
      fetch-registry: true
      register-with-eureka: true
  ```

  ```yaml
  server:
    port: 8003
  spring:
    application:
      name: eureka-server
  eureka:
    instance:
      hostname: replica2
    client:
      serviceUrl:
        defaultZone: http://replica1:8002/eureka/ #注册到另一个Eureka注册中心
      fetch-registry: true
      register-with-eureka: true
  ```

- 修改host文件

  - 用于对服务名进行ip地址映射

  ```
  127.0.0.1 replica1
  127.0.0.1 replica2
  ```

- 运行集群

  - 注意：不需要再重新添加两个服务，可以通过使用不同的配置文件来启动同一个springboot应用

  - 从原启动配置中复制两个以不同配置文件启动的配置

    ![image-20201124142528030](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201124142528030.png)

    ![image-20201124142615657](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201124142615657.png)

#### 2.3.2 客户端

- 配置文件

  - 同样添加一个额外的配置文件application-replica.yml，让springboot应用以复制配置文件启动

  - defaultZone同时添加了两个注册中心，中间用逗号隔开

    ```yaml
    server:
      port: 8102
    spring:
      application:
        name: eureka-client
    eureka:
      client:
        register-with-eureka: true
        fetch-registry: true
        service-url:
          defaultZone: http://replica1:8002/eureka/,http://replica2:8003/eureka/ #同时注册到两个注册中心
    ```

### 2.4 Eureka注册中心添加认证

#### 2.4.1 新建模块，添加依赖

- eureka-security-server

  - 这里引入的是spring-boot-starter-security，而不是spring-cloud-security

  ```xml
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
  </dependency>
  
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-security</artifactId>
  </dependency>
  ```

#### 2.4.2 配置文件

- spring.security.user来配置登录的用户名和密码，而不是直接从数据库中读取(缺点就是账号密码都是固定的)

```yaml
server:
  port: 8004
spring:
  application:
    name: eureka-security-server
  security: #配置SpringSecurity登录用户名和密码
    user:
      name: macro
      password: 123456
eureka:
  instance:
    hostname: localhost
  client:
    fetch-registry: false
    register-with-eureka: false
```

#### 2.4.3 对SpringSecurity进行配置

- 默认情况下添加SpringSecurity依赖的应用每个请求都需要添加CSRF token才能访问，Eureka客户端注册时并不会添加，所以需要配置/eureka/**路径不需要CSRF token

- ##### CSRF

  - **跨站请求伪造**-- 是一种挟制用户在当前已登录的Web应用程序上执行非本意的操作的攻击方法
  - 攻击者通过一些技术手段欺骗用户的浏览器去访问一个自己曾经认证过的网站并运行一些操作(如发邮件，发消息，甚至财产操作如转账和购买商品)，由于浏览器曾经认证过，所以被访问的网站会认为是真正的用户操作而去运行
  - 利用了web中用户身份验证的一个漏洞：**简单的身份验证只能保证请求发自某个用户的浏览器，却不能保证请求本身是用户自愿发出的**

```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().ignoringAntMatchers("/eureka/**");
        super.configure(http);
    }
}
```

- 重新请求Eureka客户端时需要登录

  ![image-20201124153036253](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201124153036253.png)

#### 2.4.4 eureka-client注册到有登录认证的注册中心

- eureka-client新建一个配置文件application-security.yml并复制一份启动配置以该文件启动

  - defaultZone的格式变更为http://${username}:${password}@${hostname}:${port}/eureka/

  ```yaml
  server:
    port: 8103
  spring:
    application:
      name: eureka-client
  eureka:
    client:
      register-with-eureka: true
      fetch-registry: true
      service-url:
        defaultZone: http://macro:123456@localhost:8004/eureka/
  ```

### 2.5 Eureka常用配置

```yaml
eureka:
  client: #eureka客户端配置
    register-with-eureka: true #是否将自己注册到eureka服务端上去
    fetch-registry: true #是否获取eureka服务端上注册的服务列表
    service-url:
      defaultZone: http://localhost:8001/eureka/ # 指定注册中心地址
    enabled: true # 启用eureka客户端
    registry-fetch-interval-seconds: 30 #定义去eureka服务端获取服务列表的时间间隔
  instance: #eureka客户端实例配置
    lease-renewal-interval-in-seconds: 30 #定义服务多久去注册中心续约
    lease-expiration-duration-in-seconds: 90 #定义服务多久不去续约认为服务失效
    metadata-map:
      zone: jiangsu #所在区域
    hostname: localhost #服务主机名称
    prefer-ip-address: false #是否优先使用ip来作为主机名
  server: #eureka服务端配置
    enable-self-preservation: false #关闭eureka服务端的保护机制
```

## 3 Ribbon(负载均衡的服务调用)

### 3.1 概述

- Ribbon主要给服务间调用及API网关转发提供负载均衡的功能，基于某种规则(简单轮询、随机连接)进行服务间的连接

### 3.2 使用RestTemplate实现服务调用

- RestTemplate是一个HTTP客户端，可以方便的调用HTTP接口，支持GET、POST、PUT、DELETE等方法

#### 3.2.1 Get请求

- 从请求的地址、请求的返回值类型以及请求参数中获取响应的结果

```java
<T> T getForObject(String url, Class<T> responseType, Object... uriVariables);

<T> ResponseEntity<T> getForEntity(String url, Class<T> responseType, Object... uriVariables);
```

#### 3.2.2 Post请求

- 从请求的地址、post提交的表单(封装的对象)、返回值类型以及请求参数中获取响应的结果

```java
<T> T postForObject(String url, @Nullable Object request, Class<T> responseType, Object... uriVariables);

<T> ResponseEntity<T> postForEntity(String url, @Nullable Object request, Class<T> responseType, Object... uriVariables);
```

### 3.3 Ribbon测试

#### 3.3.1 被调用模块(user-service)

- 依赖

  - 注册到Eureka注册中心

  ```xml
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  ```

- 配置文件

  ```yaml
  server:
    port: 8201
  spring:
    application:
      name: user-service
  eureka:
    client:
      register-with-eureka: true
      fetch-registry: true
      service-url:
        defaultZone: http://localhost:8001/eureka/
  ```

- 定义一些接口用于被调用

  ```java
  @RestController
  @RequestMapping("/user")
  public class UserController {
  
      private Logger LOGGER = LoggerFactory.getLogger(this.getClass());
  
      @Autowired
      private UserService userService;
  
      @PostMapping("/create")
      public CommonResult create(@RequestBody User user) {
          userService.create(user);
          return new CommonResult("操作成功", 200);
      }
  
      @GetMapping("/{id}")
      public CommonResult<User> getUser(@PathVariable Long id) {
          User user = userService.getUser(id);
          LOGGER.info("根据id获取用户信息，用户名称为：{}",user.getUsername());
          return new CommonResult<>(user);
      }
  }
  ```

#### 3.3.2 调用模块(ribbon-service)

- 依赖

  - 这里加入了spring-cloud-starter-netflix-ribbon用于调用其他服务

  ```xml
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
  </dependency>
  ```

- 配置文件

  - service-url.user-service是user-service服务在Eureka上的调用路径，便于后边通过@value来取出路径

  ```yaml
  server:
    port: 8301
  spring:
    application:
      name: ribbon-service
  eureka:
    client:
      register-with-eureka: true
      fetch-registry: true
      service-url:
        defaultZone: http://localhost:8001/eureka/
  service-url:
    user-service: http://user-service
  ```

- 给调用模块添加RestTemplate组件通过@LoadBalanced并赋予负载均衡的能力

  ```java
  @Configuration
  public class RibbonConfig {
  
      @Bean
      @LoadBalanced
      public RestTemplate restTemplate() {
          return new RestTemplate();
      }
  }
  ```

- 调用模块只添加controller用于调用user-service模块的service功能(通过restTemplate)

  ```java
  @RestController
  @RequestMapping("/user")
  public class UserRibbonController {
      @Autowired
      private RestTemplate restTemplate;
      @Value("${service-url.user-service}")
      private String userServiceUrl;
  
      @GetMapping("/{id}")
      public CommonResult getUser(@PathVariable Long id) {
          return restTemplate.getForObject(userServiceUrl + "/user/{1}", CommonResult.class, id);
      }
  
      @GetMapping("/getByUsername")
      public CommonResult getByUsername(@RequestParam String username) {
          return restTemplate.getForObject(userServiceUrl + "/user/getByUsername?username={1}", CommonResult.class, username);
      }
  }
  ```

### 3.4 Ribbon常用配置

#### 3.4.1 全局配置

```yaml
ribbon:
  ConnectTimeout: 1000 #服务请求连接超时时间（毫秒）
  ReadTimeout: 3000 #服务请求处理超时时间（毫秒）
  OkToRetryOnAllOperations: true #对超时请求启用重试机制
  MaxAutoRetriesNextServer: 1 #切换重试实例的最大个数
  MaxAutoRetries: 1 # 切换实例后重试最大次数
  NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule #修改负载均衡算法
```

#### 3.4.2 指定服务进行配置

```yaml
user-service:
  ribbon:
    ConnectTimeout: 1000 #服务请求连接超时时间（毫秒）
    ReadTimeout: 3000 #服务请求处理超时时间（毫秒）
    OkToRetryOnAllOperations: true #对超时请求启用重试机制
    MaxAutoRetriesNextServer: 1 #切换重试实例的最大个数
    MaxAutoRetries: 1 # 切换实例后重试最大次数
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule #修改负载均衡算法
```

#### 3.4.3 Ribbon负载均衡策略

- com.netflix.loadbalancer.RandomRule：从提供服务的实例中以随机的方式
- com.netflix.loadbalancer.RoundRobinRule：以线性轮询的方式，就是维护一个计数器，从提供服务的实例中按顺序选取，第一次选第一个，第二次选第二个，以此类推，到最后一个以后再从头来过
- com.netflix.loadbalancer.RetryRule：在RoundRobinRule的基础上添加重试机制，即在指定的重试时间内，反复使用线性轮询策略来选择可用实例
- com.netflix.loadbalancer.WeightedResponseTimeRule：对RoundRobinRule的扩展，响应速度越快的实例选择权重越大，越容易被选择
- com.netflix.loadbalancer.BestAvailableRule：选择并发较小的实例
- com.netflix.loadbalancer.AvailabilityFilteringRule：先过滤掉故障实例，再选择并发较小的实例
- com.netflix.loadbalancer.ZoneAwareLoadBalancer：采用双重过滤，同时过滤不是同一区域的实例和故障实例，选择并发较小的实例

## 4 Hystrix(服务容错保护)

### 4.1 概述

#### 4.1.1 分布式系统面临的问题

- 复杂分布式体系结构中的应用程序有数十个依赖关系，每个依赖关系在某些时候将不可避免地失败

- ##### 服务雪崩
  - 效应
    - 服务雪崩效应是一种因“服务提供者的不可用”（原因）导致“服务调用者不可用”（结果），并将不可用逐渐放大的现象
    - A为服务提供者, B为A的服务调用者, C和D是B的服务调用者. 当A的不可用,引起B的不可用,并将不可用逐渐放大C和D时, 服务雪崩就形成

  ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/20171108163132277)

  

  - 形成原因

  ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/20171108164858509)
  - 应对策略

  ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/20171108170359201)

#### 4.1.2 Hystrix

- Hystrix是一个用于处理分布式系统的延迟和容错的开源库，在分布式系统里，许多依赖不可避免的会调用失败，，比如超时、异常等，Hystrix能够保证在一个依赖出问题的情况下，不会导致整体服务失败，避免出现级联故障，提高分布式系统的弹性
- "断路器"本身是一种开关装置，当某个服务单元发生故障后，通过断路器的故障监控(类似熔断保险丝)，向调用方返回一个符合预期的、可处理的备选响应(FallBack)，而不是长时间的等待或者抛出调用方无法处理的异常

### 4.2 服务熔断、降级

#### 4.2.1 创建模块演示hystrix功能(hystrix-service)

- 依赖

  ```xml
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
  </dependency>
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  ```

- 配置文件

  - service-url.user-service同样是为了获取到user-service的服务地址

  ```yaml
  server:
    port: 8401
  spring:
    application:
      name: hystrix-service
  eureka:
    client:
      register-with-eureka: true
      fetch-registry: true
      service-url:
        defaultZone: http://localhost:8001/eureka/
  service-url:
    user-service: http://user-service
  ```

- 开启断路器功能

  ```java
  @SpringBootApplication
  @EnableDiscoveryClient
  @EnableCircuitBreaker
  public class HystrixServiceApplication {
  
      public static void main(String[] args) {
          SpringApplication.run(HystrixServiceApplication.class, args);
      }
  
  }
  ```

- 创建controller用于调用user-service服务

  ```java
  @RestController
  @RequestMapping("/user")
  public class UserHystrixController {
      @Autowired
      private UserService userService;
  
      @GetMapping("/testFallback/{id}")
      public CommonResult testFallback(@PathVariable Long id) {
          return userService.getUser(id);
      }
  }
  ```

- 实际调用user-service服务在service层

  - restTemplate是自定义负载均衡之后注入到IOC容器中的，故直接获取
  - userServiceUrl是预定义在配置文件中user-service的服务地址
  - restTemplate.getForObject调用user-service中的服务路径来获取结果

  ```java
  @Service
  public class UserService {
  
      private Logger LOGGER = LoggerFactory.getLogger(this.getClass());
  
      @Autowired
      private RestTemplate restTemplate;
  
      @Value("${service-url.user-service}")
      private String userServiceUrl;
  
      @HystrixCommand(fallbackMethod = "getDefaultUser")
      public CommonResult getUser(Long id) {
          return restTemplate.getForObject(userServiceUrl + "/user/{1}",CommonResult.class,id);
      }
  
      public CommonResult getDefaultUser(@PathVariable Long id) {
          User defaultUser = new User(-1L, "defaultUser", "123456");
          return new CommonResult<>(defaultUser);
      }
  }
  ```

- 结果

  - 可以通过hystrix-service模块调用user-service模块

    ![image-20201124211417960](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201124211417960.png)

  - 当停掉user-service服务之后，可以获取到断路器的默认结果

    ![image-20201124211543687](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201124211543687.png)

### 4.2 服务熔断解读

#### 4.2.1 @HystrixCommand

- @HystrixCommand注解出现在UserService类中，可以看出getUser方法通过restTemplate远程调用user-service，而@HystrixCommand注解用于处理当服务出现故障时默认的解决方案

- 常用参数
  - fallbackMethod：指定服务降级处理方法
  - ignoreExceptions：忽略某些异常，不发生服务降级
  - commandKey：命令名称，用于区分不同的命令
  - groupKey：分组名称，Hystrix会根据不同的分组来统计命令的告警及仪表盘信息
  - threadPoolKey：线程池名称，用于划分线程池

#### 4.2.2 Hystrix的请求缓存

#### 4.2.3 Hystrix请求合并

### 4.3 Hystrix常用配置

#### 4.3.1 全局配置

```yaml
hystrix:
  command: #用于控制HystrixCommand的行为
    default:
      execution:
        isolation:
          strategy: THREAD #控制HystrixCommand的隔离策略，THREAD->线程池隔离策略(默认)，SEMAPHORE->信号量隔离策略
          thread:
            timeoutInMilliseconds: 1000 #配置HystrixCommand执行的超时时间，执行超过该时间会进行服务降级处理
            interruptOnTimeout: true #配置HystrixCommand执行超时的时候是否要中断
            interruptOnCancel: true #配置HystrixCommand执行被取消的时候是否要中断
          timeout:
            enabled: true #配置HystrixCommand的执行是否启用超时时间
          semaphore:
            maxConcurrentRequests: 10 #当使用信号量隔离策略时，用来控制并发量的大小，超过该并发量的请求会被拒绝
      fallback:
        enabled: true #用于控制是否启用服务降级
      circuitBreaker: #用于控制HystrixCircuitBreaker的行为
        enabled: true #用于控制断路器是否跟踪健康状况以及熔断请求
        requestVolumeThreshold: 20 #超过该请求数的请求会被拒绝
        forceOpen: false #强制打开断路器，拒绝所有请求
        forceClosed: false #强制关闭断路器，接收所有请求
      requestCache:
        enabled: true #用于控制是否开启请求缓存
  collapser: #用于控制HystrixCollapser的执行行为
    default:
      maxRequestsInBatch: 100 #控制一次合并请求合并的最大请求数
      timerDelayinMilliseconds: 10 #控制多少毫秒内的请求会被合并成一个
      requestCache:
        enabled: true #控制合并请求是否开启缓存
  threadpool: #用于控制HystrixCommand执行所在线程池的行为
    default:
      coreSize: 10 #线程池的核心线程数
      maximumSize: 10 #线程池的最大线程数，超过该线程数的请求会被拒绝
      maxQueueSize: -1 #用于设置线程池的最大队列大小，-1采用SynchronousQueue，其他正数采用LinkedBlockingQueue
      queueSizeRejectionThreshold: 5 #用于设置线程池队列的拒绝阀值，由于LinkedBlockingQueue不能动态改版大小，使用时需要用该参数来控制线程数
```

#### 4.3.2 实例配置

实例配置只需要将全局配置中的default换成与之对应的key即可

```yaml
hystrix:
  command:
    HystrixComandKey: #将default换成HystrixComrnandKey
      execution:
        isolation:
          strategy: THREAD
  collapser:
    HystrixCollapserKey: #将default换成HystrixCollapserKey
      maxRequestsInBatch: 100
  threadpool:
    HystrixThreadPoolKey: #将default换成HystrixThreadPoolKey
      coreSize: 10
```

- 配置文件中相关key的说明
  - HystrixComandKey对应@HystrixCommand中的commandKey属性
  - HystrixCollapserKey对应@HystrixCollapser注解中的collapserKey属性
  - HystrixThreadPoolKey对应@HystrixCommand中的threadPoolKey属性

## 5 Hystrix Dashboard(断路器执行监控)

### 5.1 概述

- Hystrix Dashboard 是Spring Cloud中查看Hystrix实例执行情况的一种仪表盘组件，支持查看单个实例和查看集群实例
- Hystrix提供了Hystrix Dashboard来实时监控HystrixCommand方法的执行情况，Hystrix Dashboard可以有效地反映出每个Hystrix实例的运行情况，帮助快速发现系统中的问题

### 5.2 断路器执行监控

#### 5.2.1 创建模块执行监控(hystrix-dashboard)

- 依赖

  ```xml
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
  </dependency>
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>
  ```

- 配置文件

  ```yaml
  server:
    port: 8501
  spring:
    application:
      name: hystrix-dashboard
  eureka:
    client:
      register-with-eureka: true
      fetch-registry: true
      service-url:
        defaultZone: http://localhost:8001/eureka/
  ```

- 开启监控功能

  ```java
  @SpringBootApplication
  @EnableDiscoveryClient
  @EnableHystrixDashboard
  public class HystrixDashboardApplication {
  
      public static void main(String[] args) {
          SpringApplication.run(HystrixDashboardApplication.class, args);
      }
  
  }
  ```

- 查看监控信息

  - DashBoard地址:http://localhost:8501/hystrix

  ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16d5e5a49f435f29)
  - 填写监控地址，注意本地不支持https，监控地址:http://localhost:8401/actuator/hystrix.stream

  ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16d5e5a49ff037d3)
  - 注意:被监控的服务必须开启Actuator的hystrix.stream端点，也就是8401端口配置文件添加:

    ```yaml
    management:
      endpoints:
        web:
          exposure:
            include: 'hystrix.stream' #暴露hystrix监控端点
    ```

- 监控结果

  ![image-20201124215140259](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201124215140259.png)

### 5.3 Hystrix集群实例监控

- 使用Turbine来聚合hystrix-service服务的监控信息，然后hystrix-dashboard服务就可以从Turbine获取聚合好的监控信息

#### 5.3.1 创建模块聚合hystrix-service的监控信息(turbine-service)

## 6 OpenFeign(基于Ribbon和Hystrix的声明式服务调用)

### 6.1 概述

- Feign是一个声明式WebService客户端
- 在之前使用Ribbon+RestTemplate时，利用RestTemplate对http请求的封装处理，形成了一套模板化的调用方法，实际开发中，往往一个接口被多出调用，Feign在此基础上做了进一步封装，帮助定义和实现依赖服务接口的定义
- Feign集成了Ribbon通过轮询实现客户端的负载均衡

### 6.2 Feign测试

#### 6.2.1 创建模块演示Feign服务调用功能(feign-service)

- 依赖

  ```xml
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-openfeign</artifactId>
  </dependency>
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  ```

- 配置文件

  ```yaml
  server:
    port: 8701
  spring:
    application:
      name: feign-service
  eureka:
    client:
      register-with-eureka: true
      fetch-registry: true
      service-url:
        defaultZone: http://localhost:8001/eureka/
  ```

- 开启Feign客户端功能

  ```java
  @SpringBootApplication
  @EnableDiscoveryClient
  @EnableFeignClients
  public class FeignServiceApplication {
  
      public static void main(String[] args) {
          SpringApplication.run(FeignServiceApplication.class, args);
      }
  
  }
  ```

- 添加UserService接口实现对user-service服务Controller接口绑定

  - 通过@FeignClient注解实现了一个Feign客户端，其中的value为user-service表示是对user-service服务的接口调用客户端

  - UserService接口的方法本质上就是对user-service服务中的controller对应的方法删掉方法内容，只保留接口

    ```java
    @FeignClient(value = "user-service")
    public interface UserService {
        @PostMapping("/user/create")
        CommonResult create(@RequestBody User user);
    
        @GetMapping("/user/{id}")
        CommonResult<User> getUser(@PathVariable Long id);
    
        @GetMapping("/user/getByUsername")
        CommonResult<User> getByUsername(@RequestParam String username);
    
        @PostMapping("/user/update")
        CommonResult update(@RequestBody User user);
    
        @PostMapping("/user/delete/{id}")
        CommonResult delete(@PathVariable Long id);
    }
    ```

- 添加controller对UserService中的方法进行调用(省略)

- user-service重新改造一个配置文件，在启动配置上指定Active Profile启动另一个端口的user-service检测Feign的负载均衡能力

  ![image-20201124223930404](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201124223930404.png)

- 测试结果(运行在8201和8202的user-service服务交替调用)

  ![image-20201124224207246](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201124224207246.png)

#### 6.2.2 测试Feign中的服务降级

- 添加UserService接口的实现类作为服务降级类

  ```java
  public class UserServiceImpl implements UserService {
      @Override
      public CommonResult create(User user) {
          User defaultUser = new User(-1L, "defaultUser", "123456");
          return new CommonResult<>(defaultUser);
      }
  
      @Override
      public CommonResult<User> getUser(Long id) {
          User defaultUser = new User(-1L, "defaultUser", "123456");
          return new CommonResult<>(defaultUser);
      }
  
      @Override
      public CommonResult<User> getByUsername(String username) {
          User defaultUser = new User(-1L, "defaultUser", "123456");
          return new CommonResult<>(defaultUser);
      }
  
      @Override
      public CommonResult update(User user) {
          return new CommonResult("调用失败，服务被降级",500);
      }
  
      @Override
      public CommonResult delete(Long id) {
          return new CommonResult("调用失败，服务被降级",500);
      }
  }
  ```

- 指定服务降级类为UserServiceImpl

  ![image-20201124224719012](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201124224719012.png)

- 在配置文件中开启Hystrix功能

  ![image-20201124224856305](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201124224856305.png)

#### 6.2.3 Feign日志打印功能

- Feign提供了日志打印功能，可以通过配置来调整日志级别，从而了解Feign中Http请求的细节

- 日志级别

  - NONE：默认的，不显示任何日志
  - BASIC：仅记录请求方法、URL、响应状态码及执行时间
  - HEADERS：除了BASIC中定义的信息之外，还有请求和响应的头信息
  - FULL：除了HEADERS中定义的信息之外，还有请求和响应的正文及元数据

- 配置Logger.Level(Logger来源于feign.Logger)

  ```java
  @Configuration
  public class FeignConfig {
  
      @Bean
      public Logger.Level feignLoggerLevel() {
          return Logger.Level.FULL;
      }
  }
  ```

- 在配置文件中开启日志的Feign客户端

  - 配置UserService的日志级别为debug

    ```yaml
    logging:
      level:
        org.jiang.service.UserService: debug
    ```

- 查看日志

  ![image-20201124225909761](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201124225909761.png)

### 6.3 Feign常用配置

#### 6.3.1 Feign自己的配置

```yaml
feign:
  hystrix:
    enabled: true #在Feign中开启Hystrix
  compression:
    request:
      enabled: false #是否对请求进行GZIP压缩
      mime-types: text/xml,application/xml,application/json #指定压缩的请求数据类型
      min-request-size: 2048 #超过该大小的请求会被压缩
    response:
      enabled: false #是否对响应进行GZIP压缩
logging:
  level: #修改日志级别
    com.macro.cloud.service.UserService: debug
```

#### 6.3.2 Feign中Ribbon配置

- 在Feign中配置Ribbon可以直接使用Ribbon的配置

#### 6.3.3 Feign中的Hystrix配置

- 在Feign中配置Hystrix可以直接使用Hystrix的配置

## 7 Config(外部集中化配置管理)

### 7.1 概述

#### 7.1.1 分布式系统面临的配置问题

- 微服务意味着要将单体应用中的业务拆分成一个个子服务，每个服务的粒度相对较小，系统中会出现大量的配置信息用于运行，故将公共部分抽取出来成为一套集中式、动态的配置管理设施必不可少

#### 7.1.2 Spring Cloud Config

- Spring Cloud Config为微服务提供集中化的外部配置支持，配置服务器为不同微服务应用提供了一个中心化的外部配置

- Spring Cloud Config 分为服务端和客户端两个部分

  - 服务端被称为分布式配置中心，它是个独立的应用，可以从配置仓库获取配置信息并提供给客户端使用
  - 客户端可以通过配置中心来获取配置信息，在启动时加载配置
  - Spring Cloud Config 的配置中心默认采用Git来存储配置信息，所以天然就支持配置信息的版本管理，并且可以使用Git客户端来方便地管理和访问配置信息

  ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/aHR0cHM6Ly9yYXcuZ2l0aHVidXNlcmNvbnRlbnQuY29tL25pYW9uYW8vSW1hZ2VJY29uL21hc3Rlci9HaXQvMjAxOTExMTEwMS5wbmc)

### 7.2 搭建配置中心实现中心配置

#### 7.2.1 远程仓库准备配置信息

- Spring Cloud Config 需要一个存储配置信息的Git仓库，故提前在Git仓库中添加好配置文件

- git仓库地址:https://gitee.com/macrozheng/springcloud-config

- 配置仓库目录结构

  - 提供了不同的分支，不同的分支配置了不同的环境，生产、开发、测试

  ![image-20201125005040369](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201125005040369.png)

  ![image-20201125005114547](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201125005114547.png)

#### 7.2.2 创建配置中心模块演示Config配置中心功能(config-server)

- 依赖

  - 这里添加了spring-cloud-config-server依赖用来作为配置中心

  ```xml
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-config-server</artifactId>
  </dependency>
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>
  ```

- 配置文件

  - 增加了spring.cloud.config.server用于配置git仓库的信息

  ```yaml
  server:
    port: 8901
  spring:
    application:
      name: config-server
    cloud:
      config:
        server:
          git: #配置存储配置信息的Git仓库
            uri: https://gitee.com/macrozheng/springcloud-config.git
            username: macro
            password: 123456
            clone-on-start: true #开启启动时直接从git获取配置
  eureka:
    client:
      service-url:
        defaultZone: http://localhost:8001/eureka/
  ```

- 开启配置中心功能

  ```java
  @SpringBootApplication
  @EnableDiscoveryClient
  @EnableConfigServer
  public class ConfigServerApplication {
  
      public static void main(String[] args) {
          SpringApplication.run(ConfigServerApplication.class, args);
      }
  
  }
  ```

- 通过config-server获取**配置信息**

  - 获取配置文件信息的访问格式

    ```yaml
    # 获取配置信息
    /{label}/{application}-{profile}
    # 获取配置文件信息
    /{label}/{application}-{profile}.yml
    ```

  - 占位符相关解释

    - application：代表应用名称，默认为配置文件中的spring.application.name，如果配置了spring.cloud.config.name，则为该名称
    - label：代表分支名称，对应配置文件中的spring.cloud.config.label
    - profile：代表环境名称，对应配置文件中的spring.cloud.config.profile

  - 获取配置信息

    - 读取配置信息地址:http://localhost:8901/master/config-dev

    - 之所以是这个请求地址，因为git仓库中的配置文件默认格式就是{application}-{profile}.yml，也就是config就代表application
    
      ![image-20201125012247025](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201125012247025.png)

- 通过config-server获取**配置文件信息**

  - 访问地址:http://localhost:8901/master/config-dev.yml

    ![image-20201125012415794](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201125012415794.png)

### 7.3 搭建客户端用于获取配置中心配置信息

#### 7.3.1 创建模块用于访问(config-client)

- 依赖

  - 相对于配置中心增加了spring-boot-starter-web模块，说明要进行网络请求
  - 这里使用的是spring-cloud-starter-config，而配置中心使用的是spring-cloud-config-server

  ```xml
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-config</artifactId>
  </dependency>
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  ```

- 配置文件bootstrap.yml

  - bootstrap.yml

    - Spring Cloud会创建一个Bootstrap Context作为Spring应用的Application Context的**父上下文**
    - Bootstrap Context负责从**外部源加载配置属性**并解析配置，和Application Context共享一个从外部获取的环境
    - Bootstrap属性具有高优先级，默认情况下不会被本地配置覆盖，Bootstrap Context和Application Context有着不同的约定，故新增bootstrap.yml用于和Application Context配置分离
    - bootstrap.yml优先于application.yml加载

  - 配置文件内容

    - 这里没有spring.cloud.config.server，而是spring.cloud.config，因为这是客户端
    - 分别获取了配置中心的地址、配置文件名称和分支名称，也就对应了application、label和profile

    ```yaml
    server:
      port: 9001
    spring:
      application:
        name: config-client
      cloud:
        config: #Config客户端配置
          profile: dev #启用配置后缀名称
          label: dev #分支名称
          uri: http://localhost:8901 #配置中心地址
          name: config #配置文件名称
    eureka:
      client:
        service-url:
          defaultZone: http://localhost:8001/eureka/
    
    ```

- 创建controller测试获取配置文件信息

  - 为什么是@Value("${config.info}")

    ![image-20201125092321590](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201125092321590.png)

  - 从配置中心读取到的配置文件信息就是这样，所以可以使用config.info读取

  ```java
  @RestController
  public class ConfigClientController {
  
      @Value("${config.info}")
      private String configInfo;
  
      @GetMapping("/configInfo")
      public String getConfigInfo() {
          return configInfo;
      }
  }
  ```

- 测试

  - 访问地址:http://localhost:9001/configInfo

    ![image-20201125092958785](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201125092958785.png)

- 获取子目录下的配置

  - 不仅可以把每个项目的配置放在不同的Git仓库存储，也可以在一个Git仓库中存储多个项目的配置，此时就会用到在子目录中搜索配置信息的配置

  - config文件夹就代表dev分支下config应用的配置文件

    ![image-20201125094047248](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201125094047248.png)

  - 获取方式

    - 在config-server的配置文件中添加配置，用于搜索子目录的位置，使用application占位符，表示对于不同的应用，我们从对应应用名称的子目录中搜索配置，比如config子目录中的配置对应config应用

      ![image-20201125094357836](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201125094357836.png)

  - 获取结果

    - 可以发现获取的是config子目录下的配置文件信息

      ![image-20201125094433889](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201125094433889.png)

#### 7.3.2 配置刷新

- 当Git仓库中的配置信息更改后，我们可以通过SpringBoot Actuator的refresh端点来刷新客户端配置信息

##### 对config-client进行改进

- 添加依赖

  ```java
  <dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>
  ```

- 修改配置文件添加actuator配置开启refresh端点

  ```yaml
  management:
    endpoints:
      web:
        exposure:
          include: 'refresh'
  ```

- 在controller类上添加@RefreshScope注解用于刷新配置

  ```java
  @RestController
  @RefreshScope
  public class ConfigClientController {
  
      @Value("${config.info}")
      private String configInfo;
  
      @GetMapping("/configInfo")
      public String getConfigInfo() {
          return configInfo;
      }
  }
  ```

- 发送一次post请求，调用refresh端点进行配置刷新(监控版本变化)

  - 请求地址:http://localhost:9001/actuator/refresh

    ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16dca57cfef7a052)

  - 当不适用postman时，可以通过终端执行curl -X post http://localhost:9001/actuator/refresh指令

### 7.4 配置中心添加安全验证

#### 7.4.1 添加模块整合SpringSecurity(config-security-server)

- 依赖 

  ```yaml
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-config-server</artifactId>
  </dependency>
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-security</artifactId>
  </dependency>
  ```

- 配置文件

  - 增加了对security账户的配置

  ```yaml
  server:
    port: 8905
  spring:
    application:
      name: config-security-server
    cloud:
      config:
        server:
          git:
            uri: https://gitee.com/macrozheng/springcloud-config.git
            username: macro
            password: 123456
            clone-on-start: true #开启启动时直接从git获取配置
    security: #配置用户名和密码
      user:
        name: macro
        password: 123456
  ```

#### 7.4.2 修改config-client用于匹配添加了安全认证的配置中心

- 添加一个bootstrap-security.yml配置文件，用于匹配配置了用户名和密码的配置中心，使用bootstrap-security.yml启动config-client服务

  ```yaml
  server:
    port: 9002
  spring:
    application:
      name: config-client
    cloud:
      config:
        profile: dev #启用配置后缀名称
        label: dev #分支名称
        uri: http://localhost:8905 #配置中心地址
        name: config #配置文件名称
        username: macro
        password: 123456
  
  eureka:
    client:
      service-url:
        defaultZone: http://localhost:8001/eureka/
  
  management:
    endpoints:
      web:
        exposure:
          include: 'refresh'
  ```

### 7.5 config-sever集群

- 在微服务架构中，所有服务都从配置中心获取配置，配置中心一旦宕机，会发生很严重的问题；故需要搭建配置中心集群

#### 7.5.1搭建配置中心集群

- 启动两个config-server分别运行在8902和8903端口上

- 添加config-client的配置文件bootstrap-cluster.yml，主要是添加了从注册中心获取配置中心地址的配置并去除了配置中心uri的配置

  - bootstrap-cluster.yml

    ```yaml
    spring:
      cloud:
        config:
          profile: dev #启用环境名称
          label: dev #分支名称
          name: config #配置文件名称
          discovery:
            enabled: true
            service-id: config-server
    eureka:
      client:
        service-url:
          defaultZone: http://localhost:8001/eureka/
    ```

- 以bootstrap-cluster.yml启动config-client服务，注册中心显示信息

  ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16dca57d33f3b209)

# 8 Bus(消息总线)

### 8.1 概述

- Spring Cloud Bus 使用轻量级的消息代理来连接微服务架构中的各个服务，可以将其用于广播状态更改（例如配置中心配置更改）或其他管理指令
- 使用消息代理来构建一个主题，然后把微服务架构中的所有服务都连接到这个主题上去，当我们向该主题发送消息时，所有订阅该主题的服务都会收到消息并进行消费
- Spring Cloud Bus 配合 Spring Cloud Config 使用可以实现配置的动态刷新

![image-20201125114431226](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201125114431226.png)

### 8.2 安装RabbitMQ

- https://www.cnblogs.com/JustinLau/p/11738511.html
- **注意:erlang添加环境变量之后一定要重启**

### 8.3 增加动态刷新配置功能

- 使用 Spring Cloud Bus 动态刷新配置需要配合 Spring Cloud Config 一起使用，通过config-server、config-client模块演示该功能

#### 8.3.1 config-server模块修改

- 添加依赖

  ```xml
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-bus-amqp</artifactId>
  </dependency>
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>
  ```

- 配置文件修改

  - 添加配置文件application-amqp.yml，主要是添加了RabbitMQ的配置及暴露了刷新配置的Actuator端点

    ```yaml
    server:
      port: 8904
    spring:
      application:
        name: config-server
      cloud:
        config:
          server:
            git:
              uri: https://gitee.com/macrozheng/springcloud-config.git
              username: macro
              password: 123456
              clone-on-start: true # 开启启动时直接从git获取配置
      rabbitmq: #rabbitmq相关配置
        host: localhost
        port: 5672
        username: guest
        password: guest
    eureka:
      client:
        service-url:
          defaultZone: http://localhost:8001/eureka/
    management:
      endpoints: #暴露bus刷新配置的端点
        web:
          exposure:
            include: 'bus-refresh'
    ```

#### 8.3.2 config-client模块修改

- 添加依赖

  ```xml
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-bus-amqp</artifactId>
  </dependency>
  ```

- 修改配置文件

  - 添加配置文件bootstrap-amqp1.yml及bootstrap-amqp2.yml用于启动两个不同的config-client，两个配置文件只有端口号不同

    ```yaml
    server:
      port: 9004
    spring:
      application:
        name: config-client
      cloud:
        config:
          profile: dev #启用环境名称
          label: dev #分支名称
          name: config #配置文件名称
          discovery:
            enabled: true
            service-id: config-server
      rabbitmq: #rabbitmq相关配置
        host: localhost
        port: 5672
        username: guest
        password: guest
    eureka:
      client:
        service-url:
          defaultZone: http://localhost:8001/eureka/
    management:
      endpoints:
        web:
          exposure:
            include: 'refresh'
    ```

#### 8.3.3 动态刷新配置演示

- 先启动相关服务，启动eureka-server，以application-amqp.yml为配置启动config-server，以bootstrap-amqp1.yml为配置启动config-client，以bootstrap-amqp2.yml为配置再启动一个config-client，启动后注册中心显示

  ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16dd48b0e6512e49)

- 启动所有服务后，登录RabbitMQ的控制台可以发现Spring Cloud Bus 创建了一个叫springCloudBus的交换机及三个以 springCloudBus.anonymous开头的队列

  ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16dd48b113e18cdd)

![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16dd48b1120ff7b6)

- 调用注册中心的接口刷新所有配置:http://localhost:8904/actuator/bus-refresh

  ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16dd48b1199bb9b4)

- 刷新后再分别调用http://localhost:9004/configInfo 和 http://localhost:9005/configInfo 获取配置信息，发现都已经刷新
  
  - **如果只需要刷新指定实例的配置可以使用以下格式进行刷新：http://localhost:8904/actuator/bus-refresh/{destination} ，我们这里以刷新运行在9004端口上的config-client为例http://localhost:8904/actuator/bus-refresh/config-client:9004**

### 8.4 配合WebHooks实现自动刷新服务配置

- WebHooks相当于是一个钩子函数，通过配置当向Git仓库push代码时触发这个钩子函数，当向配置仓库push代码时就会自动刷新服务配置

## 9 Sleuth(分布式请求链路跟踪)

### 9.1 概述

#### 9.1.1 遇到的问题

- 在微服务框架中，一个由客户端发起的请求在后端系统中会经过很多个不同的服务节点调用来协同产生最后的请求结果，每一个前端请求都会形成一条复杂的分布式服务调用链路，链路中的任何一环出现高延时和错误都会引起整个请求最后的失败

#### 9.1.2 解决方案

- Spring Cloud Sleuth 是分布式系统中跟踪服务间调用的工具，它可以直观地展示出一次请求的调用过程

- 在分布式系统中提供追踪解决方案并且兼容支持了zipkin

- SpringCloud从F版起已不需要自己构建Zipkin server了，只需要调用jar包即可

  ![image-20201125210902611](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201125210902611.png)

### 9.2 服务添加请求链路跟踪功能

- 通过user-service和ribbon-service之间的服务调用演示该功能，将调用ribbon-service的接口时，ribbon-service会通过RestTemplate来调用user-service提供的接口

#### 9.2.1 下载搭建zipkin-server

- zipkin-server概述

  - Zipkin 是一个开放源代码分布式的跟踪系统，每个服务向zipkin报告计时数据，zipkin会根据调用关系通过Zipkin UI生成依赖关系图

  - Zipkin提供了可插拔数据存储方式：In-Memory、MySql、Cassandra以及Elasticsearch，为了方便在开发环境可以采用了In-Memory方式进行存储，生产数据量大的情况则推荐使用Elasticsearch

  - 表示一条请求链路，一条链路通过Trace Id唯一标识，Span标识发起的请求信息，各Span通过parent id关联起来

    - Trace:类似于树结构的Span集合，表示一条调用链路，存在唯一标识
    - span:表示调用链路来源，通俗的理解span就是一次请求信息

    ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/12889335-49075b7a31bf4b4a.png)

- 构建zipkin服务端

  - zipkin服务端jar包下载:https://dl.bintray.com/openzipkin/maven/io/zipkin/java/zipkin-server/

  - 启动服务端，默认INFO级别可以不设置logging

    - 命令:java -jar zipkin-server-2.10.4-exec.jar --logging.level.zipkin2=INFO

  - 启动成功

    ![image-20201125213205835](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201125213205835.png)

  - 访问服务端

    - 地址:http://localhost:9411

#### 9.2.1 user-service和ribbon-service添加请求链路跟踪功能的支持

- user-service和ribbon-service添加依赖

  ```xml
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-zipkin</artifactId>
  </dependency>
  ```

- user-service和ribbon-service修改配置文件

  - 添加了zipkin的请求地址以及sleuth抽样收集概率

  ```yaml
  server:
    port: 8201
  spring:
    application:
      name: user-service
  
    zipkin:
      base-url: http://localhost:9411
    sleuth:
      sampler:
        probability: 0.1 #设置Sleuth的抽样收集概率
  
  eureka:
    client:
      register-with-eureka: true
      fetch-registry: true
      service-url:
        defaultZone: http://localhost:8001/eureka/
  ```

- 多次调用（Sleuth为抽样收集）ribbon-service的接口http://localhost:8301/user/1 ，调用完后查看Zipkin请求链路跟踪信息

  ![image-20201125214944314](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201125214944314.png)

### 9.3 整合Elasticsearch存储跟踪信息

#### 9.3.1 Elasticsearch安装及配置

- 下载zip安装包并解压

- 运行bin目录下的elasticsearch.bat启动Elasticsearch

- 启动界面

  ![image-20201125215415141](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201125215415141.png)

- 重启zipkin并指定启动参数

  - ```yaml
    # STORAGE_TYPE：表示存储类型 ES_HOSTS：表示ES的访问地址
    java -jar zipkin-server-2.12.9-exec.jar --STORAGE_TYPE=elasticsearch --ES_HOSTS=localhost:9200 
    ```

- 重启user-service和ribbon-service服务(只有重启存储才能生效)
- zipkin启动参数参考:https://github.com/openzipkin/zipkin/tree/master/zipkin-server#elasticsearch-storage

#### 9.3.2 配合Elasticsearch的可视化工具Kibana

## 10 Gateway(新一代API网关服务)

### 10.1 概述

- Gateway是在Spring生态系统之上构建的API网关服务，基于Spring 5，Spring Boot 2和 Project Reactor等技术。Gateway旨在提供一种简单而有效的方式来对API进行路由，以及提供一些强大的过滤器功能， 例如：熔断、限流、重试等

- 为了提高网关的性能，Spring Cloud Gateway是基于WebFlux框架实现的，而WebFlux框架底层使用了高性能的**Reactor模式通信框架Netty**

- Spring Cloud Gateway的目标统一的路由的方式并且基于Filter链的方式提供了网关基本的功能，例如:安全，监控/指标和限流

- 源码架构

  - 可以看出Spring Cloud Gateway集成了webflux，而webflux又集成了netty

  ![image-20201125220853549](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201125220853549.png)

- Spring Cloud Gateway特性
  - 基于Spring Framework 5, Project Reactor 和 Spring Boot 2.0 进行构建
  - 动态路由：能够匹配任何请求属性
  - 可以对路由指定 Predicate（断言）和 Filter（过滤器）
  - 集成Hystrix的断路器功能
  - 集成 Spring Cloud 服务发现功能
  - 易于编写的 Predicate（断言）和 Filter（过滤器）
  - 请求限流功能
  - 支持路径重写
- 相关概念
  - Route（路由）：路由是构建网关的基本模块，它由ID，目标URI，一系列的断言和过滤器组成，如果断言为true则匹配该路由
  - Predicate（断言）：指的是Java 8 的 Function Predicate。 输入类型是Spring框架中的ServerWebExchange。 这使开发人员可以匹配HTTP请求中的所有内容，例如请求头或请求参数。如果请求与断言相匹配，则进行路由
  - Filter（过滤器）：指的是Spring框架中GatewayFilter的实例，使用过滤器，可以在请求被路由前后对请求进行修改

### 10.2 构建API网关

#### 10.2.1 构建api-gateway模块演示常用功能

- 依赖

  ```xml
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-gateway</artifactId>
  </dependency>
  ```

- 配置文件

  - spring.cloud.gateway.routes配置路由规则和路径等
  - 如果断言没有指定匹配路径，则表示到9201端口的请求都会被转发到8201,
    - 例如http://localhost:9201/user/1则转发到http://localhost:8201/user/1

  ```yaml
  server:
    port: 9201
  service-url:
    user-service: http://localhost:8201
  spring:
    cloud:
      gateway:
        routes:
          - id: path_route #路由的ID
            uri: ${service-url.user-service}/user/{id} #匹配后路由地址
            predicates: # 断言，路径相匹配的进行路由
              - Path=/user/{id}
  ```

- 启动eureka-server，user-service和api-gateway服务，并调用地址测试：http://localhost:9201/user/1

  - 该地址是通过Gateway路由到匹配的地址

  ![image-20201125223856235](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201125223856235.png)

### 10.3 Route Predicate的使用

- Spring Cloud Gateway将路由匹配作为Spring WebFlux HandlerMapping基础架构的一部分， Spring Cloud Gateway包括许多内置的Route Predicate工厂， 所有这些Predicate都与HTTP请求的不同属性匹配，多个Route Predicate工厂可以进行组合

#### 10.3.1 After Route Predicate

- 在指定时间之后的请求会匹配该路由

  ```yaml
  spring:
    cloud:
      gateway:
        routes:
          - id: after_route
            uri: ${service-url.user-service}
            predicates:
              - After=2019-09-24T16:30:00+08:00[Asia/Shanghai]
  ```

#### 10.3.2 Before Route Predicate

- 在指定时间之前的请求会匹配该路由

  ```yaml
  spring:
    cloud:
      gateway:
        routes:
          - id: before_route
            uri: ${service-url.user-service}
            predicates:
              - Before=2019-09-24T16:30:00+08:00[Asia/Shanghai]
  ```

#### 10.3.3 Between Route Predicate

- 在指定时间区间内的请求会匹配该路由

  ```yaml
  spring:
    cloud:
      gateway:
        routes:
          - id: before_route
            uri: ${service-url.user-service}
            predicates:
              - Between=2019-09-24T16:30:00+08:00[Asia/Shanghai], 2019-09-25T16:30:00+08:00[Asia/Shanghai]
  
  ```

#### 10.3.4 Cookie Route Predicate

- 带有指定cookie的请求会匹配该路由

  - 逗号前后分别代表key和value
  - 多个cookie可以使用多个- Cookie

  ```yaml
  spring:
    cloud:
      gateway:
        routes:
          - id: cookie_route
            uri: ${service-url.user-service}
            predicates:
              - Cookie=username,macro
              - Cookie=password,123456
  ```

- 使用curl工具发送带有cookie为`username=macro`的请求可以匹配该路由

  ```
  curl http://localhost:9201/user/1 --cookie "username=macro"
  ```

#### 10.3.5 Header Route Predicate

- 带有指定请求头的请求会匹配该路由

  - 逗号前后分别代表key和value

  ```yaml
  spring:
    cloud:
      gateway:
        routes:
        - id: header_route
          uri: ${service-url.user-service}
          predicates:
          - Header=X-Request-Id, \d+
  ```

- 使用curl工具发送带有请求头为`Host:www.macrozheng.com`的请求可以匹配该路由

  ```
  curl http://localhost:9201/user/1 -H "Host:www.macrozheng.com" 
  ```

#### 10.3.6 Method Route Predicate

- 发送指定方法的请求会匹配该路由

  ```yaml
  spring:
    cloud:
      gateway:
        routes:
        - id: method_route
          uri: ${service-url.user-service}
          predicates:
          - Method=GET
  ```

- 使用curl工具发送POST请求无法匹配该路由

  ```
  curl -X POST http://localhost:9201/user/1
  ```

#### 10.3.7 Path Route Predicate

- 发送指定路径的请求会匹配该路由

  ```yaml
  spring:
    cloud:
      gateway:
        routes:
          - id: path_route
            uri: ${service-url.user-service}/user/{id}
            predicates:
              - Path=/user/{id}
  ```

- 使用curl工具发送`/user/1`路径请求可以匹配该路由

  ```
  curl http://localhost:9201/user/1
  ```

- 使用curl工具发送`/abc/1`路径请求无法匹配该路由

  ```
  curl http://localhost:9201/abc/1
  ```

#### 10.3.8 Query Route Predicate

- 带指定查询参数的请求可以匹配该路由

  ```yaml
  spring:
    cloud:
      gateway:
        routes:
        - id: query_route
          uri: ${service-url.user-service}/user/getByUsername
          predicates:
          - Query=username
  ```

- 使用curl工具发送带`username=macro`查询参数的请求可以匹配该路由

  ```
  curl http://localhost:9201/user/getByUsername?username=macro
  复制代码
  ```

- 使用curl工具发送带不带查询参数的请求无法匹配该路由

  ```
  curl http://localhost:9201/user/getByUsername
  ```

#### 10.3.9 RemoteAddr Route Predicate

- 从指定远程地址发起的请求可以匹配该路由

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: remoteaddr_route
        uri: ${service-url.user-service}
        predicates:
        - RemoteAddr=192.168.1.1/24
```

- 使用curl工具从192.168.1.1发起请求可以匹配该路由

```
curl http://localhost:9201/user/1
```

#### 10.3.10 Weight Route Predicate

- 使用权重来路由相应请求，以下表示有80%的请求会被路由到localhost:8201，20%会被路由到localhost:8202
  - 可以看出- weight两个不同的route同属于一个组group1，后面的整数代表分配的权重

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: weight_high
        uri: http://localhost:8201
        predicates:
        - Weight=group1, 8
      - id: weight_low
        uri: http://localhost:8202
        predicates:
        - Weight=group1, 2
```

### 10.4 Route Filter的使用

- 路由过滤器可用于修改进入的HTTP请求和返回的HTTP响应，路由过滤器只能指定路由进行使用，Spring Cloud Gateway 内置了多种路由过滤器，他们都由GatewayFilter的工厂类来产生

#### 10.4.1 AddRequestParameter GatewayFilter

- 给请求添加参数的过滤器

  ```yaml
  spring:
    cloud:
      gateway:
        routes:
          - id: add_request_parameter_route
            uri: http://localhost:8201
            filters:
              - AddRequestParameter=username, macro
            predicates:
              - Method=GET
  ```

- 以上配置会对GET请求添加`username=macro`的请求参数，通过curl工具使用以下命令进行测试

  ```
  curl http://localhost:9201/user/getByUsername
  ```

- 相当于发起该请求：

  ```
  curl http://localhost:8201/user/getByUsername?username=macro
  ```

#### 10.4.2 StripPrefix GatewayFilter

- 对指定数量的路径前缀进行去除的过滤器

  ```yaml
  spring:
    cloud:
      gateway:
        routes:
        - id: strip_prefix_route
          uri: http://localhost:8201
          predicates:
          - Path=/user-service/**
          filters:
          - StripPrefix=2
  ```

- 以上配置会把以`/user-service/`开头的请求的路径去除两位，通过curl工具使用以下命令进行测试

  - 路径是通过多级目录进行请求，也就是把前面的两位目录去除

  ```
  curl http://localhost:9201/user-service/a/user/1
  ```

- 相当于发起请求

  ```
  curl http://localhost:8201/user/1
  ```

#### 10.4.3 PrefixPath GatewayFilter

- 与StripPrefix过滤器恰好相反，会对原有路径进行增加操作的过滤器

  ```yaml
  spring:
    cloud:
      gateway:
        routes:
        - id: prefix_path_route
          uri: http://localhost:8201
          predicates:
          - Method=GET
          filters:
          - PrefixPath=/user
  ```

- 以上配置会对所有GET请求添加`/user`路径前缀，通过curl工具使用以下命令进行测试

  ```
  curl http://localhost:9201/1
  ```

- 相当于发起该请求

  ```
  curl http://localhost:8201/user/1
  ```

#### 10.4.4 Hystrix GatewayFilter

- Hystrix过滤器允许将断路器功能添加到网关路由中

- 开启断路器功能，添加Hystrix相关依赖

  ```xml
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
  </dependency>
  ```

- 添加相关服务降级处理类

  ```yaml
  spring:
    cloud:
      gateway:
        routes:
          - id: hystrix_route
            uri: http://localhost:8201
            predicates:
              - Method=GET
            filters:
              - name: Hystrix
                args:
                  name: fallbackcmd
                  fallbackUri: forward:/fallback
  ```

- 关闭user-service，调用该地址进行测试：http://localhost:9201/user/1 ，发现已经返回了服务降级的处理信息

  ![image-20201126114113334](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201126114113334.png)

#### 10.4.5 RequestRateLimiter GatewayFilter

- RequestRateLimiter过滤器可以用于限流，使用**RateLimiter**实现来确定是否允许当前请求继续进行，如果请求太大默认会返回HTTP 429-太多请求状态

- 添加相关依赖

  ```xml
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
  </dependency>
  ```

- 添加限流策略的配置类，这里有两种限流策略；一种是根据请求参数中的username进行限流，另一种方式是根据访问IP进行限流

  ```java
  @Configuration
  public class RedisRateLimiterConfig {
  
      @Bean
      public KeyResolver userKeyResolver() {
          return exchange -> Mono.just(Objects.requireNonNull(exchange.getRequest().getQueryParams().getFirst("username")));
      }
  
      @Bean
      @Primary
      public KeyResolver ipKeyResolver() {
          return exchange -> Mono.just(Objects.requireNonNull(exchange.getRequest().getRemoteAddress()).getHostName());
      }
  }
  ```

- 使用Redis进行限流，需要添加Redis和RequestRateLimiter的配置，这里对所有的GET请求都进行了按IP来限流的操作

  ```yaml
  server:
    port: 9201
  spring:
    redis:
      host: localhost
      password: 123456
      port: 6379
    cloud:
      gateway:
        routes:
          - id: requestratelimiter_route
            uri: http://localhost:8201
            filters:
              - name: RequestRateLimiter
                args:
                  redis-rate-limiter.replenishRate: 1 #令牌桶每秒填充平均速率
                  redis-rate-limiter.burstCapacity: 2 #令牌桶总容量
                  key-resolver: "#{@ipKeyResolver}" #用于限流的键的解析器的Bean对象的名字，它使用SpEL表达式根据#{@beanName}从Spring容器中获取Bean对象
            predicates:
              - Method=GET
  logging:
    level:
      org.springframework.cloud.gateway: debug
  
  ```

- 限流算法

  - https://blog.csdn.net/y277an/article/details/97074272?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.control&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.control

- 多次请求该地址：http://localhost:9201/user/1 ，会返回状态码为429的错误

  ![image-20201126121352962](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201126121352962.png)

#### 10.4.6 Retry GatewayFilter

- 对路由请求进行重试的过滤器，可以根据路由请求返回的HTTP状态码来确定是否重试

  ```yaml
  spring:
    cloud:
      gateway:
        routes:
        - id: retry_route
          uri: http://localhost:8201
          predicates:
          - Method=GET
          filters:
          - name: Retry
            args:
              retries: 1 #需要进行重试的次数
              statuses: BAD_GATEWAY #返回哪个状态码需要进行重试，返回状态码为5XX进行重试
              backoff:
                firstBackoff: 10ms
                maxBackoff: 50ms
                factor: 2
                basedOnPreviousValue: false
  ```

- 当调用返回500时会进行重试，访问测试地址：http://localhost:9201/user/111 (不存在id为111的用户)

- 可以发现user-service控制台报错2次，说明进行了一次重试

  ```verilog
  2019-10-27 14:08:53.435 ERROR 2280 --- [nio-8201-exec-2] o.a.c.c.C.[.[.[/].[dispatcherServlet]    : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed; nested exception is java.lang.NullPointerException] with root cause
  
  java.lang.NullPointerException: null
  	at com.macro.cloud.controller.UserController.getUser(UserController.java:34) ~[classes/:na]
  ```

### 10.5 配合注册中心使用

- Gateway配合注册中心使用时，默认情况下Eureka会根据注册中心注册的服务列表，**以服务名为路径创建动态路由**，Gateway同样实现了该功能

#### 10.5.1 整合Eureka使用动态路由

- 在api-gateway服务下添加相关依赖

  ```xml
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>
  ```

- 添加application-eureka.yml配置文件

  - 这里spring.cloud.gateway.discovery.locator开启以服务名为路径创建动态路由

  ```yml
  server:
    port: 9201
  spring:
    application:
      name: api-gateway
    cloud:
      gateway:
        discovery:
          locator:
            enabled: true #开启从注册中心动态创建路由的功能
            lower-case-service-id: true #使用小写服务名，默认是大写
  eureka:
    client:
      service-url:
        defaultZone: http://localhost:8001/eureka/
  logging:
    level:
      org.springframework.cloud.gateway: debug
  ```

- 使用application-eureka.yml配置文件启动api-gateway服务，访问http://localhost:9201/user-service/user/1

  ![image-20201126132132632](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201126132132632.png)
  - 实际上路由到user-service的http://localhost:8201/user/1 处
  - 也就是http://localhost:9201/user-service这一部分就会路由到http://localhost:8201，再拼接/user/1

#### 10.5.2 整合Eureka使用动态路由和过滤器

- **在结合注册中心使用过滤器的时候，我们需要注意的是uri的协议为`lb`，这样才能启用Gateway的负载均衡功能**

- 修改配置文件，使用PrefixPath过滤器，为所有Get请求路径添加`/user`路径并路由

  ```yaml
  server:
    port: 9201
  spring:
    application:
      name: api-gateway
    cloud:
      gateway:
        routes:
          - id: prefixpath_route
            uri: lb://user-service #此处需要使用lb协议
            predicates:
              - Method=GET
            filters:
              - PrefixPath=/user
        discovery:
          locator:
            enabled: true
  eureka:
    client:
      service-url: 
        defaultZone: http://localhost:8001/eureka/
  logging:
    level:
      org.springframework.cloud.gateway: debug
  ```

## 11 Admin(微服务应用监控)

### 11.1 概述

Spring Boot Admin是一个开源社区项目，用于管理和监控SpringBoot应用程序，然后通过图形化界面呈现出来。Spring Boot Admin不仅可以监控单体应用，还可以和Spring Cloud的注册中心相结合来监控微服务应用

应用程序作为Spring Boot Admin Client向为Spring Boot Admin Server注册（通过HTTP）或使用SpringCloud注册中心（例如Eureka，Consul）发现， UI是的AngularJs应用程序，展示Spring Boot Admin Client的Actuator端点上的一些监控

- Spring Boot Admin 可以提供应用的以下监控信息
  - 监控应用运行过程中的概览信息
  - 度量指标信息，比如JVM、Tomcat及进程信息
  - 环境变量信息，比如系统属性、系统环境变量以及应用配置信息
  - 查看所有创建的Bean信息
  - 查看应用中的所有配置信息
  - 查看应用运行日志信息
  - 查看JVM信息
  - 查看可以访问的Web端点
  - 查看HTTP跟踪信息

### 11.2 演示监控中心功能

#### 11.2.1 创建模块作为监控中心(admin-server)

- 依赖

  ```xml
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
      <groupId>de.codecentric</groupId>
      <artifactId>spring-boot-admin-starter-server</artifactId>
  </dependency>
  ```

- 配置文件

  ```yaml
  spring:
    application:
      name: admin-server
  server:
    port: 9301
  ```

- 添加注解开启admin-server功能

  ```java
  @SpringBootApplication
  @EnableAdminServer
  public class AdminServerApplication {
  
      public static void main(String[] args) {
          SpringApplication.run(AdminServerApplication.class, args);
      }
  
  }
  ```

#### 11.2.2 创建模块作为客户端注册到监控中心(admin-server)

- 依赖

  ```xml
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
      <groupId>de.codecentric</groupId>
      <artifactId>spring-boot-admin-starter-client</artifactId>
  </dependency>
  ```

- 配置文件

  - management配置了暴露的端口以及健康状况
  - logging.file.name将日志监控保存到文件中

  ```yaml
  spring:
    application:
      name: admin-client
    boot:
      admin:
        client:
          url: http://localhost:9301 #配置admin-server地址
  server:
    port: 9305
  management:
    endpoints:
      web:
        exposure:
          include: '*'
    endpoint:
      health:
        show-details: always
  logging:
    file:
      name: admin-client.log #添加开启admin的日志监控
  ```

#### 11.2.3 监控信息演示

- 启动admin-server和admin-client服务，访问如下地址打开Spring Boot Admin的主页:http://localhost:9301

  ![image-20201126142122801](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201126142122801.png)

- 点击应用墙，选择admin-client查看监控信息

  - 监控信息概览

    ![image-20201126142434524](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201126142434524.png)

  - 度量指标信息，比如JVM、Tomcat及进程信息

    ![image-20201126142758766](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201126142758766.png)

  - 环境变量信息，比如系统属性、系统环境变量以及应用配置信息

    ![image-20201126142909059](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201126142909059.png)

  - 查看所有创建的Bean信息

    ![image-20201126142947106](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201126142947106.png)

  - 查看日志信息，需要添加以下配置才能开启

    ```yaml
    logging:
      file:
        name: admin-client.log #添加开启admin的日志监控
    ```

    ![image-20201126143111083](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201126143111083.png)

  - 查看JVM信息

    ![image-20201126143246929](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201126143246929.png)

  - 查看映射信息

    ![image-20201126143348372](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201126143348372.png)

### 11.3 结合注册中心使用

- Spring Boot Admin结合Spring Cloud 注册中心使用，只需将admin-server和注册中心整合即可，admin-server 会自动从注册中心获取服务列表，然后挨个获取监控信息

#### 11.3.1 修改admin-server模块

- 添加依赖

  ```xml
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>
  ```

- 修改配置文件，添加注册中心配置

  ```yaml
  spring:
    application:
      name: admin-server
  server:
    port: 9301
  
  eureka:
    client:
      register-with-eureka: true
      fetch-registry: true
      service-url:
        defaultZone: http://localhost:8001/eureka/
  ```

- 启动服务注册功能

  ```java
  @EnableDiscoveryClient
  @EnableAdminServer
  @SpringBootApplication
  public class AdminServerApplication {
  
      public static void main(String[] args) {
          SpringApplication.run(AdminServerApplication.class, args);
      }
  
  }
  ```

### 11.3.2 修改admin-client模块

- 添加依赖

  ```xml
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>
  ```

- 修改配置文件，删除原来的admin-server地址配置，添加注册中心配置即可

  ```yaml
  spring:
    application:
      name: admin-client
  server:
    port: 9305
  management:
    endpoints:
      web:
        exposure:
          include: '*'
    endpoint:
      health:
        show-details: always
  logging:
    file: admin-client.log #添加开启admin的日志监控
  eureka:
    client:
      register-with-eureka: true
      fetch-registry: true
      service-url:
        defaultZone: http://localhost:8001/eureka/
  ```

- 开启服务注册功能

  ```yaml
  @EnableDiscoveryClient
  @SpringBootApplication
  public class AdminClientApplication {
  
      public static void main(String[] args) {
          SpringApplication.run(AdminClientApplication.class, args);
      }
  
  }
  ```

#### 11.3.3 功能演示

- 启动eureka-server，使用application-eureka.yml配置启动admin-server，admin-client，user-service;Spring Boot Admin 主页发现可以看到服务信息:http://localhost:9301

  ![image-20201126145026182](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201126145026182.png)

### 11.4 添加登录验证

- 通过给admin-server添加Spring Security支持来获得登录认证功能

#### 11.4.1 创建模块演示带有登录验证的监控中心

- 依赖

  ```xml
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>
  <dependency>
      <groupId>de.codecentric</groupId>
      <artifactId>spring-boot-admin-starter-server</artifactId>
      <version>2.1.5</version>
  </dependency>
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-security</artifactId>
  </dependency>
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  ```

- 配置文件

  ```yaml
  spring:
    application:
      name: admin-security-server
    security: # 配置登录用户名和密码
      user:
        name: macro
        password: 123456
    boot:  # 不显示admin-security-server的监控信息
      admin:
        discovery:
          ignored-services: ${spring.application.name}
  server:
    port: 9301
  eureka:
    client:
      register-with-eureka: true
      fetch-registry: true
      service-url:
        defaultZone: http://localhost:8001/eureka/
  ```

- 添加配置类对SpringSecurity进行配置，便于admin-client可以注册

  ```java
  @Configuration
  public class SecuritySecureConfig extends WebSecurityConfigurerAdapter {
      private final String adminContextPath;
  
      public SecuritySecureConfig(AdminServerProperties adminServerProperties) {
          this.adminContextPath = adminServerProperties.getContextPath();
      }
  
      @Override
      protected void configure(HttpSecurity http) throws Exception {
          SavedRequestAwareAuthenticationSuccessHandler successHandler = new SavedRequestAwareAuthenticationSuccessHandler();
          successHandler.setTargetUrlParameter("redirectTo");
          successHandler.setDefaultTargetUrl(adminContextPath + "/");
  
          http.authorizeRequests()
                  //1.配置所有静态资源和登录页可以公开访问
                  .antMatchers(adminContextPath + "/assets/**").permitAll()
                  .antMatchers(adminContextPath + "/login").permitAll()
                  .anyRequest().authenticated()
                  .and()
                  //2.配置登录和登出路径
                  .formLogin().loginPage(adminContextPath + "/login").successHandler(successHandler).and()
                  .logout().logoutUrl(adminContextPath + "/logout").and()
                  //3.开启http basic支持，admin-client注册时需要使用
                  .httpBasic().and()
                  .csrf()
                  //4.开启基于cookie的csrf保护
                  .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                  //5.忽略这些路径的csrf保护以便admin-client注册
                  .ignoringAntMatchers(
                          adminContextPath + "/instances",
                          adminContextPath + "/actuator/**"
                  );
      }
  }
  ```

- 开启AdminServer及注册发现功能

  ```yaml
  @EnableDiscoveryClient
  @EnableAdminServer
  @SpringBootApplication
  public class AdminSecurityServerApplication {
  
      public static void main(String[] args) {
          SpringApplication.run(AdminSecurityServerApplication.class, args);
      }
  
  }
  ```

  

- 启动eureka-server，admin-security-server，访问Spring Boot Admin 主页发现需要登录才能访问:http://localhost:9301

  ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16e1cc3b8288b529)

## 12 Spring Cloud Security:OAuth2

### 12.1 概述

#### 12.1.1 简介

- Spring Cloud Security 为构建安全的SpringBoot应用提供了一系列解决方案，结合Oauth2可以实现单点登录、令牌中继、令牌交换等功能
- OAuth 2.0是用于授权的行业标准协议，OAuth 2.0为简化客户端开发提供了特定的授权流，包括Web应用、桌面应用、移动端应用等

#### 12.1.2 具体内容

[参考文档]: OAuth2、JWT.md

### 12.2 OAuth2的使用

#### 12.1 创建模块作为认证服务器(oauth2-server)

- 依赖

  ```xml
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-oauth2</artifactId>
  </dependency>
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-security</artifactId>
  </dependency>
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  ```

- 配置文件

  ```yaml
  server:
    port: 9401
  spring:
    application:
      name: oauth2-service
  ```

- 创建User实现UserDetails接口用于存储用户信息(ORM)

  ```java
  public class User implements UserDetails {
      private String username;
      private String password;
      private List<GrantedAuthority> authorities;
  
      public User(String username, String password, List<GrantedAuthority> authorities) {
          this.username = username;
          this.password = password;
          this.authorities = authorities;
      }
  
      @Override
      public Collection<? extends GrantedAuthority> getAuthorities() {
          return authorities;
      }
  
      @Override
      public String getPassword() {
          return password;
      }
  
      @Override
      public String getUsername() {
          return username;
      }
  
      @Override
      public boolean isAccountNonExpired() {
          return true;
      }
  
      @Override
      public boolean isAccountNonLocked() {
          return true;
      }
  
      @Override
      public boolean isCredentialsNonExpired() {
          return true;
      }
  
      @Override
      public boolean isEnabled() {
          return true;
      }
  }
  ```

  

- 注册PasswordEncoder和AuthenticationManager组件到容器

  ```java
  @Configuration
  public class SecureConfig extends WebSecurityConfigurerAdapter {
      @Bean
      public PasswordEncoder passwordEncoder() {
          return new BCryptPasswordEncoder();
      }
      
      @Bean
      @Override
      public AuthenticationManager authenticationManagerBean() throws Exception {
          return super.authenticationManagerBean();
      }
  }
  ```

- 添加UserService实现UserDetailsService接口，**用于加载用户信息**

  ```java
  @Service
  public class UserService implements UserDetailsService {
  
      private List<User> userList;
  
      @Autowired
      private PasswordEncoder passwordEncoder;
  
      @PostConstruct
      public void initData() {
          String password = passwordEncoder.encode("123456");
          userList = new ArrayList<>();
          userList.add(new User("macro", password, AuthorityUtils.commaSeparatedStringToAuthorityList("admin")));
          userList.add(new User("andy", password, AuthorityUtils.commaSeparatedStringToAuthorityList("client")));
          userList.add(new User("mark", password, AuthorityUtils.commaSeparatedStringToAuthorityList("client")));
      }
  
      @Override
      public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
          List<User> findUserList = userList.stream().filter(user -> user.getUsername().equals(username)).collect(Collectors.toList());
          if (!CollectionUtils.isEmpty(findUserList)) {
              return findUserList.get(0);
          } else {
              throw new UsernameNotFoundException("用户名或密码错误");
          }
      }
  }
  ```

- 添加认证服务器配置，添加@EnableAuthorizationServer注解开启认证服务器

  ```java
  @Configuration
  @EnableAuthorizationServer
  public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {
      @Autowired
      private PasswordEncoder passwordEncoder;
  
      @Autowired
      private AuthenticationManager authenticationManager;
  
      @Autowired
      private UserService userService;
  
      /**
       * 使用密码模式需要配置
       * @param endpoints
       * @throws Exception
       */
      @Override
      public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
          endpoints.authenticationManager(authenticationManager)
                  .userDetailsService(userService);
      }
  
      @Override
      public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
          clients.inMemory()
                  .withClient("admin") // 配置client_id
                  .secret("123456") // 配置client_secret
                  .accessTokenValiditySeconds(3600) // 配置token有效期
                  .refreshTokenValiditySeconds(86400) // 设置刷新token有效期
                  .redirectUris("http://www.baidu.com") // 配置登录成功跳转页面
                  .scopes("all") // 设置申请的权限范围
                  .authorizedGrantTypes("authorization_code","password"); // 配置grant_type，表示授权类型
      }
  }
  ```

- 配置资源服务器，添加@EnableResourceServer注解开启资源服务器

  ```java
  @Configuration
  @EnableResourceServer
  public class ResourceServerConfig extends ResourceServerConfigurerAdapter {
      @Override
      public void configure(HttpSecurity http) throws Exception {
          http.authorizeRequests()
                  .anyRequest()
                  .authenticated()
                  .and()
                  .requestMatchers()
                  .antMatchers("/user/**"); // 配置需要保护的资源路径
      }
  }
  ```

- 添加SpringSecurity配置，允许认证相关路径的访问及表单登录

  - PasswordEncoder和AuthenticationManager已经提前注册到容器中，主要添加的是认证配置

  ```java
  @Configuration
  @EnableWebSecurity
  public class SecureConfig extends WebSecurityConfigurerAdapter {
  
      @Bean
      public PasswordEncoder passwordEncoder() {
          return new BCryptPasswordEncoder();
      }
  
      @Bean
      @Override
      public AuthenticationManager authenticationManagerBean() throws Exception {
          return super.authenticationManagerBean();
      }
  
      @Override
      protected void configure(HttpSecurity http) throws Exception {
          http.csrf()
                  .disable()
                  .authorizeRequests()
                  .antMatchers("/oauth/**", "/login/**", "/logout/**")
                  .permitAll()
                  .anyRequest()
                  .authenticated()
                  .and()
                  .formLogin()
                  .permitAll();
      }
  }
  ```

- 添加需要登录的接口用于测试

  ```java
  @RestController
  @RequestMapping("/user")
  public class UserController {
      @GetMapping("/getCurrentUser")
      public Object getCurrentUser(Authentication authentication) {
          return authentication.getPrincipal();
      }
  }
  ```

#### 12.2.2 授权码模式使用

- 启动oauth2-server服务

- 在浏览器访问该地址进行登录授权：http://localhost:9401/oauth/authorize?response_type=code&client_id=admin&redirect_uri=http://www.baidu.com&scope=all&state=normal

- 输入账号密码进行登录操作

  ![image-20201126172106974](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201126172106974.png)

- 登录之后进行授权操作

  ![image-20201126172219306](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201126172219306.png)

- 浏览器会带着授权码跳转到指定的路径

  ![image-20201126172308660](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201126172308660.png)

- 使用授权码请求该地址获取访问令牌：http://localhost:9401/oauth/token

- 使用Basic认证通过client_id和client_secret构造一个Authorization头信息

  ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16e364d08d92274a)

- 在body中添加以下参数信息，通过POST请求获取访问令牌

  ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16e364d08f299225)

- 在请求头中添加访问令牌，访问需要登录认证的接口进行测试，http://localhost:9401/user/getCurrentUser

  ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16e364d0ad2c3eb7)

## 13 Spring Cloud Security:Oauth2结合JWT使用

### 13.1 概述

#### 13.1.1 Security结合Oauth2

- Spring Cloud Security 为构建安全的SpringBoot应用提供了一系列解决方案，结合Oauth2还可以实现更多功能，比如使用JWT令牌存储信息，刷新令牌功能

#### 13.1.2 JWT

[参考文档]: OAuth2、JWT.md

### 13.2 整合JWT

- 之前是把令牌存储在内存中的，这样如果部署多个服务，就会导致无法使用令牌的问题。 Spring Cloud Security中有两种存储令牌的方式可用于解决该问题，一种是使用Redis来存储，另一种是使用JWT来存储

#### 13.2.1 使用Redis存储令牌

- 添加依赖

  ```xml
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-redis</artifactId>
  </dependency>
  ```

- 配置文件添加redis配置

  ```yaml
  server:
    port: 9401
  spring:
    application:
      name: oauth2-service
    redis:
      host: 123.57.66.144
      port: 6379
      password: nidi1995230
  ```

- 添加redis存储令牌的配置类

  ```java
  @Configuration
  public class ReisTokenStoreConfig {
      @Autowired
      private RedisConnectionFactory redisConnectionFactory;
  
      @Bean
      public TokenStore redisTokenStore() {
          return new RedisTokenStore(redisConnectionFactory);
      }
  }
  ```

- 在认证服务器配置中指定令牌的存储策略为Redis

  - 首先在类中注入了redisTokenStore
  - 之后在endpoints中指定了token存储策略为redis存储

  ```java
  @Configuration
  @EnableAuthorizationServer
  public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {
      @Autowired
      private PasswordEncoder passwordEncoder;
  
      @Autowired
      private AuthenticationManager authenticationManager;
  
      @Autowired
      private UserService userService;
  
      @Autowired
      @Qualifier("redisTokenStore")
      private TokenStore tokenStore;
  
      /**
       * 使用密码模式需要配置
       * @param endpoints
       * @throws Exception
       */
      @Override
      public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
          endpoints.authenticationManager(authenticationManager)
                  .userDetailsService(userService)
                  .tokenStore(tokenStore); // 指定token存储策略
      }
  
      @Override
      public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
          clients.inMemory()
                  .withClient("admin") // 配置client_id
                  .secret(passwordEncoder.encode("123456")) // 配置client_secret
                  .accessTokenValiditySeconds(3600) // 配置token有效期
                  .refreshTokenValiditySeconds(86400) // 设置刷新token有效期
                  .redirectUris("http://www.baidu.com") // 配置登录成功跳转页面
                  .scopes("all") // 设置申请的权限范围
                  .authorizedGrantTypes("authorization_code","password"); // 配置grant_type，表示授权类型
      }
  }
  ```

- 使用授权码方式测试，发现已经存储到redis当中

  ![image-20201126224341906](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201126224341906.png)

#### 13.2.2 使用JWT存储令牌

- 配置JWT内容增强器

  - 可以看出是给token增加了额外的信息

  ```java
  public class JwtTokenEnhancer implements TokenEnhancer {
      @Override
      public OAuth2AccessToken enhance(OAuth2AccessToken accessToken, OAuth2Authentication authentication) {
          Map<String,Object> info = new HashMap<>();
          info.put("enhance","enhance info");
          ((DefaultOAuth2AccessToken) accessToken).setAdditionalInformation(info);
          return accessToken;
      }
  }
  ```

- 添加使用JWT存储令牌的配置类

  - 可以看出先获取JwtAccessTokenConverter并设置秘钥，之后通过JwtAccessTokenConverter生成JwtTokenStore，进而向容器中注入了TokenEnhancer

  ```java
  @Configuration
  public class JwtTokenStoreConfig {
      @Bean
      public JwtAccessTokenConverter jwtAccessTokenConverter() {
          JwtAccessTokenConverter accessTokenConverter = new JwtAccessTokenConverter();
          accessTokenConverter.setSigningKey("test_key"); //配置JWT使用的秘钥
          return accessTokenConverter;
      }
      
      @Bean
      public TokenStore jwtTokenStore() {
          return new JwtTokenStore(jwtAccessTokenConverter());
      }
      
   	@Bean()
      public JwtTokenEnhancer jwtTokenEnhancer() {
          return new JwtTokenEnhancer();
      }
  }
  ```

- 在认证服务器配置中指定令牌的存储策略为JWT

  ```java
  @Configuration
  @EnableAuthorizationServer
  public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {
      @Autowired
      private PasswordEncoder passwordEncoder;
  
      @Autowired
      private AuthenticationManager authenticationManager;
  
      @Autowired
      private UserService userService;
  
      @Autowired
      @Qualifier("jwtTokenStore")
      private TokenStore tokenStore;
  
      @Autowired
      private JwtAccessTokenConverter jwtAccessTokenConverter;
  
      @Autowired
      private JwtTokenEnhancer jwtTokenEnhancer;
  
      /**
       * 使用密码模式需要配置
       * @param endpoints
       * @throws Exception
       */
      @Override
      public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
          endpoints.authenticationManager(authenticationManager)
                  .userDetailsService(userService)
                  .tokenStore(tokenStore) // 指定token存储策略
                  .accessTokenConverter(jwtAccessTokenConverter);
      }
  
      @Override
      public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
          clients.inMemory()
                  .withClient("admin") // 配置client_id
                  .secret(passwordEncoder.encode("123456")) // 配置client_secret
                  .accessTokenValiditySeconds(3600) // 配置token有效期
                  .refreshTokenValiditySeconds(86400) // 设置刷新token有效期
                  .redirectUris("http://www.baidu.com") // 配置登录成功跳转页面
                  .scopes("all") // 设置申请的权限范围
                  .authorizedGrantTypes("authorization_code","password"); // 配置grant_type，表示授权类型
      }
  }
  ```

- 使用授权码方式测试

  - 当获取到授权码时携带授权码访问http://localhost:9401/oauth/token发现已经直接生成了Token，而不需要再通过访问令牌进行访问

    ![image-20201126230929305](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201126230929305.png)

- 从[jwt.io/](https://jwt.io/)网站解析access_token发现已经解析到了对应的内容

  ![image-20201126231159161](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201126231159161.png)

#### 13.2.3 增强JWT中存储的内容

- 添加一个类继承TokenEnhancer实现一个JWT内容增强器(13.2.2中已经提及)

  ![image-20201126231534533](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201126231534533.png)

- 创建一个JwtTokenEnhancer实例(13.2.2中已经提及)

- 在认证服务器中配置JWT内容增强器

  ```java
  @Configuration
  @EnableAuthorizationServer
  public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {
      @Autowired
      private PasswordEncoder passwordEncoder;
  
      @Autowired
      private AuthenticationManager authenticationManager;
  
      @Autowired
      private UserService userService;
  
      @Autowired
      @Qualifier("jwtTokenStore")
      private TokenStore tokenStore;
  
      @Autowired
      private JwtAccessTokenConverter jwtAccessTokenConverter;
  
      @Autowired
      private JwtTokenEnhancer jwtTokenEnhancer;
  
      /**
       * 使用密码模式需要配置
       * @param endpoints
       * @throws Exception
       */
      @Override
      public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
          TokenEnhancerChain enhancerChain = new TokenEnhancerChain();
          List<TokenEnhancer> delegates = new ArrayList<>();
          delegates.add(jwtTokenEnhancer); //配置JWT的内容增强器
          delegates.add(jwtAccessTokenConverter);
          enhancerChain.setTokenEnhancers(delegates);
          endpoints.authenticationManager(authenticationManager)
                  .userDetailsService(userService)
                  .tokenStore(tokenStore) //配置令牌存储策略
                  .accessTokenConverter(jwtAccessTokenConverter)
                  .tokenEnhancer(enhancerChain);
      }
  
      @Override
      public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
          clients.inMemory()
                  .withClient("admin") // 配置client_id
                  .secret(passwordEncoder.encode("123456")) // 配置client_secret
                  .accessTokenValiditySeconds(3600) // 配置token有效期
                  .refreshTokenValiditySeconds(86400) // 设置刷新token有效期
                  .redirectUris("http://www.baidu.com") // 配置登录成功跳转页面
                  .scopes("all") // 设置申请的权限范围
                  .authorizedGrantTypes("authorization_code","password"); // 配置grant_type，表示授权类型
      }
  }
  ```

- 运行项目后使用密码模式来获取令牌，之后对令牌进行解析，发现已经包含扩展的内容

  ```json
  {
    "user_name": "macro",
    "scope": [
      "all"
    ],
    "exp": 1572683821,
    "authorities": [
      "admin"
    ],
    "jti": "1ed1b0d8-f4ea-45a7-8375-211001a51a9e",
    "client_id": "admin",
    "enhance": "enhance info"
  }
  ```

#### 12.2.4 Java中解析JWT中的内容

- 如果需要获取JWT中的信息，可以使用一个叫jjwt的工具包

- 添加依赖

  - **注意:hutool工具包的使用**

  ```xml
  <dependency>
      <groupId>io.jsonwebtoken</groupId>
      <artifactId>jjwt</artifactId>
      <version>0.9.0</version>
  </dependency>
  <!-- https://mvnrepository.com/artifact/cn.hutool/hutool-core -->
  <dependency>
      <groupId>cn.hutool</groupId>
      <artifactId>hutool-core</artifactId>
      <version>5.5.1</version>
  </dependency>
  ```

- 修改UserController类，使用jjwt工具类来解析Authorization头中存储的JWT内容

  ```java
  @RestController
  @RequestMapping("/user")
  public class UserController {
      @GetMapping("/getCurrentUser")
      public Object getCurrentUser(Authentication authentication, HttpServletRequest request) {
          String authorization = request.getHeader("Authorization");
          String token = StrUtil.subAfter(authorization, "bearer ", false);
          return Jwts.parser()
                  .setSigningKey("test_key".getBytes(StandardCharsets.UTF_8))
                  .parseClaimsJws(token)
                  .getBody();
      }
  }
  ```

- 将令牌放入`Authorization`头中，访问如下地址获取信息：http://localhost:9401/user/getCurrentUser

  ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16e40b92f1dba453)

#### 12.2.5 刷新令牌

- 在Spring Cloud Security 中使用oauth2时，如果令牌失效了，可以使用刷新令牌通过refresh_token的授权模式再次获取access_token

- 只需修改认证服务器的配置，添加refresh_token的授权模式即可

  ```java
  @Configuration
  @EnableAuthorizationServer
  public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {
  
      @Override
      public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
          clients.inMemory()
                  .withClient("admin")
                  .secret(passwordEncoder.encode("admin123456"))
                  .accessTokenValiditySeconds(3600)
                  .refreshTokenValiditySeconds(864000)
                  .redirectUris("http://www.baidu.com")
                  .autoApprove(true) //自动授权配置
                  .scopes("all")
                  .authorizedGrantTypes("authorization_code","password","refresh_token"); //添加授权模式
      }
  }
  ```

- 使用刷新令牌模式来获取新的令牌，访问如下地址：http://localhost:9401/oauth/token

  - 第一次访问时不仅获得了access_token，同时也获得了refresh_token

    ![image-20201127010918557](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201127010918557.png)

  - 第二次通过refresh_key获取新的令牌，访问http://localhost:9401/oauth/token
  
    ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16e40b92f1c724c0)

## 13 Nacos(注册中心、配置中心)

### 13.1 概述

Nacos致力于发现、配置和管理微服务，Nacos提供了一组简单易用的特性集，快读实现动态服务发现、服务配置、服务元数据及流量管理

- Nacos特性
  - 服务发现和服务健康监测：支持基于DNS和基于RPC的服务发现，支持对服务的实时的健康检查，阻止向不健康的主机或服务实例发送请求
  - 动态配置服务：动态配置服务可以让您以中心化、外部化和动态化的方式管理所有环境的应用配置和服务配置
  - 动态 DNS 服务：动态 DNS 服务支持权重路由，让您更容易地实现中间层负载均衡、更灵活的路由策略、流量控制以及数据中心内网的简单DNS解析服务
  - 服务及其元数据管理：支持从微服务平台建设的视角管理数据中心的所有服务及元数据

### 13.2 使用Nacos作为注册中心

#### 13.2.1 创建应用注册到Nacos(nacos-user-service)

- 依赖

  如果使用Spring Cloud Alibaba的组件都需要在pom.xml中添加该配置

  ```xml
  <dependencyManagement>
      <dependencies>
          <dependency>
              <groupId>com.alibaba.cloud</groupId>
              <artifactId>spring-cloud-alibaba-dependencies</artifactId>
              <version>2.1.0.RELEASE</version>
              <type>pom</type>
              <scope>import</scope>
          </dependency>
      </dependencies>
  </dependencyManagement>
  ```

  添加Nacos相关依赖

  ```xml
  <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-actuator</artifactId>
          </dependency>
  
          <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-test</artifactId>
              <scope>test</scope>
          </dependency>
  
          <dependency>
              <groupId>com.alibaba.cloud</groupId>
              <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
          </dependency>
  
          <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  ```

- 配置文件

  - spring.cloud.nacos.discovery.server-addr代表配置nacos的注册中心地址

  ```
  server:
    port: 8206
  spring:
    application:
      name: nacos-user-service
    cloud:
      nacos:
        discovery:
          server-addr: localhost:8848 #配置Nacos地址
  management:
    endpoints:
      web:
        exposure:
          include: '*'
  ```

- 添加作为Nacos客户端服务被发现功能

  ```java
  @SpringBootApplication
  @EnableDiscoveryClient
  public class NacosUserServiceApplication {
  
      public static void main(String[] args) {
          SpringApplication.run(NacosUserServiceApplication.class, args);
      }
  
  }
  ```

- 测试

  启动nacos-user-service服务之后，访问http://localhost:8848/nacos/发现服务已存在

  ![image-20201130104347109](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201130104347109.png)

#### 13.2.2 创建模块演示Nacos负载均衡能力(nacos-user-service)

- 依赖

  ```xml
  <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-actuator</artifactId>
          </dependency>
  
          <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-test</artifactId>
              <scope>test</scope>
          </dependency>
  
          <dependency>
              <groupId>com.alibaba.cloud</groupId>
              <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
          </dependency>
  
          <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  
  <dependency>
              <groupId>org.springframework.cloud</groupId>
              <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
              <version>2.2.6.RELEASE</version>
          </dependency>
  ```

- 配置文件

  - service-url.nacos-user-service用来通过@Value注解调用user服务

  ```
  server:
    port: 8308
  spring:
    application:
      name: nacos-ribbon-service
    cloud:
      nacos:
        discovery:
          server-addr: localhost:8848
  service-url:
    nacos-user-service: http://nacos-user-service
  ```

- 添加作为Nacos客户端服务被发现功能

  ```java
  @SpringBootApplication
  @EnableDiscoveryClient
  public class NacosRibbonServiceApplication {
  
      public static void main(String[] args) {
          SpringApplication.run(NacosRibbonServiceApplication.class, args);
      }
  
  }
  ```

- 运行两个nacos-user-service(以不同配置文件启动)和一个nacos-ribbon-service，通过服务管理中心监测

  ![image-20201130110611366](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201130110611366.png)

- 通过nacos-ribbon-service的接口实际调用nacos-user-service的服务观察负载均衡，调用接口：http://localhost:8308/user/1

  ![image-20201130111007990](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201130111007990.png)

  可以看出同一服务名下的两个服务交替调用

### 13.3 使用Nacos作为配置中心

#### 13.3.1 创建模块演示配置管理(nacos-config-client)

- 依赖

  ```xml
  <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter</artifactId>
          </dependency>
          <dependency>
              <groupId>com.alibaba.cloud</groupId>
              <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
          </dependency>
  
          <dependency>
              <groupId>com.alibaba.cloud</groupId>
              <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
          </dependency>
  
          <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-web</artifactId>
          </dependency>
  
          <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-test</artifactId>
              <scope>test</scope>
          </dependency>
  ```

- 配置文件

  添加配置文件application.yml，启用的是dev环境的配置

  ```yaml
  spring:
    profiles:
      active: dev
  ```

  添加配置文件bootstrap.yml，主要是对Nacos的作为配置中心的功能进行配置

  - Nacos同config配置中心一样，需要先从云端配置中心拉取配置，故bootstrap优先级高于application

  - spring.cloud.nacos.config.server-addr表示配置中心的地址，而file-extension代表配置中心文件的类型，可以是properties或者yaml

  ```yaml
  server:
    port: 9101
  spring:
    application:
      name: nacos-config-client
    cloud:
      nacos:
        discovery:
          server-addr: localhost:8848 #Nacos地址
        config:
          server-addr: localhost:8848 #Nacos地址
          file-extension: yaml #这里我们获取的yaml格式的配置
  ```

- 创建ConfigClientController，从Nacos配置中心中获取配置信息

  - config.info代表从配置中心中获取对应的yaml配置文件(配置文件中的config.info)
  - @RefreshScope：SpringCloud原生注解实现配置自动刷新

  ```java
  @RestController
  @RefreshScope
  public class ConfigClientController {
  
      @Value("${config.info}")
      private String configInfo;
  
      @GetMapping("/configInfo")
      public String getConfigInfo() {
          return configInfo;
      }
  }
  ```

#### 13.3.2 在Nacos中添加配置

- Nacos中的dataid组成格式和SpringBoot配置文件的属性关系

  ```reStructuredText
  ${spring.application.name}-${spring.profiles.active}.${spring.cloud.nacos.config.file-extension}
  ```

  比如说我们现在要获取应用名称为`nacos-config-client`的应用在`dev`环境下的`yaml`配置

  ```reStructuredText
  nacos-config-client-dev.yaml
  ```

- 在nacos-config-client-dev.yaml添加配置

  ```yaml
  config:
    info: "config info for dev"
  ```

- 启动nacos-config-client，调用接口查看配置信息：http://localhost:9101/configInfo

  ![image-20201130114010921](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201130114010921.png)

#### 13.3.3 Nacos的动态刷新配置

只要修改下Nacos中的配置信息，再次调用查看配置的接口，就会发现配置已经刷新，Nacos和Consul一样都支持动态刷新配置；在Nacos页面上修改配置并发布后，应用会刷新配置并打印如下信息

```reStructuredText
2019-11-06 14:50:49.460  INFO 12372 --- [-localhost_8848] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.cloud.autoconfigure.ConfigurationPropertiesRebinderAutoConfiguration' of type [org.springframework.cloud.autoconfigure.ConfigurationPropertiesRebinderAutoConfiguration$$EnhancerBySpringCGLIB$$ec395f8e] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2019-11-06 14:50:49.608  INFO 12372 --- [-localhost_8848] c.a.c.n.c.NacosPropertySourceBuilder     : Loading nacos data, dataId: 'nacos-config-client-dev.yaml', group: 'DEFAULT_GROUP'
2019-11-06 14:50:49.609  INFO 12372 --- [-localhost_8848] b.c.PropertySourceBootstrapConfiguration : Located property source: CompositePropertySource {name='NACOS', propertySources=[NacosPropertySource {name='nacos-config-client-dev.yaml'}, NacosPropertySource {name='nacos-config-client.yaml'}]}
2019-11-06 14:50:49.610  INFO 12372 --- [-localhost_8848] o.s.boot.SpringApplication               : The following profiles are active: dev
2019-11-06 14:50:49.620  INFO 12372 --- [-localhost_8848] o.s.boot.SpringApplication               : Started application in 0.328 seconds (JVM running for 172.085)
2019-11-06 14:50:49.638  INFO 12372 --- [-localhost_8848] o.s.c.e.event.RefreshEventListener       : Refresh keys changed: [config.info]
```

#### 13.3.4 Nacos的分类配置

一个大型分布式微服务系统有很多个微服务子模块，而每个微服务又有相应的开发环境、测试环境和正式环境...，Nacos通过命名空间、分组名称和Dataid组成不同的配置文件

![image-20201130205410790](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201130205410790.png)

默认情况：NameSpace=public，Group=DEFAULT_GROUP，默认Cluster为DEFAULT

![](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/Alibaba的30.png)

比如说现在有三个环境，开发、测试和生产环境；可以创建三个NameSpace分别代表三种环境

Group可以把不同的微服务划分到同一个分组中

Service就是微服务，一个Service可以包含多个Cluster(集群)，Cluster是对指定微服务的一个虚拟划分

##### 配置不同的Group_Id

在配置中心配置不同分组的配置文件之后，只需要在微服务的bootstrap.yml中指定group即可

![image-20201130211226947](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201130211226947.png)

##### 配置不同的NameSpace

配置中心新建命名空间时会生成随机的命名空间ID

![image-20201130211621997](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201130211621997.png)

在bootstrap.yml配置文件中指定命名空间

![image-20201130211746245](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201130211746245.png)

### 13.4 Nacos集群部署

Nacos默认自带嵌入式数据库derby，但如果是集群模式就不能使用自带数据库，不然就会导致每个节点一个数据库，数据不统一，因此需要使用外部mysql数据库

#### 13.4.1 目标架构

![image-20201130213152078](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201130213152078.png)

#### 13.4.2 从derby数据库转向mysql

- 在nacos-server/conf找到nacos-mysql.sql脚本文件，创建nacos_config数据库，并在数据库中执行脚本

  ![image-20201130213550381](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201130213550381.png)

  ![image-20201130213842589](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201130213842589.png)

- 在nacos-server-1.1.4\nacos\conf目录下找到application.properties，在文件最后追加msyql数据源配置

  ```properties
  spring.datasource.platform=mysql
  
  db.num=1
  db.url.0=jdbc:mysql://123.57.66.144:3306/nacos_config?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true
  db.user=root
  db.password=youdontknow
  ```

- 重启Nacos

  当在配置中心新建一个配置文件时

  ![image-20201130214854606](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201130214854606.png)

  mysql数据库就会更新该配置信息

  ![image-20201130214927773](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201130214927773.png)

#### 13.4.3 Nacos集群部署

# 14 Sentinel(熔断与限流)

### 14.1 概述

随着微服务的流行，服务之间的稳定性变得越来越重要，Sentinel以流量为切入点，从流量控制、熔断降级、系统负载保护等多个维度保护服务的稳定性

Sentinel特性

- 丰富的应用场景：承接了阿里巴巴近 10 年的双十一大促流量的核心场景，例如秒杀，可以实时熔断下游不可用应用
- 完备的实时监控：同时提供实时的监控功能，可以在控制台中看到接入应用的单台机器秒级数据，甚至 500 台以下规模的集群的汇总运行情况
- 广泛的开源生态：提供开箱即用的与其它开源框架/库的整合模块，例如与 Spring Cloud、Dubbo、gRPC 的整合
- 完善的 SPI 扩展点：提供简单易用、完善的 SPI 扩展点，您可以通过实现扩展点，快速的定制逻辑

![image-20201130114910079](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201130114910079.png)

#### 14.2 安装Sentinel控制台

Sentinel控制台是一个轻量级的控制台应用，它可用于实时查看单机资源监控及集群资源汇总，并提供了一系列的规则管理功能，如流控规则、降级规则、热点规则等

- 官网下载Sentinel控制台，地址：https://github.com/alibaba/Sentinel/releases

- 终端输入命令运行控制台

  ```bash
  java -jar sentinel-dashboard-1.8.0.jar
  ```

- Sentinel控制台默认运行在8080端口上，登录账号密码均为`sentinel`，通过如下地址可以进行访问：[http://localhost:8080](http://localhost:8080/)

  ![image-20201130121656512](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201130121656512.png)

- Sentinel控制台可以查看单台机器的实时监控数据

  ![image-20201130221225530](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201130221225530.png)

### 14.3 演示Sentinel熔断和限流功能

#### 14.3.1 创建模块演示注册到Sentinel(sentinel-service)

- 添加依赖，使用nacos作为注册中心

  ```xml
  <dependency>
              <groupId>com.alibaba.cloud</groupId>
              <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
          </dependency>
          <dependency>
              <groupId>com.alibaba.cloud</groupId>
              <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
          </dependency>
          <dependency>
              <groupId>org.springframework.cloud</groupId>
              <artifactId>spring-cloud-starter-openfeign</artifactId>
          </dependency>
          <dependency>
              <groupId>com.alibaba.csp</groupId>
              <artifactId>sentinel-datasource-nacos</artifactId>
          </dependency>
          <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-actuator</artifactId>
          </dependency>
          <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-web</artifactId>
          </dependency><dependencies>
          <dependency>
              <groupId>com.alibaba.cloud</groupId>
              <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
          </dependency>
          <dependency>
              <groupId>com.alibaba.cloud</groupId>
              <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
          </dependency>
          <dependency>
              <groupId>org.springframework.cloud</groupId>
              <artifactId>spring-cloud-starter-openfeign</artifactId>
          </dependency>
          <dependency>
              <groupId>com.alibaba.csp</groupId>
              <artifactId>sentinel-datasource-nacos</artifactId>
          </dependency>
          <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-actuator</artifactId>
          </dependency>
          <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-web</artifactId>
          </dependency>
          <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-test</artifactId>
              <scope>test</scope>
          </dependency>
          <dependency>
              <groupId>cn.hutool</groupId>
              <artifactId>hutool-all</artifactId>
              <version>4.6.3</version>
          </dependency>
      </dependencies>
  ```

- 配置文件，主要配置Nacos和Sentinel控制台的地址

  - spring.cloud.sentinel.transport.dashboard配置dashborad地址，port配置sentinel的端口号

  ```yaml
  server:
    port: 8401
  spring:
    application:
      name: sentinel-service
    cloud:
      nacos:
        discovery:
          server-addr: localhost:8848 #配置Nacos地址
      sentinel:
        transport:
          dashboard: localhost:8080 #配置sentinel dashboard地址
          port: 8719 #默认8719，假如被占用了会自动从8719开始依次+1扫描。直至找到未被占用的端口
  service-url:
  	user-service: http://nacos-user-service       
  management:
    endpoints:
      web:
        exposure:
          include: "*"
  ```

- 测试

  当启动sentinel-service服务时，发现Sentinel Dashboard并没有出现对sentinel-service服务的监控

  原因是Sentinel采用懒加载的方式，只有当该服务被调用时才会加载到Dashboard页面

  ![image-20201130130422586](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201130130422586.png)

### 14.3.2 演示Sentinel限流功能

##### 流量控制规则

![image-20201130221312641](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201130221312641.png)

![](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/sentinel的4.png)

Sentinel Starter 默认为所有的 HTTP 服务提供了限流埋点，同样可以通过使用@SentinelResource来自定义一些限流行为

- 配置Ribbon负载均衡

  ```java
  package org.jiang.config;
  @Configuration
  public class RibbonConfig {
  
      @Bean
      @SentinelRestTemplate
      public RestTemplate restTemplate(){
          return new RestTemplate();
      }
  }
  ```

- 超过限流出现异常时使用自定义异常处理类CustomBlockHandler

  ```java
  package org.jiang.handler;
  public class CustomBlockHandler {
  
      public CommonResult handleException(BlockException exception){
          return new CommonResult("自定义限流信息",200);
      }
  }
  ```

- 创建RateLimitController测试熔断限流功能

  ```java
  @RestController
  @RequestMapping("/rateLimit")
  public class RateLimitController {
  
      /**
       * 按资源名称限流，需要指定限流处理逻辑
       */
      @GetMapping("/byResource")
      @SentinelResource(value = "byResource",blockHandler = "handleException")
      public CommonResult byResource() {
          return new CommonResult("按资源名称限流", 200);
      }
  
      /**
       * 按URL限流，有默认的限流处理逻辑
       */
      @GetMapping("/byUrl")
      @SentinelResource(value = "byUrl",blockHandler = "handleException")
      public CommonResult byUrl() {
          return new CommonResult("按url限流", 200);
      }
  
      /**
       * 自定义通用的限流处理逻辑
       */
      @GetMapping("/customBlockHandler")
      @SentinelResource(value = "customBlockHandler", blockHandler = "handleException",blockHandlerClass = CustomBlockHandler.class)
      public CommonResult blockHandler() {
          return new CommonResult("限流成功", 200);
      }
  
      public CommonResult handleException(BlockException exception){
          return new CommonResult(exception.getClass().getCanonicalName(),200);
      }
  
  }
  ```


##### 按照资源名称限流

根据@SentinelResource注解中定义的value(资源名称)来进行限流操作，并指定限流处理逻辑规则

- 在Sentinel控制台配置流控规则，根据@SentinelResource注解的value值

  ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16e7eb0f2553a327)

- 当超过每秒钟1次访问接口之后，发现返回了自定义的限流处理信息

  接口：http://localhost:8401/rateLimit/byResource

  ![image-20201130222740441](SpringCloud.assets/image-20201130222740441.png)

##### 根据URL限流

- 在Sentinel控制台配置流控规则，使用访问的URL

  ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16e7eb0f89604604)

当超过每秒钟1次访问接口之后，发现返回了自定义的限流处理信息

接口：http://localhost:8401/rateLimit/byUrl

![image-20201130223910879](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201130223910879.png)

##### 关联

当与一个接口A关联的接口B达到了域值，就对A进行限流；比如说支付模块达到了域值，就对订单模块进行限流，防止新的订单产生

- 在Sentinel控制台配置流控规则，

  ![image-20201130223428807](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201130223428807.png)

- 当byResource资源达到域值之后，byUrl资源返回了自定义的限流处理信息

  ![image-20201130223918172](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201130223918172.png)

#### 14.3.3 演示Sentinel降级功能

##### 降级规则

**Sentinel的断路器没有半开状态**(半开状态系统自动去检测请求是否有异常，没有异常就关闭断路器恢复使用，有异常就继续打开断路器不可用)

![image-20201130224636924](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201130224636924.png)

- RT(平均响应时间，秒级)

  平均响应时间**超出域值**且在**时间窗口内通估计的请求>=5**两个条件同时满足后出发降级

  窗口期过后关闭断路器(就是窗口期类的请求平均响应时间小于域值)

  RT最大为4900(更大的需要通过-Dcsp.sentinel.static.max.rt=xxxx才能生效)

- 异常比例

  QPS》=5且异常比例(秒级统计)超过域值后出发降级；时间窗口结束后关闭降级

- 异常数

  异常数(分钟统计)超过阈值后出发降级；时间窗口结束后关闭降级

##### 配置RT

![image-20201130230306269](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201130230306269.png)

降级规则当一秒钟请求数量超过5个，最大的响应时间超过200毫秒时就发生服务熔断，熔断时间为1秒钟

##### 配置异常比例

![image-20201130230929603](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201130230929603.png)

降级规则为一秒钟内请求的数据超过5个，且异常请求的比例超过50%就发生服务熔断，熔断时间为1秒钟

##### 配置异常数

![image-20201130231236061](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201130231236061.png)

降级规则为一秒钟内请求的数据超过5个，且异常请求的数量超过3个就发生服务熔断，熔断时间为1秒钟

#### 14.3.4 Sentinel热点限

##### 热点

经常访问的数据，大多数时候希望统计某个热点数据中访问频次最高的Top K数据，并对其访问进行限制

- 商品ID为参数，统计一段时间内最长购买的商品ID进行限制
- 用户ID为参数，统计一段时间内频繁访问的客户ID进行限制

##### 配置普通热点限流

![image-20201130232501861](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201130232501861.png)

限流规则请求的资源且添加了第0个索引(类似于www.baidu.com/query，query就是第0个参数)，在一秒钟之内请求的次数超过1次，则会进行限流

注意：如果没有配置blockHandler方法，就会返回error_page页面

```java
@SentinelResource(value = "byUrl",blockHandler = "handleException")
```

##### 配置特别限流规则(配置参数例外项)

![image-20201130233203224](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201130233203224.png)

当第0个参数的值为5时限流规则就发生了改变，只有一秒钟之内请求的次数超过了200次才会进行限流

#### 14.3.5 自定义限流处理逻辑

通过@SentinelResource可以指定自定义通用的限流处理逻辑

- 创建CustomBlockHandler类用于自定义限流处理逻辑

```java
public class CustomBlockHandler {

    public CommonResult handleException(BlockException exception){
        return new CommonResult("自定义限流信息",200);
    }
}
```

- 在RateLimitController中使用自定义限流处理逻辑

  在@SentinelResource注解中首先指定了资源名为customBlockHandler，进而指定了限流处理的类为CustomBlockHandler，限流处理方法为handleException

  ```java
  @RestController
  @RequestMapping("/rateLimit")
  public class RateLimitController {
  
      /**
       * 自定义通用的限流处理逻辑
       */
      @GetMapping("/customBlockHandler")
      @SentinelResource(value = "customBlockHandler", blockHandler = "handleException",blockHandlerClass = CustomBlockHandler.class)
      public CommonResult blockHandler() {
          return new CommonResult("限流成功", 200);
      }
  
  }
  ```

#### 14.3.6 Sentinel熔断功能

Sentinel支持对服务间调用进行保护，对故障应用进行熔断操作

##### 通过RestTemplate来调用nacos-user-service服务所提供的接口进行演示

- 使用@SentinelRestTemplate包装RestTemplate实例

  ```java
  @Configuration
  public class RibbonConfig {
  
      @Bean
      @SentinelRestTemplate
      public RestTemplate restTemplate(){
          return new RestTemplate();
      }
  }
  ```

- 定义CircleBreakerController类，定义对nacos-user-service提供接口的调用

  ```java
  /**
   * 熔断功能
   * Created by macro on 2019/11/7.
   */
  @RestController
  @RequestMapping("/breaker")
  public class CircleBreakerController {
  
      private Logger LOGGER = LoggerFactory.getLogger(CircleBreakerController.class);
      @Autowired
      private RestTemplate restTemplate;
      @Value("${service-url.user-service}")
      private String userServiceUrl;
  
      @RequestMapping("/fallback/{id}")
      @SentinelResource(value = "fallback",fallback = "handleFallback")
      public CommonResult fallback(@PathVariable Long id) {
          return restTemplate.getForObject(userServiceUrl + "/user/{1}", CommonResult.class, id);
      }
  
      @RequestMapping("/fallbackException/{id}")
      @SentinelResource(value = "fallbackException",fallback = "handleFallback2", exceptionsToIgnore = {NullPointerException.class})
      public CommonResult fallbackException(@PathVariable Long id) {
          if (id == 1) {
              throw new IndexOutOfBoundsException();
          } else if (id == 2) {
              throw new NullPointerException();
          }
          return restTemplate.getForObject(userServiceUrl + "/user/{1}", CommonResult.class, id);
      }
  
      public CommonResult handleFallback(Long id) {
          User defaultUser = new User(-1L, "defaultUser", "123456");
          return new CommonResult<>(defaultUser,"服务降级返回",200);
      }
  
      public CommonResult handleFallback2(@PathVariable Long id, Throwable e) {
          LOGGER.error("handleFallback2 id:{},throwable class:{}", id, e.getClass());
          User defaultUser = new User(-2L, "defaultUser2", "123456");
          return new CommonResult<>(defaultUser,"服务降级返回",200);
      }
  }
  ```

- 启动nacos-user-service和sentinel-service服务

- 由于我们并没有在nacos-user-service中定义id为4的用户，所有访问如下接口会返回服务降级    请求地址：http://localhost:8401/breaker/fallback/4

  ```java
  {
  	"data": {
  		"id": -1,
  		"username": "defaultUser",
  		"password": "123456"
  	},
  	"message": "服务降级返回",
  	"code": 200
  }
  ```

- 当使用了exceptionsToIgnore参数忽略了NullPointerException，所以我们访问接口报空指针时不会发生服务降级：http://localhost:8401/breaker/fallbackException/2

  ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16e7eb0fb3df7c01)

#### 14.3.7 @SentinelResource中的fallback和brokerHandler对比

##### fallback

当程序运行时出现了运行时异常，如果指定了fallback对应的兜底方法，就会按照兜底方法返回数据；如果没有配置就返回错误页面

##### fallback和brokerHandler同时存在

当程序运行时出现运行时异常，而且没有达到sentinel服务端配置的降级要求，就会按照fallback方法执行；一旦达到降级要求，就按照brokerHandler指定的方法执行

#### 14.3.8 Sentinel和Feign结合

- 在sentinel-service服务上添加依赖

  ```xml
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-openfeign</artifactId>
  </dependency>
  ```

- 修改配置文件打开sentinel对feign的支持

  ```yaml
  feign:
    sentinel:
      enabled: true #打开sentinel对feign的支持
  ```

- 在应用启动类上添加@EnableFeignClients启动Feign的功能

- 创建一个UserService接口，用于定义对nacos-user-service服务的调用

  ```java
  @FeignClient(value = "nacos-user-service",fallback = UserFallbackService.class)
  public interface UserService {
      @PostMapping("/user/create")
      CommonResult create(@RequestBody User user);
  
      @GetMapping("/user/{id}")
      CommonResult<User> getUser(@PathVariable Long id);
  
      @GetMapping("/user/getByUsername")
      CommonResult<User> getByUsername(@RequestParam String username);
  
      @PostMapping("/user/update")
      CommonResult update(@RequestBody User user);
  
      @PostMapping("/user/delete/{id}")
      CommonResult delete(@PathVariable Long id);
  }
  ```

- 创建UserFallbackService类实现UserService接口，用于处理服务降级逻辑

  注意：**@Component注解一定要加上，将组建注册到容器中**

  ```java
  @Component
  public class UserFallbackService implements UserService {
      @Override
      public CommonResult create(User user) {
          User defaultUser = new User(-1L, "defaultUser", "123456");
          return new CommonResult<>(defaultUser,"服务降级返回",200);
      }
  
      @Override
      public CommonResult<User> getUser(Long id) {
          User defaultUser = new User(-1L, "defaultUser", "123456");
          return new CommonResult<>(defaultUser,"服务降级返回",200);
      }
  
      @Override
      public CommonResult<User> getByUsername(String username) {
          User defaultUser = new User(-1L, "defaultUser", "123456");
          return new CommonResult<>(defaultUser,"服务降级返回",200);
      }
  
      @Override
      public CommonResult update(User user) {
          return new CommonResult("调用失败，服务被降级",500);
      }
  
      @Override
      public CommonResult delete(Long id) {
          return new CommonResult("调用失败，服务被降级",500);
      }
  }
  ```

- 在UserFeignController中使用UserService通过Feign调用nacos-user-service服务中的接口

  ```
  @RestController
  @RequestMapping("/user")
  public class UserFeignController {
      @Autowired
      private UserService userService;
  
      @GetMapping("/{id}")
      public CommonResult getUser(@PathVariable Long id) {
          return userService.getUser(id);
      }
  
      @GetMapping("/getByUsername")
      public CommonResult getByUsername(@RequestParam String username) {
          return userService.getByUsername(username);
      }
  
      @PostMapping("/create")
      public CommonResult create(@RequestBody User user) {
          return userService.create(user);
      }
  
      @PostMapping("/update")
      public CommonResult update(@RequestBody User user) {
          return userService.update(user);
      }
  
      @PostMapping("/delete/{id}")
      public CommonResult delete(@PathVariable Long id) {
          return userService.delete(id);
      }
  }
  ```

- 调用http://localhost:8401/user/4接口会发生服务降级，返回服务降级处理信息

  ```json
  {
  	"data": {
  		"id": -1,
  		"username": "defaultUser",
  		"password": "123456"
  	},
  	"message": "服务降级返回",
  	"code": 200
  }
  ```

#### 14.3.9 使用Nacos持久化存储Sentinel配置规则

默认情况下，当我们在Sentinel控制台中配置规则时，控制台推送规则方式是通过API将规则推送至客户端并直接更新到内存中。一旦我们重启应用，规则将消失

##### 原理

![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16e7eb102ba54d69)

- 首先直接在配置中心创建规则，配置中心将规则推送到客户端
- Sentinel控制台也从配置中心去获取配置信息

##### 持久化功能演示(在sentinel-service服务演示)

- 添加依赖

  ```xml
  <dependency>
      <groupId>com.alibaba.csp</groupId>
      <artifactId>sentinel-datasource-nacos</artifactId>
  </dependency>
  ```

- 修改配置文件，添加Nacos数据源配置

  ds1代表datasource1，server-addr就是nacos服务器地址，dataID、groupId和data-type组合起来就对应着一个配置文件，rule-tpype代表的是规则

  ```yaml
  spring:
    cloud:
      sentinel:
        datasource:
          ds1:
            nacos:
              server-addr: localhost:8848
              dataId: ${spring.application.name}-sentinel
              groupId: DEFAULT_GROUP
              data-type: json
              rule-type: flow
  ```

- Nacos中添加配置

  ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16e7eb10215f0bce)

- 配置信息

  ```json
  [
      {
          "resource": "/rateLimit/byUrl",
          "limitApp": "default",
          "grade": 1,
          "count": 1,
          "strategy": 0,
          "controlBehavior": 0,
          "clusterMode": false
      }
  ]
  ```

  - resource：资源名称

  - limitApp：来源应用

  - grade：阈值类型，0表示线程数，1表示QPS

  - count：单机阈值

  - strategy：流控模式，0表示直接，1表示关联，2表示链路

  - controlBehavior：流控效果，0表示快速失败，1表示Warm Up，2表示排队等待

  - clusterMode：是否集群

- 再次登录sentinel控制台已经有了对应限流规则

  ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16e7eb103687e544)