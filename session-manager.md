# session 管理

Spring Security 通过 http 元素下的子元素 session-management 提供了对 Http Session 管理的支持。
 
## 检测 session 超时

Spring Security 可以在用户使用已经超时的 sessionId 进行请求时将用户引导到指定的页面。这个可以通过如下配置来实现。

```
   <security:http>
      ...
      <!-- session 管理，invalid-session-url 指定使用已经超时的 sessionId 进行请求需要重定向的页面 -->
      <security:session-management invalid-session-url="/session_timeout.jsp"/>
      ...
   </security:http>
``` 

需要注意的是 session 超时的重定向页面应当是不需要认证的，否则再重定向到 session 超时页面时会直接转到用户登录页面。此外如果你使用这种方式来检测 session 超时，当你退出了登录，然后在没有关闭浏览器的情况下又重新进行了登录，Spring Security 可能会错误的报告 session 已经超时。这是因为即使你已经退出登录了，但当你设置 session 无效时，对应保存 session 信息的 cookie 并没有被清除，等下次请求时还是会使用之前的 sessionId 进行请求。解决办法是显示的定义用户在退出登录时删除对应的保存 session 信息的 cookie。

```
   <security:http>
      ...
      <!-- 退出登录时删除 session 对应的 cookie -->
      <security:logout delete-cookies="JSESSIONID"/>
      ...
   </security:http>
```

此外，Spring Security 并不保证这对所有的 Servlet 容器都有效，到底在你的容器上有没有效，需要你自己进行实验。
 
## concurrency-control

通常情况下，在你的应用中你可能只希望同一用户在同时登录多次时只能有一个是成功登入你的系统的，通常对应的行为是后一次登录将使前一次登录失效，或者直接限制后一次登录。Spring Security 的 session-management 为我们提供了这种限制。

首先需要我们在 web.xml 中定义如下监听器。

```
   <listener>
   <listener-class>org.springframework.security.web.session.HttpSessionEventPublisher</listener-class>
   </listener>
``` 

在 session-management 元素下有一个 concurrency-control 元素是用来限制同一用户在应用中同时允许存在的已经通过认证的 session 数量。这个值默认是 1，可以通过 concurrency-control 元素的 max-sessions 属性来指定。

```
   <security:http auto-config="true">
      ...
      <security:session-management>
         <security:concurrency-control max-sessions="1"/>
      </security:session-management>
      ...
   </security:http>
```

当同一用户同时存在的已经通过认证的 session 数量超过了 max-sessions 所指定的值时，Spring Security 的默认策略是将先前的设为无效。如果要限制用户再次登录可以设置 concurrency-control 的 error-if-maximum-exceeded 的值为 true。
 
```
   <security:http auto-config="true">
      ...
      <security:session-management>
         <security:concurrency-control max-sessions="1" error-if-maximum-exceeded="true"/>
      </security:session-management>
      ...
   </security:http>
```

设置 error-if-maximum-exceeded 为 true 后如果你之前已经登录了，然后想再次登录，那么系统将会拒绝你的登录，同时将重定向到由 form-login 指定的 authentication-failure-url。如果你的再次登录是通过 Remember-Me 来完成的，那么将不会转到 authentication-failure-url，而是返回未授权的错误码 401 给客户端，如果你还是想重定向一个指定的页面，那么你可以通过 session-management 的 session-authentication-error-url 属性来指定，同时需要指定该 url 为不受 Spring Security 管理，即通过 http 元素设置其 secure=”none”。

```
   <security:http security="none" pattern="/none/**" />
   <security:http>
      <security:form-login/>
      <security:logout/>
      <security:intercept-url pattern="/**" access="ROLE_USER"/>
      <!-- session-authentication-error-url 必须是不受 Spring Security 管理的 -->
      <security:session-management session-authentication-error-url="/none/session_authentication_error.jsp">
         <security:concurrency-control max-sessions="1" error-if-maximum-exceeded="true"/>
      </security:session-management>
      <security:remember-me data-source-ref="dataSource"/>
   </security:http>
```

在上述配置中我们配置了 session-authentication-error-url 为 “/none/session\_authentication_error.jsp”，同时我们通过 <security:http security="none" pattern="/none/**" /> 指定了以 “/none” 开始的所有 URL 都不受 Spring Security 控制，这样当用户进行登录以后，再次通过 Remember-Me 进行自动登录时就会重定向到 “/none/session\_authentication\_error.jsp” 了。

在上述配置中为什么我们需要通过 <security:http security="none" pattern="/none/\*\*" /> 指定我们的 session-authentication-error-url 不受 Spring Security 控制呢？把它换成 <security:intercept-url pattern="/none/\*\*" access="IS\_AUTHENTICATED\_ANONYMOUSLY"/> 不行吗？这就涉及到之前所介绍的它们两者之间的区别了。前者表示不使用任何 Spring Security 过滤器，自然也就不需要通过 Spring Security 的认证了，而后者是会被 Spring Security 的 FilterChain 进行过滤的，只是其对应的 URL 可以匿名访问，即不需要登录就可访问。使用后者时，REMEMBER\_ME\_FILTER 检测到用户没有登录，同时其又提供了 Remember-Me 的相关信息，这将使得 REMEMBER\_ME\_FILTER 进行自动登录，那么在自动登录时由于我们限制了同一用户同一时间只能登录一次，后来者将被拒绝登录，这个时候将重定向到 session-authentication-error-url，重定向访问 session-authentication-error-url 时，经过 REMEMBER_ME_FILTER 时又会自动登录，这样就形成了一个死循环。所以 session-authentication-error-url 应当使用 <security:http security="none" pattern="/none/\*\*" /> 设置为不受 Spring Security 控制，而不是使用 <security:intercept-url pattern="/none/\*\*" access="IS\_AUTHENTICATED\_ANONYMOUSLY"/>。
 

此外，可以通过 expired-url 属性指定当用户尝试使用一个由于其再次登录导致 session 超时的 session 时所要跳转的页面。同时需要注意设置该 URL 为不需要进行认证。

```
   <security:http auto-config="true">
      <security:form-login/>
      <security:logout/>
      <security:intercept-url pattern="/expired.jsp" access="IS_AUTHENTICATED_ANONYMOUSLY"/>
      <security:intercept-url pattern="/**" access="ROLE_USER"/>
      <security:session-management>
         <security:concurrency-control max-sessions="1" expired-url="/expired.jsp" />
      </security:session-management>
   </security:http>
``` 

## session 固定攻击保护

session 固定是指服务器在给客户端创建 session 后，在该 session 过期之前，它们都将通过该 session 进行通信。session 固定攻击是指恶意攻击者先通过访问应用来创建一个 session，然后再让其他用户使用相同的 session 进行登录（比如通过发送一个包含该 sessionId 参数的链接），待其他用户成功登录后，攻击者利用原来的 sessionId 访问系统将和原用户获得同样的权限。Spring Security 默认是对 session 固定攻击采取了保护措施的，它会在用户登录的时候重新为其生成一个新的 session。如果你的应用不需要这种保护或者该保护措施与你的某些需求相冲突，你可以通过 session-management 的 session-fixation-protection 属性来改变其保护策略。该属性的可选值有如下三个。

- migrateSession：这是默认值。其表示在用户登录后将新建一个 session，同时将原 session 中的 attribute 都 copy 到新的 session 中。
- none：表示继续使用原来的 session。
- newSession：表示重新创建一个新的 session，但是不 copy 原 session 拥有的 attribute。