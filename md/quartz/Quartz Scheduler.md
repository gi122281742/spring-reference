# Quartz Scheduler
Springboot 为Quartz scheduler提供了一些方便，包含在spring-boot-starter-quartz中。如果启用了，Scheduler 将会通过SchedulerFactoryBean抽象类自动配置。

    以下几个类型，将会被Springboot自动挑选并与Scheduler相关联。
> JobDetail:定义了特定的JOB，JobDetail实例可以通过JobBuilder  API创建。

> Calendar

> Trigger:定义了何时触发JOB

默认情况下，JobStore 存储在内存中，然而你也可以保存在基于JDBC的存储库，前提是DataSource Bean可用以及spring.quartz.job-store-type呗相应的配置，比如:

>   spring.quartz.job-store-type=jdbc

也可以设置每次启动时，初始化数据库

> spring.quartz.jdbc.initialize-schema=always

默认，database会Quartz 的标准脚本通过自动检测并初始化。脚本会删除表并清空所有出发器在每一次重启时。可以设置spring.quartz.jdbc.schema设置自定义的脚本。

让Quartz 使用主应用程序之外的Datasource，声明一个Quartz ，并添加@Bean以及@QuartzDataSource注解。这样做可以确保Quartz特定数据源被所有SchedulerFactoryBean 使用 以及数据库的初始化。
By Default，通过configuration 创建的job不会覆盖已经注册保并持久化保存的job，可以设置spring.quartz.overwrite-existing-jobs启用覆盖。
Quartz Scheduler可以通过spring.quartz属性或者SchedulerFactoryBeanCustomizer 配置，它可以编程化定制SchedulerFactoryBean 。进阶配置可以通过spring.quartz.properties.*。

需要注意的是，Executor Bean 并不会和scheduler 相关，因为Quartz 提供了通过spring.quartz.properties来配置的方法。如果需要定制一个 task executor,考虑实现SchedulerFactoryBeanCustomizer吧。
# Task Execution and Scheduling

若context不存在TaskExecutor ，SpringBoot将会自动配置一个ThreadPoolTaskExecutor 以及springboot觉得挺聪明的默认（自动配置与asynchronous task execution关联，Spring MVC异步请求处理）。
这个线程是八核的，根据核心增长或者减小。可以通过spring.task.execution进行一些很细的配置。
> spring.task.execution.pool.max-threads=16
> 
> spring.task.execution.pool.queue-capacity=100
> 
> spring.task.execution.pool.keep-alive=10s

按照上面这个配置，会使线程池变成一个有界的序列，以至于当序列满了时（100任务），这个线程池会增加到最大值16线程。收缩线程池的时候也很迅速，因为如果超过10s的空闲线程就会被回收（默认60s）。
ThreadPoolTaskScheduler 也会被自动配置，如果他需要与scheduled task execution 相关联（@EnableScheduling）。线程池默认使用一个线程，它可以通过spring.task.scheduling进行配置。
TaskExecutorBuilder 以及TaskSchedulerBuilder  Bean可以用来创建一个自定义的executor 或者 scheduler 。


