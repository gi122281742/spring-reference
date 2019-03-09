# Spring Cloud Commons: Common Abstractions

类似service discovery，load balancing， circuit breakers等模式适用于可以被所有sc client消费的抽象层，独立于实现（比如通过Eureka or Consul发现）。

## 3.1 @EnableDiscoveryClient

sc 公共提供@EnableDiscoveryClient注解。他使用META-INF/spring.factories寻找DiscoveryClient 接口的实现。Discovery Client的实现添加一个配置类给spring.factories，在org.springframework.cloud.client.discovery.EnableDiscoveryClien这个key 下。
DiscoveryClient 实现类的例子包括了Spring Cloud Netflix Eureka, Spring Cloud Consul Discovery以及Spring Cloud Zookeeper Discovery.
默认，DiscoveryClient 实现类使用远程发现服务器自动注册到本地 sb server。这个行为可以通过在@EnableDiscoveryClient.设置autoRegister=false禁用。

> @EnableDiscoveryClient不再需要。你可以放置一个DiscoveryClient 实现类在classpath，来引导sb程序使用服务发现服务器注册。

### 3.1.1  Health Indicator

公用创建一个DiscoveryClient 实现类 可以通过实现DiscoveryHealthIndicator 参加的HealthIndicator 。为了禁用HealthIndicator，设置spring.cloud.discovery.client.composite-indicator.enabled=false。一个基于DiscoveryClient的通用HealthIndicator  自动配置（DiscoveryClientHealthIndicator）。禁用他，设置spring.cloud.discovery.client.health-indicator.enabled=false.禁用DiscoveryClientHealthIndicator的filed的描述，设置spring.cloud.discovery.client.health-indicator.include-description=false。否则，它可以像卷起的HealthIndicator的描述一样冒出来。
### 3.1.2 Ordering DiscoveryClient instances

DiscoveryClient 接口继承了Ordered。但你使用多个discovery 有用。它允许你定义返回discovery client的顺序，和你定义bean加载顺序一样。默认，DiscoveryClient 的order为0.如果你希望设置不同顺序，只需要重写getOrder()方法。除此之外，你可以使用属性来设置通过sc的DiscoveryClient 的实现类提供的order。比如ConsulDiscoveryClient，EurekaDiscoveryClient ，ZookeeperDiscoveryClient。可以设置spring.cloud.{clientIdentifier}.discovery.order属性。（或者eureka.client.order）属性来设置。

## 3.2 ServiceRegistry

公共现在提供了一个ServiceRegistry 接口，它提供了register(Registration) 和 deregister(Registration)方法。它使你提供定制的注册服务。Registration 是一个标记接口。
eg:
```java
@Configuration
@EnableDiscoveryClient(autoRegister=false)
public class MyConfiguration {
    private ServiceRegistry registry;

    public MyConfiguration(ServiceRegistry registry) {
        this.registry = registry;
    }

    // called through some external process, such as an event or a custom actuator endpoint
    public void register() {
        Registration registration = constructRegistration();
        this.registry.register(registration);
    }
}
```
每个ServiceRegistry 实现类都有他自己的Registry 实现类
- ZookeeperRegistration 使用ZookeeperServiceRegistry
- EurekaRegistration 使用EurekaServiceRegistry
- ConsulRegistration 使用ConsulServiceRegistry

如果你使用ServiceRegistry 接口，你需要使用正确的ServiceRegistry 的Registry 实现类。

### 3.2.1 ServiceRegistry Auto-Registration
默认，ServiceRegistry 实现类自动注册 正在运行的服务。禁用此行为，设置@EnableDiscoveryClient(autoRegister=false)来永久禁用自动注册。spring.cloud.service-registry.auto-registration.enabled=false来通过配置禁用此行为。

> ServiceRegistry Auto-Registration Events
存在两个当service自动注册时会触发的事件，一个是InstancePreRegisteredEvent，在service 注册前。InstanceRegisteredEvent在service注册之后触发。你可以注册一个ApplicationListener来监听并响应这些事件。

这些事件在spring.cloud.service-registry.auto-registration.enabled为false时不会触发。

### 3.2.2  Service Registry Actuator Endpoint
sc 公用提供/service-registry 执行器端点。这个端点依赖于Spring application context的Registration bean。通过GET调用/service-registry，返回Registration的状态。POST一个json数据给这个端点可以改变当前的Registration 的状态。json body需要包含status 字段以及期望值。看ServiceRegistry 实现类的文档，你可以看到更新状态时允许的值，以及状态的返回值。比如Eureka’s 支持的状态有UP, DOWN, OUT_OF_SERVICE, 和 UNKNOWN。
## 3.3 Spring RestTemplate as a Load Balancer Client
RestTemplate 可以使用ribbon自动配置。创建一个负载均衡的RestTemplate，创建一个RestTemplate @Bean 以及使用@LoadBalanced限定符。eg：
```java
@Configuration
public class MyConfiguration {

    @LoadBalanced
    @Bean
    RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

public class MyClass {
    @Autowired
    private RestTemplate restTemplate;

    public String doOtherStuff() {
        String results = restTemplate.getForObject("http://stores/stores", String.class);
        return results;
    }
}
```
> 注意：一个RestTemplate  Bean不再通过自动配置创建。独立的application必须创建他。

## 3.4 Spring WebClient as a Load Balancer Client

WebClient 可以使用LoadBalancerClient自动配置。创建一个负载均衡WebClient，需要创建一个WebClient.Builder @Bean以及使用@LoadBalanced限定符。eg：
```java
@Configuration
public class MyConfiguration {

	@Bean
	@LoadBalanced
	public WebClient.Builder loadBalancedWebClientBuilder() {
		return WebClient.builder();
	}
}

public class MyClass {
    @Autowired
    private WebClient.Builder webClientBuilder;

    public Mono<String> doOtherStuff() {
        return webClientBuilder.build().get().uri("http://stores/stores")
        				.retrieve().bodyToMono(String.class);
    }
}
```
这个URL需要使用虚拟host名（那是一个服务名，不是host名）。Ribbon client被用来创建一个全物理地址。
### 3.4.1 Retrying Failed Requests

一个负载均衡的RestTemplate 可以配置重试失败请求。默认，这个login被禁用。你可以通过添加 Spring Retry到类路径来启动它。这个 load-balanced RestTemplate 表现出一些与重试失败请求相关的Ribbon配置值。你可以使用client.ribbon.MaxAutoRetries，client.ribbon.MaxAutoRetriesNextServer, and client.ribbon.OkToRetryOnAllOperations属性。如果你想在Spring Retry在类路径时，禁用这个重试logic，你可以设置spring.cloud.loadbalancer.retry.enabled=false。查看 Ribbon文档相关描述。
如果你希望在重试时继承BackOffPolicy 策略，你需要创建一个LoadBalancedRetryFactory 类型的bean，并且重写createBackOffPolicy 方法。
```java
@Configuration
public class MyConfiguration {
    @Bean
    LoadBalancedRetryFactory retryFactory() {
        return new LoadBalancedRetryFactory() {
            @Override
            public BackOffPolicy createBackOffPolicy(String service) {
        		return new ExponentialBackOffPolicy();
        	}
        };
    }
}
```
client 在之前的例子需要替换ribbon client名称。

如果你希望添加一个或者更多RetryListener 实现类在你的重试功能中，你需要创建一个LoadBalancedRetryListenerFactory bean，并且返回你希望用来提供服务的RetryListener 数组。eg：
```java
@Configuration
public class MyConfiguration {
    @Bean
    LoadBalancedRetryListenerFactory retryListenerFactory() {
        return new LoadBalancedRetryListenerFactory() {
            @Override
            public RetryListener[] createRetryListeners(String service) {
                return new RetryListener[]{new RetryListener() {
                    @Override
                    public <T, E extends Throwable> boolean open(RetryContext context, RetryCallback<T, E> callback) {
                        //TODO Do you business...
                        return true;
                    }

                    @Override
                     public <T, E extends Throwable> void close(RetryContext context, RetryCallback<T, E> callback, Throwable throwable) {
                        //TODO Do you business...
                    }

                    @Override
                    public <T, E extends Throwable> void onError(RetryContext context, RetryCallback<T, E> callback, Throwable throwable) {
                        //TODO Do you business...
                    }
                }};
            }
        };
    }
}
```
## 3.5 Multiple RestTemplate objects
如果你需要一个不是负载均衡的RestTemplate ，创建一个RestTemplate  bean并且注入它。为了访问这个负载均衡的RestTemplate，创建@bean时使用@LoadBalanced 标识符，eg：
```java
@Configuration
public class MyConfiguration {

    @LoadBalanced
    @Bean
    RestTemplate loadBalanced() {
        return new RestTemplate();
    }

    @Primary
    @Bean
    RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

public class MyClass {
    @Autowired
    private RestTemplate restTemplate;

    @Autowired
    @LoadBalanced
    private RestTemplate loadBalanced;

    public String doOtherStuff() {
        return loadBalanced.getForObject("http://stores/stores", String.class);
    }

    public String doStuff() {
        return restTemplate.getForObject("http://example.com", String.class);
    }
}
```
> 使用@Primary在普通RestTemplate 上表明在上一个例子消除了不合格的@Autowired 注入歧义。

> 如果你看见类似java.lang.IllegalArgumentException: Can not set org.springframework.web.client.RestTemplate field com.my.app.Foo.restTemplate to com.sun.proxy.$Proxy89的错误，尝试注入RestOperations 或者设置spring.aop.proxyTargetClass=true。

## 3.6 Spring WebFlux WebClient as a Load Balancer Client

WebClient 可以使用LoadBalancerClient来配置。LoadBalancerExchangeFilterFunction 是自动配置的当spring-webflux在classpath时。接下来的例子表明了如何配置WebClient 负载均衡:
```java
public class MyClass {
    @Autowired
    private LoadBalancerExchangeFilterFunction lbFunction;

    public Mono<String> doOtherStuff() {
        return WebClient.builder().baseUrl("http://stores")
            .filter(lbFunction)
            .build()
            .get()
            .uri("/stores")
            .retrieve()
            .bodyToMono(String.class);
    }
}
```
URL需要使用虚拟host（service名而不是host名）。LoadBalancerClient 用来创建全物理地址名。
## 3.7 Ignore Network Interfaces
有时，忽略某些命令的网络接口有用，以至于他们可以从Service Discovery registration中除去（比如，运行在docker容器时）。许多正则表达式可以用来设置希望被ignore的网络接口。下面的配置忽略了docker0接口以及所有的以veth开头的接口。

**application.yml.**
```
spring:
  cloud:
    inetutils:
      ignoredInterfaces:
        - docker0
        - veth.* 
```
你也可以使用一些正则表达式强制只有指定的网络接口，eg：
```
spring:
  cloud:
    inetutils:
      preferredNetworks:
        - 192.168
        - 10.0
```
你可以强制仅使用site-local地址，eg:
```
spring:
  cloud:
    inetutils:
      useOnlySiteLocalInterfaces: true
```
查看 Inet4Address.html.isSiteLocalAddress()更多细节关于 site-local address的构成。
## 3.8 HTTP Client Factories
sc common提供了一个bean用来创建apache HTTP 客户端 (ApacheHttpClientFactory)以及OKHTTP客户端(OkHttpClientFactory)。OkHttpClientFactory  bean只有OK HTTP在classpath才会被创建。除此之外，sc common 提供了bean用来创建连接管理所有的clients：
ApacheHttpClientConnectionManagerFactory 针对Apache HTTP client，OkHttpClientConnectionPoolFactory 针对OK HTTP客户端。如果你需要定制HTTP clients如何在下游项目创建，你可以提供你的这些bean的实现。如果你提供了一个HttpClientBuilder 或者OkHttpClient.Builder类型的bean，默认的工厂使用这些builder作为返回给下游项目的builder的基础。你也可以禁用这些bean的创建通过设置spring.cloud.httpclientfactories.apache.enabled or spring.cloud.httpclientfactories.ok.enabled to false.
## 3.9 Enabled Features
sc common 提供了/features 执行器端点。这个端点返回了类路径可用的功能不管他们是否启用。这个信息返回了功能 的type, name, version, and vendor。
### 3.9.1 Feature types
有两个特点类型：abstract 和 named。
Abstract  是在一个接口或者抽象类被定义，并且有一个实现类被创建，比如DiscoveryClient, LoadBalancerClient, or LockService。抽象类或者接口被用来发现在context中该type 的 bean。版本展示是bean.getClass().getPackage().getImplementationVersion()。

named 是 不需要实现特定的类，比如"Circuit Breaker", "API Gateway", "Spring Cloud Bus", and others. 这些特点要求一个名字和bean的类型。
### 3.9.2 Declaring features

任何模组都可以声明任何数量的HasFeature ，比如：
```
@Bean
public HasFeatures commonsFeatures() {
  return HasFeatures.abstractFeatures(DiscoveryClient.class, LoadBalancerClient.class);
}

@Bean
public HasFeatures consulFeatures() {
  return HasFeatures.namedFeatures(
    new NamedFeature("Spring Cloud Bus", ConsulBusAutoConfiguration.class),
    new NamedFeature("Circuit Breaker", HystrixCommandAspect.class));
}

@Bean
HasFeatures localFeatures() {
  return HasFeatures.builder()
      .abstractFeature(Foo.class)
      .namedFeature(new NamedFeature("Bar Feature", Bar.class))
      .abstractFeature(Baz.class)
      .build();
}
```
其中每一个bean都需要进入一个适当的@Configuration保护。

### 3.10 Spring Cloud Compatibility Verification

由于一些用户有一些关于设置sc 应用程序的问题，我们决定添加一个兼容性验证机制。当你的当前设置不兼容sc 要求时它将会中断，伴随着一个记录，标记着什么出错了。
记录 eg:
```
***************************
APPLICATION FAILED TO START
***************************

Description:

Your project setup is incompatible with our requirements due to following reasons:

- Spring Boot [2.1.0.RELEASE] is not compatible with this Spring Cloud release train


Action:

Consider applying the following actions:

- Change Spring Boot version to one of the following versions [1.2.x, 1.3.x] .
You can find the latest Spring Boot versions here [https://spring.io/projects/spring-boot#learn].
If you want to learn more about the Spring Cloud Release train compatibility, you can visit this page [https://spring.io/projects/spring-cloud#overview] and check the [Release Trains] section.
```
为了禁用这个功能，可以设置
spring.cloud.compatibility-verifier.enabled to false。如果你需要覆盖这个兼容sb版本，只需要设置spring.cloud.compatibility-verifier.compatible-boot-versions属性用逗号分割的一组兼容sb 版本。