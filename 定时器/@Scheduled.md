## 开启流程

```text
@EnableScheduling -> SchedulingConfiguration -> ScheduledAnnotationBeanPostProcessor -> processScheduled -> scheduledTasks
-> ContextRefreshedEvent -> finishRegistration -> ScheduledTaskRegistrar.afterPropertiesSet -> scheduleTasks -> addScheduledTask
-> ThreadPoolTaskScheduler.schedule
```

![流程](./img/EnableScheduling.png)

![流程](https://img-blog.csdnimg.cn/img_convert/837b276cc0400f626feda24c88edd272.webp?x-oss-process=image/format,png)



@Import注解是在springboot启动时将对应类以beanDefinition对象存储到spring的BeanFactory中