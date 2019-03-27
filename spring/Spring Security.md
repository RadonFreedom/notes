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

ProviderManager 是 AuthenticationManager 的实现类。ProviderManager 并没有实现对 Authentication 的校验功能，而是采用代理模式将校验功能交给 AuthenticationProvider 去实现。这样设计是因为在 Web 环境中可能会支持多种不同的验证方式，比如用户名密码登录、短信登录、指纹登录等等，如果每种验证方式的代码都写在 ProviderManager 中，想想都是灾难。因此为每种验证方式提供对应的 AuthenticationProvider，ProviderManager 将验证任务代理给对应的 AuthenticationProvider，这是一种不错的解决方案。在 ProviderManager 中可以找到以下代码，如清单 7 所示:

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

## 核心组件的交互和源码分析

### `@EnableWebSecurity`

Javadoc: 

Add this annotation to an @Configuration class to have the Spring Security configuration defined in any WebSecurityConfigurer or more likely by extending the WebSecurityConfigurerAdapter base class and overriding individual methods:

```java
   @Configuration
   @EnableWebSecurity
   public class MyWebSecurityConfiguration extends WebSecurityConfigurerAdapter {

   	@Override
   	public void configure(WebSecurity web) throws Exception {
   		web.ignoring()
   		// Spring Security should completely ignore URLs starting with /resources/
   				.antMatchers("/resources/**");
   	}

   	@Override
   	protected void configure(HttpSecurity http) throws Exception {
   		http.authorizeRequests().antMatchers("/public/**").permitAll().anyRequest()
   				.hasRole("USER").and()
   				// Possibly more configuration ...
   				.formLogin() // enable form based log in
   				// set permitAll for all URLs associated with Form Login
   				.permitAll();
   	}

   	@Override
   	protected void configure(AuthenticationManagerBuilder auth) throws Exception {
   		auth
   		// enable in memory based authentication with a user named "user" and "admin"
   		.inMemoryAuthentication().withUser("user").password("password").roles("USER")
   				.and().withUser("admin").password("password").roles("USER", "ADMIN");
   	}

   	// Possibly more overridden methods ...
   }
```

