# Spring Security

## Spring Security 核心组件

Spring Security 有五个核心组件：SecurityContext、SecurityContextHolder、Authentication、Userdetails 和 AuthenticationManager。下面分别介绍一下各个组件。

### SecurityContext

SecurityContext 即安全上下文，关联当前用户的安全信息。用户通过 Spring Security 的校验之后，SecurityContext 会存储验证信息，下文提到的 Authentication 对象包含当前用户的身份信息。SecurityContext 的接口签名如清单 1 所示:

##### 清单 1. SecurityContext 的接口签名

```JAVA
public interface SecurityContext extends Serializable {
       Authentication getAuthentication();
       void setAuthentication(Authentication authentication);
}
```

SecurityContext 存储在 SecurityContextHolder 中。

### SecurityContextHolder

SecurityContextHolder 存储 SecurityContext 对象。SecurityContextHolder 是一个存储代理，有三种存储模式分别是：

- MODE_THREADLOCAL：SecurityContext 存储在线程中。
- MODE_INHERITABLETHREADLOCAL：SecurityContext 存储在线程中，但子线程可以获取到父线程中的 SecurityContext。
- MODE_GLOBAL：SecurityContext 在所有线程中都相同。

SecurityContextHolder 默认使用 MODE_THREADLOCAL 模式，SecurityContext 存储在当前线程中。调用 SecurityContextHolder 时不需要显示的参数传递，在当前线程中可以直接获取到 SecurityContextHolder 对象。但是对于很多 C 端的应用（音乐播放器，游戏等等），用户登录完毕，在软件的整个生命周期中只有当前登录用户，面对这种情况 SecurityContextHolder 更适合采用 MODE_GLOBAL 模式，SecurityContext 相当于存储在应用的进程中，SecurityContext 在所有线程中都相同。

### Authentication

Authentication 即验证，表明当前用户是谁。什么是验证，比如一组用户名和密码就是验证，当然错误的用户名和密码也是验证，只不过 Spring Security 会校验失败。Authentication 接口签名如清单 2 所示:

##### 清单 2. Authentication 的接口签名

```JAVA
public interface Authentication extends Principal, Serializable {
       Collection<? extends GrantedAuthority> getAuthorities();
       Object getCredentials();
       Object getDetails();
       Object getPrincipal();
       boolean isAuthenticated();
       void setAuthenticated(boolean isAuthenticated);
}
```

Authentication 是一个接口，实现类都会定义 authorities，credentials，details，principal，authenticated 等字段，具体含义如下：

- `getAuthorities`: 获取用户权限，一般情况下获取到的是用户的角色信息。
- `getCredentials`: 获取证明用户认证的信息，通常情况下获取到的是密码等信息。
- `getDetails`: 获取用户的额外信息，比如 IP 地址、经纬度等。
- `getPrincipal`: 获取用户身份信息，在未认证的情况下获取到的是用户名，在已认证的情况下获取到的是 UserDetails (暂时理解为，当前应用用户对象的扩展)。
- `isAuthenticated`: 获取当前 Authentication 是否已认证。
- `setAuthenticated`: 设置当前 Authentication 是否已认证。

在验证前，principal 填充的是用户名，credentials 填充的是密码，detail 填充的是用户的 IP 或者经纬度之类的信息。通过验证后，Spring Security 对 Authentication 重新注入，principal 填充用户信息（包含用户名、年龄等）, authorities 会填充用户的角色信息，authenticated 会被设置为 true。重新注入的 Authentication 会被填充到 SecurityContext 中。

### UserDetails

UserDetails 提供 Spring Security 需要的用户核心信息。UserDetails 的接口签名如清单 3 所示:

##### 清单 3. UserDetails 的接口签名

```JAVA
public interface UserDetails extends Serializable {
       Collection<? extends GrantedAuthority> getAuthorities();
       String getPassword();
       String getUsername();
       boolean isAccountNonExpired();
       boolean isAccountNonLocked();
       boolean isCredentialsNonExpired();
       boolean isEnabled();
}
```

UserDetails 用 `isAccountNonExpired`, `isAccountNonLocked`，`isCredentialsNonExpired`，`isEnabled` 表示用户的状态（与下文中提到的 `DisabledException`，`LockedException`，`BadCredentialsException` 相对应），具体含义如下：

- `getAuthorites`：获取用户权限，本质上是用户的角色信息。
- `getPassword`: 获取密码。
- `getUserName`: 获取用户名。
- `isAccountNonExpired`: 账户是否过期。
- `isAccountNonLocked`: 账户是否被锁定。
- `isCredentialsNonExpired`: 密码是否过期。
- `isEnabled`: 账户是否可用。

UserDetails 也是一个接口，实现类都会继承当前应用的用户信息类，并实现 UserDetails 的接口。假设应用的用户信息类是 User，自定义的 CustomUserdetails 继承 User 类并实现 UserDetails 接口。

### AuthenticationManager

AuthenticationManager 负责校验 Authentication 对象。在 AuthenticationManager 的 authenticate 函数中，开发人员实现对 Authentication 的校验逻辑。如果 authenticate 函数校验通过，正常返回一个重新注入的 Authentication 对象；校验失败，则抛出 AuthenticationException 异常。authenticate 函数签名如清单 4 所示:

##### 清单 4. authenticate 函数签名

```JAVA
Authentication authenticate(Authentication authentication)throws AuthenticationException;
```

AuthenticationManager 可以将异常抛出的更加明确：

- 当用户不可用时抛出 `DisabledException`。
- 当用户被锁定时抛出 `LockedException`。
- 当用户密码错误时抛出 `BadCredentialsException`。

重新注入的 Authentication 会包含当前用户的详细信息，并且被填充到 SecurityContext 中，这样 Spring Security 的验证流程就完成了，Spring Security 可以识别到 "你是谁"。

### 基本校验流程示例

下面采用 Spring Security 的核心组件写一个最基本的用户名密码校验示例，如清单 5 所示:

##### 清单 5. Spring Security 核心组件伪代码

```JAVA
AuthenticationManager amanager = new CustomAuthenticationManager();
Authentication namePwd = new CustomAuthentication(“name”, “password”);
try {
       Authentication result = amanager.authenticate(namePwd);
       SecurityContextHolder.getContext.setAuthentication(result);
} catch(AuthenticationException e) {
       // TODO 验证失败
}
```

Spring Security 的核心组件易于理解，其基本校验流程是: 验证信息传递过来，验证通过，将验证信息存储到 SecurityContext 中；验证失败，做出相应的处理。

## Spring Security 在 Web 中的核心组件

#### FilterChainProxy

FilterChaniProxy 是 FilterChain 代理。

FilterChain 维护了一个 Filter 队列，这些 Filter 为 Spring Security 提供了强大的功能。

**Spring Security 在 Web 中的入口是 `javax.servlet.Filter`。**

- Spring Security 在 Filter 中创建 Authentication 对象，并调用 AuthenticationManager 进行校验。
- Spring Security 选择 Filter，而没有采用上文中 Controller 的方式有以下优点。
- Spring Security 依赖 J2EE 标准，无需依赖特定的 MVC 框架。
- 另一方面 Spring MVC 通过 Servlet 做请求转发，如果 Spring Security 采用 Servlet，那么 Spring Security 和 Spring MVC 的集成会存在问题。
- FilterChain 维护了很多 Filter，每个 Filter 都有自己的功能，因此在 Spring Security 中添加新功能时，推荐通过 Filter 的方式来实现。

#### ProviderManager

**ProviderManager 是 AuthenticationManager 的实现类。**

ProviderManager 并没有实现对 Authentication 的校验功能，而是**采用代理模式将校验功能交给 AuthenticationProvider 去实现。**

这样设计是因为在 Web 环境中可能会支持多种不同的验证方式，比如用户名密码登录、短信登录、指纹登录等等，如果每种验证方式的代码都写在 ProviderManager 中，想想都是灾难。因此为每种验证方式提供对应的 AuthenticationProvider，ProviderManager 将验证任务代理给对应的 AuthenticationProvider，这是一种不错的解决方案。在 ProviderManager 中可以找到以下代码，如清单 7 所示:

##### 清单 7. ProviderManager 代码片段

```JAVA
private List<AuthenticationProvider> providers;
public Authentication authenticate(Authentication authentication)
                      throws AuthenticationException {
       ......
       for (AuthenticationProvider provider : getProviders()) {
              if (!provider.supports(toTest)) {
                      continue;
              }
              try {
                      result = provider.authenticate(authentication);
                      if (result != null) {
                             copyDetails(authentication, result);
                             break;
                      }
              }
       }
}
```

ProviderManager 维护了一个 AuthenticationProvider 队列。当 Authentication 传递进来时，ProviderManager 通过 supports 函数查找支持校验的 AuthenticationProvider。如果没有找到支持的 AuthenticationProvider 将抛出`ProviderNotFoundException` 异常。

#### AuthenticationProvider

AuthenticationProvider 是在 Web 环境中真正对 Authentication 进行校验的组件。其接口签名如清单 8 所示:

##### 清单 8. AuthenticationProvider 的接口签名

```JAVA
public interface AuthenticationProvider {
       Authentication authenticate(Authentication authentication)
                      throws AuthenticationException;
       boolean supports(Class<?> authentication);
}
```

其中，authenticate 函数用于校验 Authentication 对象；supports 函数用于判断 provider 是否支持校验 Authentication 对象。

当应用添加新的验证方式时，验证逻辑需要写在对应 AuthenticationProvider 中的 authenticate 函数中。验证通过返回一个重新注入的 Authentication，验证失败抛出 `AuthenticationException` 异常。



## 核心组件的源码分析

### `@EnableWebSecurity`: 从配置到过滤器链`springSecurityFilterChain`

Javadoc: 

> Add this annotation to an @Configuration class to have the Spring Security configuration defined in any WebSecurityConfigurer or more likely by extending the WebSecurityConfigurerAdapter base class and overriding individual methods.

可以知道, 这个注解开启了一个继承`WebSecurityConfigurerAdapter` 的`@Configuration`类中的 Spring Security 配置. 

看一下源码: 

```JAVA
@Import({ 
    	//Spring Boot中Security的关键配置类
    	WebSecurityConfiguration.class,
    	//Used by EnableWebSecurity to conditionally import WebMvcSecurityConfiguration
    	//when the DispatcherServlet is present on the classpath.
		SpringWebMvcImportSelector.class,
    	//Used by EnableWebSecurity to conditionally import OAuth2ClientConfiguration 
    	//when the spring-security-oauth2-client module is present on the classpath.
		OAuth2ImportSelector.class })

@EnableGlobalAuthentication //源码是@Import(AuthenticationConfiguration.class)
//被注解类会被加入容器中
@Configuration
public @interface EnableWebSecurity {
	//是否开启debug
	boolean debug() default false;
}
```

接下来对`WebSecurityConfiguration`和`AuthenticationConfiguration`作说明.

#### `WebSecurityConfiguration`

Javadoc:

> Uses a `WebSecurity` to create the `FilterChainProxy` that performs the web based security for Spring Security. It then exports the necessary beans. 
>
> Customizations can be made to WebSecurity by extending WebSecurityConfigurerAdapter and exposing it as a Configuration or implementing WebSecurityConfigurer and exposing it as a Configuration. This configuration is imported when using EnableWebSecurity.

关键的源码:

```JAVA
@Configuration
public class WebSecurityConfiguration implements ImportAware, BeanClassLoaderAware {
	//下面两个私有域在setFilterChainProxySecurityConfigurer方法中赋值
    private WebSecurity webSecurity;
    private List<SecurityConfigurer<Filter, WebSecurity>> webSecurityConfigurers;
    
    /*
    A class used to get all the WebSecurityConfigurer instances from the current 
    ApplicationContext but ignoring the parent.
    */
    @Bean //加入容器
	public static AutowiredWebSecurityConfigurersIgnoreParents
        autowiredWebSecurityConfigurersIgnoreParents(
			ConfigurableListableBeanFactory beanFactory) {
		return new AutowiredWebSecurityConfigurersIgnoreParents(beanFactory);
	}
    
    //Autowired注解在方法上相当于这个Bean的初始化方法之一
    //会在该Bean(WebSecurityConfiguration)初始化时执行
	@Autowired(required = false)
	public void setFilterChainProxySecurityConfigurer(
			ObjectPostProcessor<Object> objectPostProcessor,
        /*
        调用autowiredWebSecurityConfigurersIgnoreParents.getWebSecurityConfigurers()
        来获取webSecurityConfigurers, 其中autowiredWebSecurityConfigurersIgnoreParents
        就是上面注册过的Bean, 可以从里面获取当前ApplicationContext中的所有WebSecurityConfigurer
        */
			@Value("#{@autowiredWebSecurityConfigurersIgnoreParents.getWebSecurityConfigurers()}") List<SecurityConfigurer<Filter, WebSecurity>> webSecurityConfigurers)
			throws Exception {
        //符合Javadoc描述, 创建WebSecurity并加入到域中
		webSecurity = objectPostProcessor
				.postProcess(new WebSecurity(objectPostProcessor));
		if (debugEnabled != null) {
			webSecurity.debug(debugEnabled);
		}
		//对获取到的webSecurityConfigurers进行排序
		Collections.sort(webSecurityConfigurers, AnnotationAwareOrderComparator.INSTANCE);

		Integer previousOrder = null;
		Object previousConfig = null;
        //迭代所有配置类的实例, 判断其order必须不相同, 否则抛出异常
        //因为要想应用配置, 必须保证优先级, 否则无法确定配置覆盖的顺序
		for (SecurityConfigurer<Filter, WebSecurity> config : webSecurityConfigurers) {
			Integer order = AnnotationAwareOrderComparator.lookupOrder(config);
			if (previousOrder != null && previousOrder.equals(order)) {
				throw new IllegalStateException(
						"@Order on WebSecurityConfigurers must be unique. Order of "
								+ order + " was already used on " + previousConfig + ", so it cannot be used on "
								+ config + " too.");
			}
			previousOrder = order;
			previousConfig = config;
		}
        
        //按照顺序应用所有webSecurityConfigurers中的元素到webSecurity.apply方法
		for (SecurityConfigurer<Filter, WebSecurity> webSecurityConfigurer : webSecurityConfigurers) {
            //这个方法其实还是把所有webSecurityConfigurer加入到WebSecurity的域中
            //并回调 SecurityConfigurerAdapter.setBuilder(SecurityBuilder)
            //最后代理给webSecurity.build()来构建Filter Chain
            //具体参见下一节
			webSecurity.apply(webSecurityConfigurer);
		}
        //加入到域中
		this.webSecurityConfigurers = webSecurityConfigurers;
	}

    //Creates the Spring Security Filter Chain, 作为Bean加入到容器中
    //Bean的默认名称是springSecurityFilterChain
	@Bean(name = AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME)
	public Filter springSecurityFilterChain() throws Exception {
		boolean hasConfigurers = webSecurityConfigurers != null
				&& !webSecurityConfigurers.isEmpty();
		if (!hasConfigurers) {
            //没有自定义webSecurityConfigurers, 创建空配置 new WebSecurityConfigurerAdapter()
			WebSecurityConfigurerAdapter adapter = objectObjectPostProcessor
					.postProcess(new WebSecurityConfigurerAdapter() {
					});
			webSecurity.apply(adapter);
		}
        
        //对应Java doc, 代理给webSecurity来构建Spring Security Filter Chain
		return webSecurity.build();
	}
}
```

#### `WebSecurity`

Java doc :

> The `WebSecurity` is created by `WebSecurityConfiguration` to create the `FilterChainProxy` known as the Spring Security Filter Chain (Bean name: springSecurityFilterChain, 可以直接从配置中获取). The springSecurityFilterChain is the Filter that the DelegatingFilterProxy delegates to.

先看看继承关系, 因为下面有很多概念和其父类相关, 先看看`WebSecurity`的父类: 

![1553674661221](images/Spring Security/1553674661221.png)

下面这个继承关系就是自定义Web安全配置时需要继承的适配器.

![1553674710253](images/Spring Security/1553674710253.png)

很奇怪的一点是这两个类都实现了SecurityBuilder接口, 这个接口的功能只是返回一个创建的对象:

```JAVA
//Interface for building an Object
public interface SecurityBuilder<O> {
// Builds the object and returns it or null.
   O build() throws Exception;
}
```



先粗略看一下apply方法.

```JAVA
//Applies a SecurityConfigurerAdapter to this SecurityBuilder and invokes 
// SecurityConfigurerAdapter.setBuilder(SecurityBuilder).
public <C extends SecurityConfigurerAdapter<O, B>> C apply(C configurer)
      throws Exception {
   configurer.addObjectPostProcessor(objectPostProcessor);
    //符合Javadoc
   configurer.setBuilder((B) this);
    
	//Adds SecurityConfigurer ensuring that it is allowed and invoking 
   	//SecurityConfigurer.init(SecurityBuilder) immediately if necessary.
   add(configurer);
   return configurer;
}
```



再看一下这个类核心的`WebSecurity#build()`方法.

它通过一个CAS操作判断是否已经构建过(本对象的`WebSecurity#build()`是否已经被调用过). 若没有, 调用`WebSecurity#doBuild()`完成过滤器链的创建.

```JAVA
	public final O build() throws Exception {
		if (this.building.compareAndSet(false, true)) {
			this.object = doBuild();
			return this.object;
		}
		throw new AlreadyBuiltException("This object has already been built");
	}
```

`WebSecurity#doBuild()`, Javadoc : 

> Executes the build using the SecurityConfigurer's that have been applied using the following steps:
>
> (有点绕, 翻译一下, 使用之前调用`WebSecurity#apply()`加入进来的`SecurityConfigurer`, 按照以下步骤, 来build Filter Chain)
>
> - Invokes `beforeInit()` for any subclass to hook into
> - Invokes `SecurityConfigurer.init(SecurityBuilder)` for any `SecurityConfigurer` that was applied to this builder.
> - Invokes `beforeConfigure()` for any subclass to hook into
> - Invokes `performBuild()` which actually builds the Object

```JAVA
@Override
protected final O doBuild() throws Exception {
   synchronized (configurers) {
      buildState = BuildState.INITIALIZING;

      //根据Javadoc, 这是一个hook
      beforeInit();
       //回调 SecurityConfigurer.init(SecurityBuilder)
      init();

      buildState = BuildState.CONFIGURING;

       //依旧是hook
      beforeConfigure();
       //配置
      configure();

      buildState = BuildState.BUILDING;
		//最后执行构建
      O result = performBuild();

      buildState = BuildState.BUILT;

      return result;
   }
}
```

所以, 下面按照顺序说明 `init()`,  `configure()`, `performBuild()`

1. `init()`回调`SecurityConfigurer.init(SecurityBuilder)`

   如果继承`WebSecurityConfigurerAdapter`自定义配置, 会回调它的init方法

   ```JAVA
   public abstract class WebSecurityConfigurerAdapter implements
   		WebSecurityConfigurer<WebSecurity> {
       
   	public void init(final WebSecurity web) throws Exception {
           //获取HttpSecurity和更多的复杂操作, 见下段代码
   		final HttpSecurity http = getHttp();
          
           //下面是对WebSecurity的两个操作
   		web
               //1. 通过web.addSecurityFilterChainBuilder方法把获取到的HttpSecurity实例
           	//	赋值给WebSecurity的securityFilterChainBuilders属性
               //  会在performBuild步骤中调用HttpSecurity#build()
               .addSecurityFilterChainBuilder(http)
               //2. 为WebSecurity追加了一个postBuildAction(build完成后的后置动作)，
           	//	在build都完成后从http中拿出FilterSecurityInterceptor对象并赋值给WebSecurity 
               .postBuildAction(new Runnable() {
   			public void run() {
   				FilterSecurityInterceptor securityInterceptor = http
   						.getSharedObject(FilterSecurityInterceptor.class);
   				web.securityInterceptor(securityInterceptor);
   			}
   		});
   	}
   ```

   ```JAVA
       /*
       getHttp()这个方法在当我们使用默认配置时（大多数情况下）
       会为我们追加各种SecurityConfigurer的具体实现类到httpSecurity中，
       如exceptionHandling()方法会追加一个ExceptionHandlingConfigurer，
       sessionManagement()方法会追加一个SessionManagementConfigurer,
       securityContext()方法会追加一个SecurityContextConfigurer对象，
       这些SecurityConfigurer的具体实现类最终会为我们配置各种具体的filter(在perfromBuild方法中)
       */
       protected final HttpSecurity getHttp() throws Exception {
   		if (http != null) {
   			return http;
   		}
   
   		DefaultAuthenticationEventPublisher eventPublisher = objectPostProcessor
   				.postProcess(new DefaultAuthenticationEventPublisher());
   		localConfigureAuthenticationBldr.authenticationEventPublisher(eventPublisher);
   
           //获取authenticationManager及其他复杂操作, 见下段代码
   		AuthenticationManager authenticationManager = authenticationManager();
   		authenticationBuilder.parentAuthenticationManager(authenticationManager);
   		authenticationBuilder.authenticationEventPublisher(eventPublisher);
   		Map<Class<? extends Object>, Object> sharedObjects = createSharedObjects();
   
   		http = new HttpSecurity(objectPostProcessor, authenticationBuilder,
   				sharedObjects);
   		if (!disableDefaults) {
               // HTTP默认配置如下
   			// @formatter:off
   			http
   				.csrf().and()
   				.addFilter(new WebAsyncManagerIntegrationFilter())
   				.exceptionHandling().and()
   				.headers().and()
   				.sessionManagement().and()
   				.securityContext().and()
   				.requestCache().and()
   				.anonymous().and()
   				.servletApi().and()
   				.apply(new DefaultLoginPageConfigurer<>()).and()
   				.logout();
   			// @formatter:on
   			ClassLoader classLoader = this.context.getClassLoader();
   			List<AbstractHttpConfigurer> defaultHttpConfigurers =
   					SpringFactoriesLoader.loadFactories
                   (AbstractHttpConfigurer.class, classLoader);
   
   			for (AbstractHttpConfigurer configurer : defaultHttpConfigurers) {
   				http.apply(configurer);
   			}
   		}
               
           //在这里调用configure(HttpSecurity http)自定义配置来覆盖上面的默认配置
   		configure(http);
   		return http;
   	}
   ```

   

   ```java
   protected AuthenticationManager authenticationManager() throws Exception {
      if (!authenticationManagerInitialized) {
          //调用自定义配置的configure(AuthenticationManagerBuilder auth)
          //来自定义AuthenticationManager
          //默认操作为disableLocalConfigureAuthenticationBldr = true
          //意思是关闭自定义的AuthenticationManagerBuilder, 因为没有自定义配置
          //因此默认会进行下一个if里面的内容
         configure(localConfigureAuthenticationBldr);
         if (disableLocalConfigureAuthenticationBldr) {
             //调用AuthenticationConfiguration#getAuthenticationManager();
             //关于AuthenticationConfiguration见下节
            authenticationManager = authenticationConfiguration
                  .getAuthenticationManager();
         }
         else {
             //相反, 如果覆盖了configure(AuthenticationManagerBuilder auth)方法
             //会走到else, 用自定义的authenticationManagerBuilder来build
            authenticationManager = localConfigureAuthenticationBldr.build();
         }
         authenticationManagerInitialized = true;
      }
      return authenticationManager;
   }
   ```

   支持自定义配置的`configure(AuthenticationManagerBuilder auth)`源码如下:

   > Used by the default implementation of authenticationManager() to attempt to obtain an AuthenticationManager. If overridden, the AuthenticationManagerBuilder should be used to specify the AuthenticationManager.
   > The authenticationManagerBean() method can be used to expose the resulting AuthenticationManager as a Bean. The userDetailsServiceBean() can be used to expose the last populated UserDetailsService that is created with the AuthenticationManagerBuilder as a Bean. The UserDetailsService will also automatically be populated on HttpSecurity.getSharedObject(Class) for use with other SecurityContextConfigurer (i.e. RememberMeConfigurer )
   > For example, the following configuration could be used to register in memory authentication that exposes an in memory UserDetailsService:
   >
   > ```JAVA
   > 	   @Override
   > 	   protected void configure(AuthenticationManagerBuilder auth) {
   > 	   	auth
   > 	   	// enable in memory based authentication with a user named
   > 	   	// "user" and "admin"
   > 	   	.inMemoryAuthentication().withUser("user").password("password").roles("USER").and()
   > 	   			.withUser("admin").password("password").roles("USER", "ADMIN");
   > 	   }
   > 	  
   > 	   // Expose the UserDetailsService as a Bean
   > 	   @Bean
   > 	   @Override
   > 	   public UserDetailsService userDetailsServiceBean() throws Exception {
   > 	   	return super.userDetailsServiceBean();
   > 	   }
   > ```

   ```JAVA
       protected void configure(AuthenticationManagerBuilder auth) throws Exception {
   		this.disableLocalConfigureAuthenticationBldr = true;
   	}
   }
   ```

   

2. `configure()`

   回调`WebSecurityConfigurerAdapter#configure(WebSecurity web)`方法, 自定义配置写在其中.

3. `performBuild()`

```JAVA
protected Filter performBuild() throws Exception {
   Assert.state(
         !securityFilterChainBuilders.isEmpty(),
         () -> "At least one SecurityBuilder<? extends SecurityFilterChain> needs to be specified. "
               + "Typically this done by adding a @Configuration that extends WebSecurityConfigurerAdapter. "
               + "More advanced users can invoke "
               + WebSecurity.class.getSimpleName()
               + ".addSecurityFilterChainBuilder directly");
   int chainSize = ignoredRequests.size() + securityFilterChainBuilders.size();
    //把所有从配置中提取出来的, 对应配置中请求模式的
    //构造成不同的SecurityFilterChain, 全都加入到list里面
   List<SecurityFilterChain> securityFilterChains = new ArrayList<>(
         chainSize);
    //加入被忽略的请求到securityFilterChains
   for (RequestMatcher ignoredRequest : ignoredRequests) {
       							//注意, 和下面的链不是同一个链
      securityFilterChains.add(new DefaultSecurityFilterChain(ignoredRequest));
   }
    //调用所有securityFilterChainBuilders.build()来构造另一个链
    //见下面的继承关系图可知, 被调用的是HttpSecurity对象
    //
   for (SecurityBuilder<? extends SecurityFilterChain> securityFilterChainBuilder : securityFilterChainBuilders) {
      securityFilterChains.add(securityFilterChainBuilder.build());
   }
   FilterChainProxy filterChainProxy = new FilterChainProxy(securityFilterChains);
   if (httpFirewall != null) {
      filterChainProxy.setFirewall(httpFirewall);
   }
   filterChainProxy.afterPropertiesSet();

   Filter result = filterChainProxy;
   if (debugEnabled) {
      logger.warn("\n\n"
            + "********************************************************************\n"
            + "**********        Security debugging is enabled.       *************\n"
            + "**********    This may include sensitive information.  *************\n"
            + "**********      Do not use in a production system!     *************\n"
            + "********************************************************************\n\n");
      result = new DebugFilter(filterChainProxy);
   }
    //回调初始化时加入的postBuildAction
   postBuildAction.run(
				FilterSecurityInterceptor securityInterceptor = http
						.getSharedObject(FilterSecurityInterceptor.class);
				web.securityInterceptor(securityInterceptor);
		);
   return result;
}
```

![1553680794115](images/Spring Security/1553680794115.png)

![1553681013386](images/Spring Security/1553681013386.png)





























