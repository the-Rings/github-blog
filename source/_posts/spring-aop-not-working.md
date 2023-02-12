---
title: Spring AOP不生效
date: 2023-01-20 12:17:21
categories:
- Spring
tags:
- AOP
- Spring
---

{% post_link spring-aop 'Spring AOP基础' %}，这里不在赘述.
### AOP不生效的案例
在生产上遇到Spring Cache的注解@CacheEvict不生效，究其原因是在定时任务注解@Scheduled修饰的方法调用了其他方法，这些方法也被@CacheEvict修饰，因为这两个注解都是用Spring AOP来实现。
```Java
// 定时任务类
@Component
@RequiredArgsConstructor
public class ScheduledTask {
  private final TreasuryFuturesBinJob treasuryFuturesBinJob;
  @Scheduled(cron = "0 0/1 * * * ?")
  public void treasuryFuturesProcess() {
  	// run方法被@CacheEvict修饰
    treasuryFuturesBinJob.run();
  }
}

@Service
public class TreasuryFuturesBinJob {
  @CacheEvict(cacheNames = {AuthorizationMenu._TREASURYFUTURES, AuthorizationMenu._CFFEXMEMBER}, allEntries = true)
  public void run() {
    // do something ...
  }
}
```
### 不生效的原因分析
Spring利用动态代理实现AOP，Runtime时动态生成代理类，加载到JVM中
