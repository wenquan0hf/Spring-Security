# 匿名认证

对于匿名访问的用户，Spring Security 支持为其建立一个匿名的 AnonymousAuthenticationToken 存放在 SecurityContextHolder 中，这就是所谓的匿名认证。这样在以后进行权限认证或者做其它操作时我们就不需要再判断 SecurityContextHolder 中持有的 Authentication 对象是否为 null 了，而直接把它当做一个正常的 Authentication 进行使用就 OK 了。
 
## 配置

使用 NameSpace 时，http 元素的使用默认就会启用对匿名认证的支持，不过我们也可以通过设置 http 元素下的 anonymous 元素的 enabled 属性为 false 停用对匿名认证的支持。以下是 anonymous 元素可以配置的属性，以及它们的默认值。

```
      <security:anonymous enabled="true" key="doesNotMatter" username="anonymousUser" granted-authority="ROLE_ANONYMOUS"/>
```

key 用于指定一个在 AuthenticationFilter 和 AuthenticationProvider 之间共享的值。username 用于指定匿名用户所对应的用户名，granted-authority 用于指定匿名用户所具有的权限。

与匿名认证相关的类有三个，AnonymousAuthenticationToken 将作为一个 Authentication 的实例存放在 SecurityContextHolder 中；过滤器运行到 AnonymousAuthenticationFilter 时，如果 SecurityContextHolder 中持有的 Authentication 还是空的，则 AnonymousAuthenticationFilter 将创建一个 AnonymousAuthenticationToken 并存放在 SecurityContextHolder 中。最后一个相关的类是 AnonymousAuthenticationProvider，其会添加到 ProviderManager 的 AuthenticationProvider 列表中，以支持对 AnonymousAuthenticationToken 的认证。AnonymousAuthenticationToken 的认证是在 AbstractSecurityInterceptor 中的 beforeInvocation() 方法中进行的。使用 http 元素定义时这些 bean 都是会自动定义和添加的。如果需要手动定义这些 bean 的话，那么可以如下定义：

```
   <bean id="anonymousAuthFilter"
   class="org.springframework.security.web.authentication.AnonymousAuthenticationFilter">
      <property name="key" value="doesNotMatter" />
      <property name="userAttribute" value="anonymousUser,ROLE_ANONYMOUS" />
   </bean>
 
   <bean id="anonymousAuthenticationProvider"
   class="org.springframework.security.authentication.AnonymousAuthenticationProvider">
      <property name="key" value="doesNotMatter" />
   </bean>
```

key 是在 AnonymousAuthenticationProvider 和 AnonymousAuthenticationFilter 之间共享的，它们必须保持一致，AnonymousAuthenticationProvider 将使用本身拥有的 key 与传入的 AnonymousAuthenticationToken 的 key 作比较，相同则认为可以进行认证，否则将抛出异常 BadCredentialsException。userAttribute 属性是以 usernameInTheAuthenticationToken,grantedAuthority[,grantedAuthority] 的形式进行定义的。
 
## AuthenticationTrustResolver

AuthenticationTrustResolver 是一个接口，其中定义有两个方法，isAnonymous() 和 isRememberMe()，它们都接收一个 Authentication 对象作为参数。它有一个默认实现类 AuthenticationTrustResolverImpl，Spring Security 就是使用它来判断一个 SecurityContextHolder 持有的 Authentication 是否 AnonymousAuthenticationToken 或 RememberMeAuthenticationToken。如当 ExceptionTranslationFilter 捕获到一个 AccessDecisionManager 后就会使用它来判断当前 Authentication 对象是否为一个 AnonymousAuthenticationToken，如果是则交由 AuthenticationEntryPoint 处理，否则将返回 403 错误码。