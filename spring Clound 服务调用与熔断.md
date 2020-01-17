# spring Clound 服务调用与熔断

## 编码流程

1. 依赖spring clound open feign  依赖
2.  `@EnableFeignClients`
3. 编写接口使用`@FeignClient`注解
4. 编写方法使用`@RequestMapping`注解



spring 对Feign Client 做了代理

代理使用了`feign.ReflectiveFeign.FeignInvocationHandler`

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





通过下面这个类来创建FeignClient

`org.springframework.cloud.openfeign.FeignClientFactoryBean`



