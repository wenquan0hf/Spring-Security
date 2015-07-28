# 缓存 UserDetails

Spring Security 提供了一个实现了可以缓存 UserDetails 的 UserDetailsService 实现类，CachingUserDetailsService。该类的构造接收一个用于真正加载 UserDetails 的 UserDetailsService 实现类。当需要加载 UserDetails 时，其首先会从缓存中获取，如果缓存中没有对应的 UserDetails 存在，则使用持有的 UserDetailsService 实现类进行加载，然后将加载后的结果存放在缓存中。UserDetails 与缓存的交互是通过 UserCache 接口来实现的。CachingUserDetailsService 默认拥有 UserCache 的一个空实现引用，NullUserCache。以下是 CachingUserDetailsService 的类定义。

```
public class CachingUserDetailsService implements UserDetailsService {
    private UserCache userCache = new NullUserCache();
    private final UserDetailsService delegate;
 
    CachingUserDetailsService(UserDetailsService delegate) {
        this.delegate = delegate;
    }
 
    public UserCache getUserCache() {
        return userCache;
    }
 
    public void setUserCache(UserCache userCache) {
        this.userCache = userCache;
    }
 
    public UserDetails loadUserByUsername(String username) {
        UserDetails user = userCache.getUserFromCache(username);
 
        if (user == null) {
            user = delegate.loadUserByUsername(username);
        }
 
        Assert.notNull(user, "UserDetailsService" + delegate + "returned null for username" + username + "." +
                "This is an interface contract violation");
 
        userCache.putUserInCache(user);
 
        return user;
    }
}
```

我们可以看到当缓存中不存在对应的 UserDetails 时将使用引用的 UserDetailsService 类型的 delegate 进行加载。加载后再把它存放到 Cache 中并进行返回。除了 NullUserCache 之外，Spring Security 还为我们提供了一个基于 Ehcache 的 UserCache 实现类，EhCacheBasedUserCache，其源码如下所示。

```
public class EhCacheBasedUserCache implements UserCache, InitializingBean {
 
    private static final Log logger = LogFactory.getLog(EhCacheBasedUserCache.class);
 
    private Ehcache cache;
 
    public void afterPropertiesSet() throws Exception {
        Assert.notNull(cache, "cache mandatory");
    }
 
    public Ehcache getCache() {
        returncache;
    }
 
    public UserDetails getUserFromCache(String username) {
        Element element = cache.get(username);
        if (logger.isDebugEnabled()) {
            logger.debug("Cache hit:" + (element != null) + "; username:" + username);
        }
        if (element == null) {
            returnnull;
        } else {
            return (UserDetails) element.getValue();
        }
    }
 
    public void putUserInCache(UserDetails user) {
        Element element = new Element(user.getUsername(), user);
        if (logger.isDebugEnabled()) {
            logger.debug("Cache put:" + element.getKey());
        }
        cache.put(element);
    }
 
    public void removeUserFromCache(UserDetails user) {
        if (logger.isDebugEnabled()) {
            logger.debug("Cache remove:" + user.getUsername());
        }
        this.removeUserFromCache(user.getUsername());
    }
 
    public void removeUserFromCache(String username) {
        cache.remove(username);
    }
 
    public void setCache(Ehcache cache) {
        this.cache = cache;
    }
}
```

从上述源码我们可以看到 EhCacheBasedUserCache 所引用的 Ehcache 是空的，所以，当我们需要对 UserDetails 进行缓存时，我们只需要定义一个 Ehcache 实例，然后把它注入给 EhCacheBasedUserCache 就可以了。接下来我们来看一下定义一个支持缓存 UserDetails 的 CachingUserDetailsService 的示例。

```
   <security:authentication-manager alias="authenticationManager">
      <!-- 使用可以缓存 UserDetails 的 CachingUserDetailsService -->
      <security:authentication-provider
         user-service-ref="cachingUserDetailsService" />
   </security:authentication-manager>
   <!-- 可以缓存 UserDetails 的 UserDetailsService -->
   <bean id="cachingUserDetailsService" class="org.springframework.security.config.authentication.CachingUserDetailsService">
      <!-- 真正加载 UserDetails 的 UserDetailsService -->
      <constructor-arg ref="userDetailsService"/>
      <!-- 缓存 UserDetails 的 UserCache -->
      <property name="userCache">
         <bean class="org.springframework.security.core.userdetails.cache.EhCacheBasedUserCache">
            <!-- 用于真正缓存的 Ehcache 对象 -->
            <property name="cache" ref="ehcache4UserDetails"/>
         </bean>
      </property>
   </bean>
   <!-- 将使用默认的 CacheManager 创建一个名为 ehcache4UserDetails 的 Ehcache 对象 -->
   <bean id="ehcache4UserDetails" class="org.springframework.cache.ehcache.EhCacheFactoryBean"/>
   <!-- 从数据库加载 UserDetails 的 UserDetailsService -->
   <bean id="userDetailsService"
      class="org.springframework.security.core.userdetails.jdbc.JdbcDaoImpl">
      <property name="dataSource" ref="dataSource" />
   </bean>
```

在上面的配置中，我们通过 EhcacheFactoryBean 定义的 Ehcache bean 对象采用的是默认配置，其将使用默认的 CacheManager，即直接通过 CacheManager.getInstance() 获取当前已经存在的 CacheManager 对象，如不存在则使用默认配置自动创建一个，当然这可以通过 cacheManager 属性指定我们需要使用的 CacheManager，CacheManager 可以通过 EhCacheManagerFactoryBean 进行定义。此外，如果没有指定对应缓存的名称，默认将使用 beanName，在上述配置中即为 ehcache4UserDetails，可以通过 cacheName 属性进行指定。此外，缓存的配置信息也都是使用的默认的。更多关于 Spring 使用 Ehcache 的信息可以参考我的另一篇文章[《Spring 使用 Cache》](http://haohaoxuexi.iteye.com/blog/2123030)。