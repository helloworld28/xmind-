# spring Clound 服务调用与熔断

## 编码流程

1. 依赖spring clound open feign  依赖
2.  `@EnableFeignClients`
3. 编写接口使用`@FeignClient`注解
4. 如果需要熔断则需要加配置`feign.hystrix.enabled=true`
5. 编写方法使用`@RequestMapping`注解



## 原理分析

spring 对Feign Client 做了代理

没用使用到熔断机制，则代理使用了`feign.ReflectiveFeign.FeignInvocationHandler`

根据方法签名调用到

feign.SynchronousMethodHandler#invoke

```
public Object invoke(Object[] argv) throws Throwable {
    RequestTemplate template = buildTemplateFromArgs.create(argv);
    Options options = findOptions(argv);
    Retryer retryer = this.retryer.clone();
    while (true) {
      try {
        return executeAndDecode(template, options);
      } catch (RetryableException e) {
        try {
          retryer.continueOrPropagate(e);
        } catch (RetryableException th) {
      ...
      }
    }
  }
```

feign.SynchronousMethodHandler#executeAndDecode

org.springframework.cloud.openfeign.ribbon.LoadBalancerFeignClient#execute

```
public Response execute(Request request, Request.Options options) throws IOException {
		try {
			URI asUri = URI.create(request.url());
			String clientName = asUri.getHost();
			URI uriWithoutHost = cleanUrl(request.url(), clientName);
			FeignLoadBalancer.RibbonRequest ribbonRequest = new FeignLoadBalancer.RibbonRequest(
					this.delegate, request, uriWithoutHost);

			IClientConfig requestConfig = getClientConfig(options, clientName);
			//lbClient(clientName)就根据clientName返回对应的FeignLoadBalancer
			return lbClient(clientName)
					.executeWithLoadBalancer(ribbonRequest, requestConfig).toResponse();
		}
		...
	}

```



com.netflix.client.AbstractLoadBalancerAwareClient#executeWithLoadBalancer(S, com.netflix.client.config.IClientConfig)

```
public T executeWithLoadBalancer(final S request, final IClientConfig requestConfig) throws ClientException {

		//通过这个command来选取Server
        LoadBalancerCommand<T> command = buildLoadBalancerCommand(request, requestConfig);

        try {
        	
            return command.submit(
                new ServerOperation<T>() {
                    @Override
                    public Observable<T> call(Server server) {
                    	//把选完的Server回调这个方法触发请求
                        URI finalUri = reconstructURIWithServer(server, request.getUri());
                        S requestForServer = (S) request.replaceUri(finalUri);
                        try {
                            return Observable.just(AbstractLoadBalancerAwareClient.this.execute(requestForServer, requestConfig));
                        } 
                        catch (Exception e) {
                            return Observable.error(e);
                        }
                    }
                })
                .toBlocking()
                .single();
        } catch (Exception e) {
          ...
        }
        
    }
```





选完server后调用

org.springframework.cloud.openfeign.ribbon.FeignLoadBalancer#execute

```
public RibbonResponse execute(RibbonRequest request, IClientConfig configOverride)
			throws IOException {
		Request.Options options;
		if (configOverride != null) {
			RibbonProperties override = RibbonProperties.from(configOverride);
			options = new Request.Options(override.connectTimeout(this.connectTimeout),
					override.readTimeout(this.readTimeout));
		}
		else {
			options = new Request.Options(this.connectTimeout, this.readTimeout);
		}
		//发起HTTP调用
		Response response = request.client().execute(request.toRequest(), options);
		return new RibbonResponse(request.getUri(), response);
	}
```



最后调用FeignClient的转换并发送Http请求

feign.Client.Default#convertAndSend



##如何装配-使用FeignClientFactoryBean

1. `@EnableFeignClients`注解驱动`@Import(FeignClientsRegistrar.class)`

2. `org.springframework.cloud.openfeign.FeignClientsRegistrar`

   org.springframework.cloud.openfeign.FeignClientsRegistrar#registerFeignClients

   ```
   public void registerFeignClients(AnnotationMetadata metadata,
   			BeanDefinitionRegistry registry) {
   		...
   		Map<String, Object> attrs = metadata
   				.getAnnotationAttributes(EnableFeignClients.class.getName());
   		AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(
   				FeignClient.class);
   		...
   
   ```

   它的会扫描`@FeignClient`注解的类，然后开始构建Beanfinition并注入使用方法`registerFeignClient`

   ```
   private void registerFeignClient(BeanDefinitionRegistry registry,
   			AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
   		String className = annotationMetadata.getClassName();
   		
   		//@FeignClient注解的接口会使用FeignClientFactoryBean来创建FeignClient的代理类
   		BeanDefinitionBuilder definition = BeanDefinitionBuilder
   				.genericBeanDefinition(FeignClientFactoryBean.class);
   		validate(attributes);
   		definition.addPropertyValue("url", getUrl(attributes));
   		definition.addPropertyValue("path", getPath(attributes));
   		String name = getName(attributes);
   		definition.addPropertyValue("name", name);
   		String contextId = getContextId(attributes);
   		definition.addPropertyValue("contextId", contextId);
   		definition.addPropertyValue("type", className);
   		definition.addPropertyValue("decode404", attributes.get("decode404"));
   		definition.addPropertyValue("fallback", attributes.get("fallback"));
   		definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
   		...
   		BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,new String[] { alias });
   		BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
   	}
   ```



创建一个FeignClient过程使用`org.springframework.cloud.openfeign.FeignClientFactoryBean#getTarget`

```
<T> T getTarget() {
		//这个FeignContext很重要，里面存在Feign的默认配置
		FeignContext context = this.applicationContext.getBean(FeignContext.class);
		Feign.Builder builder = feign(context);

		if (!StringUtils.hasText(this.url)) {
			if (!this.name.startsWith("http")) {
				this.url = "http://" + this.name;
			}
			else {
				this.url = this.name;
			}
			this.url += cleanPath();
			//负载均衡是在这里加上
			return (T) loadBalance(builder, context,
					new HardCodedTarget<>(this.type, this.name, this.url));
		}
		...
	}
```

其中`FeignContext`很重要里面会创建`FeignClientsConfiguration`它有如下默认配置

```

@Configuration(proxyBeanMethods = false)
public class FeignClientsConfiguration {

	...
	@Bean
	@ConditionalOnMissingBean
	public Decoder feignDecoder() {
		return new OptionalDecoder(
				new ResponseEntityDecoder(new SpringDecoder(this.messageConverters)));
	}
	
	@Bean
	@ConditionalOnMissingBean
	@ConditionalOnMissingClass("org.springframework.data.domain.Pageable")
	public Encoder feignEncoder() {
		return new SpringEncoder(this.messageConverters);
	}

	@Bean
	@ConditionalOnClass(name = "org.springframework.data.domain.Pageable")
	@ConditionalOnMissingBean
	public Encoder feignEncoderPageable() {
		PageableSpringEncoder encoder = new PageableSpringEncoder(
				new SpringEncoder(this.messageConverters));
		if (springDataWebProperties != null) {
			encoder.setPageParameter(
					springDataWebProperties.getPageable().getPageParameter());
			encoder.setSizeParameter(
					springDataWebProperties.getPageable().getSizeParameter());
			encoder.setSortParameter(
					springDataWebProperties.getSort().getSortParameter());
		}
		return encoder;
	}

	@Bean
	@ConditionalOnMissingBean
	public Contract feignContract(ConversionService feignConversionService) {
		//扩展Feign的Contract来实现SringMVC的注解来实现Feign接口配置
		return new SpringMvcContract(this.parameterProcessors, feignConversionService);
	}

	@Bean
	public FormattingConversionService feignConversionService() {
		FormattingConversionService conversionService = new DefaultFormattingConversionService();
		for (FeignFormatterRegistrar feignFormatterRegistrar : this.feignFormatterRegistrars) {
			feignFormatterRegistrar.registerFormatters(conversionService);
		}
		return conversionService;
	}

	@Bean
	@ConditionalOnMissingBean
	public Retryer feignRetryer() {
		return Retryer.NEVER_RETRY;
	}

	@Bean
	@Scope("prototype")
	@ConditionalOnMissingBean
	public Feign.Builder feignBuilder(Retryer retryer) {
		return Feign.builder().retryer(retryer);
	}

	@Bean
	@ConditionalOnMissingBean(FeignLoggerFactory.class)
	public FeignLoggerFactory feignLoggerFactory() {
		return new DefaultFeignLoggerFactory(this.logger);
	}

	@Bean
	@ConditionalOnClass(name = "org.springframework.data.domain.Page")
	public Module pageJacksonModule() {
		return new PageJacksonModule();
	}

	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass({ HystrixCommand.class, HystrixFeign.class })
	protected static class HystrixFeignConfiguration {

		@Bean
		@Scope("prototype")
		@ConditionalOnMissingBean
		@ConditionalOnProperty(name = "feign.hystrix.enabled")
		public Feign.Builder feignHystrixBuilder() {
			return HystrixFeign.builder();
		}

	}

}

```

如上面代码有含有Decoder, Encoder,Contract,Retryer,HystrixFeign熔断



它的创建者是`org.springframework.cloud.openfeign.FeignAutoConfiguration`

而这个是由spring-cloud-openfeign-core-2.2.1.RELEASE.jar 包里的spring.factories

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.openfeign.ribbon.FeignRibbonClientAutoConfiguration,\
org.springframework.cloud.openfeign.hateoas.FeignHalAutoConfiguration,\
org.springframework.cloud.openfeign.FeignAutoConfiguration,\
org.springframework.cloud.openfeign.encoding.FeignAcceptGzipEncodingAutoConfiguration,\
org.springframework.cloud.openfeign.encoding.FeignContentGzipEncodingAutoConfiguration,\
org.springframework.cloud.openfeign.loadbalancer.FeignLoadBalancerAutoConfiguration

```

这个配置文件的会在spring容器启动时被加载