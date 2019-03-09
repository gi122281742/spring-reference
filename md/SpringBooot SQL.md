# Spring SQL

## Working with SQL Databases
Spring提供了很多SQL数据库的支持，比如直接从JDBC访问使用JdbcTemplate 来完成对象关系映射；Spring Data提供了额外的方法,创建Repository 直接实现接口，通过其约定的方法名来生成查询。
### Configure a DataSource
javax.sql.DataSource接口提供了于数据库链接工作的标准的方法，通常，一个DataSource需要一个URL和一些凭证来建立数据库连接。
#### Embedded Database Support
通常在开发中使用内存嵌入的数据库会比较方便，但是，他们并不支持持久化保存，你需要在你的应用 程序启动时填充数据库以及在结束时抛出一些数据。

SpringBoot能够自动配置H1,HSQL,Derby嵌入式数据库，你不需要提供任何URLs，你只需要包含他们在你的依赖中。
如果你在test中使用这些功能，你需要注意不管你的应用程序context有多少，相同的database会被重用，如果你需要每个context隔离开来，你需要设置spring.datasource.generate-unique-name为true。
需要依赖spring-jdbc才会自动配置嵌入式数据库。
当你配置嵌入式数据库URL时，无论什么原因，都不要启动数据库的自动关闭功能。这样可以使SpringBoot完全控制他的关闭以确认不需要再访问该数据库。

### Connection to a Production Database
1.生产数据库连接会被自动配置连接池，Springboot使用一下算法来选择实例：
> 优先HikariCP。
> 其次Tomcat pooling DataSource
>  最后Commons DBCP2

如果使用spring-boot-starter-jdbc or spring-boot-starter-data-jpa，会自动获得依赖优先HikariCP。
你也可以使用spring.datasource.type来绕过这些直接指定连接池，比如指定tomcat-jdbc。其他的数据连接池需要手动配置，如果手动配置了，就不会自动配置。
数据源配置基于一些属性：
```
spring.datasource.url=jdbc:mysql://localhost/test
spring.datasource.username=dbuser
spring.datasource.password=dbpass
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```
至少指定一个url，否则会使用嵌入的数据库。通常不需要写spring.datasource.driver-class-name，因为SpringBoot会通过url推断className。
DataSourceProperties提供更多支持的属性，也可以通过添加后缀来微调连接池属性，比如(spring.datasource.hikari.*, spring.datasource.tomcat.*, and spring.datasource.dbcp2.*)
### Connection to a JNDI DataSource
如果通过springboot开发一个应用程序，你可能需要通过应用程序服务内部配置以及访问它来管理你的数据源。
spring.datasource.jndi-name替代url，username，password属性，从一个特定的地址来访问数据源。比如，下面这个例子可以通过访问一个JBoss定义的数据库。
```
spring.datasource.jndi-name=java:jboss/datasources/customers
```

## Using JdbcTemplate
JdbcTemplate 和NamedParameterJdbcTemplate 都是自动配置，你可以使用@Autowire直接在Bean使用它。
也可以为他设置一些属性
```
spring.jdbc.template.*
```
如果存在多个JdbcTemplate 并且没有主要候选项,NamedParameterJdbcTemplate 不会自动配置。

## JPA and Spring Data JPA
> spring-boot-starter-data-jpa提供了以下依赖
- Hibernate
- Spring Data JPA
- Spring ORMs

### Entity Classes

传统上，JPA实体类通常指定在persistence.xml，springboot使用Entity Scanning替代了它，默认，所有在主程序下的包都会被扫描，主类即具有@EnableAutoConfiguration o或@SpringBootApplication的类。

### Spring Data JPA Repositories
Spring Data JPA Repositories是一个用来定义访问数据的接口，JPA查询会根据方法名自动创建。
比如 CityRepository 接口声明了一个findAllByState方法，该方法将会通过state寻找cities。对于一些复查的查询，你可以看看[query](https://docs.spring.io/spring-data/jpa/docs/current/api/org/springframework/data/jpa/repository/Query.html)注解,Spring Data repositories 通常继承至 Repository or CrudRepository接口，如果你需要自动配置，他会自动扫描包下的所有配置类。
Spring Data JPA repositories 支持三种启动模式：default, deferred, and lazy.通过spring.data.jpa.repositories.bootstrap-mode属性设置为deffered或者lazy，可以修改。这时EntityManagerFactoryBuilder 将使用异步任务执行器来自动配置。
这只是表面，更多细节请参考[Spring Data JPA reference documentation](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)。
### Creating and Dropping JPA Databases
    默认情况下，当你使用嵌入式数据库时，才会自动创建JPA Databases。你可以明确指定spring.jpa.*属性。
    也可以通过spring.jpa.properties.A来指定A的jpa设置。

# Spring Nosql
Spring Data提供了需要Nosql支持，但都需要自己配置。
## Redis
Redis是一个集缓存，信息经纪，丰富功能的键值对仓库。springBoot为Lettuce and Jedis 客户端库自动配置，以及在他们之上抽象了一个[Spring Data Redis](https://github.com/spring-projects/spring-data-redis);在spring-boot-starter-data-redis中，收集了一些方便的依赖，默认使用Lettuce。还提供了spring-boot-starter-data-redis-reactive以便与其他stores的相应一致。
### Connecting to Redis
    你可以注入自动配置的RedisConnectionFactory，StringRedisTemplate或者vanilla RedisTemplate实例。默认情况下，他们尝试连接localhost:6379
你可以使用LettuceClientConfigurationBuilderCustomizer ，如果使用Jefis，则使用JedisClientConfigurationBuilderCustomizer 进一步配置他们。



