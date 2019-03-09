# Spring Using the Quartz
Quartz使用trigger，job,JobDetail对象来实现job的调度。为了方便的目的，spring提供了一些简单使用Quartz的类。
# 1.使用JobDetailFactoryBean

Quartz JobDetail包含了所有job的信息。spring提供了JobDetailFactoryBean ，它为XML配置提供了bean-格式的属性。比如：
```
<bean name="exampleJob" class="org.springframework.scheduling.quartz.JobDetailFactoryBean">
    <property name="jobClass" value="example.ExampleJob"/>
    <property name="jobDataAsMap">
        <map>
            <entry key="timeout" value="5"/>
        </map>
    </property>
</bean>
```
job detail配置具有所有需要运行的job的信息。timeout在job data map中指定。job data map是一个可以通过JobExecutionContext 获取 （在运行时传递给你），但是JobDetail也可以从job data maop获取这些属性。所以，在接下来的例子，ExampleJob 包含了一个属性名 named ，以及 JobDetail 会自动应用。
```
package example;

public class ExampleJob extends QuartzJobBean {

    private int timeout;

    /**
     * Setter called after the ExampleJob is instantiated
     * with the value from the JobDetailFactoryBean (5)
     */
    public void setTimeout(int timeout) {
        this.timeout = timeout;
    }

    protected void executeInternal(JobExecutionContext ctx) throws JobExecutionException {
        // do the actual work
    }

}
```
所有额外的job data map的属性都可以使用。
## 2.Using the MethodInvokingJobDetailFactoryBean

通常你只是需要调用一个特定对象的某个方法。可以使用MethodInvokingJobDetailFactoryBean，你可以明确这个，比如：
```
<bean id="jobDetail" class="org.springframework.scheduling.quartz.MethodInvokingJobDetailFactoryBean">
    <property name="targetObject" ref="exampleBusinessObject"/>
    <property name="targetMethod" value="doIt"/>
</bean>
```
上面这个例子会使exampleBusinessObject上调用doIt方法。
```
public class ExampleBusinessObject {

    // properties and collaborators

    public void doIt() {
        // do the actual work
    }
}
```
```
<bean id="exampleBusinessObject" class="examples.ExampleBusinessObject"/>
```
使用MethodInvokingJobDetailFactoryBean，你无需创建仅仅需要调用一个方法的job。你只需要创建实际的食物对象以及装配detail对象。
默认，quartz job无国籍，可能导致job之间相互干扰。如果你需要为一个JobDetail特定两个trigger，可能会导致，在第一个job完成之前，第二个开始。如果jobDetail继承了Stateful 接口，这就不会发生。第二个并不会在第一个完成之前启动。要让MethodInvokingJobDetailFactoryBean不并发，设置concurrent 为false：
```
<bean id="jobDetail" class="org.springframework.scheduling.quartz.MethodInvokingJobDetailFactoryBean">
    <property name="targetObject" ref="exampleBusinessObject"/>
    <property name="targetMethod" value="doIt"/>
    <property name="concurrent" value="false"/>
</bean>
```
默认并发。
## 3.Wiring up Jobs by Using Triggers and SchedulerFactoryBean

我们已经创建了job和jobdetail。回顾一下调用特定对象某个方法的方便的Bean。当然，我们仍然需要调度job本身。这个动作是通过trigger和SchedulerFactoryBean完成。几个trigger在quartz中可用，spring提供了两个方便的quartz FactoryBean：CronTriggerFactoryBean 和SimpleTriggerFactoryBean。
trigger需要被调度。spring提供了SchedulerFactoryBean ，它暴露了trigger可以被设置的属性。SchedulerFactoryBean 调度实际的job和那些trigger。
下面的例子是使用SimpleTriggerFactoryBean 和CronTriggerFactoryBean：
```
<bean id="simpleTrigger" class="org.springframework.scheduling.quartz.SimpleTriggerFactoryBean">
    <!-- see the example of method invoking job above -->
    <property name="jobDetail" ref="jobDetail"/>
    <!-- 10 seconds -->
    <property name="startDelay" value="10000"/>
    <!-- repeat every 50 seconds -->
    <property name="repeatInterval" value="50000"/>
</bean>

<bean id="cronTrigger" class="org.springframework.scheduling.quartz.CronTriggerFactoryBean">
    <property name="jobDetail" ref="exampleJob"/>
    <!-- run every morning at 6 AM -->
    <property name="cronExpression" value="0 0 6 * * ?"/>
</bean>
```
上面的例子设置了两个trigger，一个没50秒一次延迟10s启动，一个每天6.00 am。完成这一切，我们需要设置SchedulerFactoryBean，就像下面一样：
```
<bean class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
    <property name="triggers">
        <list>
            <ref bean="cronTrigger"/>
            <ref bean="simpleTrigger"/>
        </list>
    </property>
</bean>
```
更多属性对于SchedulerFactoryBean可用，就像通过job detail使用calendars 一样，用于自定义quartz的属性，以及其他。查看SchedulerFactoryBean 更多信息。