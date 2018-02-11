---
title: Springboot整合Quartz定时任务调度
date: 2017-03-16 06:27:29
categories:
- 技术杂谈
tags:
- Java
---
最近因为项目中涉及到了分布式任务执行的问题，考虑到自己去利用线程池去实现这种定时的任务调度重复造轮子，从整个服务的量级和需求来看，决定在项目中使用Quartz来实现这种分布式的定时任务调度。

>> 本文中的Springboot版本为1.5.4.RELEASE，Quartz版本为2.2.3

## Quartz能做什么
Quartz是一个开源的作业调度框架，它完全由Java写成，并设计用于J2SE和J2EE应用中。它提供了巨大的灵 活性而不牺牲简单性。你能够用它来为执行一个作业而创建简单的或复杂的调度。它有很多特征，如：数据库支持，集群，插件，EJB作业预构 建，JavaMail及其它，支持cron-like表达式等等。

Quartz同时也提供了极为广泛的特性如持久化任务，集群和分布式任务等，其优点如下：

* 完全由Java实现，方便集成(Spring)
* 伸缩性
* 负载均衡
* 高可用性

下面进行一个简单的示范例子，设定每分钟在终端打印出一个Hello World.

```java
public class QuartzTest implements Job {

    @Override
    public void execute(JobExecutionContext jobExecutionContext) throws JobExecutionException {
        System.out.println("Hello World");
    }

    public static void main(String[] args) {
        try {
            // 获取调度器
            Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();

            // 创建Job执行类
            JobDetail job = JobBuilder.newJob(QuartzTest.class)
                                .storeDurably()
                                .withIdentity("hello world ", "test")
                                .build();
            // 创建trigger,每分钟运行一次,无限循环
            Trigger trigger = TriggerBuilder.newTrigger()
                    .withIdentity("hello world ", "test")
                    .withSchedule(SimpleScheduleBuilder.simpleSchedule().withIntervalInMinutes(1).repeatForever())
                    .build();

            scheduler.scheduleJob(job, trigger);

            // and start it off
            scheduler.start();
            Thread.sleep(2000);
            scheduler.shutdown();

        } catch (SchedulerException se) {
            se.printStackTrace();
        } catch (InterruptedException ie) {
            ie.printStackTrace();
        }
    }
}
```

## Quartz体系结构
从上面的例子很好的覆盖了Quartz最重要的3个基本要素:

* Scheduler(调度器)：代表一个Quartz的独立运行容器， Trigger和JobDetail可以注册到Scheduler中， 两者在Scheduler中拥有各自的组及名称， 组及名称是Scheduler查找定位容器中某一对象的依据， Trigger的组及名称必须唯一， JobDetail的组和名称也必须唯一（但可以和Trigger的组和名称相同，因为它们是不同类型的）。Scheduler定义了多个接口方法， 允许外部通过组及名称访问和控制容器中Trigger和JobDetail。
* JobDetail & Job(任务数据): Quartz每次调度Job时， 都重新创建一个Job实例， 所以它不直接接受一个Job的实例，相反它接收一个Job实现类(JobDetail:描述Job的实现类及其它相关的静态信息，如Job名字、描述、关联监听器等信息)，以便运行时通过newInstance()的反射机制实例化Job。
* Trigger(触发器)：是一个类，描述触发Job执行的时间触发规则。主要有SimpleTrigger和CronTrigger这两个子类。当且仅当需调度一次或者以固定时间间隔周期执行调度，SimpleTrigger是最适合的选择；而CronTrigger则可以通过Cron表达式定义出各种复杂时间规则的调度方案

### Quartz集群架构图
Quartz支持单机上的程序调度，其在分布式集群中也有较好的表现，集群的架构如下图所示，其通过数据库的状态统一来保证多个节点的应用每次只被调度一次（即某一时刻的调度任务只由其中一台服务器执行）。

![](http://wx3.sinaimg.cn/mw690/78d85414ly1foa1h2wwuwj20e205gt93.jpg "图1 Quartz集群架构")

Quartz集群中的每个节点是一个独立的Quartz应用，它又管理着其他的节点。该集群需要分别对每个节点分别启动或停止，不像应用服务器的集群，独立的Quartz节点并不与另一个节点或是管理节点通信。Quartz应用是通过数据库表来感知到另一应用。只有使用持久的JobStore才能完成Quqrtz集群。

### Quartz集群数据库表
Quartz的集群部署方案在架构上是分布式的，没有负责集中管理的节点，而是利用数据库锁的方式来实现集群环境下进行并发控制。BTW，分布式部署时需要保证各个节点的系统时间一致。其数据库表如下：

| Table Name | Description |
|--------|---------|
|QRTZ_CALENDARS|存储Quartz的Calendar信息|
|QRTZ_CRON_TRIGGERS	|存储CronTrigger，包括Cron表达式和时区信息|
|QRTZ_FIRED_TRIGGERS|	存储与已触发的Trigger相关的状态信息，以及相联Job的执行信息|
|QRTZ_PAUSED_TRIGGER_GRPS|	存储已暂停的Trigger组的信息|
|QRTZ_SCHEDULER_STATE|	存储少量的有关Scheduler的状态信息，和别的Scheduler实例|
|QRTZ_LOCKS|	存储程序的悲观锁的信息|
|QRTZ_JOB_DETAILS|	存储每一个已配置的Job的详细信息|
|QRTZ_JOB_LISTENERS|	存储有关已配置的JobListener的信息|
|QRTZ_SIMPLE_TRIGGERS|	存储简单的Trigger，包括重复次数、间隔、以及已触的次数|
|QRTZ_BLOG_TRIGGERS|	Trigger作为Blob类型存储|
|QRTZ_TRIGGER_LISTENERS|	存储已配置的TriggerListener的信息|
|QRTZ_TRIGGERS|	存储已配置的Trigger的信息|

### Quartz线程模型

在Quartz中有两类线程：Scheduler调度线程和任务执行线程。

* 任务执行线程：Quartz不会在主线程(QuartzSchedulerThread)中处理用户的Job。Quartz把线程管理的职责委托给ThreadPool，一般的设置使用SimpleThreadPool。
* SimpleThreadPool创建了一定数量的WorkerThread实例来使得Job能够在线程中进行处理。WorkerThread是定义在SimpleThreadPool类中的内部类，它实质上就是一个线程。

## Springboot集成Quartz
虽然Springboot有自己的定时器模块，但其功能无法适用于动态的添加任务的需求，我们在项目中集成Quartz来满足动态实时的添加任务的功能。

### Quartz配置及加载
Quartz配置文件为quartz.properties，其具体的配置项目可以参考官方的文档，内容如下：

```
#============================================================================
# Configure Main Scheduler Properties 设置全局的调度器属性
#============================================================================
org.quartz.scheduler.instanceName = ClusteredScheduler
org.quartz.scheduler.instanceId = auto

#============================================================================
# Configure Datasources
#============================================================================
org.quartz.dataSource.adrule.driver = com.mysql.jdbc.Driver
org.quartz.dataSource.adrule.URL = jdbc:mysql://yourdatabase/dbname?zeroDateTimeBehavior=convertToNull&useUnicode=true&characterEncoding=utf8&autoReconnect=true&failOverReadOnly=false
org.quartz.dataSource.adrule.user : user
org.quartz.dataSource.adrule.password : password
org.quartz.dataSource.adrule.maxConnections : 30


#============================================================================
# Configure JobStore 这里设置存储方式为数据库
#============================================================================
org.quartz.jobStore.class = org.quartz.impl.jdbcjobstore.JobStoreTX
org.quartz.jobStore.driverDelegateClass = org.quartz.impl.jdbcjobstore.StdJDBCDelegate
org.quartz.jobStore.dataSource = adrule
org.quartz.jobStore.misfireThreshold = 25000
org.quartz.jobStore.isClustered = true
org.quartz.jobStore.clusterCheckinInterval = 20000

#============================================================================
# Configure ThreadPool Quartz的任务是利用线程池执行的
#============================================================================
org.quartz.threadPool.class = org.quartz.simpl.SimpleThreadPool
org.quartz.threadPool.makeThreadsDaemons = true
org.quartz.threadPool.threadCount = 20
org.quartz.threadPool.threadPriority = 5

```
对于配置文件的加载如下，在这里需要一个问题：Spring容器可以管理Bean，但是Quartz的job是自己管理的，如果在Job中注入Spring管理的Bean，需要先把Quartz的Job也让Spring管理起来，因此，我们需要重写JobFactory如下
```java
@Component
public class SpringJobFactory extends SpringBeanJobFactory implements ApplicationContextAware {

    private transient AutowireCapableBeanFactory beanFactory;

    @Override
    public void setApplicationContext(final ApplicationContext context) {
        beanFactory = context.getAutowireCapableBeanFactory();
    }

    @Override
    protected Object createJobInstance(final TriggerFiredBundle bundle) throws Exception {
        final Object job = super.createJobInstance(bundle);
        beanFactory.autowireBean(job);
        return job;
    }

}
```

加载quartz.properties，生成全局唯一的Scheduler和SchedulerFactoryBean.
```java
@Configuration
public class QuartzSchedulerConfig {

    @Autowired
    private SpringJobFactory springJobFactory;

    /**
     * create scheduler
     */
    @Bean
    public SchedulerFactoryBean schedulerFactoryBean() throws IOException {

        SchedulerFactoryBean factory = new SchedulerFactoryBean();

        // 用于quartz集群,QuartzScheduler 启动时更新己存在的Job，这样就不用每次修改targetObject后删除qrtz_job_details表对应记录了
        Properties properties = properties();
        factory.setBeanName(properties.getProperty("org.quartz.scheduler.instanceName"));
        factory.setOverwriteExistingJobs(true);
        factory.setQuartzProperties(properties);

        // 如果你的job需要使用springboot的依赖注入，则使用spring的JobFactory，否则，使用默认的
        factory.setJobFactory(springJobFactory);
        factory.setAutoStartup(true);
        return factory;
    }

    @Bean(name = "scheduler")
    public Scheduler scheduler() throws IOException, SchedulerException {
        val scheduler = schedulerFactoryBean().getScheduler();
        return scheduler;
    }

    /**
     * Configure quartz using properties file
     */
    @Bean
    public Properties properties() throws IOException {
        Properties prop = new Properties();
        prop.load(new ClassPathResource("/quartz.properties").getInputStream());
        return prop;
    }
}

```
### JobDetail定义
编写定时任务
```Java
@DisallowConcurrentExecution
@Component
public class JobExample extends QuartzJobBean implements InitializingBean {

    // 此处只是表示可以使用springboot的依赖注入获取到调度器
     @Autowired
    Scheduler scheduler;

    @Override
    public void afterPropertiesSet() throws Exception {

    }

    @Override
    protected void executeInternal(JobExecutionContext context) throws JobExecutionException {
        // todo your job
        System.out.println("hello world, springboot with quartz");
    }
}
```

### 任务调度
举例对上面定义的定时任务进行调度
```Java
@Service
public class QuartzJobTest {

    @Autowired
    Scheduler scheduler;

    public boolean test() throws InterruptedException {
        // 创建JobExample执行类
        JobDetail job = JobBuilder.newJob(JobExample.class)
                                .storeDurably()
                                .withIdentity("hello world ", "test")
                                .build();
        
        // 可以给job里面传递参数
        job.getJobDataMap().put("jobData", "jobdata");
        
        // 创建trigger,每分钟运行一次,无限循环
        Trigger trigger = TriggerBuilder.newTrigger()
                    .withIdentity("hello world ", "test")
                    .withSchedule(SimpleScheduleBuilder.simpleSchedule().withIntervalInMinutes(1).repeatForever())
                    .build();

        // 此处不需要进行start了，springboot默认会启动调度
        scheduler.scheduleJob(job, trigger);
        return true;
    }
}
```

## 参考文献
1. [Quartz Documentation](http://quartz-scheduler.org/documentation)
2. [基于Quartz开发企业级任务调度应用](http://www.ibm.com/developerworks/cn/opensource/os-cn-quartz)












