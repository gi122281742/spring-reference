# Quartz

## Quick Start
### Download and Install

最开始当然是要下载，并添加进classpath。使用maven直接依赖应该就OK了。官方太多不看。

### Configuration

为了快速启动，使用以下配置:
```
org.quartz.scheduler.instanceName = MyScheduler
org.quartz.threadPool.threadCount = 3
org.quartz.jobStore.class = org.quartz.simpl.RAMJobStore
```
- org.quartz.scheduler.instanceName 指定了该调度的名称。
- org.quartz.threadPool.threadCount 制定了线程池中线程的数量，也就是说最多有这么多JOB同时运行。
- org.quartz.jobStore.class 所有的Quartz的数据，job的细节以及触发器，都保存在内存中（而不是database）。即使你有一个数据库并且希望使用在quartz，官方建议在使用数据库之前先使用RamJobStore 。估计是方便练习吧。

### Starting a Sample Application


下面的代码是官方的例子，启动与停止调度：
```
  import org.quartz.Scheduler;
  import org.quartz.SchedulerException;
  import org.quartz.impl.StdSchedulerFactory;
  import static org.quartz.JobBuilder.*;
  import static org.quartz.TriggerBuilder.*;
  import static org.quartz.SimpleScheduleBuilder.*;

  public class QuartzTest {

      public static void main(String[] args) {

          try {
              // Grab the Scheduler instance from the Factory
              Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();

              // and start it off
              scheduler.start();

              scheduler.shutdown();

          } catch (SchedulerException se) {
              se.printStackTrace();
          }
      }
  }
```
> 一旦你通过StdSchedulerFactory.getDefaultScheduler(),获得一个Scheduler，除非你调用scheduler.shutdown()，否则他不会关闭，因为他会活跃在线程中。

## Lesson 1: Using Quartz
当你使用scheduler时，他需要初始化。完成初始化可以使用SchedulerFactory。一些Quartz用户会把factory一个实例保存在JNDI store，其他的可以发现直接的初始化并且使用工厂实例会更加简单（比如以下的例子）。
一旦scheduler初始化，他就可以被启动，安置在闲置模式，以及关闭。值得注意的是，一旦scheduler shutdown，他不能在未初始化的情况下重启。Trigger不会启动（Job不会执行）直到scheduler启动，也不会在在暂停状态时触发。
下面是例子：
```
  SchedulerFactory schedFact = new org.quartz.impl.StdSchedulerFactory();

  Scheduler sched = schedFact.getScheduler();

  sched.start();

  // define the job and tie it to our HelloJob class
  JobDetail job = newJob(HelloJob.class)
      .withIdentity("myJob", "group1")
      .build();

  // Trigger the job to run now, and then every 40 seconds
  Trigger trigger = newTrigger()
      .withIdentity("myTrigger", "group1")
      .startNow()
      .withSchedule(simpleSchedule()
          .withIntervalInSeconds(40)
          .repeatForever())
      .build();

  // Tell quartz to schedule the job using our trigger
  sched.scheduleJob(job, trigger);
```
## Lesson 2: The Quartz API, Jobs And Triggers
### Quartz API 关键接口:
- Scheduler : 主要与调度互动的API；
- Job ： 通过希望被调度执行的组件实现的接口；
- JobDetail ： 用来定义Job的接口；
- Trigger ： 一个定义给定job执行计划的组件；
- JobBuilder ： 用来创建/定义JobDetail实例，其中定义了Job的实例；
- TriggerBuilder ： 用来创建/定义Trigger 实例。
    
   Scheduler的生命周期受限于他的创建，引用SchedulerFactory 并调用他的shutdown（）方法。一旦Scheduler 被创建，它可以被添加，移除，以及列出job和trigger，执行其他调度相关的操作（比如暂停trigger）。然而，Scheduler 实际上并不会在start() 之前执行任何trigger。
   上面的例子引用了许多静态类的方法：
   ```
    import static org.quartz.JobBuilder.*;
    import static org.quartz.SimpleScheduleBuilder.*;
    import static org.quartz.CronScheduleBuilder.*;
    import static org.quartz.CalendarIntervalScheduleBuilder.*;
    import static org.quartz.TriggerBuilder.*;
    import static org.quartz.DateBuilder.*;
   ```
### Jobs and Triggers
    job就是一个实现了Job接口并实现其方法的类。
    当Job的trigger启动方法在scheduler的worker的一个线程中被调用，传递给这个方法的JobExecutionContext 会提供job 实例的关于运行时环境的信息（Scheduler执行的句柄，一个Trigger触发的句柄，job的JobDetail，还有一些其他items）。
    JobDetail被Quartz 创建在Job被添加到Scheduler时，他包含了job的每个设置熟属性，以及JobDataMap。JobDataMap可以用来为给定的job实例保存状态信息。他实质上是定义了一个Job实例，进一步讨论在下一章。
    Trigger用来fire job，当你希望调用一个Job，你初始化一个Trigger并调试他的属性以提供你希望的调度时间表。Trigger可能有一个JobDataMap 关联他们，它可以用来传递参数给被trigger启动的指定的job。Quartz 有少数的trigger类型，但是通常只使用SimpleTrigger and CronTrigger.
- SimpleTrigger在你一次性执行job时或者在给定时间启动，执行N次时，每次执行间隔T时间时比较有用。
- CronTrigger 通常用于类似日历的日安排，比如每周五十点这样。

    为何会有Job和Scheduler？许多job调度没有区分job和trigger。一些定义了一个job就像简单的执行时间和一些小工作的标识符，其他的则像整合了Quartz的job和trigger对象。当开发Quartz时，我们认为区分日程表以及日程表执行的job十分有意义。举个例子，job可以被创建以及保存在依赖于trigger的job scheduler，以及一些trigger可以关联一些相同的job。其他的解耦的好处就是可以配置触发器过期了但是仍然保存在Scheduler的job，以至于它可以直接改期晚一点，而不需要重新定义他，它允许你定义或者替换触发器而不需要重新定义job。
 ### Identities

    Job以及Trigger在注册Scheduler时都被给了一个识别符。job和trigger的keys允许他们可以被安置在'group'中，这样对组织他们在不同的类别比较有用。job和trigger的key的名称部分必须在'group'中唯一，也就是说，完整的key或者说识别符是通过'group'和'name'组成。

## Lesson 3: More About Jobs and Job Details

正如Lesson 2看到的那样，job比较容易实现，这个接口只有一个execute方法。你可能需要明白关于job的性质，关于execute，以及关于JobDetails一些细节。
虽然你实现的job类具有知道如何执行特定类型job的实际工作的代码，但是Quartz需要被通知关于每个你希望job实例具有的参数。这样会引用JobDetail 类。
JobDetail 是通过JobBuilder 来创建，你通常会引入
```
import static org.quartz.JobBuilder.*;
```
在Quartz中关于job的性质以及生命周期，查看下面的代码:
```
  // define the job and tie it to our HelloJob class
  JobDetail job = newJob(HelloJob.class)
      .withIdentity("myJob", "group1") // name "myJob", group "group1"
      .build();

  // Trigger the job to run now, and then every 40 seconds
  Trigger trigger = newTrigger()
      .withIdentity("myTrigger", "group1")
      .startNow()
      .withSchedule(simpleSchedule()
          .withIntervalInSeconds(40)
          .repeatForever())            
      .build();

  // Tell quartz to schedule the job using our trigger
  sched.scheduleJob(job, trigger);
```
HelloJob类:
```
  public class HelloJob implements Job {

    public HelloJob() {
    }

    public void execute(JobExecutionContext context)
      throws JobExecutionException
    {
      System.err.println("Hello!  HelloJob is executing.");
    }
  }
```
请注意我们给了Scheduler一个JobDetail实例，它知道被执行的job类型是通过在创建JobDetail时提供的job class。每一次scheduler执行job，他会在execute之前创建一个新的该class实例，当execute完成后，对该job类的引用会被销毁，并且实例随后会被垃圾回收。
这个行为其中之一的后果是job必须有一个无参构造器（使用默认JobFactory 实现时），另外一个后果是定义在job的状态数据字段没有意义，因为他并不会在job执行之间保存。
你可能会问“如何为job实例提供属性”，“如何在执行之间跟踪job状态”。答案的关键都是JobDataMap，他是JobDetail的一部分。

### JobDataMap
    JobDataMap 在执行时支持任何数量的你希望可用的的数据对象，JobDataMap 继承自Map，添加了一些方便主要类型的保存以及检索方法。

    以下是在定义/创建 JobDataMap 时将数据put进JobDataMap的代码片段，优先添加job进scheduler：
```
    // define the job and tie it to our DumbJob class
    JobDetail job = newJob(DumbJob.class)
    .withIdentity("myJob", "group1") // name "myJob", group "group1"
    .usingJobData("jobSays", "Hello World!")
    .usingJobData("myFloatValue", 3.141f)
    .build();
```
下面是从JobDataMap获取一些数据:
```
public class DumbJob implements Job {

    public DumbJob() {
    }

    public void execute(JobExecutionContext context)
      throws JobExecutionException
    {
      JobKey key = context.getJobDetail().getKey();

      JobDataMap dataMap = context.getJobDetail().getJobDataMap();

      String jobSays = dataMap.getString("jobSays");
      float myFloatValue = dataMap.getFloat("myFloatValue");

      System.err.println("Instance " + key + " of DumbJob says: " + jobSays + ", and val is: " + myFloatValue);
    }
  }
```
如果你使用持久性JobStore，你需要注意你放置了什么在JobDataMap，因为object在里面会被序列化，他们因此容易发生类版本问题。标准的java类型十分安全，但是除此之外，任何时间一些人修改了保存的class定义，注意不要破坏它的兼容性。你也可以put进JDBC-JobStore以及JobDataMap只存主要类型以及String类型。因此消除可能的序列化问题。如果添加了setter方法在你的job类中，相应的key也会存在JobDataMap中。Quartz的默认JobFactory 实现会在job初始化时，自动调用那些setter方法，因此无需在execute 方法中显示从map中获取值。
Triggers 同样有一个JobDataMaps 关联他，当你有一个job存储在有多个trigger调用他的scheduler时比较有用，对于每个依赖的trigger，你可以应用job使用不同的输入数据。
JobDataMap可以在Job execution服务中的JobExecutionContext 中找到，他合并了JobDetail 的JobDataMap 以及Trigger的的JobDataMap，后者值会覆盖前者。
下面是从job’s execution的JobExecutionContext的合并获取数据的例子：
```
public class DumbJob implements Job {

    public DumbJob() {
    }

    public void execute(JobExecutionContext context)
      throws JobExecutionException
    {
      JobKey key = context.getJobDetail().getKey();

      JobDataMap dataMap = context.getMergedJobDataMap();  // Note the difference from the previous example

      String jobSays = dataMap.getString("jobSays");
      float myFloatValue = dataMap.getFloat("myFloatValue");
      ArrayList state = (ArrayList)dataMap.get("myStateData");
      state.add(new Date());

      System.err.println("Instance " + key + " of DumbJob says: " + jobSays + ", and val is: " + myFloatValue);
    }
  }
```
当你希望依赖JobFactory 注入data map值时，也可以这样：
```
 public class DumbJob implements Job {


    String jobSays;
    float myFloatValue;
    ArrayList state;

    public DumbJob() {
    }

    public void execute(JobExecutionContext context)
      throws JobExecutionException
    {
      JobKey key = context.getJobDetail().getKey();

      JobDataMap dataMap = context.getMergedJobDataMap();  // Note the difference from the previous example

      state.add(new Date());

      System.err.println("Instance " + key + " of DumbJob says: " + jobSays + ", and val is: " + myFloatValue);
    }

    public void setJobSays(String jobSays) {
      this.jobSays = jobSays;
    }

    public void setMyFloatValue(float myFloatValue) {
      myFloatValue = myFloatValue;
    }

    public void setState(ArrayList state) {
      state = state;
    }

  }
```
### Job “Instances”
你可以定义一个job class，然后通过创建多个JobDetail实例（每个实例都有自己的属性以及JobDetailMap，并添加进scheduler）存储他的多个实例定义在scheduler中。
当trigger启动时，他关联的JobDetail会被加载，以及他引用的job class是通过在scheduler的JobFactory 配置初始化。默认JobFactory 简单的在job class调用newInstance方法，然后从调用class的setter方法匹配JobDataMap中的key的name。你可能希望创建你自己的JobFactory 实现类，来完成诸如IOC容器或者DI的一些功能。

### Job State and Concurrency
关于job state 数据以及并发的一些额外说明。提供了一对注解可以添加到job class上以尊重他们影响这些行为。

> @DisallowConcurrentExecution是一个告诉quartz不需要并发执行多重给定job实例。也就是说给定时刻只有一个该job class的实例正在运行。该约束基于job class而不是实例，所以需要对类注解，因为回为类的编码方式会产生影响。
> @PersistJobDataAfterExecution会告诉Quartz在完成execute（）后更新JobDetail’s JobDataMap复制的store，这样下一次同样job执行时会收到该更新，而不是原本的store值。和上面那个注解一样，应用于class而不是instance。会对编码产生影响。
使用@PersistJobDataAfterExecution时，通常应该使用@DisallowConcurrentExecution。这样可以避免一些疑惑，比如当两个相同job的实例并发执行时，该保存哪一个的数据。
### Other Attributes Of Jobs
一些可以通过JobDetail对象定义的 job 实例的属性：
- Durability ：如果job是non-Durability，他会在没有关联active trigger时从scheduler删除。也就是说，non-Durability 的声明周期取决去他的trigger。
- RequestsRecovery ：如果一个job是“requests recovery”，他会在scheduler“硬关闭时执行”，然后再次启动scheduler时重新执行它。这种情况下，JobExecutionContext.isRecovering() 会返回true。
### JobExecutionException
只有JobExecutionException可以被抛出，你通常需要使用try-catch包裹整个execute块，可以阅读文档，来使用它来为scheduler提供各种如何处理异常的指令。

## More About Triggers
有许多类型的trigger，你可以选择以匹配不同scheduler的需要。
### Common Trigger Attributes
    除了每个trigger 都具有TriggerKey属性用来追踪他们的身份外，他们还具有一些其他的共用属性。这些属性在创建trigger 定义时，通过TriggerBuilder 设置。
如下列表:
- jobKey：需要在启动trigger执行时的job的标识符。
- startTime：trigger首次执行的时间。值为定义了给定日历日期的 java.util.Date对象。对于一些trigger类型，trigger 会在starttime启动，对于另一些，只是简单标记这个应该遵循scheduler的时间。这意味着你可以存储带有时间表的trigger。
- endTime : end time后trigger不再有效。
### Priority
当你拥有许多trigger时，Quartz可能没有足够的资源同时启动所有trigger。这是你可能需要控制trigger触发的优先级。默认值为5，可正可负。
### Misfire Instructions
如果由于scheduler关闭或者在quartz线程池中没有可用的线程来执行job，导致持久的trigger错失了它应该执行的时间，则会发生misfire 。不同的trigger对于它们有不同的misfire instructions 。默认，他们使用 ‘smart policy’ instruction- 它具有基于trigger类型和配置的动态行为。当scheduler 启动时，他会搜索任何misfire的持久trigger，然后他会基于他们配置的失火说明更新每一个trigger。当你开始使用Quartz 时，你需要熟悉定义在给定trigger类型的失火指令，然后再javadoc解释。详细信息在每个trigger章节。
### Calendars
Quartz Calendar并不是 java.util.Calendar objects，它可以在trigger被定义以及存储在scheduler时关联trigger。Calendars 对于在trigger启动时间表中排除时间块非常有用。比如，你创建一个trigger在每天9.30，它可以添加Calendars 来排除节假日。
Calendars提供了两个方法，参数均为timestamps的毫秒 long类型。大多数情况下，你可能需要阻止一整天，为了方便，Quartz包含了HolidayCalendar，他就是这样的。
Calendars必须初始化并且通过scheduler的addCalendar注册。如果你使用HolidayCalendar，在初始化它之后，你需要使用他的addExcludedDate（），用你希望排除在外的Date填充进去。同一个calendar可以用于多个trigger。
org.quartz.impl.calendar有许多实现类方便使用。
## SimpleTrigger
SimpleTrigger当你需要执行job一次，或者重复N次在相同的间隔比较有用。
SimpleTrigger包括：start-time，end-time，repeat count，repeat interval属性。这些属性都是让你期望他们如何执行，只有一些与end-time属性相关的特别的注释。
repeat count可以为0，正数或者常量SimpleTrigger.REPEAT_INDEFINITELY。
repeat interval 必须为0或者一个正数long，单位为毫秒。请注意：repeat interval 为0时会导致触发器的repeat count同时发生（或者接近同时，就像scheduler能够管理的一样）。
它对于计算触发器的启动时间很有帮助，依赖于你创建它时的startTime（或者endTime）。
endTime覆盖了repeat count。你可以简单的指定end-time，然后使用REPEAT_INDEFINITELY指定repeat count（你可以指定任何巨大的数字，以确保比它在end-time之前执行的次数多）。
SimpleTrigger实例通过TriggerBuilder（对于trigger的主要属性）以及SimpleScheduleBuilder（对于SimpleTrigger的特定属性）。
使用DSL方式，导入static 类
```
import static org.quartz.TriggerBuilder.*;
import static org.quartz.SimpleScheduleBuilder.*;
import static org.quartz.DateBuilder.*:
```
查看TriggerBuilder and SimpleScheduleBuilder获取更多信息。
TriggerBuilder (以及 Quartz的其他 builders) 在你为明确设置时会选择一个合理的属性值。比如：如果你没有调用withIdentity（），TriggerBuilder 会生成一个随机的名字给trigger。如果未调用startAt(..)，则会设置为当前启动。
### SimpleTrigger Misfire Instructions
SimpleTrigger有一些可以用来通知Quartz 在misfire时它应该做什么的指令。这些指令被定义为SimpleTrigger 的常量。
有以下一些指令：
> MISFIRE_INSTRUCTION_IGNORE_MISFIRE_POLICY
MISFIRE_INSTRUCTION_FIRE_NOW
MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_EXISTING_REPEAT_COUNT
MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_REMAINING_REPEAT_COUNT
MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_REMAINING_COUNT
MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_EXISTING_COUNT

默认值为Trigger.MISFIRE_INSTRUCTION_SMART_POLICY ，smart policy启用时，SimpleTrigger 会动态选择他们之中的MISFIRE 指令，基于配置以及给定
SimpleTrigger实例的状态。SimpleTrigger.updateAfterMisfire()可以查看更多。
创建SimpleTriggers时，misfire指令通常作为simple schedule的一部分。

## CronTrigger
在你需要job-firing 基于类日历的重复而不是特定时间间隔时，他比SimpleTrigger更有用。
通过它，你可以指定启动时间表为“每周五晚”，“每周工作日9.30”，甚至“五月每周星期一，二，四9点至10点，每五分钟重复一次”。
即便如此，CronTrigger 也有startTime 指定何时生效，endTime 何时不再继续。

### Cron Expressions
Cron Expressions 用来配置CronTrigger实例。
Cron-Expressions是一个由七个子表达式组成的字符串，它描述了每个独立的scheduler细节，通过空格分割。分别代表：

1. Seconds
2. Minutes
3. Hours
4. Day-of-Month
5. Month
6. Day-of-Week
7. Year (optional field)
  
  举个例子“0 0 12 ? * WED”就代表“每周wed的12：00：00 pm”。
  每个子表达式可以包含一个范围，或者多个值。比如“MON-FRI”等于“MON,WED,FRI”，也可以混合表达 “MON-WED,SAT”。
  > Wild-cards可以用来代表任何一种可能。上个例子的*代表每个月。
  时间值均从0开始，秒最大59，以此类推。

  > 每月几号是1-31。
  月可以使用JAN, FEB, MAR, APR, MAY, JUN, JUL, AUG, SEP, OCT, NOV,DEC;

  > 星期几是1-7：SUN, MON, TUE, WED, THU, FRI and SAT。

  > “/”可以用来表示增量。比如‘0/15’ 表示从0开始每15一次。“/35”等于“0/35”；

  > ‘?’只能在day-of-month 和 day-of-week中使用。这代表没有特定值。其中之一有用时比较有用。

  > ‘L’day-of-month 和 day-of-week在day-of-month and day-of-week中有用，表示last，代表每月最后一天或者每周的7也就是星期六。但是如果在day-of-week的值后面添加L，
  比如“6L”，代表每月最后一个星期5。day of the month也可以添加位移“L-3”代表每月倒数第三天。使用L时不要使用lists或者范围，这样坑你会得到以外的结果。

  > ‘W’用来指定最接近的“星期一到星期五”。比如day-of-month的“15W”代表最接近本月第十五天的工作日。

  >  ‘#’ 代表weekday of the month。比如day-of-week的“6#3”代表每个月的第三个星期五。

  更多信息在javadoc org.quartz.CronExpression。

### Example Cron Expressions
“0 0/5 * * * ?”：每五分钟一次；
“10 0/5 * * * ?”：每五分钟一次，10秒开始；
“0 30 10-13 ? * WED,FRI”：每周Wednesday and Friday，10-13点的，30分钟一次。
“0 0/30 8-9 5,20 * ?”：每月5，25的8-9点，30分钟一次。
### Building CronTriggers
CronTrigger 通过TriggerBuilder（主要属性） 
以及CronScheduleBuilder （CronTrigger特定属性）创建。使用DSL方式，静态导入：
```
import static org.quartz.TriggerBuilder.*;
import static org.quartz.CronScheduleBuilder.*;
import static org.quartz.DateBuilder.*:
```
### CronTrigger Misfire Instructions
下面的说明可以用来通过quartz在misfire时，它应该做什么。这些指令被定义为一个CronTrigger 的常量。包括：
```
MISFIRE_INSTRUCTION_IGNORE_MISFIRE_POLICY
MISFIRE_INSTRUCTION_DO_NOTHING
MISFIRE_INSTRUCTION_FIRE_NOW
```
每个trigger都有Trigger.MISFIRE_INSTRUCTION_SMART_POLICY指令可用，这个指令是所有trigger的默认值。‘smart policy’解释为CronTrigger的MISFIRE_INSTRUCTION_FIRE_NOW。CronTrigger.updateAfterMisfire() 有更多细节。创建时，可通过schedule 的一部分设置misfire instruction。

### TriggerListeners and JobListeners

Listeners  是一个创建用来基于scheduler某些event发生来执行的动作。TriggerListeners 接受triggers有关的event，JobListeners 接受有关jobs的事件。
Trigger-related events包括： trigger firings, trigger mis-firings，trigger completions（trigger完成了从而解雇job）。
Job-related events 包括:job将要执行的通知，以及job已经完成的通知。

### Using Your Own Listeners
创建一个listener，只需要实现org.quartz.TriggerListener 或者org.quartz.JobListener 。Listeners 会在运行期注册到cheduler，以及必须给一个名字（必须可以通过getName()获取他的名字）。
更方便的话，除了实现这两个接口，还可以继承JobListenerSupport 或者TriggerListenerSupport 类，只重写你感兴趣的方法。
Listeners 和Matcher （用来匹配什么job/trigger的事件可以被监听）被注册到scheduler的ListenerManager。
> Listeners 在运行时注册，并不会和jobs 以及triggers存储在JobStore 。这是因为listeners 是应用成语的一个集成点。因此，没次应用程序运行时，listeners 需要重新注册在sheduler。
> 
Adding a JobListener that is interested in a particular job:
```
scheduler.getListenerManager().addJobListener(myJobListener, KeyMatcher.jobKeyEquals(new JobKey("myJobName", "myJobGroup")));
```
你可能需要导入一些静态类
```
import static org.quartz.JobKey.*;
import static org.quartz.impl.matchers.KeyMatcher.*;
import static org.quartz.impl.matchers.GroupMatcher.*;
import static org.quartz.impl.matchers.AndMatcher.*;
import static org.quartz.impl.matchers.OrMatcher.*;
import static org.quartz.impl.matchers.EverythingMatcher.*;
...etc.
```
简洁了许多:
```
scheduler.getListenerManager().addJobListener(myJobListener, jobKeyEquals(jobKey("myJobName", "myJobGroup")));
```
Adding a JobListener that is interested in all jobs of a particular group:
```
scheduler.getListenerManager().addJobListener(myJobListener, jobGroupEquals("myJobGroup"));
```
Adding a JobListener that is interested in all jobs of two particular groups:
```
scheduler.getListenerManager().addJobListener(myJobListener, or(jobGroupEquals("myJobGroup"), jobGroupEquals("yourGroup")));
```
Adding a JobListener that is interested in all jobs:
```
scheduler.getListenerManager().addJobListener(myJobListener, allJobs());
```
注册TriggerListeners 就像上面这样。
Listeners 大多数情况下不会使用，但如果需要事件通知时这回很方便，不需要job明确的通知程序。

## SchedulerListeners
SchedulerListeners 与TriggerListeners 和 JobListeners十分相像,它接受Scheduler 本身的事件通知，而不是trigger 或 job相关联的事件。
Scheduler-related events包括： the addition of a job/trigger, the removal of a job/trigger, a serious error within the scheduler, notification of the scheduler being shutdown, and others.
Adding a SchedulerListener:
```
scheduler.getListenerManager().addSchedulerListener(mySchedListener);
```
Removing a SchedulerListener:
```
scheduler.getListenerManager().removeSchedulerListener(mySchedListener);
```


##  Job Stores
JobStore负责追踪所有的你提供给scheduler的工作数据：jobs，trigger，calendars。为你的Quartz选择合适的JobStore 十分重要。幸运的时，一旦你明白了他们的区别选择就会十分简单。你声明你的scheduler应该使用哪个JobStore （通过配置设置）在你提供给用于生产scheduler 实例的SchedulerFactory 的属性文件中。
不要在代码中直接使用JobStore 。由于一些原因，许多人希望这样使用。JobStore 是Quartz 幕后工作者。你需要告诉Quartz （通过配置）哪个JobStore 被使用，但是然后你只需要在代码中和这个Scheduler 接口工作就行了。

### RAMJobStore
RAMJobStore是一个简单的JobStore ，它的性能最高（就CPU时间来说）。RAMJobStore 通过明显的方法获取他的名字：它保存这些数据在RAM中。这也是为什么他这个快，以及为什么配置这么简单。
他的缺点是当你的应用程序结束或者崩溃时，所有的scheduling 信息都会丢失。这意味着RAMJobStore 不能实现jobs以及trigger的配置的"non-volatility"性。对于一些程序来说，它可以被接受，甚至期望这样的行为发生。但是对于一些其他的程序来说，它的代价时很大的。
使用RAMJobStore （假设你使用StdSchedulerFactory）简单的指定类名org.quartz.simpl.RAMJobStore就像 你配置quartz的JobStore 类的属性一样。

Configuring Quartz to use RAMJobStore：
```
org.quartz.jobStore.class = org.quartz.simpl.RAMJobStore
```
无需其他任何设置。
### JDBCJobStore
JDBCJobStore 正如其名，通过JDBC保存在database。 他比RAMJobStore复杂一点，而且没那么快。然而，他的性能缺陷没有那么严重，特别是当你创建数据库的主键时。在比较现代，一整套以及以及一个正常的LAN，取回以及更新一个fireing trigger通常小于10毫秒。
JDBCJobStore 可以和大部分SQL一起工作，包括了Oracle, PostgreSQL, MySQL, MS SQLServer, HSQLDB, and DB2。使用JDBCJobStore，你首先必须为Quartz创建一个表。你可以发现表的创建SQL在Quartz distribution的“docs/dbTables”里面。需要注意的是，表明均有前缀“QRTZ_”。这个前缀可以是任何你喜欢的东西，只要你告诉了JDBCJobStore 前缀是什么（在Quartz属性中）。使用不同的前缀可能对创建多套表有帮助，多个scheduler实例，在一个数据库里面。
一旦你完成了表格创建，你有一个更重要的决定在配置以及启动JDBCJobStore之前。你需要决定什么类型的事务是你的应用程序所需要的。如果你不需要绑定你的scheduling 命令（比如添加，删除trigger）给其他的事物。你可以使用JobStoreTX作为JobStore，从而使Quartz 管理这个事物（通常会选择这个）。
如果你需要Quartz和其他事物一起工作（在J2EE服务程序里），你可以使用JobStoreCMT 。这是Quartz会使app 服务包含管理这个事物。
最后一个是为JDBCJobStore能够连接数据库设置DataSource 。DataSource 定义域Quartz属性，使用有点不同的方法。一个方法是Quartz创建以及管理数据库本身，通过提供所有的数据库连接信息。另一个方式是Quartz使用一个被Quartz运行中的app server管理的数据库，需要提供JDBCJobStore 的数据库的JNDI名字。更多细节参考“docs/config”。
使用JDBCJobStore （假设你使用StdSchedulerFactory），你首先需要设置Quartz配置的JobSotre class属性为org.quartz.impl.jdbcjobstore.JobStoreTX或者org.quartz.impl.jdbcjobstore.JobStoreCMT。
Configuring Quartz to use JobStoreTx
```
org.quartz.jobStore.class = org.quartz.impl.jdbcjobstore.JobStoreTX
```
接下来，你需要选择DriverDelegate 给JobStore使用。DriverDelegate 用来负责任何可能需要特定数据库的JDBC工作。StdJDBCDelegate 是使用 “vanilla” JDBC代码（以及SQL statements） 执行工作的委托。如果没有其他特定于你的数据库的而委托，请使用这个委托-我们只为使用StdJDBCDelegate 发现问题的数据库创建数据库特定的委托。其他的委托可以在“org.quartz.impl.jdbcjobstore”中找到。其他的委托包括了DB2v6Delegate (for DB2 version 6 and earlier), HSQLDBDelegate (for HSQLDB), MSSQLDelegate (for Microsoft SQLServer), PostgreSQLDelegate (for PostgreSQL), WeblogicDelegate (for using JDBC drivers made by Weblogic), OracleDelegate (for using Oracle), and others.
一旦你选择了委托，设置他的类名为JDBCJobStore 的delegate 来使用。
```
org.quartz.jobStore.driverDelegateClass = org.quartz.impl.jdbcjobstore.StdJDBCDelegate
```
设置前缀
```
org.quartz.jobStore.tablePrefix = QRTZ_
```
最后，你需要设置JobStore应该使用哪个数据源。这个命名数据源必须定义于Quartz属性。这种情况下，我们指定Quartz应该使用数据库名为“myDS”（它定义在配置文件的别处。）
配置JDBCJobStore 的数据源：
```
org.quartz.jobStore.dataSource = myDS
```
> 如果你的Scheduler很忙（即 几乎总是需要执行数量相同于线程池容量相同的jobs，你需要设置数据源的连接数量大概是线程池容量+2）
> org.quartz.jobStore.useProperties的配置参数可悲设置为true（默认false），用来知道JDBCJobStore所有的JobDataMaps 的值都是字符串，因此能够被存储为键值对，而不是复杂的序列化的BLOB字段。这样在长期来看十分安全，避免了将非字符串class类序列化为BLOB的class版本问题。

### TerracottaJobStore
TerracottaJobStore 提供了不使用数据库的压缩性以及稳定性。这意味着你的数据库可以免于从Quartz加载，并且可以为你的应用程序其余部分保存所有资源。
TerracottaJobStore 可以应用于集群或非集群。在其中一种情况下，为你持久在重启之间的的job 数据提供一个存储介质，因为这个数据存储在Terracotta 服务。它比数据库快，但是比RAMJobStore慢比较多。
使用TerracottaJobStore （假设使用StdSchedulerFactory）简单的指定类名org.quartz.jobStore.class = org.terracotta.quartz.TerracottaJobStore ，就像你用来配置QURTZ的JobStore属性一样。然后添加一个额外的一行配置指定的Terracotta server location。
```
org.quartz.jobStore.class = org.terracotta.quartz.TerracottaJobStore
org.quartz.jobStore.tcConfigUrl = localhost:9510
```
更多信息点击[查看](http://www.terracotta.org/quartz)。

## Configuration, Resource Usage and SchedulerFactory

Quartz 的模块化，因此运行需要几个的组件拼接在一起，幸运的是有一些帮手。
主要的在Quartz工作之前需要配置的组件：
- ThreadPool
- JobStore
- DataSources (if necessary)
- The Scheduler itself
ThreadPool 为Quartz执行job时提供了一套线程。越多的线程在池里，更多的job能够并发执行。然而太多的线程停滞在系统中。大多数的Quartz用户发现5个左右线程足够了，因为他们通常只有小于100job在任何时间，这个job通常不需要在相同时间执行，并且生命周期较短。其他的的用户发现需要10，50甚至更多线程，因为他们各个scheduler有成千上万个trigger，最后平均下来有10-100任务需要在特定时刻执行。线程池的size取决与你使用scheduler做什么。没有什么确切的规则，尽量使线程池越小越好（节省资源），但是需要确保你有足够的用来执行job。请记住如果trigger启动时间来到，且没有可用线程。Quartz会block（pause）直到线程可用，然后job执行。可能会有毫秒级别的延迟。这也许会导致线程misfire-如果没有可用的线程在“misfire threshold”间。
一个ThreadPool 接口被定义于org.quartz.spi 包，你可以用任何方式创建ThreadPool 实现类。Quartz附带一个简单（但是十分安全）的线程池，名叫org.quartz.simpl.SimpleThreadPool。这个ThreadPool 简单的维护固定线程池中的线程，不会增加线程池也不会收缩。但是它除此自外相当强大，并且经过很好的测试。大多数人都用它。
JobStores 以及DataSources 已被提到过。值得注意的是，JobStores 实现了org.quartz.spi.JobStore interface接口，如果其中一个绑定的JobStores不适合你的需求，你可以创建自己的。
最后，你需要创建一个自己的Scheduler 实例。这个Scheduler 本身需要给一个名字，告诉他的RMI 设置，以及处理JobStore的实例和ThreadPool。RMI设置包括了scheduler是否应该为RMI创建他本身想server object一样，什么host，port被使用，等等。StdSchedulerFactory 可以生成用来代理Scheduler 实例，它实际上是代理在远程创建的Schedulers 。
### StdSchedulerFactory
StdSchedulerFactory是org.quartz.SchedulerFactory的实现类，它使用一套属性来创建并初始化Quartz Scheduler。这个属性通常保存和读取从文件。但是也可以被你的程序创建以及直接交给工厂。简单的调用getScheduler()会生成一个scheduler，初始化它（他的线程池，jobstore，以及数据源），以及返回他的公共接口的句柄。
有一些例子配置在“docs/config” 目录，在“Configuration” 手册的 “Reference” 章节有说。
### DirectSchedulerFactory
DirectSchedulerFactory  是另外一个SchedulerFactory 实现类。他在用编程化的方式创建Scheduler 比较有用。有以下两个缺点：1.他需要用户明白理解他做什么的。2.它不允许声明式配置，必须硬编码方式scheduler的设置。
### Logging
Quartz 使用SLF4J 框架，为了更精细的logging设置，自己看额外的文档。
如果需要捕获额外的管旭trigger启动和job执行的信息，你可能对启动org.quartz.plugins.history.LoggingJobHistoryPlugin或者org.quartz.plugins.history.LoggingTriggerHistoryPlugin比较感兴趣。
## Advanced (Enterprise) Features
### Clustering
Clustering通常和JDBC-Jobstore以及TerracottaJobStore一起工作。特征包括了加载均衡，以及故障转移（如果JobDetail的“request recovery”设置为trie）
Clustering 和JobStoreTX 或JobStoreCMT 启用时通过设置“org.quartz.jobStore.isClustered”为true。每一个cluster 的实例应该使用相同的quartz属性文件。他们必须使用相同属性文件，允许一些例外：不同的线程池size，不同的org.quartz.scheduler.instanceId属性。每一个节点在cluster 必须有唯一的instanceId，这非常容易做到，设置属性值为AUTO即可。

> 不要使用clustering 在分离的机器，除非他们的时间是同步的（使用某种时间同步服务守护进程），他们的时间差需要在一秒以内。
> 不要针对正在运行其他实例的同一套table启动非non-clustered实例。你可能需要获取严重的数据错误，又不稳定的行为。

每次启动只有一个节点触发一个job。如果一个job有一个每十秒一次的重复的trigger，在12点整一个节点将会触发这个job，12.00：10又会有一个节点启动这个job。每次不一定都是同一个节点。或多或少是随机的节点启动它。负载均衡机制关于忙 scheduler是接近随机，但是好处是相同的节点只会激活不忙的scheduler。
Clustering 与TerracottaJobStore 简单的配置scheduler使用TerracottaJobStore ，然后你的scheduler会自动设置为集群。
你可能会考虑如何设置Terracotta 服务。特别是配置选项，可以打开持久性等功能，并为HA运行Terracotta 阵列。
企业版的TerracottaJobStore 提供了进阶的Quartz 特征，它允许将作业职能定位到合适的集群节点。

### JTA Transactions

JobStoreCMT 允许在较大的JTA事务中执行Quartz Scheduer 操作。
Jobs 能够通过设置“org.quartz.scheduler.wrapJobExecutionInUserTransaction”为“true”来执行在JTA事务中。设置该属性后，一个JTA事务将在job执行之前启动，在调用执行中断之后提交。应用于所有job。
如果你需要每一个job是否是JTA事务，则需要包装它的execution，你需要使用@ExecuteInJTATransaction在每个job类上。
除了Quartz自动包装Job executions 在JTA事务，在scheduler接口上进行的调用，也会使在使用JobStoreCMT时加入事务。只要确定你在调用方法前启动了事务。你可以直接这样做，通过使用UserTransaction，或者使用你的使用容器管理事务的SessionBean 里面的scheduler代码。

## Miscellaneous Features of Quartz

### Plug-Ins
Quartz提供了接口org.quartz.spi.SchedulerPlugin用来扩展额外的方法。
Quartz附加的org.quartz.spi.SchedulerPlugin提供了各个有效的功能，能够在org.quartz.plugins中找到。提供了在scheduler启动时 job的自动调度，logging历史事件，以及在JVM退出时确保scheduler关闭清除。
### JobFactory
当一个trigger启动时，它关联的job会通过Scheduler的JobFactory 配置初始化。默认JobFactory 只是简单的调用newInstance()在job类上。你可能需要创建自己的JobFactory 实现类来完成一些类似IOC，DI生成初始化job实例的事情。
查看org.quartz.spi.JobFactory 接口，以及相关的Scheduler.setJobFactory(fact) 方法。
‘Factory-Shipped’ Jobs
Quartz提供了许多有用的job，你可以使用在程序中用来做比如发送邮件，调用EJBS的事情。这些都是开箱即用的，可以在 org.quartz.jobs 找到。


## CronTrigger Tutorial

### 介绍
cron 是一个很久远的UNIX工具，他的调度功能十分强大。CronTrigger 类基于他的调度功能实现的。
CronTrigger 使用 “cron expressions”,它能够创建启动时间表，比如每周五之类的。 
“cron expressions”非常强大，但是容易迷惑。这个教程就是解除迷惑。
### format
一个 “cron expressions”表达式时6-7个通过空格隔开的子表达式组成。field可以包含任何允许的值。


|Field Name | Mandatory   | Allowed Values | 	Allowed Special Characters |
| ----- | --------- | ----------- | ------- |
| Seconds      | YES | 0-59             | , - * /       |
| ------------ | --- | ---------------- | ------------- |
| Minutes      | YES | 0-59             | , - * /       |
| Hours        | YES | 0-23             | , - * /       |
| Day of month | YES | 1-31             | , - * ? / L W |
| Month        | YES | 1-12 or JAN-DEC  | , - * /       |
| Day of week  | YES | 1-7 or SUN-SAT   | , - * ? / L # |
| Year         | NO  | empty, 1970-2099 | , - * /       |

### Special characters

- *：表示任何可能
- ?：表示无确定值，在一种之一生效时，他比较 有用。比如，如果一个trigger每月十号，不关心他一天是星期几。则在day-of-week设置？即可。
- -:表示范围
- ，：多个值分割
- /：增量，每隔多久执行一次。5/15等于5，20，35，50
- L：代表last，代表最后。一个月的最后X天，或者本月倒数第X个星期T。最好不要和list以及range一起使用。
- W：用来表示最接近这一天的工作日（周一至周五）。
LW可以表示该月最后一个工作日。
- #：指定第几个星期几。比如6#3，表示第三个星期五。
法律字符，以及月，星期几不区分大小写。
### Notes
- 支持指定星期几和几号未完成（必须使用?在其中一个）
- 在语言环境使用夏令时时启动事件需要注意（在US，通常在2.00 am之后）。因为时间会便宜导致跳过或者重复。你可能需要在维基百科寻找细节。