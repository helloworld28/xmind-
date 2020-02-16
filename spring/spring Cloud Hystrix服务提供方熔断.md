# spring Cloud Hystrix服务提供方熔断

## 用法

1. 增加依赖 

   ```
    <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
           </dependency>
   ```

2. 使用注解启用熔断器`@EnableCircuitBreaker`

3. 使用注解`@HystrixCommand`

   设置策略有：

   * 超时策略（使用线程池与Future来做）
   * 信号量（使用JUC里的信号量来做会阻塞，起限流作用）

   设置回调方法`fallbackMethod`

[https://cloud.spring.io/spring-cloud-static/Finchley.SR4/single/spring-cloud.html#_circuit_breaker_hystrix_clients]



## Spring 整合Hystrix

（整合的项目名称Hystrix-javanica）

使用了AOP来实现

```
com.netflix.hystrix.contrib.javanica.aop.aspectj.HystrixCommandAspect
```

```
@Aspect
public class HystrixCommandAspect {
    private static final Map<HystrixPointcutType, MetaHolderFactory> META_HOLDER_FACTORY_MAP;
   ....
@Pointcut("@annotation(com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand)")

    public void hystrixCommandAnnotationPointcut() {
    }

    @Pointcut("@annotation(com.netflix.hystrix.contrib.javanica.annotation.HystrixCollapser)")
    public void hystrixCollapserAnnotationPointcut() {
    }

    @Around("hystrixCommandAnnotationPointcut() || hystrixCollapserAnnotationPointcut()")
    public Object methodsAnnotatedWithHystrixCommand(final ProceedingJoinPoint joinPoint) throws Throwable {
        Method method = getMethodFromTarget(joinPoint);
        Validate.notNull(method, "failed to get method from joinPoint: %s", joinPoint);
        if (method.isAnnotationPresent(HystrixCommand.class) && method.isAnnotationPresent(HystrixCollapser.class)) {
            throw new IllegalStateException("method cannot be annotated with HystrixCommand and HystrixCollapser " +
                    "annotations at the same time");
        }
        MetaHolderFactory metaHolderFactory = META_HOLDER_FACTORY_MAP.get(HystrixPointcutType.of(method));
        MetaHolder metaHolder = metaHolderFactory.create(joinPoint);
        HystrixInvokable invokable = HystrixCommandFactory.getInstance().create(metaHolder);
        ExecutionType executionType = metaHolder.isCollapserAnnotationPresent() ?
                metaHolder.getCollapserExecutionType() : metaHolder.getExecutionType();

        Object result;
        try {
            if (!metaHolder.isObservable()) {
                result = CommandExecutor.execute(invokable, executionType, metaHolder);
            } else {
                result = executeObservable(invokable, executionType, metaHolder);
            }
        } catch (HystrixBadRequestException e) {
            throw e.getCause();
        } catch (HystrixRuntimeException e) {
            throw hystrixRuntimeExceptionToThrowable(metaHolder, e);
        }
        return result;
    }
```



## 自定义尝试实现



通过Spring AOP来自定义注解，并使用线程池来实现  

> https://docs.spring.io/spring/docs/5.2.3.RELEASE/spring-framework-reference/core.html#aop-ataspectj

```

@Configuration
@Aspect
public class MethodTimeoutAOP {
    ExecutorService executorService = Executors.newCachedThreadPool();
    @Around(" @annotation(com.powerjun.springclound.serviceprivider.annotation.MyTimeout)")
    public Object doBasicProfiling(ProceedingJoinPoint pjp) throws Throwable {

        Object[] args = pjp.getArgs();

        String name = pjp.getSignature().getName();

        Class[] classes = Stream.of(pjp.getArgs())
                .map(Object::getClass)
                .toArray(Class[]::new);

        Method method = pjp.getTarget().getClass().getMethod(name, classes);
        MyTimeout annotation = method.getAnnotation(MyTimeout.class);
        if (annotation != null) {
            Future<Object> future = executorService.submit(new Callable<Object>() {
                @Override
                public Object call() throws Exception {
                    try {
                        return pjp.proceed(args);
                    } catch (Throwable throwable) {
                        throwable.printStackTrace();
                    }
                    return null;
                }
            });
            try {
               return future.get(annotation.value(), annotation.timeUnit());
            } catch (TimeoutException e) {
                return invokeFallBackMethod(pjp.getTarget(), method, annotation.fallbackMethod(), args);
            }

        } else {
            return pjp.proceed();
        }
    }


    private Object invokeFallBackMethod(Object bean, Method method, String fallbackName, Object[] args) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {
        Method fallbackMethod = bean.getClass().getMethod(fallbackName, method.getParameterTypes());
        return fallbackMethod.invoke(bean, args);
    }
}

```

