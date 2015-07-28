# Remember-Me 功能

## 概述

Remember-Me 是指网站能够在 Session 之间记住登录用户的身份，具体来说就是我成功认证一次之后在一定的时间内我可以不用再输入用户名和密码进行登录了，系统会自动给我登录。这通常是通过服务端发送一个 cookie 给客户端浏览器，下次浏览器再访问服务端时服务端能够自动检测客户端的 cookie，根据 cookie 值触发自动登录操作。Spring Security 为这些操作的发生提供必要的钩子，并且针对于 Remember-Me 功能有两种实现。一种是简单的使用加密来保证基于 cookie 的 token 的安全，另一种是通过数据库或其它持久化存储机制来保存生成的 token。

需要注意的是两种实现都需要一个 UserDetailsService。如果你使用的 AuthenticationProvider 不使用 UserDetailsService，那么记住我将会不起作用，除非在你的 ApplicationContext 中拥有一个 UserDetailsService 类型的 bean。
 
## 基于简单加密 token 的方法

当用户选择了记住我成功登录后，Spring Security 将会生成一个 cookie 发送给客户端浏览器。cookie 值由如下方式组成：

base64(username+":"+expirationTime+":"+md5Hex(username+":"+expirationTime+":"+password+":"+key))

- username：登录的用户名。
- password：登录的密码。
- expirationTime：token 失效的日期和时间，以毫秒表示。
- key：用来防止修改 token 的一个 key。

这样用来实现 Remember-Me 功能的 token 只能在指定的时间内有效，且必须保证 token 中所包含的 username、password 和 key 没有被改变才行。需要注意的是，这样做其实是存在安全隐患的，那就是在用户获取到实现记住我功能的 token 后，任何用户都可以在该 token 过期之前通过该 token 进行自动登录。如果用户发现自己的 token 被盗用了，那么他可以通过改变自己的登录密码来立即使其所有的记住我 token 失效。如果希望我们的应用能够更安全一点，可以使用接下来要介绍的持久化 token 方式，或者不使用 Remember-Me 功能，因为 Remember-Me 功能总是有点不安全的。

使用这种方式时，我们只需要在 http 元素下定义一个 remember-me 元素，同时指定其 key 属性即可。key 属性是用来标记存放 token 的 cookie 的，对应上文提到的生成 token 时的那个 key。

```
   <security:http auto-config="true">
      <security:form-login/>
      <!-- 定义记住我功能 -->
      <security:remember-me key="elim"/>
      <security:intercept-url pattern="/**" access="ROLE_USER" />
   </security:http>
```

这里有两个需要注意的地方。第一，如果你的登录页面是自定义的，那么需要在登录页面上新增一个名为 “_spring_security_remember_me” 的 checkbox，这是基于 NameSpace 定义提供的默认名称，如果要自定义可以自己定义 TokenBasedRememberMeServices 或 PersistentTokenBasedRememberMeServices 对应的 bean，然后通过其 parameter 属性进行指定，具体操作请参考后文关于《Remember-Me 相关接口和实现类》部分内容。第二，上述功能需要一个 UserDetailsService，如果在你的 ApplicationContext 中已经拥有一个了，那么 Spring Security 将自动获取；如果没有，那么当然你需要定义一个；如果拥有在 ApplicationContext 中拥有多个 UserDetailsService 定义，那么你需要通过 remember-me 元素的 user-service-ref 属性指定将要使用的那个。如：

```
   <security:http auto-config="true">
      <security:form-login/>
      <!-- 定义记住我功能，通过 user-service-ref 指定将要使用的 UserDetailsService-->
      <security:remember-me key="elim" user-service-ref="userDetailsService"/>
      <security:intercept-url pattern="/**" access="ROLE_USER" />
   </security:http>
  
   <bean id="userDetailsService" class="org.springframework.security.core.userdetails.jdbc.JdbcDaoImpl">
      <property name="dataSource" ref="dataSource"/>
   </bean>
```

## 基于持久化 token 的方法

持久化 token 的方法跟简单加密 token 的方法在实现 Remember-Me 功能上大体相同，都是在用户选择了 “记住我” 成功登录后，将生成的 token 存入 cookie 中并发送到客户端浏览器，待到下次用户访问系统时，系统将直接从客户端 cookie 中读取 token 进行认证。所不同的是基于简单加密 token 的方法，一旦用户登录成功后，生成的 token 将在客户端保存一段时间，如果用户不点击退出登录，或者不修改密码，那么在 cookie 失效之前，他都可以使用该 token 进行登录，哪怕该 token 被别人盗用了，用户与盗用者都同样可以进行登录。而基于持久化 token 的方法采用这样的实现逻辑：

1. 用户选择了 “记住我” 成功登录后，将会把 username、随机产生的序列号、生成的 token 存入一个数据库表中，同时将它们的组合生成一个 cookie 发送给客户端浏览器。
2. 当下一次没有登录的用户访问系统时，首先检查 cookie，如果对应 cookie 中包含的 username、序列号和 token 与数据库中保存的一致，则表示其通过验证，系统将重新生成一个新的 token 替换数据库中对应组合的旧 token，序列号保持不变，同时删除旧的 cookie，重新生成包含新生成的 token，就的序列号和 username 的 cookie 发送给客户端。
3. 如果检查 cookie 时，cookie 中包含的 username 和序列号跟数据库中保存的匹配，但是 token 不匹配。这种情况极有可能是因为你的 cookie 被人盗用了，由于盗用者使用你原本通过认证的 cookie 进行登录了导致旧的 token 失效，而产生了新的 token。这个时候 Spring Security 就可以发现 cookie 被盗用的情况，它将删除数据库中与当前用户相关的所有 token 记录，这样盗用者使用原有的 cookie 将不能再登录，同时提醒用户其帐号有被盗用的可能性。
4. 如果对应 cookie 不存在，或者包含的 username 和序列号与数据库中保存的不一致，那么将会引导用户到登录页面。

从以上逻辑我们可以看出持久化 token 的方法比简单加密 token 的方法更安全，因为一旦你的 cookie 被人盗用了，你只要再利用原有的 cookie 试图自动登录一次，原有的 token 将失效导致盗用者不能再使用原来盗用的 cookie 进行登录了，同时用户可以发现自己的 cookie 有被盗用的可能性。但因为 cookie 被盗用后盗用者还可以在用户下一次登录前顺利的进行登录，所以如果你的应用对安全性要求比较高就不要使用 Remember-Me 功能了。

使用持久化 token 方法时需要我们的数据库中拥有如下表及其表结构。

```
create table persistent_logins (username varchar(64) not null,
                                    series varchar(64) primary key,
                                    token varchar(64) not null,
                                    last_used timestamp not null)
```

然后还是通过 remember-me 元素来使用，只是这个时候我们需要其 data-source-ref 属性指定对应的数据源，同时别忘了它也同样需要 ApplicationContext 中拥有 UserDetailsService，如果拥有多个，请使用 user-service-ref 属性指定 remember-me 使用的是哪一个。
   
```
<security:http auto-config="true">
      <security:form-login/>
      <!-- 定义记住我功能 -->
      <security:remember-me data-source-ref="dataSource"/>
      <security:intercept-url pattern="/**" access="ROLE_USER" />
   </security:http>
```

## Remember-Me 相关接口和实现类

在上述介绍中，我们实现 Remember-Me 功能是通过 Spring Security 为了简化 Remember-Me 而提供的 NameSpace 进行定义的。而底层实际上还是通过 RememberMeServices、UsernamePasswordAuthenticationFilter 和 RememberMeAuthenticationFilter 的协作来完成的。RememberMeServices 是 Spring Security 为 Remember-Me 提供的一个服务接口，其定义如下。

```
publicinterface RememberMeServices {
    /**
     * 自动登录。在实现这个方法的时候应该判断用户提供的 Remember-Me cookie 是否有效，如果无效，应当直接忽略。
     * 如果认证成功应当返回一个 AuthenticationToken，推荐返回 RememberMeAuthenticationToken；
     * 如果认证不成功应当返回 null。
     */
    Authentication autoLogin(HttpServletRequest request, HttpServletResponse response);
    /**
     * 在用户登录失败时调用。实现者应当做一些类似于删除 cookie 之类的处理。
     */
    void loginFail(HttpServletRequest request, HttpServletResponse response);
    /**
     * 在用户成功登录后调用。实现者可以在这里判断用户是否选择了 “Remember-Me” 登录，然后做相应的处理。
     */
    void loginSuccess(HttpServletRequest request, HttpServletResponse response,
        Authentication successfulAuthentication);
}
``` 

UsernamePasswordAuthenticationFilter 拥有一个 RememberMeServices 的引用，默认是一个空实现的 NullRememberMeServices，而实际当我们通过 remember-me 定义启用 Remember-Me 时，它会是一个具体的实现。用户的请求会先通过 UsernamePasswordAuthenticationFilter，如认证成功会调用 RememberMeServices 的 loginSuccess() 方法，否则调用 RememberMeServices 的 loginFail() 方法。UsernamePasswordAuthenticationFilter 是不会调用 RememberMeServices 的 autoLogin() 方法进行自动登录的。之后运行到 RememberMeAuthenticationFilter 时如果检测到还没有登录，那么 RememberMeAuthenticationFilter 会尝试着调用所包含的 RememberMeServices 的 autoLogin() 方法进行自动登录。关于 RememberMeServices Spring Security 已经为我们提供了两种实现，分别对应于前文提到的基于简单加密 token 和基于持久化 token 的方法。
 
### TokenBasedRememberMeServices

TokenBasedRememberMeServices 对应于前文介绍的使用 namespace 时基于简单加密 token 的实现。TokenBasedRememberMeServices 会在用户选择了记住我成功登录后，生成一个包含 token 信息的 cookie 发送到客户端；如果用户登录失败则会删除客户端保存的实现 Remember-Me 的 cookie。需要自动登录时，它会判断 cookie 中所包含的关于 Remember-Me 的信息是否与系统一致，一致则返回一个 RememberMeAuthenticationToken 供 RememberMeAuthenticationProvider 处理，不一致则会删除客户端的 Remember-Me cookie。TokenBasedRememberMeServices 还实现了 Spring Security 的 LogoutHandler 接口，所以它可以在用户退出登录时立即清除 Remember-Me cookie。
 
如果把使用 namespace 定义 Remember-Me 改为直接定义 RememberMeServices 和对应的 Filter 来使用的话，那么我们可以如下定义。

```
   <security:http>
      <security:form-login login-page="/login.jsp"/>
      <security:intercept-url pattern="/login*.jsp*" access="IS_AUTHENTICATED_ANONYMOUSLY"/>
      <security:intercept-url pattern="/**" access="ROLE_USER" />
      <!-- 把 usernamePasswordAuthenticationFilter 加入 FilterChain -->
      <security:custom-filter ref="usernamePasswordAuthenticationFilter" before="FORM_LOGIN_FILTER"/>
      <security:custom-filter ref="rememberMeFilter" position="REMEMBER_ME_FILTER"/>
   </security:http>
   <!-- 用于认证的 AuthenticationManager -->
   <security:authentication-manager alias="authenticationManager">
      <security:authentication-provider
         user-service-ref="userDetailsService"/>
      <security:authentication-provider ref="rememberMeAuthenticationProvider"/>
   </security:authentication-manager>
 
   <bean id="userDetailsService"
      class="org.springframework.security.core.userdetails.jdbc.JdbcDaoImpl">
      <property name="dataSource" ref="dataSource" />
   </bean>
 
   <bean id="usernamePasswordAuthenticationFilter" class="org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter">
      <property name="rememberMeServices" ref="rememberMeServices"/>
      <property name="authenticationManager" ref="authenticationManager"/>
      <!-- 指定 request 中包含的用户名对应的参数名 -->
      <property name="usernameParameter" value="username"/>
      <property name="passwordParameter" value="password"/>
      <!-- 指定登录的提交地址 -->
      <property name="filterProcessesUrl" value="/login.do"/>
   </bean>
   <!-- Remember-Me 对应的 Filter -->
   <bean id="rememberMeFilter"
   class="org.springframework.security.web.authentication.rememberme.RememberMeAuthenticationFilter">
      <property name="rememberMeServices" ref="rememberMeServices" />
      <property name="authenticationManager" ref="authenticationManager" />
   </bean>
   <!-- RememberMeServices 的实现 -->
   <bean id="rememberMeServices"
   class="org.springframework.security.web.authentication.rememberme.TokenBasedRememberMeServices">
      <property name="userDetailsService" ref="userDetailsService" />
      <property name="key" value="elim" />
      <!-- 指定 request 中包含的用户是否选择了记住我的参数名 -->
      <property name="parameter" value="rememberMe"/>
   </bean>
   <!-- key 值需与对应的 RememberMeServices 保持一致 -->
   <bean id="rememberMeAuthenticationProvider"
   class="org.springframework.security.authentication.RememberMeAuthenticationProvider">
      <property name="key" value="elim" />
   </bean>
```

需要注意的是 RememberMeAuthenticationProvider 在认证 RememberMeAuthenticationToken 的时候是比较它们拥有的 key 是否相等，而 RememberMeAuthenticationToken 的 key 是 TokenBasedRememberMeServices 提供的，所以在使用时需要保证 RememberMeAuthenticationProvider 和 TokenBasedRememberMeServices 的 key 属性值保持一致。需要配置 UsernamePasswordAuthenticationFilter 的 rememberMeServices 为我们定义好的 TokenBasedRememberMeServices，把 RememberMeAuthenticationProvider 加入 AuthenticationManager 的 providers 列表，并添加 RememberMeAuthenticationFilter 和 UsernamePasswordAuthenticationFilter 到 FilterChainProxy。
 
### PersistentTokenBasedRememberMeServices

PersistentTokenBasedRememberMeServices 是 RememberMeServices 基于前文提到的持久化 token 的方式实现的。具体实现逻辑跟前文介绍的以 NameSpace 的方式使用基于持久化 token 的 Remember-Me 是一样的，这里就不再赘述了。此外，如果单独使用，其使用方式和上文描述的 TokenBasedRememberMeServices 是一样的，这里也不再赘述了。

需要注意的是 PersistentTokenBasedRememberMeServices 是需要将 token 进行持久化的，所以我们必须为其指定存储 token 的 PersistentTokenRepository。Spring Security 对此有两种实现，InMemoryTokenRepositoryImpl 和 JdbcTokenRepositoryImpl。前者是将 token 存放在内存中的，通常用于测试，而后者是将 token 存放在数据库中。PersistentTokenBasedRememberMeServices 默认使用的是前者，我们可以通过其 tokenRepository 属性来指定使用的 PersistentTokenRepository。

使用 JdbcTokenRepositoryImpl 时我们可以使用在前文提到的默认表结构。如果需要使用自定义的表，那么我们可以对 JdbcTokenRepositoryImpl 进行重写。定义 JdbcTokenRepositoryImpl 时需要指定一个数据源 dataSource，同时可以通过设置参数 createTableOnStartup 的值来控制是否要在系统启动时创建对应的存入 token 的表，默认创建语句为 “create table persistent_logins (username varchar(64) not null, series varchar(64) primary key, token varchar(64) not null, last_used timestamp not null)”，但是如果自动创建时对应的表已经存在于数据库中，则会抛出异常。createTableOnStartup 属性默认为 false。

直接显示地使用 PersistentTokenBasedRememberMeServices 和上文提到的直接显示地使用 TokenBasedRememberMeServices 的方式是一样的，我们只需要将上文提到的配置中 RememberMeServices 实现类 TokenBasedRememberMeServices 换成 PersistentTokenBasedRememberMeServices 即可。

```
   <!-- RememberMeServices 的实现 -->
   <bean id="rememberMeServices"
   class="org.springframework.security.web.authentication.rememberme.PersistentTokenBasedRememberMeServices">
      <property name="userDetailsService" ref="userDetailsService" />
      <property name="key" value="elim" />
      <!-- 指定 request 中包含的用户是否选择了记住我的参数名 -->
      <property name="parameter" value="rememberMe"/>
      <!-- 指定 PersistentTokenRepository -->
      <property name="tokenRepository">
         <bean class="org.springframework.security.web.authentication.rememberme.JdbcTokenRepositoryImpl">
            <!-- 数据源 -->
            <property name="dataSource" ref="dataSource"/>
            <!-- 是否在系统启动时创建持久化 token 的数据库表 -->
            <property name="createTableOnStartup" value="false"/>
         </bean>
      </property>
   </bean>
```