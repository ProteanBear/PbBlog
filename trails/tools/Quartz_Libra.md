# Quartz之一：任务调度的动态处理

Quartz是一个完全由java编写的功能丰富的开源作业调度库，可以集成到几乎任何Java应用程序中，小到独立应用程序，大到大型的电子商务系统。Quartz可以用来创建执行数十，数百乃至数万个作业的简单或复杂的计划；作业的任务被定义为标准的Java组件，它可以执行几乎任何你可能编程的任务。而且Quartz Scheduler包含许多企业级功能，例如支持JTA事务和集群。

任务调度是很多系统都会用到的功能，比如需要定期执行调度生成报表、或者博客定时更新之类的，都可以靠Quartz来完成。而需求中经常会出现，需要对调度任务进行动态添加、关闭、启动和删除，在这里本文就对Spring+Quartz实现任务调度动态处理进行说明。

> 开篇就彩蛋：其实我已将Spring+Quartz动态任务调度封装为框架工具，不想看文章解释编码可以直接转到[我的GitHub](https://github.com/ProteanBear)上的项目[Libra](https://github.com/ProteanBear/libra)。



## TL;DR

- 简单了解一下Quartz中的概念和运行原理
- 根据原理解析如何实现动态任务调度
- 在Spring中配置Quartz框架，并自定义Quartz作业工厂类
- 利用自定义注解实现动态任务类，并实现自动注入
- 实现Dubbo环境下注解的自动注入
- 附加：Libra如何来使用



## 理解Quartz的原理

先来直接看Quartz官网提供的单程序中使用示例：

```java
  SchedulerFactory schedFact = new org.quartz.impl.StdSchedulerFactory();
  Scheduler scheduler = schedFact.getScheduler();

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
  scheduler.scheduleJob(job, trigger);
  scheduler.start();
```

通过示例就可以展开了解一下Quartz中涉及到的几个类概念：

- **SchedulerFactory**：调度器工厂。这是一个接口，用于调度器的创建和管理。示例中使用的是Quartz中的默认实现。
- **Scheduler**：任务调度器。它表示一个Quartz的独立运行容器，里面注册了多个触发器（Trigger）和任务实例（JobDetail）。两者分别通过各自的组（group）和名称（name）作为容器中定位某一对象的唯一依据，因此组和名称必须唯一（触发器和任务实例的组和名称可以相同，因为对象类型不同）。
- **Job**：是一个接口，只有一个方法`void execute(JobExecutionContext context)`，开发者实现该接口定义运行任务，JobExecutionContext类提供了调度上下文的各种信息。Job运行时的信息保存在JobDataMap实例中。
- **JobDetail**：Job实例。Quartz在每次执行Job时，都重新创建一个Job实例，所以它不直接接受一个Job的实例，相反它接收一个Job实现类，以便运行时通过newInstance()的反射机制实例化Job。因此需要通过一个类来描述Job的实现类及其它相关的静态信息，如Job名字、描述、关联监听器等信息，JobDetail承担了这一角色。
- **Trigger**：触发器，描述触发Job执行的时间触发规则。
  - SimpleTrigger：当仅需触发一次或者以固定时间间隔周期执行，SimpleTrigger是最适合的选择。
  - CronTrigger：通过Cron表达式定义出各种复杂时间规则的调度方案：如每早晨9:00执行，周一、周三、周五下午5:00执行等。

我们可以简单的理解到整个**Quartz运行调度的流程**如下：

1. 通过触发器工厂（SchedulerFactory的实现类）创建一个调度器（Scheduler）;
2. 创建一个任务实例（JobDetail），为它指定实现了Job接口的实现类（示例中的HelloWord.class），并指定唯一标识（Identity）组（示例中的“group1”）和名称（示例中的“myJob”）；
3. 创建一个触发器（Trigger），为它指定时间触发规则（示例中的`simpleSchedule()`生成的SimpleTrigger），并指定唯一标识（Identity）组（示例中的“group1”）和名称（示例中的“myTrigger”）
4. 最后通过调度器（Scheduler）将任务实例（JobDetail）和触发器（Trigger）绑定在一起，并通过`start()`方法开启任务调度。

> Tips：当然Quartz还涉及到线程池等其他内容，你可以通过“[Quartz原理揭秘和源码解读](https://www.jianshu.com/p/bab8e4e32952)”或“[Quartz原理解析](https://www.cnblogs.com/zhangchengzhangtuo/p/5705672.html)”对Quartz的原理有更深入的了解，本篇文章就不详细展开了。



## 如何实现动态任务调度

了解了原理就可以开始构思如何实现动态任务调度了，其实从上面可以看出Quartz已经很好的支持了任务的动态创建、修改和删除，而我们的重点是如何抽象它以及想达到什么样的程度。

我们的需求是通过数据库（关系型或非关系型）来记录当前执行的任务处理，已经配置好的任务在系统启动时应该自动运行起来。另外，可以在系统运行时添加新的任务、对任务进行启动或停止、重新设置任务运行时间，还可以删除任务。

这里在系统运行时可以添加就涉及到两个不同的情况：

1. 添加任务指定的实现类（实现了Quartz的`Job`接口）已经存在于项目中
2. 添加任务指定的实现类（实现了Quartz的`Job`接口）不存在于项目中，需要让系统动态加载jar包

而对于实现类的处理也产生了两种方案：

- **方案一**：业务相关的实现类直接实现`Job`接口
- **方案二**：编写统一的实现类实现`Job`接口，在统一实现类中使用反射指定到某个类方法执行

这里基于**脱离任务状态区分**、**更好解耦**及**方便拓展**等方面考虑最终选择了方案二。

> Tips：在后面可以看到，方案二更加灵活。

> 补充一下Quartz中的任务状态：
>
> **无状态Job**：默认情况下都为此类型，可以并发执行。
>
> **有状态Job（StatefulJob）**：同一个实例（JobDetail）不能同时运行。在Quartz旧版本中实现StatefulJob接口，新版已经废弃了此接口，使用注解（`@DisallowConcurrentExecution`）实现。
>
> 这里使用方案二会创建一个无状态和一个有状态的统一任务实现类，但真正的业务任务类可以是一个。

确定好方案，便整理一下我们的具体实现思路如下：

1. 实现两个统一的任务实现类（有状态和无状态），通过JobDataMap传递配置数据，根据数据通过反射运行指定的真正实现类中的方法。
2. 因为要传递配置数据，则单独创建一个配置Bean用于传递。
3. 而在解决如何收集业务实现类以提供选择的问题上，决定使用自定义注解+Spring提供的注解扫描方式。
4. 要实现统一的任务实现类到真正的任务实现类的数据传递，并实现Spring环境下的自动注入。
5. 系统启动时，需要一个初始化方法访问数据并生成配置项，将配置好的任务全部添加调度。
6. 提供任务增、删、改、查以及启动停止等方法工具，便于数据接口对任务进行处理。

不过，首先我们想要将Quartz和Spring配置在一起。



## Spring结合Quartz的配置

### 添加依赖

Spring和Quartz运行在各自的容器中，需要一定的配置才能将二者融合在一起。Spring对Quartz的支持放在`context-support`包中，因此项目除了引入Quartz还需引入`spring-context-support`，要是Maven项目需要在pom.xml中添加如下内容：

```xml
<!-- quartz -->
<dependency>
    <groupId>org.quartz-scheduler</groupId>
    <artifactId>quartz</artifactId>
    <version>2.3.0</version>
</dependency>

<!-- Spring -->
...
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context-support</artifactId>
    <version>5.0.0.RELEASE</version>
</dependency>
```

> Tips：版本看你自己，我这里是最新的5.0.0

### 添加配置

Spring的配置使用xml或者Configuration方式都可以。都是要设置以下几个内容：

- 载入Quartz的属性配置文件`quartz.properties`
- 设置Bean`SchedulerFactoryBean`将Spring和Quartz连接起来
- 添加一个我们自定义的一个工厂类，这个类可以将Spring的上下文环境传导到Quartz中

我们先添加这个自定义工厂类，内容为：

```java
/**
 * Custom JobFactory, so that Job can be added to the Spring Autowired (Autowired)
 *
 * @author ProteanBear
 */
public class AutowiringSpringBeanJobFactory extends AdaptableJobFactory
{
    /**
     * Spring Context
     */
    @Autowired
    private AutowireCapableBeanFactory capableBeanFactory;

    /**
     * Create a Job instance, add Spring injection
     *
     * @param bundle A simple class (structure) used for returning execution-time data from the JobStore to the QuartzSchedulerThread
     * @return Job instance
     * @throws Exception throw from createJobInstance
     */
    @Override
    protected Object createJobInstance(TriggerFiredBundle bundle) throws Exception
    {
        Object jobInstance=super.createJobInstance(bundle);
        //Manually execute Spring injection
        capableBeanFactory.autowireBean(jobInstance);
        return jobInstance;
    }
}
```

这个类继承自`AdaptableJobFactory`（一个由Spring提供的工厂类），在构建它时它存在于Spring上下文中，我们自动注入Spring提供的`AutowireCapableBeanFactory`，通过它就可以使用代码在我们创建Job实例时进行自动注入处理。

这样就可以开始添加配置文件（xml）或者Configuration类，二者选择一个就好。

###### spring-quartz.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans-4.0.xsd">

    <!-- Load the configuration -->
    <bean id="quartzProperties" class="org.springframework.beans.factory.config.PropertiesFactoryBean">
        <property name="locations">
            <list>
                <value>classpath*:quartz.properties</value>
            </list>
        </property>
        <property name="fileEncoding" value="UTF-8"></property>
    </bean>

    <!-- With a custom JobFactory, you can inject a Spring-related job into your job -->
    <bean id="jobFactory" class="com.github.proteanbear.libra.framework.AutowiringSpringBeanJobFactory">
    </bean>

    <!-- Task scheduling factory -->
    <bean id="schedulerFactoryBean" class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
        <property name="jobFactory" ref="jobFactory"/>
        <property name="overwriteExistingJobs" value="true"/>
        <property name="quartzProperties" ref="quartzProperties"/>
        <!-- After the application is started, start the task by 5 seconds delay -->
        <property name="startupDelay" value="5"/>
        <!-- Configure the spring context through the applicationContextSchedulerContextKey property -->
        <property name="applicationContextSchedulerContextKey">
            <value>applicationContext</value>
        </property>
    </bean>
</beans>
```

###### Configuration

```java
/**
 * Spring integrated timing task framework quartz configuration
 *
 * @author ProteanBear
 */
@Configuration
public class LibraQuartzConfiguration
{
    /**
     * Load the configuration
     *
     * @return the configuration properties
     * @throws IOException The properties set is error.
     */
    @Bean
    public Properties quartzProperties() throws IOException
    {
        PropertiesFactoryBean propertiesFactoryBean=new PropertiesFactoryBean();
        propertiesFactoryBean.setLocation(new ClassPathResource("quartz.properties"));
        propertiesFactoryBean.afterPropertiesSet();
        return propertiesFactoryBean.getObject();
    }

    /**
     * With a custom JobFactory, you can inject a Spring-related job into your job.
     *
     * @param applicationContext the application context
     * @return the job factory
     */
    @Bean
    public JobFactory jobFactory(ApplicationContext applicationContext)
    {
        return new AutowiringSpringBeanJobFactory();
    }

    /**
     * Task scheduling factory
     *
     * @param jobFactory the job factory
     * @return the scheduler factory
     * @throws IOException The properties set is error.
     */
    @Bean
    public SchedulerFactoryBean schedulerFactoryBean(JobFactory jobFactory)
            throws IOException
    {
        SchedulerFactoryBean schedulerFactoryBean=new SchedulerFactoryBean();

        //configuration
        schedulerFactoryBean.setOverwriteExistingJobs(true);
        //After the application is started, start the task by 5 seconds delay
        schedulerFactoryBean.setStartupDelay(5);
        //Job factory
        schedulerFactoryBean.setJobFactory(jobFactory);
        //load properties
        schedulerFactoryBean.setQuartzProperties(quartzProperties());
        //Configure the spring context through the applicationContextSchedulerContextKey property
        schedulerFactoryBean.setApplicationContextSchedulerContextKey("applicationContext");

        return schedulerFactoryBean;
    }
}
```

###### quartz.properties

```properties
# Quartz配置
# 设置org.quartz.scheduler.skipUpdateCheck的属性为true来跳过更新检查
org.quartz.scheduler.skipUpdateCheck=true
# 线程池配置
org.quartz.threadPool.threadCount=8
```

> Tips：属性配置我这里只配置了两项，欲知详情请参见【[官方文档](http://www.quartz-scheduler.org/documentation/quartz-2.x/configuration/)】。

使用xml的话，最后不要忘记载入配置文件，也有两种方式：

- 在web.xml中，载入Spring配置（不是SpringMVC的地方）的地方，即`<context-param>`下的`<param-value>`中增加一项`classpath*:spring-quartz.xml`，当然具体位置看你放的位置。
- 在Spring的配置xml中（如`application-context.xml`）添加`<import resource="classpath*:spring-quartz.xml"/>`。





## 自定义注解与自动注入

Spring+Quartz就配置好了，下面才开始真正的编码开发。

### 创建两个统一的任务实现类

分别为有状态的`QuartzJobDispatcherDisallow`和无状态的`QuartzJobDispatcher`，但其实二者主要是注解的区别，因此添加一个抽象父类`AbstractQuartzJobDispatcher`来统一反射调用方法。

```java
/**
 * Central task super class to implement general configuration information parsing
 * and execution methods
 *
 * @author ProteanBear
 */
public abstract class AbstractQuartzJobDispatcher
{
    /**
     * Get the log object
     *
     * @return the log object
     */
    abstract protected Logger getLogger();

    /**
     * Get Spring injection factory class
     *
     * @return Spring injection factory class
     */
    abstract protected AutowireCapableBeanFactory getCapableBeanFactory();

    /**
     * Invoke the specified method by reflection
     *
     * @param jobTaskBean The job config
     * @param jobDataMap the job data
     */
    protected void invokeJobMethod(JobTaskBean jobTaskBean,JobDataMap jobDataMap)
    {
        ...
    }

    /**
     * Calculate the running time
     *
     * @param startTime The start time
     * @param endTime   The end time
     * @return The running time description
     */
    protected String calculateRunTime(Date startTime,Date endTime)
    {
        ...
    }
}
```

> Tips：两个抽象方法有子类实现，`getLogger()`可以让子类提供在日志中可以区分任务是否有状态；`getCapableBeanFactory()`将自动注入放入子类中，并获取后继续手动传递到真正反射指向的实现类中。

`QuartzJobDispatcher`继承抽象父类`AbstractQuartzJobDispatcher`实现接口`Job`（`QuartzJobDispatcherDisallow`内容一样只是多了`@DisallowConcurrentExecution`注解），并实现`execute`方法。

```java
/**
 * Generic stateless task,
 * which is responsible for the method of dispatching execution
 * through configuration information
 *
 * @author ProteanBear
 */
public class QuartzJobDispatcher extends AbstractQuartzJobDispatcher implements Job
{
    /**
     * Log
     */
    private static final Logger logger=LoggerFactory.getLogger(QuartzJobDispatcher.class);

    /**
     * Spring injection factory class
     */
    @Autowired
    private AutowireCapableBeanFactory capableBeanFactory;

    /**
     * Get the log object
     *
     * @return the log object
     */
    @Override
    protected Logger getLogger(){return logger;}

    /**
     * Get Spring injection factory class
     *
     * @return Spring injection factory class
     */
    @Override
    protected AutowireCapableBeanFactory getCapableBeanFactory(){return capableBeanFactory;}

    /**
     * Actuator
     *
     * @param context the job execution context
     * @throws JobExecutionException the exception
     */
    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException
    {
        ...
    }
}
```

方法里我们需要从`JobDataMap`中获取传递的数据，这个数据描述了实际要执行（即反射调用）的类的方法。

### 创建Bean描述任务类

###### JobTaskBean

| 属性              | 类型               | 说明                                 |
| ----------------- | ------------------ | ------------------------------------ |
| key               | String             | 任务标识                             |
| title             | String             | 任务显示名称                         |
| group             | String             | 任务组名                             |
| description       | String             | 任务描述                             |
| taskClass         | Class              | 任务对应的实现类                     |
| methodList        | List<Method>       | 任务实现类中需要执行的方法           |
| fieldSetMethodMap | Map<String,Method> | 任务实现类属性设置方法，用于传递数据 |
| concurrent        | boolean            | 任务是有状态还是无状态               |

两个任务类（有状态和无状态）中获取数据并调用抽象父类中的反射执行方法：

```java
public void execute(JobExecutionContext context) throws JobExecutionException
{
	//Get recorded task configuration information
	JobTaskBean jobTaskBean=(JobTaskBean)context.getMergedJobDataMap().get(LibraKey.CONFIG.toString());

	//Execute the method specified by the configuration information
	invokeJobMethod(jobTaskBean,context.getMergedJobDataMap());
}
```

### 利用反射调用真正的实现类中的方法

`invokeJobMethod`方法主要是两部分，第一通过反射将JobDataMap中同名称的数据（通过`fieldSetMethodMap`）传递到指定的实现类（`taskClass`）中；第二是通过反射调用`methodList`中指定的方法：

```java
protected void invokeJobMethod(JobTaskBean jobTaskBean,JobDataMap jobDataMap)
{
    //Job class
    Class jobClass=null;
    //Job method
    Object job=null;

    try
    {
        //Load config
        String name=jobTaskBean.getTitle();
        getLogger().info("Ready run job task for:"+jobTaskBean.toString());

        //Get the specified class and object
        jobClass=jobTaskBean.getTaskClass();
        job=jobClass.newInstance();
        if(job==null)
        {
            throw new Exception("Task【"+name+"】Task initialization error,dead start ！");
        }
        //Spring autowire
        getCapableBeanFactory().autowireBean(job);

        //Pass the data
        Method setMethod=null;
        Object data=null;
        for(String field : jobTaskBean.getFieldSetMethodMap().keySet())
        {
            //Get the data
            data=jobDataMap.get(field);
            if(data==null) continue;
            setMethod=jobTaskBean.getFieldSetMethodMap().get(field);

            //Set the data
            try
            {
                setMethod.invoke(job,data);
            }
            catch(Exception ex)
            {
                ex.printStackTrace();
                getLogger().error(ex.getMessage());
            }
        }

        //Traverse execution of all annotation methods
        Method method=null;
        String methodName=null;
        for(int i=0, size=jobTaskBean.getMethodList().size();i<size;i++)
        {
            method=jobTaskBean.getMethodList().get(i);
            methodName=method.getName();

            //Invoke method
            getLogger().info("Start invoke job \""+name+"\"'s method:"+methodName);
            Date startTime=new Date();
            method.invoke(job);
            getLogger().info("Invoke method "+methodName+" of job "+name+" success.Use time "+calculateRunTime(
                startTime,new Date()));
        }
    }
    catch(Exception ex)
    {
        getLogger().error(ex.getMessage(),ex);
    }
}
```

### 通过自定义注解+Spring注解扫描，获取任务实现类

上面可以看到反射其实很简单，关键是我们怎么扫描到系统中的我们编写实现类呢？

这里就用到了Spring提供的自定义注解扫描机制。首先添加一个自定义注解来标注我们的任务实现类，就叫做`JobTask`。并在为`JobTask`注解指定Spring的`@Component`，这样Spring启动时就会扫描到这个注解，而我们就可以通过Spring的上下文`applicationContext`的`getBeansWithAnnotation`方法获取到标注了`@JobTask`注解的所有类。

我们再创建两个自定义注解（`JobTaskExecute`和`JobTaskData`）分别用来标注要执行的任务方法和需要传递进来的数据属性。这样在系统启动时运行初始化方法获取到Spring上下文中标注了`@JobTask`注解的所有类，并遍历它的属性和方法查找自定义注解，最终生成`JobTaskBean`任务类描述并缓存在HashMap中。

###### 初始化及遍历查找注解生成任务类描述（来自`JobTaskUtils`）

```java
/**
 * Initialization
 */
private void init()
{
    logger.info("Start init job task!");

    if(this.applicationContext==null)
    {
        logger.error("ApplicationContext is null!");
        return;
    }

    //Initialization
    jobTaskMap=(jobTaskMap==null)?(new HashMap<>(20)):jobTaskMap;

    //Gets all the classes in the container with @JobTask annotations
    Map<String,Object> jobTaskBeanMap=this.applicationContext.getBeansWithAnnotation(JobTask.class);
    //Traverse all classes to generate task description records
    loadJobTaskBeans(jobTaskBeanMap);

    logger.info("Init job task success!");
}

/**
 * Load jobTaskBeans from map by the annotation
 *
 * @param jobTaskBeanMap the class map
 */
public void loadJobTaskBeans(Map<String,Object> jobTaskBeanMap)
{
    //Traverse all classes to generate task description records
    JobTask jobTaskAnnotation=null;
    JobTaskData jobTaskData=null;
    JobTaskExecute executeAnnotation=null;
    String key="";
    Method[] curMethods=null;
    Field[] curFields=null;
    Collection collection=jobTaskBeanMap.values();
    logger.info("Get class map at JobTask annotation by Spring,size is "+collection.size()+"!");
    for(Object object : collection)
    {
        //Get the class
        Class curClass=(object instanceof Class)
            ?((Class)object)
            :object.getClass();
        //Get annotation
        jobTaskAnnotation=(JobTask)curClass.getAnnotation(JobTask.class);
        if(jobTaskAnnotation==null) continue;

        //Get method annotation
        curMethods=curClass.getDeclaredMethods();
        List<Method> methodList=new ArrayList<>();
        for(int i=0, length=curMethods.length;i<length;i++)
        {
            Method curMethod=curMethods[i];
            executeAnnotation=curMethod.getAnnotation(JobTaskExecute.class);
            if(executeAnnotation!=null)
            {
                methodList.add(curMethod);
            }
        }
        //No method of operation, directly skip
        if(methodList.isEmpty()) continue;

        //Get field annotation @JobTaskData
        curFields=curClass.getDeclaredFields();
        Map<String,Method> fieldSetMethodMap=new HashMap<>();
        for(int i=0, length=curFields.length;i<length;i++)
        {
            Field curField=curFields[i];
            jobTaskData=curField.getAnnotation(JobTaskData.class);
            if(jobTaskData==null) continue;

            //Saved name
            String name=(StringUtils.isBlank(jobTaskData.value())?curField.getName():jobTaskData.value());
            //Get Field set method name
            String setMethodName="set"+curField.getName().substring(0,1).toUpperCase()+curField.getName()
                .substring(1);

            //Get set method
            Method setMethod=null;
            try
            {
                setMethod=curClass.getMethod(setMethodName,curField.getType());
            }
            catch(NoSuchMethodException e)
            {
                e.printStackTrace();
                logger.error(e.getMessage());
                continue;
            }
            if(setMethod==null) continue;

            //Put into map
            fieldSetMethodMap.put(name,setMethod);
        }

        //Generate a key
        key=("".equals(jobTaskAnnotation.value().trim()))?getDefaultTaskKey(curClass):jobTaskAnnotation.value();

        //Create a task description
        JobTaskBean jobTaskBean=new JobTaskBean(key,jobTaskAnnotation);
        //Set the current class
        jobTaskBean.setTaskClass(curClass);
        //Set all running methods
        jobTaskBean.setMethodList(methodList);
        //Set all field set method
        jobTaskBean.setFieldSetMethodMap(fieldSetMethodMap);
        //Set is jar class url
        jobTaskBean.setJarClassUrl(jarClassUrl(curClass));

        //Record
        jobTaskMap.put(key,jobTaskBean);

        logger.info("Record job task("+key+") for content("+jobTaskBean.toString()+")!");
    }
}
```

> Tips：这个工具类（`JobTaskUtils`）实现了Spring的`ApplicationContextAware`这样可以方便获得ApplicationContext中的所有bean；并且这个类使用`@Component`注册为Spring组件，使用`@Scope`标注为单例。

### 创建工具类操作任务

接下来我们创建一个工具类，将对任务的操作代码封装起来。可以通过它设置任务启动、暂停、删除以及立即运行。在此之前先要增加一个Bean，用来描述任务的配置属性：

###### TaskConfigBean

| 属性               | 类型         | 说明                                                         |
| ------------------ | ------------ | ------------------------------------------------------------ |
| taskId             | String       | 作为任务的唯一标识，默认使用Java的UUID生成                   |
| taskKey            | String       | 任务实现类标识，即`@JobTask`注解中的value                    |
| taskStatus         | int          | 任务状态，0为禁用、1为启用；为0时会删除指定的任务            |
| taskCron           | String       | Cron表达式，此字段不为空时使用Cron触发器，为空时为使用间隔触发器 |
| jobDataMap         | JobDataMap   | 记录要传递到任务实现类中的全部数据                           |
| taskInterval       | Integer      | taskCron为空时有效，指定间隔触发器的间隔时间                 |
| taskIntervalType   | IntervalType | taskCron为空时有效，指定间隔类型                             |
| taskIntervalRepeat | Integer      | taskCron为空时有效，指定间隔重复频率                         |

> Tips：`IntervalType`是个枚举类型，包括SECOND（秒）,MINUTE（分）,HOUR（小时）。

然后Bean里需要实现一个生成当前任务时间配置的方法，即根据当前的设置生成一个时间生成器（Quartz里的`ScheduleBuilder`），具体内容如下：

```java
public ScheduleBuilder scheduleBuilder() throws SchedulerException
{
    //Cron
    if(StringUtils.isNotBlank(taskCron))
    {
        return CronScheduleBuilder.cronSchedule(taskCron);
    }
    //Simple
    ScheduleBuilder result=null;
    if(taskInterval==null)
    {
        throw new SchedulerException("Timing settings must not be null!");
    }
    switch(taskIntervalType)
    {
        case SECOND:
            result=SimpleScheduleBuilder.simpleSchedule()
                    .withIntervalInSeconds(taskInterval)
                    .withRepeatCount(taskIntervalRepeat);
            break;
        case MINUTE:
            result=SimpleScheduleBuilder.simpleSchedule()
                    .withIntervalInMinutes(taskInterval)
                    .withRepeatCount(taskIntervalRepeat);
            break;
        case HOUR:
            result=SimpleScheduleBuilder.simpleSchedule()
                    .withIntervalInHours(taskInterval)
                    .withRepeatCount(taskIntervalRepeat);
    }
    return result;
}
```

然后就可以编写工具类中的设置任务方法了：

###### ScheduleJobUtils.java

```java
public final void set(TaskConfigBean jobConfig) throws SchedulerException
{
    if(jobConfig==null)
    {
        throw new SchedulerException("Object jobConfig is null.");
    }

    //Get the corresponding task class record
    JobTaskBean jobTaskBean=jobTaskUtils.getJobTask(jobConfig.getTaskKey());
    if(jobTaskBean==null) return;

    //Read the parameters
    //Name for the task key + configuration task id
    String name=key(jobConfig.getTaskId(),jobConfig.getTaskKey());
    String group=jobTaskBean.getGroup();
    Integer status=jobConfig.getTaskStatus();
    boolean concurrent=jobTaskBean.isConcurrent();

    //Build the task
    Scheduler scheduler=schedulerFactoryBean.getScheduler();
    TriggerKey triggerKey=TriggerKey.triggerKey(name,group);
    Trigger trigger=scheduler.getTrigger(triggerKey);

    //The task already exists, then delete the task first
    if(trigger!=null)
    {
        logger.info("Delete job task(name:"+name+",group:"+group+")");
        scheduler.deleteJob(JobKey.jobKey(name,group));
    }

    //Create a task
    Class jobClass=concurrent?QuartzJobDispatcherDisallow.class:QuartzJobDispatcher.class;
    JobDetail jobDetail=JobBuilder.newJob(jobClass)
            .withIdentity(name,group).build();
    //Set the transmission data
    jobDetail.getJobDataMap().put(LibraKey.CONFIG.toString(),jobTaskBean);
    jobDetail.getJobDataMap().putAll(jobConfig.getJobDataMap());

    //Create a timer
    trigger=TriggerBuilder.newTrigger().withIdentity(name,group)
            .withSchedule(jobConfig.scheduleBuilder()).build();

    //Add tasks to the schedule
    scheduler.scheduleJob(jobDetail,trigger);
    logger.info("Add job task(name:"+name+",group:"+group+")");

    //Task is disabled, pause the job
    if(status==0) pauseJob(jobConfig.getTaskId(),jobConfig.getTaskKey());
}

private final String key(String taskId,String taskKey)
{
    return taskKey+"_"+taskId;
}

private final JobKey jobKey(String taskId,String taskKey) throws SchedulerException
{
    //Get the task properties
    JobTaskBean jobTaskBean=jobTaskUtils.getJobTask(taskKey);
    if(jobTaskBean==null)
    {
        throw new SchedulerException("No task class for key:"+taskKey);
    }

    return JobKey.jobKey(key(taskId,taskKey),jobTaskBean.getGroup()+"");
}

private final TriggerKey triggerKey(String taskId,String taskKey) throws SchedulerException
{
    //Get the task properties
    JobTaskBean jobTaskBean=jobTaskUtils.getJobTask(taskKey);
    if(jobTaskBean==null)
    {
        throw new SchedulerException("No task class for key:"+taskKey);
    }

    return TriggerKey.triggerKey(key(taskId,taskKey),jobTaskBean.getGroup()+"");
}
```

> Tips：这里只展示了设置方法，以及Key生成相关的私有方法，诸如启动、暂停、删除等操作不过是生成Key，然后调用Quartz提供的方法处理就好了。

这样动态任务的框架封装完成，你可以自己设计数据库（关系型或者Key-value都可以），在Spring启动时读取数据然后生成`TaskConfigBean`初始化任务；并且为数据操作提供增删改查时调用`ScheduleJobUtils`对实际运行的任务进行处理。

> Tips：启动时初始化可以使用Spring的`@PostConstruct`注解。
>
> Tips：注意`ScheduleJobUtils`调用的Quartz中的`resumeJob`和`deleteJob`都是必须任务代码执行完成后才会暂停和删除。

## Dubbo环境下的自动注入

前文看到Spring提供了非常方便的方法为Bean手动实现自动注入，只需注入`AutowireCapableBeanFactory`然后执行：

```java
capableBeanFactory.autowireBean(job);
```

但是笔者在工作中使用时遇到了项目使用了Dubbo框架，也就是我们的任务调度服务中实现的任务处理类需要使用`@Reference`注解调用远程服务，所以需要解决的是这类服务类自动注入的问题。查询后发现Dubbo并未提供类似的方法（我们那时是Dubbo2，不知道Dubbo3会不会有）。那如何处理呢？！众所周知Dubbo是使用的RPC协议，并且是使用Zookeeper作为注册中心的由阿里提供的开源框架……所以说重点在于……开源啊，直接找源码来看看Dubbo自己是怎么注入的呢（吐！那说其他的一堆有啥用！）。经过一番努力和整理后，在统一任务类的父类（`AbstractQuartzJobDispatcher`）中添加如下注入方法：

```java
/**
 * Instantiate Dubbo service annotation @Reference.
 *
 * @param bean the bean
 */
private void referenceBean(Object bean)
{
    //Set by the set method
    Method[] methods=bean.getClass().getMethods();
    for(Method method : methods)
    {
        String name=method.getName();
        if(name.length()>3 && name.startsWith("set")
                && method.getParameterTypes().length==1
                && Modifier.isPublic(method.getModifiers())
                && !Modifier.isStatic(method.getModifiers()))
        {
            try
            {
                Reference reference=method.getAnnotation(Reference.class);
                if(reference!=null)
                {
                    Object value=refer(reference,method.getParameterTypes()[0]);
                    if(value!=null)
                    {
                        method.invoke(bean,new Object[]{});
                    }
                }
            }
            catch(Throwable e)
            {
                getLogger().error("Failed to init remote service reference at method "+name+" in class "+bean
                        .getClass().getName()+", cause: "+e.getMessage(),e);
            }
        }
    }

    //Through the property settings
    Field[] fields=bean.getClass().getDeclaredFields();
    for(Field field : fields)
    {
        try
        {
            if(!field.isAccessible())
            {
                field.setAccessible(true);
            }

            Reference reference=field.getAnnotation(Reference.class);
            if(reference!=null)
            {
                //Refer method interested can see for themselves, involving zk and netty
                Object value=refer(reference,field.getType());
                if(value!=null)
                {
                    field.set(bean,value);
                }
            }
        }
        catch(Throwable e)
        {
            getLogger().error(
                    "Failed to init remote service reference at filed "+field.getName()+" in class "+bean.getClass()
                            .getName()+", cause: "+e.getMessage(),e);
        }
    }
}

/**
 * Instantiate the corresponding Dubbo service object.
 *
 * @param reference The annotation of Reference
 * @param referenceClass The class of Reference
 * @return
 */
private Object refer(Reference reference,Class<?> referenceClass)
{
    //Get the interface name
    String interfaceName;
    if(!"".equals(reference.interfaceName()))
    {
        interfaceName=reference.interfaceName();
    }
    else if(!void.class.equals(reference.interfaceClass()))
    {
        interfaceName=reference.interfaceClass().getName();
    }
    else if(referenceClass.isInterface())
    {
        interfaceName=referenceClass.getName();
    }
    else
    {
        throw new IllegalStateException(
                "The @Reference undefined interfaceClass or interfaceName, and the property type "
                        +referenceClass.getName()+" is not a interface.");
    }

    //Get service object
    String key=reference.group()+"/"+interfaceName+":"+reference.version();
    ReferenceBean<?> referenceConfig=referenceConfigs.get(key);
    //Configuration does not exist, find service
    if(referenceConfig==null)
    {
        referenceConfig=new ReferenceBean<Object>(reference);
        if(void.class.equals(reference.interfaceClass())
                && "".equals(reference.interfaceName())
                && referenceClass.isInterface())
        {
            referenceConfig.setInterface(referenceClass);
        }

        ApplicationContext applicationContext=getApplicationContext();
        if(applicationContext!=null)
        {
            referenceConfig.setApplicationContext(applicationContext);

            //registry
            if(reference.registry()!=null && reference.registry().length>0)
            {
                List<RegistryConfig> registryConfigs=new ArrayList<RegistryConfig>();
                for(String registryId : reference.registry())
                {
                    if(registryId!=null && registryId.length()>0)
                    {
                        registryConfigs
                                .add(applicationContext.getBean(registryId,RegistryConfig.class));
                    }
                }
                referenceConfig.setRegistries(registryConfigs);
            }

            //consumer
            if(reference.consumer()!=null && reference.consumer().length()>0)
            {
                referenceConfig.setConsumer(applicationContext.getBean(reference.consumer(),ConsumerConfig.class));
            }

            //monitor
            if(reference.monitor()!=null && reference.monitor().length()>0)
            {
                referenceConfig.setMonitor(
                        (MonitorConfig)applicationContext.getBean(reference.monitor(),MonitorConfig.class));
            }

            //application
            if(reference.application()!=null && reference.application().length()>0)
            {
                referenceConfig.setApplication((ApplicationConfig)applicationContext
                        .getBean(reference.application(),ApplicationConfig.class));
            }

            //module
            if(reference.module()!=null && reference.module().length()>0)
            {
                referenceConfig.setModule(applicationContext.getBean(reference.module(),ModuleConfig.class));
            }

            //consumer
            if(reference.consumer()!=null && reference.consumer().length()>0)
            {
                referenceConfig.setConsumer(
                        (ConsumerConfig)applicationContext.getBean(reference.consumer(),ConsumerConfig.class));
            }

            try
            {
                referenceConfig.afterPropertiesSet();
            }
            catch(RuntimeException e)
            {
                throw e;
            }
            catch(Exception e)
            {
                throw new IllegalStateException(e.getMessage(),e);
            }
        }

        //Configuration
        referenceConfigs.putIfAbsent(key,referenceConfig);
        referenceConfig=referenceConfigs.get(key);
    }
    return referenceConfig.get();
}
```

在反射方法（`invokeJobMethod`）中调用：

```java
//Spring autowire
getCapableBeanFactory().autowireBean(job);
//Dubbo service injection
referenceBean(job);
```

经过如上的封装后，等于是在Quartz的基础上搭建了一层与具体业务脱离的动态任务处理层，它可以使用在任何有此需求的项目（因为基于Quartz所以对大型的分布式项目支持就一般）中，前面也说过我已经将它开源出来，取名为Libra。

## 关于Libra

> Libra：这里的取的意思是天秤座，没啥深层含义就是取个名字。

**开源项目地址**：[Github-ProteanBear-Libra](https://github.com/ProteanBear/libra)

**项目最新版本**：[v1.1.1](https://github.com/ProteanBear/libra/releases)

**关于项目分支**：[develop](https://github.com/ProteanBear/libra/tree/develop)(开发)、[master](https://github.com/ProteanBear/libra)(主干)、[libra-dubbo](https://github.com/ProteanBear/libra/tree/libra-dubbo)(带Dubbo支持)

**关于项目结构**：

- [libra-test-dubbo-client](https://github.com/ProteanBear/libra/tree/libra-dubbo/libra-test-dubbo-client)：测试项目，Dubbo服务消费者；仅libra-dubbo分支。
  - src/main
    - java/com/github/proteanbear/test
      - TaskService.java：任务初始化
      - TestLongTimeRunTask.java：任务实现类
    - resources
      - application-context.xml：Spring配置
      - dubbo-consumer.xml：Dubbo服务消费者配置
      - log4j.properties：日志配置
      - quartz.properties：Quartz配置
    - webapp/WEB-INF
      - web.xml
  - pom.xml：Maven项目配置
- [libra-test-dubbo-common](https://github.com/ProteanBear/libra/tree/libra-dubbo/libra-test-dubbo-common)：测试项目，Dubbo服务接口定义；仅libra-dubbo分支。
  - src/main/java/com/github/proteanbear/test
    - DubboHelloService：Dubbo测试服务接口定义
  - pom.xml：Maven项目配置
- [libra-test-dubbo-service](https://github.com/ProteanBear/libra/tree/libra-dubbo/libra-test-dubbo-service)：测试项目，Dubbo服务提供者；仅libra-dubbo分支。
  - src/main
    - java/com/github/proteanbear/test
      - DubboHelloServiceImpl：Dubbo测试服务接口实现
    - resources
      - application-context.xml：Spring配置
      - log4j.properties：日志配置
      - spring-dubbo.xml：Dubbo服务提供者配置
    - webapp/WEB-INF
      - web.xml
  - pom.xml：Maven项目配置
- [libra-test-jar-scan](https://github.com/ProteanBear/libra/tree/master/libra-test-jar-scan)：测试项目，用于生成通用的服务类以及任务实现类的Jar包；其他测试项目使用`ScanUtils`扫描工具动态加载Jar包内容并设置任务。
  - src/main/java/com/github/proteanbear/test
    - HelloService：Spring的Service，“Hello Libra!”
    - TestLongTimeRunTask.java：任务实现类
  - pom.xml：Maven项目配置
- [libra-test-jar-xml](https://github.com/ProteanBear/libra/tree/master/libra-test-jar-xml)：测试项目，使用jar包载入依赖，使用xml配置。
  - src/main
    - java/com/github/proteanbear/test
      - TaskService：任务初始化
    - resources
      - application-context.xml：Spring配置
      - log4j.properties：日志配置
      - quartz.properties：Quartz配置
    - webapp/WEB-INF
      - web.xml
  - pom.xml：Maven项目配置
- [libra-test-maven-configuration](https://github.com/ProteanBear/libra/tree/master/libra-test-maven-configuration)：测试项目，使用Maven载入项目依赖，使用Configuration配置。
  - src/main
    - java/com/github/proteanbear/test
      - TaskService：任务初始化
    - resources
      - application-context.xml：Spring配置
      - log4j.properties：日志配置
      - quartz.properties：Quartz配置
    - webapp/WEB-INF
      - web.xml
  - pom.xml：Maven项目配置
- [libra-test-maven-xml](https://github.com/ProteanBear/libra/tree/master/libra-test-maven-xml)：测试项目，使用Maven载入项目依赖，使用xml配置。
  - src/main
    - java/com/github/proteanbear/test
      - TaskService：任务初始化
    - resources
      - application-context.xml：Spring配置
      - log4j.properties：日志配置
      - quartz.properties：Quartz配置
    - webapp/WEB-INF
      - web.xml
  - pom.xml：Maven项目配置
- [libra](https://github.com/ProteanBear/libra/tree/master/libra)：框架项目
  - src/main
    - java/com/github/proteanbear/libra
      - configuration
        - LibraQuartzConfiguration：Spring+Quartz的Configuration配置
      - framework
        - AbstractQuartzJobDispatcher：统一任务类的抽象父类，实现反射方法
        - AutowiringSpringBeanJobFactory：自定义Spring中的任务生成工厂
        - JobTask：自定义注解，标注任务实现类
        - JobTaskBean：任务实现类信息描述
        - JobTaskData：自定义注解，标注任务实现类数据传输属性
        - JobTaskExecute：自定义注解，标注任务执行方法
        - LibraKey：枚举，指定项目中的通用常量
        - QuartzJobDispatcher：统一任务类，无状态
        - QuartzJobDispatcherDisallow：统一任务类型，有状态
        - TaskConfigBean：任务配置描述
      - utils
        - JobTaskUtils：任务实现类管理工具，Spring组件、单例
        - ScanUtils：文件夹及Jar包动态扫描工具，Spring组件、单例
        - ScheduleJobUtils：任务管理工具，Spring组件
        - StringUtils：字符串处理工具，全静态方法
    - resources
      - spring-quartz.xml：Spring+Quartz的xml配置
  - pom.xml：Maven项目配置

**关于两种配置**：

- Spring+Quartz的xml配置引入，Spring配置xml中添加：

```xml
<import resource="classpath*:spring-quartz.xml"/>
```

- Spring+Quartz的Configuration配置引入，Spring配置xml中添加：

```xml
<context:annotation-config />
```

> Tips：注意一定要在使用项目中添加quartz.proterties文件。
>
> Tips：SpringBoot默认支持Configuration方式。

**关于动态载入**：

在v1.1.0版本中加入了`ScanUtils`支持Jar包的动态载入，使用`scanUtils.scan(jarFile)`就可以动态载入Jar包中的内容。

测试项目中`libra-test-jar-scan`就是用于生成Jar包的项目，测试项目`libra-test-maven-xml`中配置了Maven依赖`libra-test-jar-scan`所以可以直接调用项目`libra-test-jar-scan`中的任务实现类；而测试项目`libra-test-maven-configuration`未在Maven配置中依赖此项目，但是在初始化方法中通过在使用`ScanUtils`动态加载了Jar包，同样可以调用项目`libra-test-jar-scan`中的任务实现类。

> Tips：在测试要自己去生成`libra-test-jar-scan`项目的Jar包，并在扫描前注意一下文件位置是否正确。

Quartz的动态任务调度就到这里了，这里主要还是集中于单机环境下，再有一章计划是探讨一下分布式任务调度。

> 文章有些长，如果是从头看到尾，那在这里真心感谢您的支持！