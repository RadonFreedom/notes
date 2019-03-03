# Spring Boot 笔记

## Mvc 与 Spring Boot 自动配置

### `WebMvcAutoConfiguration`做了些什么

- The auto-configuration adds the following features on top of Spring’s defaults:
  - Inclusion of `ContentNegotiatingViewResolver` and `BeanNameViewResolver` beans.
  - Support for serving static resources, including support for WebJars (covered [later in this document](https://docs.spring.io/spring-boot/docs/2.1.3.RELEASE/reference/htmlsingle/#boot-features-spring-mvc-static-content))).
  - Automatic registration of `Converter`, `GenericConverter`, and `Formatter` beans.
  - Support for `HttpMessageConverters` (covered [later in this document](https://docs.spring.io/spring-boot/docs/2.1.3.RELEASE/reference/htmlsingle/#boot-features-spring-mvc-message-converters)).
  - Automatic registration of `MessageCodesResolver` (covered [later in this document](https://docs.spring.io/spring-boot/docs/2.1.3.RELEASE/reference/htmlsingle/#boot-features-spring-message-codes)).
  - Static `index.html` support.
  - Custom `Favicon` support (covered [later in this document](https://docs.spring.io/spring-boot/docs/2.1.3.RELEASE/reference/htmlsingle/#boot-features-spring-mvc-favicon)).
  - Automatic use of a `ConfigurableWebBindingInitializer` bean (covered [later in this document](https://docs.spring.io/spring-boot/docs/2.1.3.RELEASE/reference/htmlsingle/#boot-features-spring-mvc-web-binding-initializer)).

这些配置都在org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration这个Configuration组件中，下面具体分析Spring Boot中的视图解析器 `ContentNegotiatingViewResolver`

------

#### `ContentNegotiatingViewResolver`

1. 这个Bean作用是组合所有加入容器中的`ViewResolver`，来适配容器中所有的`ViewResolver#resolveViewName()`方法
2. 这个Bean在`WebMvcAutoConfiguration`中的配置如下：

```java
@Bean
@ConditionalOnBean(ViewResolver.class)
@ConditionalOnMissingBean(name = "viewResolver", value = ContentNegotiatingViewResolver.class)
public ContentNegotiatingViewResolver viewResolver(BeanFactory beanFactory) {
   ContentNegotiatingViewResolver resolver = new ContentNegotiatingViewResolver();
   resolver.setContentNegotiationManager(
         beanFactory.getBean(ContentNegotiationManager.class));
   // ContentNegotiatingViewResolver uses all the other view resolvers to locate
   // a view so it should have a high precedence
   resolver.setOrder(Ordered.HIGHEST_PRECEDENCE);
   return resolver;
}
```

3. 由IOC容器原理可以知道，在容器启动时会初始化所有的 Singleton Bean，过程为：
   - 解析、注册Bean definition
   - 利用Bean definition实例化Bean
   - 利用Bean definition为Bean注入属性值
   - 初始化Bean（调用初始化方法）
4. 因为这个`ContentNegotiatingViewResolver`继承了`ApplicationContextAware`，所以会在自定义初始化方法之前，实例化之后，调用其`ApplicationContextAware#setApplicationContext()`方法
5. 上述方法有如下的调用栈和被执行的栈顶源码：

![1551603784820](images/Spring Boot/1551603784820.png)

```java
@Override
protected void initServletContext(ServletContext servletContext) {
  
	//从ApplicationContext中获取所有是ViewResolver.class的Bean
   Collection<ViewResolver> matchingBeans =
         BeanFactoryUtils.beansOfTypeIncludingAncestors(obtainApplicationContext(), ViewResolver.class).values();
   if (this.viewResolvers == null) {
      this.viewResolvers = new ArrayList<>(matchingBeans.size());
      for (ViewResolver viewResolver : matchingBeans) {
         if (this != viewResolver) {
            this.viewResolvers.add(viewResolver);
         }
      }
   }
   else {
      for (int i = 0; i < this.viewResolvers.size(); i++) {
         ViewResolver vr = this.viewResolvers.get(i);
         if (matchingBeans.contains(vr)) {
            continue;
         }
         String name = vr.getClass().getName() + i;
         obtainApplicationContext().getAutowireCapableBeanFactory().initializeBean(vr, name);
      }
   }
   AnnotationAwareOrderComparator.sort(this.viewResolvers);
   this.cnmFactoryBean.setServletContext(servletContext);
}
```

6. **发现其实`ContentNegotiatingViewResolver`自己会在被初始化时机扫描所有容器中的所有`ViewResolver`并加入到自己的`viewResolvers`域中，当然除了自己本身**

7. `ContentNegotiatingViewResolver`毕竟是一个`ViewResolver`，当`WebMvc`需要视图解析器时，会调用它的`resolveViewName`方法，而在这个方法中`ContentNegotiatingViewResolver`遍历调用了自己的`viewResolvers`域的所有`ViewResolver#resolveViewName()`，所以在解析视图时只需要调用这一个`ViewResolver`即`ContentNegotiatingViewResolver`的方法就够了

```java
@Override
@Nullable
public View resolveViewName(String viewName, Locale locale) throws Exception {
   RequestAttributes attrs = RequestContextHolder.getRequestAttributes();
   Assert.state(attrs instanceof ServletRequestAttributes, "No current ServletRequestAttributes");
   List<MediaType> requestedMediaTypes = getMediaTypes(((ServletRequestAttributes) attrs).getRequest());
   if (requestedMediaTypes != null) {
        //使用viewResolvers域中的所有解析器解析出视图，加入到candidateViews中
      List<View> candidateViews = getCandidateViews(viewName, locale, requestedMediaTypes);
        //找到最合适的视图
      View bestView = getBestView(candidateViews, requestedMediaTypes, attrs);
      if (bestView != null) {
         return bestView;
      }
   }

   String mediaTypeInfo = logger.isDebugEnabled() && requestedMediaTypes != null ?
         " given " + requestedMediaTypes.toString() : "";
   if (this.useNotAcceptableStatusCode) {
      if (logger.isDebugEnabled()) {
         logger.debug("Using 406 NOT_ACCEPTABLE" + mediaTypeInfo);
      }
      return NOT_ACCEPTABLE_VIEW;
   }
   else {
      logger.debug("View remains unresolved" + mediaTypeInfo);
      return null;
   }
}
```

8. 配置自定义的`ViewResolver`

   ```JAVA
   @Component
   public class MyViewResolver implements ViewResolver {
       @Override
       public View resolveViewName(String viewName, Locale locale) throws Exception {
           return null;
       }
   }
   ```

   只需要这样就可以了，在上面的代码中打一个断点就能发现自定义的`ViewResolver`已经被加入到`ContentNegotiatingViewResolver#viewResolvers`之中了，可以发挥自己的解析视图的功能

   ![1551604949798](images/Spring Boot/1551604949798.png)

其实其他的所有配置也一样，只需要在容器中加入自定义配置的类就可以工作了。



### Spring Boot 中的 Mvc 配置原理

由上面的例子可以看出，如果想在在Spring Boot 中加入用户自定义配置，只需要把组件加入到容器中就行了

- 如果这个配置是单个配置类决定的，自定义配置会覆盖Spring Boot自动配置
- 如果像`ViewResolver`一样可以由多个类来决定，那么Spring Boot也会使用到自定义组件

为什么可以这样呢？在学Spring Mvc的时候是这么配置的：

```java
@Configuration
@EnableWebMvc
@ComponentScan(basePackages = {"shown.controller"})
public class ServletConfig implements WebMvcConfigurer {

    @Override
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
        configurer.enable();
    }

    @Bean
    public ViewResolver viewResolver() {

        InternalResourceViewResolver resolver =
                new InternalResourceViewResolver();
        resolver.setPrefix("/WEB-INF/view/");
        resolver.setSuffix(".jsp");
        resolver.setExposeContextBeansAsAttributes(true);
        return resolver;
    }
}
```

通过`@EnableWebMvc` Java doc 可知：

- `@EnableWebMvc`将会开启mvc模块的配置，使用`WebMvcConfigurationSupport`中提供的web Mvc默认配置
- 如果想要自定义配置，应继承`WebMvcConfigurer`，它当中的方法会覆盖`WebMvcConfigurationSupport`默认配置

下面看一下源码和继承关系：

```JAVA
@Import(DelegatingWebMvcConfiguration.class)
public @interface EnableWebMvc {
}
```

![1551605860403](images/Spring Boot/1551605860403.png)

```JAVA
// A subclass of WebMvcConfigurationSupport that detects and delegates to all beans of type WebMvcConfigurer allowing them to customize the configuration provided by WebMvcConfigurationSupport. This is the class actually imported by @EnableWebMvc.

@Configuration
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {

	private final WebMvcConfigurerComposite configurers = new WebMvcConfigurerComposite();


	@Autowired(required = false)
	public void setConfigurers(List<WebMvcConfigurer> configurers) {
		if (!CollectionUtils.isEmpty(configurers)) {
			this.configurers.addWebMvcConfigurers(configurers);
		}
	}
    
    ...
}
```

从源码和继承关系以及``DelegatingWebMvcConfiguration`Java doc中可以得到：

1. `@EnableWebMvc`通过`@Import`引入`DelegatingWebMvcConfiguration`来进行mvc默认配置的原因是，`DelegatingWebMvcConfiguration`继承了`WebMvcConfigurationSupport`

2. `@Configuration`类可以通过继承`WebMvcConfigurer`并重写方法来实现对默认配置的覆盖的原因是，容器中所有`WebMvcConfigurer`组件都会被加入到`DelegatingWebMvcConfiguration`域中，并调用其中自定义的配置方法，这也**体现出`DelegatingWebMvcConfiguration`的代理作用**

   

转过来看一下Spring Boot 的`WebMvcAutoConfiguration$WebMvcAutoConfigurationAdapter`和

`WebMvcAutoConfiguration$EnableWebMvcConfiguration`

```java
	// Defined as a nested config to ensure WebMvcConfigurer is not read when not
	// on the classpath
	@Configuration
	@Import(EnableWebMvcConfiguration.class)
	@EnableConfigurationProperties({ WebMvcProperties.class, ResourceProperties.class })
	@Order(0)
	public static class WebMvcAutoConfigurationAdapter
			implements WebMvcConfigurer, ResourceLoaderAware {
    }
```

```JAVA
// Configuration equivalent to @EnableWebMvc.
	@Configuration
	public static class EnableWebMvcConfiguration extends DelegatingWebMvcConfiguration {
    }
```

可以发现：

- `WebMvcAutoConfigurationAdapter`继承了`WebMvcConfigurer`并引入了相当于`@EnableWebMvc`注解的`EnableWebMvcConfiguration`

- 因为`EnableWebMvcConfiguration`继承了`DelegatingWebMvcConfiguration`，Java doc 才说其与`@EnableWebMvc`等效



### 自定义配置 mvc 或者完全覆盖 MVC

到现在一切都很明朗了：

- 如果想要自定义部分的配置，只需要加入相对应的组件，或者写自己的`WebMvcConfigurer`并加入到容器中。可以参考官方文档。
  - If you want to keep Spring Boot MVC features and you want to add additional [MVC configuration](https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/web.html#mvc) (interceptors, formatters, view controllers, and other features), you can add your own `@Configuration` class of type `WebMvcConfigurer` but **without** `@EnableWebMvc`. If you wish to provide custom instances of `RequestMappingHandlerMapping`, `RequestMappingHandlerAdapter`, or `ExceptionHandlerExceptionResolver`, you can declare a `WebMvcRegistrationsAdapter` instance to provide such components.



- 如果需要完全接管 mvc 配置，参考官方文档。
  - If you want to take complete control of Spring MVC, you can add your own `@Configuration` annotated with `@EnableWebMvc`.









