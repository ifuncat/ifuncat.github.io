---
title: 开发Tips系列01-日志添加traceId从而快速定位方法调用链
author: ifuncat
date: 2021-05-08 20:22:22 +0800
categories: [开发Tips]
tags: [开发Tips,日志优化]
---
<style>
img{
    padding-left: 3%;
}
</style>

### 问题场景描述
- 一般情况下,通过日志排查问题时,需要确定方法的调用链, 并发低时同一调用链路上的日志往往相隔在一起, 结合关键词容易找到目标日志信息. 
- 如果并发比较高, 多个线程对应的调用链路的日志会糅杂在一起, 给寻找目标日志带来麻烦

### 解决思路
   如果给同一调用链路上的每条日志加上同一唯一的traceId, 那么通过这个traceId就能迅速定位整个调用链路上的所有日志, traceId+关键词就会快速找到目标日志信息.

### 实现方案
Springboot默认日志facade为slfj2, 利用slfj2.mdc(Mapped Diagnostic Context)机制为日志增加traceId, 适用于logback, log4j等slf4j的日志实现, 具体实现方式有以下四种, 同时需要调整logger.pattern, 加入traceId.
```properties
###控制台日志pattern, 带有traceId
logging.pattern.console="%d{yyyy-MM-dd HH:mm:ss} [%thread] %highlight([%X{traceId}]) %logger{36} - [%p]  %highlight(%C:%L) : %m %n"
```

#### 1. 利用serlvet的filter
```java
/**
 * filter只能处理访问servlet的的请求,如http请求, 不能支持如定时任务等异步触发的请求, 需要结合spring的切面使用
 */
@Component //使用了该注解,filter才会初始化并且生效
@WebFilter(urlPatterns = "/*", filterName = "logMdcFilter") //匹配所有
public class LogMdcFilter implements Filter {
    private static final String UNIQUE_ID = "traceId";

    @Override
    public void init(FilterConfig filterConfig) {
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        MDC.put(UNIQUE_ID, UUID.randomUUID().toString().replace("-", "").toLowerCase());
        try {
            chain.doFilter(request, response);
        } finally {
            MDC.remove(UNIQUE_ID); //注意mdc这个线程本地变量需要在这个线程执行完毕后,去掉这个traceId的数据
        }
    }

    @Override
    public void destroy() {
    }
}
```

#### 2. 利用spring的interceptor
```java
/**
 * interceptor属于Spring, 便于Spring容器管理, 与filter类似, 只能处理访问servlet/netty的请求, 如从前端发起的请求, 不能支持定时/异步任务
 */
@Component //注意: LogTraceInterceptor只是被初始化且放到容器中,但还需要注册到WebMVCConfig中,否则不会生效
public class LogTraceInterceptor extends HandlerInterceptorAdapter {
    private static final String UNIQUE_ID = "traceId";

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        MDC.put(UNIQUE_ID, UUID.randomUUID().toString().replace("-", "").toLowerCase());
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        MDC.remove(UNIQUE_ID);
    }
}

@Configuration
public class WebMVCConfig extends WebMvcConfigurationSupport {
    @Autowired
    private LogTraceInterceptor logTraceInterceptor;

    /**
     * 注意: 还需要将interceptor注册到WebMvcConfig中
     */
    @Override
    protected void addInterceptors(InterceptorRegistry registry) {
        InterceptorRegistration registration = registry.addInterceptor(logTraceInterceptor);
        registration.excludePathPatterns("/error", "/login"); //排除
        registration.addPathPatterns("/**"); //拦截上述排除以外的所有路径
    }
}
```

#### 3. 利用spring的aop解决定时/异步任务
```java
@Aspect
@Component
public class LogTraceAspect {
    private static final String UNIQUE_ID = "traceId";

    /**
     * 匹配使用了@Async 或者 @Scheduled 的方法
     */
    @Pointcut("@annotation(org.springframework.scheduling.annotation.Async) || @annotation(org.springframework.scheduling.annotation.Scheduled)")
    public void combinCut() {
    }

    @Around("combinCut()")
    public Object around(ProceedingJoinPoint point) throws Throwable {
        MDC.put(UNIQUE_ID, UUID.randomUUID().toString().replace("-", "").toLowerCase());
        Object result = point.proceed();// 执行方法
        MDC.remove(UNIQUE_ID);
        return result;
    }
}
```

#### 4. 增强ThreadPoolTaskExecutor实现MDC功能
   通过重写ThreadPoolTaskExecutor的execute和submit方法, 加入mdc功能,  使用该线程池则对应的方法执行日志上带有traceId
```java
/**
 * ThreadPoolTaskExecutor的子类,增加了日志打印线程traceId的功能
 */
public class MdcThreadPoolTaskExecutor extends ThreadPoolTaskExecutor {
    private static final String TRACE_ID = "traceId";
    private static final long serialVersionUID = 1L;

    MdcThreadPoolTaskExecutor() {
        super();
    }

    @Override
    public void execute(Runnable command) {
        super.execute(wrapExecute(command));
    }

    @Override
    public <T> Future<T> submit(Callable<T> task) {
        return super.submit(wrapSubmit(task));
    }

    private <T> Callable<T> wrapSubmit(Callable<T> task) {
        return () -> {
            if (MDC.get(TRACE_ID) == null) {
                MDC.put(TRACE_ID, UUID.randomUUID().toString().replace("-", "").toUpperCase());
            }
            try {
                return task.call();
            } finally {
                MDC.remove(TRACE_ID);
            }
        };
    }

    private Runnable wrapExecute(final Runnable runnable) {
        return () -> {
            if (MDC.get(TRACE_ID) == null) {
                MDC.put(TRACE_ID, UUID.randomUUID().toString().replace("-", "").toUpperCase());
            }
            try {
                runnable.run();
            } finally {
                MDC.remove(TRACE_ID);
            }
        };
    }
}


/**
 * 自定义配置线程池
 */
@Configuration
public class ThreadPoolConfig {
    @Bean
    @Primary
    public ThreadPoolTaskExecutor executor() {
        //使用带有MDC功能的ThreadPoolTaskExecutor类
        ThreadPoolTaskExecutor executor = new MdcThreadPoolTaskExecutor();
        int count = Runtime.getRuntime().availableProcessors(); //cpu核心数
        executor.setCorePoolSize(count); //核心线程数
        executor.setMaxPoolSize(2 * count + 1); //最大线程数
        executor.setKeepAliveSeconds(3); //核心线程数外的线程存活时间，单位s, 可以使用配置项
        executor.setQueueCapacity(40); //任务队列
        executor.setThreadNamePrefix("pool-"); //线程名称前缀
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy()); //线程拒止策略，执行者回调策略
        executor.initialize();
        return executor;
    }
}
```

### 总结
- 以上实现方案中只适用于单一系统服务中, 如果要追踪多个服务间的调用链路, 还无法做到, 需要考虑分布式链路追踪方案如skywalking, zipkin等.
- slfj2.MDC底层使用了ThreadLocal变量, 详见: LogbackMDCAdapter.java:55

### 参考
1. filter,　interceptor的区别: [https://blog.csdn.net/zzhongcy/article/details/102498081](https://blog.csdn.net/zzhongcy/article/details/102498081)
2. mdc+filter: [https://blog.csdn.net/xingbaozhen1210/article/details/89230570](https://blog.csdn.net/xingbaozhen1210/article/details/89230570)
3. properties文件配置log: [https://www.cnblogs.com/zhuwenjoyce/p/13200501.html](https://www.cnblogs.com/zhuwenjoyce/p/13200501.html)
4. MDC高级使用1: [https://blog.csdn.net/yangcheng33/article/details/80796129](https://blog.csdn.net/yangcheng33/article/details/80796129)
5. MDC高级使用2: [https://blog.csdn.net/qq_36535538/article/details/109158143](https://blog.csdn.net/qq_36535538/article/details/109158143)

