---
title: 开发Tips系列02-AOP+自定义注解实现日志打印方法入参返回值
author: ifuncat
date: 2021-05-08 22:22:22 +0800
categories: [开发Tips]
tags: [开发Tips,日志优化,SpringAOP]
---
<style>
img{
    padding-left: 3%;
}
</style>

### 概述
利用SpringAOP+自定义注解实现打印方法的入参和返回值, 避免在方法前后编写重复的输出日志代码.

### 实现过程介绍
实现过程主要分为两步:

#### 1. 自定义以下两个个注解
- @MethodLog<br/>
  可以用于方法或类上, 用在方法上表明需要对该方法打印入参和返回结果, 用在类上表明需要
  对该类下的所有public方法打印入参和返回结果.<br/>
  有一个属性httpInfoFlag, 默认false, 即日志中不打印http方法信息, true则相反.
- @MethodLogIgnore<br/>
  用于方法, 用在方法上表明无需对该方法打印入参和返回结果.<br/>
  如果使用了@MethodLog类的某个方法不想打印入参和返回结果, 就可以使用该注解.

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD, ElementType.TYPE})
public @interface MethodLog {
    boolean httpInfoFlag() default false; //false: 日志中不打印http方法信息, true则相反
}

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
public @interface MethodLogIgnore {
}
```

#### 2. 定义日志切面
- 确定切点: 切点为使用了@MethodLog的类的public方法和使用了@MethodLog的方法.
- 使用环绕通知: 需要记录方法的入参和返回结果.
- 确定是否需要对切点打印日志: 如果某一切点上同时使用了@MethodLogIgnore, 则不对该方法打印入参/返回值日志.
- 确定是否需要打印方法的http信息: 使用反射拿到@MethodLog的httpInfoFlag属性的值, 如果为false, 即日志中不打印http方法信息, true则相反.
- 根据反射拿到方法的静态信息, 参数列表, 执行方法获得返回值, 结合方法执行耗时, 组装一个对象, 日志打印成这个对象的json串.

```java
@Component
@Aspect
@Slf4j //lombok
public class MethodLogAspect {
    /**
     * 作用于类上使用@MethodLog的该类下的所有方法, 或者作用于方法使用@MethodLog的该方法
     */
    @Around("@within(com.ifuncat.demo.export.methodLogAop.annotation.MethodLog) || @annotation(com.ifuncat.demo.export.methodLogAop.annotation.MethodLog)")
    public Object doAround(ProceedingJoinPoint joinPoint) throws Throwable {
        //方法反射对象
        Method method = ((MethodSignature) joinPoint.getSignature()).getMethod();
        //方法所在的类的实例对象
        Object target = joinPoint.getTarget();

        //判断是否打印方法日志,日志中是否需要http方法信息
        Map<String, Boolean> logHttpFlagMap = checkLogHttpInfo(method);
        Boolean methodLogFlag = logHttpFlagMap.get("methodLogFlag");
        Boolean httpInfoFlag = logHttpFlagMap.get("httpInfoFlag");
        //日志信息
        MethodLogInfo methodLogInfo = new MethodLogInfo();

        try {
            long start = System.currentTimeMillis();
            //java原生反射调用,如果该切面的注解被其他子工程引用,joinPoint.proceed()的执行结果都为null,故使用java原生反射调用
            Object result = method.invoke(target, joinPoint.getArgs());

            if (methodLogFlag) {
                //组装方法信息
                buildMethodInfo(methodLogInfo, joinPoint, httpInfoFlag);
                methodLogInfo.setTimeCost((System.currentTimeMillis() - start) + "ms");
                methodLogInfo.setResult(result);
                log.info("---MethodLog: {}", JSON.toJSONString(methodLogInfo));
            }

            return result;
        } catch (Throwable e) {
            if (methodLogFlag) {
                //组装方法信息,发生异常必须记录http信息
                buildMethodInfo(methodLogInfo, joinPoint, true);
                log.error("---MethodLog error: {}", JSON.toJSONString(methodLogInfo));
            }
            throw e;
        }
    }

    /**
     * 判断是否打印方法日志, 及判断是否打印http方法信息
     */
    private Map<String, Boolean> checkLogHttpInfo(Method method) {
        //在方法所在类上使用@MethodLog, 打印日志
        MethodLog typeAnnotation = method.getDeclaringClass().getAnnotation(MethodLog.class);
        //在方法上使用@MethodLog, 打印日志, 如果该方法所在类也使用了@MethodLog, 以方法的注解为准
        MethodLog methodAnnotation = method.getAnnotation(MethodLog.class);
        //在方法上使用@MethodLogIgnore, 不打印日志
        MethodLogIgnore methodLogIgnore = method.getAnnotation(MethodLogIgnore.class);

        boolean methodLogFlag = true; //默认打印日志
        boolean httpInfoFlag = false; //默认不打印http信息
        if (null != methodLogIgnore) {
            methodLogFlag = false;
        }
        if (null != typeAnnotation) {
            httpInfoFlag = typeAnnotation.httpInfoFlag();
        }
        if (null != methodAnnotation) {
            httpInfoFlag = methodAnnotation.httpInfoFlag();
        }

        Map<String, Boolean> map = new HashMap<>(4);
        map.put("methodLogFlag", methodLogFlag);
        map.put("httpInfoFlag", httpInfoFlag);

        return map;
    }

    /**
     * 设置方法信息,如参数,方法名等
     */
    private void buildMethodInfo(MethodLogInfo methodLogInfo, ProceedingJoinPoint joinPoint, boolean httpInfoFlag) {
        //设置请求头信息
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        if (null != attributes && httpInfoFlag) {
            HttpServletRequest request = attributes.getRequest();
            Map<String, String> httpMap = new HashMap<>(4);
            httpMap.put("ip", request.getRemoteAddr());
            httpMap.put("url", request.getRequestURL().toString());
            httpMap.put("httpMethod", request.getMethod());
            methodLogInfo.setHttpInfo(httpMap);
        }

        //方法所在的类名(非全类名)
        String simpleClassName = joinPoint.getSignature().getDeclaringTypeName();
        //方法名
        String shortMethodName = joinPoint.getSignature().getName();
        //类名#方法名
        String classMethod = simpleClassName + "#" + shortMethodName;

        Map<String, Object> paramMap = getRequestParamsByJoinPoint(joinPoint);

        methodLogInfo.setClassMethod(classMethod);
        methodLogInfo.setParamMap(paramMap);
    }

    /**
     * 获得方法入参信息,key: 参数名, value: 参数值
     */
    private Map<String, Object> getRequestParamsByJoinPoint(JoinPoint joinPoint) {
        //参数名
        String[] paramNames = ((MethodSignature) joinPoint.getSignature()).getParameterNames();
        //参数值
        Object[] paramValues = joinPoint.getArgs();

        //key:参数名, value:参数值
        Map<String, Object> requestParams = new HashMap<>();
        for (int i = 0; i < paramNames.length; i++) {
            Object value = paramValues[i];

            //如果是文件对象
            if (value instanceof MultipartFile) {
                MultipartFile file = (MultipartFile) value;
                value = file.getOriginalFilename();  //获取文件名
            }

            //排除方法入参中这两类参数
            if (value instanceof HttpServletRequest || value instanceof HttpServletResponse) {
                continue;
            }
            requestParams.put(paramNames[i], value);
        }

        return requestParams;
    }

    @Data //lombok
    private static class MethodLogInfo {
        private Map<String, String> httpInfo; //http请求信息
        private String classMethod; //方法所在的类+方法名
        private Map<String, Object> paramMap; //key: 参数名,value: 参数值
        private Object result; //返回结果对象
        private String timeCost; //方法耗时ms
    }
}
```

### 使用demo
```java
@MethodLog(httpInfoFlag = true) //记录http方法信息
@PostMapping("/testUserInfoDto")
public UserInfoDto testUserInfoDto(@RequestBody UserInfoDto userInfoDto) {
    return UserInfoDto.builder().idNo("8888888888").idType(userInfoDto.getIdType()).build(); //UserInfoDto使用了lombok的@Builder
}

//---打印日志如下
//---MethodLog: {"classMethod":"com.ifuncat.demo.export.methodLogAop.test.TestMethodLogAopController#testUserInfoDto","httpInfo":{"httpMethod":"POST","url":"http://localhost:8080/export/test/methodLogAop/testUserInfoDto","ip":"127.0.0.1"},"paramMap":{"userInfoDto":{"idNo":"123456","idType":"01","userName":"Tom"}},"result":{"idNo":"8888888888","idType":"01"},"timeCost":"0ms"}
```

### 注意点
- SpringAOP的切点都是public方法, 因此不能用于private, default, protected修饰的方法
- 如果需要记录http信息, 设置@MethodLog的属性httpInfoFlag = true
- 如果方法的入参/返回值信息较大, 谨慎使用该切面, 可以单独在该方法上使用@MethodIgnore
