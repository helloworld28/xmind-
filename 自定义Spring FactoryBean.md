# 自定义Spring FactoryBean

## 什么是FactoryBean 

factoryBean 从名字上理解也能理解它就是一个Factory，然后作为Bean的形式的存在于Spring的BeanFactory，换句话说它就是一个起工厂作用的Bean。

## FactoryBean有什么用

​		它的作用其实从名称上也能大概知道它的作用就是起一个工厂作用，就是使用了工厂模式，封装了创建对象的复杂过程，当所要创建的对象有复杂实例化过程代码，官网就推荐使用自定义`FactoryBean`的方式，把这个自定义的FactoryBean注入到容器中。

​		在大多数Spring集成其他框架时就是通过扩展这个FactoryBean的方式来用的， 比如集成Feign，Ribbon等框架，就目前来看这个FactoryBean的实现有超过50多个，所以当创建某些类复杂时，就可以利用自定义FactoryBean的这个Spring容器扩展点来做。

## 如何自定义FactoryBean

大概的过程分为两个一是实现Spring的FactoryBean接口，二是通知spring容器哪些类可由自定义的FactoryBean来创建对象，下面就是示例的代码

### ![image-20200120105432844](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200120105432844.png)

1. FooFactoryBean.java

   ```
   public class FooFactoryBean implements FactoryBean {
       private static final Logger logger = LoggerFactory.getLogger(FooFactoryBean.class);
       @Override
       public Object getObject() throws Exception {
           logger.info("FooFactoryBean create foo...");
           return new FooBean();
       }
       @Override
       public Class<?> getObjectType() {
           return FooBean.class;
       }
       @Override
       public boolean isSingleton() {
           return false;
       }
   }
   
   ```

   这个FactoryBean只是简单的创建对象

   

2. 自定义注解，能这个注解用来标识哪些是需要使用自定义的FactoryBean来创建

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface FooAnnotation {
}
```

3. 需要创建 的Bean,使用了自定的注解来标识

   ```
   @FooAnnotation
   public class FooBean {
       public String echoMessage(String message) {
           return "ECHO:" + message;
       }
   }
   
   ```

4. 绑定Bean与FactoryBean的关联

   ```
   public class FooFactoryBeanRegister implements ImportBeanDefinitionRegistrar, ResourceLoaderAware, EnvironmentAware {
       private ResourceLoader resourceLoader;
       private Environment environment;
   
       @Override
       public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
       	//扫描含有自定义注解FooAnnotation的的所有类
           ClassPathScanningCandidateComponentProvider provider = new ClassPathScanningCandidateComponentProvider(false, environment);
           provider.addIncludeFilter(new AnnotationTypeFilter(FooAnnotation.class));
           String basePackage = FooAnnotation.class.getPackage().getName();
           Set<BeanDefinition> candidateComponents = provider.findCandidateComponents(basePackage);
   
           for (BeanDefinition beanDefinition : candidateComponents) {
               AnnotationMetadata annotationMetadata = ((AnnotatedBeanDefinition) beanDefinition).getMetadata();
               registerBeanDefinition(registry, annotationMetadata);
           }
       }
   
       public void registerBeanDefinition(BeanDefinitionRegistry registry,
                                          AnnotationMetadata annotationMetadata) {
           String className = annotationMetadata.getClassName();
           //在这里把FooFactoryBean与Bean绑定起来
           BeanDefinitionBuilder definition = BeanDefinitionBuilder
                   .genericBeanDefinition(FooFactoryBean.class);
           AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();
           BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className);
           BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
       }
   
       @Override
       public void setEnvironment(Environment environment) {
           this.environment = environment;
       }
   
       @Override
       public void setResourceLoader(ResourceLoader resourceLoader) {
           this.resourceLoader = resourceLoader;
       }
   }
   ```

5. 简单引用示例

   ```
   @RestController
   @RequestMapping("/foo")
   public class FooController {
   
       @Autowired
       private FooBean fooBean;
   
       @GetMapping("/{message}")
       public String bar(@PathVariable String message) {
           return fooBean.echoMessage(message)
       }
   
   }
   ```

6. 启动类时要加上`@Import(FooFactoryBeanRegister.class)`

```
@SpringBootApplication
@Import(FooFactoryBeanRegister.class)
public class FactoryBeanDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(FactoryBeanDemoApplication.class, args);
    }

}

```

## 总结

本次的自定义Factory的源由是阅读SpringClound集成Feign的源码，集成Feign就是使用这个自定义的`FeignClientFactoryBean`，理解这个机制对后续阅读其他源码，或自己开发是遇到问题或许可通过这种方式 很容易就解决了问题，多了解一下就能相当于口袋里多了一个工具，什么时候顺手就可能用上了。