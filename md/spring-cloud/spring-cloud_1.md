# Features
  spring cloud 关注于为典型用例提供较好的开箱即用，以及可扩展去覆盖他们。
- Distributed/versioned configuration
- Service registration and discovery
- Routing
- Service-to-service calls
- Load balancing
- Circuit Breakers
- Distributed messaging
  
  # I.Cloud Native Applications
    Cloud  Native是一种开发风格，它鼓励在持续开发和价值驱动开发中采用最好的时间。一个相关的学科是建立12个应用程序。在这个开发实践中，是与交付和作业目标一致的。比如，使用声明式的程序，管理以及监控。
    Spring Cloud在一些特定方式促进这种风格。起点是一套功能，所有的分布式系统都能够轻松访问。
    许多功能在sc中都被sb覆盖了。一些更多的功能被发布在两个库：Spring Cloud Context and Spring Cloud Commons。Spring Cloud Context在sc应用程序中为ApplicationContext提供了公用和特定的服务（比如bootstrap context, encryption, refresh scope, and environment endpoints），Spring Cloud Commons则是一套抽象的共用类，使用不同的sc实现（比如Spring Cloud Netflix and Spring Cloud Consul）。
    如果你得到了一个exception，是由于（Illegal key size），并且你使用sun的JDK。你需要安装Java Cryptography Extension （JCE）武强县都管辖政策文件。
    具体看官网。
    将文件解压到JDK/JRE/LIB/security 文件夹，以获取你使用的JRE/JDK x64/x86版本。
    
> sc发布于无限制的apache 2.0 license。如果你希望贡献这章节的文档，或者发现一个error，你可以去github。
## 2.Spring Cloud Context: Application Context Services 
sb是自以为是的，sc构建于此基础，并添加了一些可能是系统中所有组件都需要使用或者偶尔需要的功能。
### 2.1 The Bootstrap Application Context

 sc应用程序通过创建bootstrap context来运行，该context是主应用程序的父context。他负责从外部源加载属性配置和解密本地外部配置文件。这两个context共享一个environment，这个环境是任何spring应用程序的额外属性源。默认，bootstrap属性（不是bootstapproperties,而是在启动时加载的属性）添加具有最高优先级，因此他不能被本地配置覆盖。
 这个bootstrap context使用了不同与于main context的约定来定位外部配置。替代了
application.yml (or .properties)，你可以使用bootstrap.yml，保持bootstrap的外部配置与main context分离。eg：
```
spring:
  application:
    name: foo
  cloud:
    config:
      uri: ${SPRING_CONFIG_URI:http://localhost:8888}
```
如果你的程序需要来自服务的特定应用程序配置，设置在spring.application.name (in bootstrap.yml or application.yml)是个好办法。
你可以完全禁用bootstrap 进程，通过spring.cloud.bootstrap.enabled=false（比如在系统属性中）
### 2.2 Application Context Hierarchies
  当你从SpringApplication 或者 SpringApplicationBuilder 创建一个应用程序，Bootstrap context会被添加为这个context的父级。spring的特点是子context集成父级的属性源以及profile。因此main应用程序context包含了额外的属性源，比起没有scConfig创建的context。有以下额外的属性源：
  - "bootstrap":如果任何在bootstrap context中找到的PropertySourceLocators 并且是非空属性。一个可选的CompositePropertySource 具有高优先级。sc Config Server就是一个例子。查看2.6，有介绍如何定制这个属性源的内容。
  - "applicationConfig":"[classpath:bootstrap.yml]"(以及相关被激活的profile)，如果你有一个bootstrap.yml（或者.properties），哪些属性会被使用在bootstrap Context。然后会在设置parent时被添加到子context。优先级比application.yml (or .properties) 要低，作为创建sb程序的正常一部分添加到任何其他属性源。查看2.3设置该属性源内容。
  
  因为属性源的排序规则，bootstrap优先。然而，这并不包含任何bootstrap.yml的数据，因为它具有低优先级，单可以用来设置默认值。

  你可以通过设置任何applicationContext的父context继承context级别。比如，使用他自己的接口或者SpringApplicationBuilder 的方法（parent(), child() and sibling()）。这个bootstap context是你创建的最根本的祖宗context。每一个context都有他自己的“bootstap”（有可能空）属性源来避免从父级到后代无意间提升值。如果存在一个 config server，每一个context有一个不同的spring.application.name，于是对应不同的远程属性源。通常spring程序 context用来解析属性的行为规则：子context属性覆盖父的，根据名字和属性源名称（如果一个子有一个属性源和父的name相同，这个属性则不会被包含在子中）。
  记住SpringApplicationBuilder 可以让你分享 Environment 在整个层级中，但是他不是默认的。因此，特别是sibling context，不需要任何相同的profile或者属性源，即使他们可能分享与父级相同的值。
  ### 2.3 Changing the Location of Bootstrap Properties
  这个bootstrap.yml（或者.properties)位置可以通过spring.cloud.bootstrap.name来指定（默认bootstrap）,或者spring.cloud.bootstrap.location指定（默认空）。在系统属性中，他的行为类似spring.config.*的变体，他们会被用来设置bootstrap ApplicationContext 。如果通过spring.profiles.active或者Environment 的API有激活profile，名字也遵循之前的规则比如bootstrap-development.properties。

### 2.4 Overriding the Values of Remote Properties

属性源被添加到你的程序通过bootstrap通常是“remote”（比如Spring Cloud Config Server）。默认，他们不会被本地覆盖。如果你需要覆盖远程属性通过系统属性或者config 文件，远程属性源需要发放许可证通过设置spring.cloud.config.allowOverride=true（设置在本地并不会工作）。一旦flag设置，两个细粒度设置控制相关的系统属性的位置和程序本地配置。
- spring.cloud.config.overrideNone=true：任何本地属性都会覆盖。
- spring.cloud.config.overrideSystemProperties=false。只有系统属性，命令行参数，以及环境变量（不包括本地配置文件）应该覆盖远程设置。

### 2.5 Customizing the Bootstrap Configuration

bootstrap context可以被设置用来做任何你想做的事通过添加/META-INF/spring.factories的 入口在一个叫 org.springframework.cloud.bootstrap.BootstrapConfiguration 的key下面。他支持逗号分割用来创建context的spring的@Configuration类。任何你希望可用于main application context的自动装配的bean可在这里创建。对于@Beans的ApplicationContextInitializer类型有一个特别的合约。如果你希望控制启动序列，类可以被标记@Order（默认last）。
> 当添加自定义的BootstrapConfiguration，要小心在不需要时，不是@ComponentScanned的类错误的进入main application context。为sb使用一个分离的包名以及确认这个名没有被@ComponentScan or @SpringBootApplication注解配置类覆盖。

bootstrap进程通过注入初始化给SpringApplication 实例来结束（通常的sb启动序列，无论是否独立程序启动还是部署在程序服务），首先，从spring.factories中的类创建一个bootstrap context，然后所有的@Beans的ApplicationContextInitializer 类型被添加到主SpringApplication 在started之前。

### 2.6  Customizing the Bootstrap Property Sources

默认通过bootstap过程添加的外部配置的属性源是 Spring Cloud Config Server，但是你可以添加额外的源通过添加PropertySourceLocator 类型的bean给bootstrap context（通过spring.factories）。比如，你可以从不同的server或者database插入额外的属性。
```
@Configuration
public class CustomPropertySourceLocator implements PropertySourceLocator {

    @Override
    public PropertySource<?> locate(Environment environment) {
        return new MapPropertySource("customProperty",
                Collections.<String, Object>singletonMap("property.from.sample.custom.source", "worked as intended"));
    }

}
```
这个传入的环境是applicationContext即将创建的环境。换句话说，这个是我们为其提供其他属性源的环境。他已经有一个通常sb提供的属性源，因此可以使用那些来定位一个指定给这个环境的属性源（比如，在spring.application.name键入它，就像在默认sc config服务属性源发现者那样。）
如果你创建一个包含class的jar，并添加了一个META-INF/spring.factories 包含了下面所列内容，这个customProperty PropertySource 会出现在任何包含这个jar的程序中：
```
org.springframework.cloud.bootstrap.BootstrapConfiguration=sample.custom.CustomPropertySourceLocator
```

### 2.7 Logging Configuration

如果你使用sb来配置log，你需要把配置放在bootstrap.yml|properties，如果你希望应用所有事件。
> 对于sc初始化logging配置，你不能使用任何自定义前缀，比如使用custom.loggin.logpath将不会被sc认可在初始化logging系统时。

### 2.8 Environment Changes

这个应用程序监听一个EnvironmentChangeEvent 以及以集中标准方式相应change（额外的ApplicationListeners 通常会通过用户被添加为@Beans）。当EnvironmentChangeEvent被发现，它有一列被改变的键值对，以及程序使用他们：
- re-bind 任何context中的@ConfigurationProperties bean
- 通过logging.level.*为任何属性设置logger等级

记住config客户端不会，默认，poll环境的change。通常，我们不推荐检测change的方法（虽然你可以通过@Scheduled设置它。）如果你有一个扩展的客户端程序，它使用广播EnvironmentChangeEvent 给所有实例来替代poll change要好一些（比如，[通过 Spring Cloud Bus](https://github.com/spring-cloud/spring-cloud-bus)）

这个 EnvironmentChangeEvent 涵盖了一大类刷新用例，只要你可以创建一个Environment 的change并且发布这个事件。注意哪些API是公共的，也是spring core的一部分。你可以校验这个change是否绑定到@ConfigurationProperties bean通过访问/configprops 端点。（一个平常的sb Actuator特点 ）。比如，一个DataSource 可以有他自己的maxPoolSize change在运行时（默认通过sb创建的DataSource 是一个@ConfigurationProperties bean），以及动态增加容量。re-bind @ConfigurationProperties并不会覆盖任何其他的大类用例，在你需要更多控制刷新和需要在整个applicationContext 原子级修改时。解决这些问题，我们有@RefreshScope。

### 2.9 Refresh Scope

当一个配置修改时，一个被标记为@RefreshScope的spring @Bean会得到特别的对待。这个特点解决了有状态的bean只会在初始化时获得注入配置的问题。比如，如果一个DataSource 有一个开放的连接，当database url通过Environment改变时，你可能系统连接的持有者可以完成它正在做的，然后，下一次一些事情借用pool的连接，他会获得一个新的连接。
有时，他可能甚至必须应用@RefreshScope注解在一些只会被初始化一次的beans上。如果一个bean是一直不变的，你可以有一个@RefreshScope注解或者指定classname在属性 key spring.cloud.refresh.extra-refreshable下。

> 如果你创建一个你自己的DataSource并且实现了HikariDataSource，返回最明确的类型，在这个例子是HikariDataSource中。否则，你为需要设置spring.cloud.refresh.extra-refreshable=javax.sql.DataSource.

刷新域 bean是懒代理，也就是在使用时初始化，scope行为旧想初始化值的cache一样。强制一个bean再次初始化在下次方法调用时，你必须废除他的缓存入口。
RefreshScope 是一个context的bean，以及具有一个refreshAll()方法来刷新所有的在这个scope的bean通过清除目标的缓存。/refresh 端口暴露了这个方法（over HTTP or JMX）。通过name刷新一个独立的bean，还有一个refresh(String)方法。
暴露的/refresh端点，你需要添加下面的配置。
```
management:
  endpoints:
    web:
      exposure:
        include: refresh
``` 
  @RefreshScope 工作（技术上）在@Configuration类上，但是他可能会引导一些以外的行为。比如，这并不意味着该类所有的@Beans本身都在@RefreshScope中。具体来说，任何依赖那些bean都不会依靠他们来更新在刷新时，除非它本身是@RefreshScope。在这种情况下，它会在刷新时被重建，以及它的依赖会被重新注入。在这点，他们会从refreshed的  @Configuration初始化。

### 2.10 Encryption and Decryption    

sc有一个Environment  pre-processor 对于本地解密属性值。它遵循与config server相同的规则，并且具有相同的额外配置通过encrypt.*。因此，你可以使用加密值以{cipher}*的形式，只要存在有效的key，它可以解密在main context获取environment设置之前。使用加密，你需要包含Spring Security RSA在你的classpath（maven："org.springframework.security:spring-security-rsa"），并且你还需要全部的JCE 扩展在JVM中。
如果获得了 "Illegal key size" 在你的jdk，你需要安装Java Cryptography Extension 无限强度管理策略文件。参考文档下载连接以及相关事项。

### 2.11 Endpoints

对于sb Actuator 程序，一些额外的管理端点可用。你可以使用:
- POST 至/actuator/env 来更新environment以及重新绑定@ConfigurationProperties，log级别。
- /actuator/refresh 用来重新加载boot strap context和刷新@RefreshScope bean
- /actuator/restart 关闭applicationContext以及重启（默认不可用）
- /actuator/pause以及/actuator/resume 来调用Lifecycle 方法（ApplicationContext的stop() and start()）
如果你禁用/actuator/restart 端口，然后/actuator/pause and /actuator/resume 端口都会被禁用。因为他是/actuator/restart特别的情况。