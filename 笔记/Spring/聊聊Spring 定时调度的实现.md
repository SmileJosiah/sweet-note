# 聊聊Spring 定时调度的实现

## 引言

在日常项目中定时调度是必不可少的功能，市面上有许多的成熟的定时调度框架非常的优秀，例如Quartz、XXL-Job的等，但是这些都不在我们今天要讨论的重点。

Spring中的大部分组件的使用方式都是非常容易上手，但是也是因为这种易上手易操作性，导致这些组件呈现给用户的时候就不够“透明，里面实现的一些机理需要我们下点功夫去理解。今天我们聊一聊Spring自带定时调度的实现

## 使用

我使用的Spring Boot的版本是`2.5.6`

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

启动类

```java
@EnableScheduling//添加该注解表明开启了定时调度
@SpringBootApplication
public class GuoRenCockPitServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(GuoRenCockPitServiceApplication.class, args);
    }

}
```

定时任务类

```java
@Component
public class Task {

    @Scheduled(cron = "0/1 * * * * ?") //每秒执行一次
    public void test() {
        System.out.println("当前时间:" + new Date());
    }
}
```



看控制台的输出结果：

我们可以看到程序正是以我们想要的执行方式在执行。所以一个最简单的定时任务就这样被我搭建起来了，是不是很神奇呢？区区两个注解就完成了一个定时调度，接下来我们通过从源码角度分析整个过程

## 源码分析



首先我们知道Spring Boot是通过自动配置的方式加载启动组件的，所以我们进入```@EnableScheduling```这个注解的内部一探究竟

### @EnableScheduling

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Import({SchedulingConfiguration.class}) //这一行是重点
@Documented
public @interface EnableScheduling {
}
```

在该注解引入了`SchedulingConfiguration`,说明该类`SchedulingConfiguration`是`Spring Boot`启动的时候加载定时调度任务的入口所在，我们继续往下看

### SchedulingConfiguration

```java
@Configuration(
    proxyBeanMethods = false
)
@Role(2)
public class SchedulingConfiguration {
    public SchedulingConfiguration() {
    }

    @Bean(
        name = {"org.springframework.context.annotation.internalScheduledAnnotationProcessor"}
    )
    @Role(2)
    public ScheduledAnnotationBeanPostProcessor scheduledAnnotationProcessor() {
        //初始化了一个 ScheduledAnnotationBeanPostProcessor
        return new ScheduledAnnotationBeanPostProcessor();
    }
}
```

这个配置类的最终作用是初始化了一个名为`ScheduledAnnotationBeanPostProcessor` 类

翻译一下这个类名就是：`调度注解Bean的后置处理器`，这么一翻译我感觉一下就来劲，它就是我心动的目标。

话不多说，接着往下看，这个`ScheduledAnnotationBeanPostProcessor`究竟是什么来头？

### ScheduledAnnotationBeanPostProcessor

这个类里面的代码很多，这里之列举一些最重要的代码出来，梳理整个流程即可，细节不究。 

#### postProcessAfterInitialization 方法

该方法属于Spring中一个非常重要的接口`BeanPostProcessor`的实，**作用是在Bean初始化完成后执行**

```JAVA
	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) {
		...
		Class<?> targetClass = AopProxyUtils.ultimateTargetClass(bean);
		//如果该Bean有注解，并且该Bean中使用到了 Scheduled注解或者Schedules注解
        if (!this.nonAnnotatedClasses.contains(targetClass) &&
				AnnotationUtils.isCandidateClass(targetClass, Arrays.asList(Scheduled.class, Schedules.class))) {
            // 生成一个以方法为Key，Scheduled注解为Value的Map
			Map<Method, Set<Scheduled>> annotatedMethods = MethodIntrospector.selectMethods(targetClass,
					(MethodIntrospector.MetadataLookup<Set<Scheduled>>) method -> {
						Set<Scheduled> scheduledAnnotations = AnnotatedElementUtils.getMergedRepeatableAnnotations(
								method, Scheduled.class, Schedules.class);
						return (!scheduledAnnotations.isEmpty() ? scheduledAnnotations : null);
					});
  			//没有使用到定时调度
			if (annotatedMethods.isEmpty()) {
				this.nonAnnotatedClasses.add(targetClass);
				if (logger.isTraceEnabled()) {
					logger.trace("No @Scheduled annotations found on bean class: " + targetClass);
				}
			}
			else {
                //关键代码，遍历上面的Map，执行的重要方法就是 processScheduled
				annotatedMethods.forEach((method, scheduledAnnotations) ->
						scheduledAnnotations.forEach(scheduled -> processScheduled(scheduled, method, bean)));
			}
		}
		return bean;
	}
    
```

#### processScheduled

这个方式是Spring 定时调度的核心代码所在

```JAVA
protected void processScheduled(Scheduled scheduled, Method method, Object bean) {
		try {
            //将需要定时调度的方法包装为一个可执行的任务
			Runnable runnable = createRunnable(bean, method);
			boolean processedSchedule = false;

			Set<ScheduledTask> tasks = new LinkedHashSet<>(4);

			//获取Scheduled注解中的配置信息，计算任务的延时启动时间
			long initialDelay = convertToMillis(scheduled.initialDelay(), scheduled.timeUnit());
			String initialDelayString = scheduled.initialDelayString();
			if (StringUtils.hasText(initialDelayString)) {
				Assert.isTrue(initialDelay < 0, "Specify 'initialDelay' or 'initialDelayString', not both");
				if (this.embeddedValueResolver != null) {
					initialDelayString = this.embeddedValueResolver.resolveStringValue(initialDelayString);
				}
				if (StringUtils.hasLength(initialDelayString)) {
					try {
						initialDelay = convertToMillis(initialDelayString, scheduled.timeUnit());
					}
					catch (RuntimeException ex) {
						throw new IllegalArgumentException(
								"Invalid initialDelayString value \"" + initialDelayString + "\" - cannot parse into long");
					}
				}
			}

			//校验Cron表达式
			String cron = scheduled.cron();
			if (StringUtils.hasText(cron)) {
				String zone = scheduled.zone();
				if (this.embeddedValueResolver != null) {
					cron = this.embeddedValueResolver.resolveStringValue(cron);
					zone = this.embeddedValueResolver.resolveStringValue(zone);
				}
				if (StringUtils.hasLength(cron)) {
                    //如果配置了Cron表达式那么，延时启动就无效
					Assert.isTrue(initialDelay == -1, "'initialDelay' not supported for cron triggers");
					processedSchedule = true;
					if (!Scheduled.CRON_DISABLED.equals(cron)) {
						TimeZone timeZone;
						if (StringUtils.hasText(zone)) {
							timeZone = StringUtils.parseTimeZoneString(zone);
						}
						else {
							timeZone = TimeZone.getDefault();
						}
                        //创建一个scheduleCronTask调度任务，执行CronTask传入需要执行的方法和Cron触发器
                        //然后在加入全局的调度任务列表中
						tasks.add(this.registrar.scheduleCronTask(new CronTask(runnable, new CronTrigger(cron, timeZone))));
					}
				}
			}

			
			if (initialDelay < 0) {
				initialDelay = 0;
			}
	
			long fixedDelay = convertToMillis(scheduled.fixedDelay(), scheduled.timeUnit());
			if (fixedDelay >= 0) {
				Assert.isTrue(!processedSchedule, errorMessage);
				processedSchedule = true;
                 //创建一个scheduleCronTask调度任务，执行FixedDelayTask传入需要执行的方法和Cron触发器
                  //然后在加入全局的调度任务列表中
				tasks.add(this.registrar.scheduleFixedDelayTask(new FixedDelayTask(runnable, fixedDelay, initialDelay)));
			}

			
			long fixedRate = convertToMillis(scheduled.fixedRate(), scheduled.timeUnit());
			if (fixedRate >= 0) {
				Assert.isTrue(!processedSchedule, errorMessage);
				processedSchedule = true;
				tasks.add(this.registrar.scheduleFixedRateTask(new FixedRateTask(runnable, fixedRate, initialDelay)));
			}
			String fixedRateString = scheduled.fixedRateString();
			if (StringUtils.hasText(fixedRateString)) {
				if (this.embeddedValueResolver != null) {
					fixedRateString = this.embeddedValueResolver.resolveStringValue(fixedRateString);
				}
				if (StringUtils.hasLength(fixedRateString)) {
					Assert.isTrue(!processedSchedule, errorMessage);
					processedSchedule = true;
					try {
						fixedRate = convertToMillis(fixedRateString, scheduled.timeUnit());
					}
					catch (RuntimeException ex) {
						throw new IllegalArgumentException(
								"Invalid fixedRateString value \"" + fixedRateString + "\" - cannot parse into long");
					}
                   //创建一个scheduleCronTask调度任务，执行FixedRateTask传入需要执行的方法和Cron触发器
                  //然后在加入全局的调度任务列表中
					tasks.add(this.registrar.scheduleFixedRateTask(new FixedRateTask(runnable, fixedRate, initialDelay)));
				}
			}
		
			Assert.isTrue(processedSchedule, errorMessage);

			synchronized (this.scheduledTasks) {
				Set<ScheduledTask> regTasks = this.scheduledTasks.computeIfAbsent(bean, key -> new LinkedHashSet<>(4));
				regTasks.addAll(tasks);
			}
		}
		catch (IllegalArgumentException ex) {
			throw new IllegalStateException(
					"Encountered invalid @Scheduled method '" + method.getName() + "': " + ex.getMessage());
		}
	}
```

其实这个方法是很简单的，里面的多个if判断都是为了根据Scheduled注解里面的参数生成不同类型的调度任务而已，本质上都是生成调度任务。

现在我们已经看到了如何生成调度任务了，那么任务又是由谁在执行呢？

那么我们就需要关注这句代码```this.registrar.scheduleCronTask(XXX)```,用到了另外一个关键的类`ScheduledTaskRegistrar`

### ScheduledTaskRegistrar

这个类不光可以执行通过注解扫描进入Spring 容器的调度任务，还允许我们以代码的形式创建调度任务来执行。下面是`ScheduledTaskRegistrar`的说明，为了方便大家阅读，我是用了机器翻译

> 用于向TaskScheduler注册任务的 Helper bean，通常使用 cron 表达式。
> 从 Spring 3.1 开始，当与@EnableAsync注释及其SchedulingConfigurer回调接口结合使用时， ScheduledTaskRegistrar具有更加突出的面向用户的角色。
> 自从：
> 3.0
> 也可以看看：
> org.springframework.scheduling.annotation.EnableAsync ， org.springframework.scheduling.annotation.SchedulingConfigurer

言归正传，继续讨论通过注解扫描的调度任务是如何执行的呢，下面的这个方法就能解答所有的疑惑

```JAVA
@Nullable
public ScheduledTask scheduleCronTask(CronTask task) {
	ScheduledTask scheduledTask = this.unresolvedTasks.remove(task);
	boolean newTask = false;
	if (scheduledTask == null) {
		scheduledTask = new ScheduledTask(task);
		newTask = true;
	}
	if (this.taskScheduler != null) {
		scheduledTask.future = this.taskScheduler.schedule(task.getRunnable(), task.getTrigger());
	}
	else {
		addCronTask(task);
		this.unresolvedTasks.put(task, scheduledTask);
	}
	return (newTask ? scheduledTask : null);
}
```







