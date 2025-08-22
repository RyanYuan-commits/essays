---
type: Java 框架
finished: "true"
---
## 1 使用方式

```java
// Application.java (主应用类)
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

在启动类上添加 `@EnableScheduling` 以启动调度功能.

```java
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import java.text.SimpleDateFormat;
import java.util.Date;

@Component
public class ScheduledTasks {

    private static final SimpleDateFormat dateFormat = new SimpleDateFormat("HH:mm:ss");
    
    @Scheduled(cron = "0 30 9 * * ?") // 每天上午9点30分执行
    public void cronTask() {
        System.out.println(dateFormat.format(new Date()));
    }
    
}
```

在 Bean 方法上添加注解并指定执行计划.

## 2 基本原理

### 2.1 注册原理

`ScheduledAnnotationBeanPostProcessor#postProcessAfterInitialization`

```java
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) {
	// ...

	Class<?> targetClass = AopProxyUtils.ultimateTargetClass(bean);
	if (!this.nonAnnotatedClasses.contains(targetClass) &&
			AnnotationUtils.isCandidateClass(targetClass, Arrays.asList(Scheduled.class, Schedules.class))) {

		// 构建 annotatedMethods, key 为 method, value 为与其绑定的 @Scheduled 注解

		if (annotatedMethods.isEmpty()) {
			// ......
		}
		else {
			// Non-empty set of methods
			annotatedMethods.forEach((method, scheduledMethods) ->
					scheduledMethods.forEach(scheduled -> processScheduled(scheduled, method, bean)));
			// log
		}
	}
	return bean;
}
```

在 Bean 初始化后, 所有绑定了 `@Scheduled` 的方法, 执行 `processScheduled`.

### 2.2 Cron job 注册流程

```java
String cron = scheduled.cron();
if (StringUtils.hasText(cron)) {
	String zone = scheduled.zone();
	if (this.embeddedValueResolver != null) {
		cron = this.embeddedValueResolver.resolveStringValue(cron);
		zone = this.embeddedValueResolver.resolveStringValue(zone);
	}
	if (StringUtils.hasLength(cron)) {
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
			tasks.add(this.registrar.scheduleCronTask(new CronTask(runnable, new CronTrigger(cron, timeZone))));
		}
	}
}
```

关键代码:

```java
tasks.add(
	this.registrar.scheduleCronTask(
		new CronTask(runnable, new CronTrigger(cron, timeZone))
	)
);
```

将配置包装成 `ScheduledTask` 并调用 `registrar.scheduleCronTask` 方法.

`scheduleCronTask` 会被触发两次, 当第二次触发的时候, 正式执行调度逻辑:

```java
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

正式指定调度逻辑的代码:

```java
scheduledTask.future = this.taskScheduler.schedule(task.getRunnable(), task.getTrigger());
```

```java
public ScheduledFuture<?> schedule(Runnable task, Trigger trigger) {  
    ScheduledExecutorService executor = getScheduledExecutor();  
    try {  
       ErrorHandler errorHandler = this.errorHandler;  
       if (errorHandler == null) {  
          errorHandler = TaskUtils.getDefaultErrorHandler(true);  
       }  
       return new ReschedulingRunnable(task, trigger, this.clock, executor, errorHandler).schedule();  
    }  
    catch (RejectedExecutionException ex) {  
       throw new TaskRejectedException("Executor [" + executor + "] did not accept task: " + task, ex);  
    }  
}
```

```java
@Nullable  
public ScheduledFuture<?> schedule() {  
    synchronized (this.triggerContextMonitor) {  
       this.scheduledExecutionTime = this.trigger.nextExecutionTime(this.triggerContext);  
       if (this.scheduledExecutionTime == null) {  
          return null;  
       }  
       long delay = this.scheduledExecutionTime.getTime() - this.triggerContext.getClock().millis();  
       this.currentFuture = this.executor.schedule(this, delay, TimeUnit.MILLISECONDS);  
       return this;  
    }  
}
```

核心执行代码:

```java
this.currentFuture = this.executor.schedule(this, delay, TimeUnit.MILLISECONDS);
```

具体的 run 方法:

```java
@Override  
public void run() {  
    Date actualExecutionTime = new Date(this.triggerContext.getClock().millis());  
    super.run();  
    Date completionTime = new Date(this.triggerContext.getClock().millis());  
    synchronized (this.triggerContextMonitor) {  
       Assert.state(this.scheduledExecutionTime != null, "No scheduled execution");  
       this.triggerContext.update(this.scheduledExecutionTime, actualExecutionTime, completionTime);  
       if (!obtainCurrentFuture().isCancelled()) {  
          schedule();  
       }  
    }  
}
```

需要注意下这一部分, 如果没有被取消, 再次触发 `schedule` 方法

```java
synchronized (this.triggerContextMonitor) {  
   Assert.state(this.scheduledExecutionTime != null, "No scheduled execution");  
   this.triggerContext.update(this.scheduledExecutionTime, actualExecutionTime, completionTime);  
   if (!obtainCurrentFuture().isCancelled()) {  
	  schedule();  
   }  
}  
```

```java
@Override  
public void run() {  
    try {  
       this.delegate.run();  
    }  
    catch (UndeclaredThrowableException ex) {  
       this.errorHandler.handleError(ex.getUndeclaredThrowable());  
    }  
    catch (Throwable ex) {  
       this.errorHandler.handleError(ex);  
    }  
}
```

最终负责执行的代码:

```java
@Override  
public void run() {  
    try {  
       ReflectionUtils.makeAccessible(this.method);  
       this.method.invoke(this.target);  
    }  
    catch (InvocationTargetException ex) {  
       ReflectionUtils.rethrowRuntimeException(ex.getTargetException());  
    }  
    catch (IllegalAccessException ex) {  
       throw new UndeclaredThrowableException(ex);  
    }  
}
```

