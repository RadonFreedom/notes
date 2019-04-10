# Spring 问题与细节

### `@Autowired`

不同情况下的`@Autowired`调用时机不同.

1. 在方法上标注, 调用时机为Bean初始化. 这说明被标注的方法一定会在初始化过程中执行一次. 因此这种方式下可以不正当地自定义初始化方法.

   ![1553654867950](images/Spring/1553654867950.png)

   ![1553654926340](images/Spring/1553654926340.png)

2. 在域上标注, 赋值时机也为Bean初始化.

   ![1553655092032](images/Spring/1553655092032.png)

   ![1553655185676](images/Spring/1553655185676.png)

3. 标注在方法参数上, 直到方法被调用时才会自动装配, 图略.



### 循环依赖

如果一个配置类需要注入`MyBean`, 而恰巧`MyBean`又被`@Bean`注册在配置类中.

```JAVA
@EnableAuthorizationServer
@EnableConfigurationProperties(Oauth2ServerClientsProperties.class)
@Configuration
public class Oauth2ServerConfig extends AuthorizationServerConfigurerAdapter {


    private final AuthenticationManager authenticationManager;
    private final Oauth2ServerClientsProperties oauth2ServerClientsProperties;
    private final TokenStore tokenStore;

    //唯一构造器, 需要tokenStore这个Bean
    @Autowired
    public Oauth2ServerConfig(AuthenticationManager authenticationManager, Oauth2ServerClientsProperties oauth2ServerClientsProperties, TokenStore tokenStore) {
        this.authenticationManager = authenticationManager;
        this.oauth2ServerClientsProperties = oauth2ServerClientsProperties;
        this.tokenStore = tokenStore;
    }

    //这个Bean注册在这个配置类中
	@Bean
    @Autowired
    public TokenStore redisTokenStore(RedisConnectionFactory redisConnectionFactory) {
    	return new RedisTokenStore(redisConnectionFactory);
    }
}
```

循环依赖出现的原因是, 如果要实例化`redisTokenStore`这个Bean, 必须先实例化一个`Oauth2ServerConfig`再调用其@Bean方法; 而`Oauth2ServerConfig`的唯一构造器恰恰需要`tokenStore`这个Bean, 导致两个组件都没办法被初始化.

#### 解决方法

1. 可以将导致循环依赖的`@Bean`配置写在另一个配置类中.

   ```JAVA
   @EnableAuthorizationServer
   @EnableConfigurationProperties(Oauth2ServerClientsProperties.class)
   @Configuration
   public class Oauth2ServerConfig extends AuthorizationServerConfigurerAdapter {
   
   
       private final AuthenticationManager authenticationManager;
       private final Oauth2ServerClientsProperties oauth2ServerClientsProperties;
       private final TokenStore tokenStore;
   
       @Autowired
       public Oauth2ServerConfig(AuthenticationManager authenticationManager, Oauth2ServerClientsProperties oauth2ServerClientsProperties, TokenStore tokenStore) {
           this.authenticationManager = authenticationManager;
           this.oauth2ServerClientsProperties = oauth2ServerClientsProperties;
           this.tokenStore = tokenStore;
       }
   
       //写在另一个配置类中
       @Configuration
       public static class TokenStoreConfig {
           @ConditionalOnProperty(name = "security.oauth2.server.enable-jwt-token", havingValue = "false")
           @Bean
           @Autowired
           public TokenStore redisTokenStore(RedisConnectionFactory redisConnectionFactory) {
               return new RedisTokenStore(redisConnectionFactory);
           }
       }
   }
   ```

   

2. 可以写一个无参构造器, 但是不推荐这么做(Spring推荐构造器Autowire).