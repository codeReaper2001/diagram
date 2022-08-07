## Spring定时任务配置

在配置类上添加 `@EnableScheduling` 注解：

```java
// gulimall-seckill -> /config/ScheduledConfig.java
@EnableAsync
@EnableScheduling // <--
@Configuration
public class ScheduledConfig {
}
```

开启一个定时任务，使用 `@Scheduled` 注解，value填上cron表达式：

```java
@Async
@Scheduled(cron = "*/5 * * ? * 4")
public void hello() {
    log.info("hello...");
    try { TimeUnit.SECONDS.sleep(3); } 
    catch (InterruptedException e) { e.printStackTrace(); }
}
```

Spring定时任务的一些注意点：

1. 在Spring中表达式是**6位组成**，不允许第七位的年份

2. 在周几的的位置，**1-7代表周一到周日**

3. 定时任务不该阻塞，默认是阻塞的
   
     1. 可以让业务以异步的方式，自己提交到线程池（这里使用的线程池为自己手动设置的线程池）
     
        ```java
        CompletableFuture.runAsync(() -> {
            ...
        },execute);
        ```
        
     2. 支持定时任务线程池，设置 `TaskSchedulingPropertie`（该设置某版本无法生效）：
        
        ```properties
        spring.task.scheduling.pool.size: 5
        ```
        
     3. 让定时任务异步执行**【项目采用】**
         异步任务，方法上添加 `@Async` 注解
         解决：使用异步任务 + 定时任务来完成定时任务不阻塞的功能
         
            - 配置类添加`@EnableAsync`注解
            - 定时任务方法上添加`@Async`注解
        ```java
        @EnableAsync // <-- 添加注解EnableAsync
        @EnableScheduling
        @Configuration
        public class ScheduledConfig {
        }
        ```
        ```java
        @Async // <-- 添加注解Async
        @Scheduled(cron = "*/5 * * ? * 4")
        public void hello() {
            log.info("hello...");
            try { TimeUnit.SECONDS.sleep(3); } 
            catch (InterruptedException e) { e.printStackTrace(); }
        }
        ```
        使用的线程池大小默认为8，自动配置类为`TaskExecutionAutoConfiguration`

