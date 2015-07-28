# 认证简介

## 认证过程

1. 用户使用用户名和密码进行登录。
2. Spring Security 将获取到的用户名和密码封装成一个实现了 Authentication 接口的 UsernamePasswordAuthenticationToken。
3. 将上述产生的 token 对象传递给 AuthenticationManager 进行登录认证。
4. AuthenticationManager 认证成功后将会返回一个封装了用户权限等信息的 Authentication 对象。
5. 通过调用 SecurityContextHolder.getContext().setAuthentication(...) 将 AuthenticationManager 返回的 Authentication 对象赋予给当前的 SecurityContext。
 
上述介绍的就是 Spring Security 的认证过程。在认证成功后，用户就可以继续操作去访问其它受保护的资源了，但是在访问的时候将会使用保存在 SecurityContext 中的 Authentication 对象进行相关的权限鉴定。
 
## Web 应用的认证过程

如果用户直接访问登录页面，那么认证过程跟上节描述的基本一致，只是在认证完成后将跳转到指定的成功页面，默认是应用的根路径。如果用户直接访问一个受保护的资源，那么认证过程将如下：

1. 引导用户进行登录，通常是重定向到一个基于 form 表单进行登录的页面，具体视配置而定。
2. 用户输入用户名和密码后请求认证，后台还是会像上节描述的那样获取用户名和密码封装成一个 UsernamePasswordAuthenticationToken 对象，然后把它传递给 AuthenticationManager 进行认证。
3. 如果认证失败将继续执行步骤 1，如果认证成功则会保存返回的 Authentication 到 SecurityContext，然后默认会将用户重定向到之前访问的页面。
4. 用户登录认证成功后再次访问之前受保护的资源时就会对用户进行权限鉴定，如不存在对应的访问权限，则会返回 403 错误码。
 
在上述步骤中将有很多不同的类参与，但其中主要的参与者是 ExceptionTranslationFilter。
 
### ExceptionTranslationFilter

ExceptionTranslationFilter 是用来处理来自 AbstractSecurityInterceptor 抛出的 AuthenticationException 和 AccessDeniedException 的。AbstractSecurityInterceptor 是 Spring Security 用于拦截请求进行权限鉴定的，其拥有两个具体的子类，拦截方法调用的 MethodSecurityInterceptor 和拦截 URL 请求的 FilterSecurityInterceptor。当 ExceptionTranslationFilter 捕获到的是 AuthenticationException 时将调用 AuthenticationEntryPoint 引导用户进行登录；如果捕获的是 AccessDeniedException，但是用户还没有通过认证，则调用 AuthenticationEntryPoint 引导用户进行登录认证，否则将返回一个表示不存在对应权限的 403 错误码。
 
### 在 request 之间共享 SecurityContext

可能你早就有这么一个疑问了，既然 SecurityContext 是存放在 ThreadLocal 中的，而且在每次权限鉴定的时候都是从 ThreadLocal 中获取 SecurityContext 中对应的 Authentication 所拥有的权限，并且不同的 request 是不同的线程，为什么每次都可以从 ThreadLocal 中获取到当前用户对应的 SecurityContext 呢？在 Web 应用中这是通过 SecurityContextPersistentFilter 实现的，默认情况下其会在每次请求开始的时候从 session 中获取 SecurityContext，然后把它设置给 SecurityContextHolder，在请求结束后又会将 SecurityContextHolder 所持有的 SecurityContext 保存在 session 中，并且清除 SecurityContextHolder 所持有的 SecurityContext。这样当我们第一次访问系统的时候，SecurityContextHolder 所持有的 SecurityContext 肯定是空的，待我们登录成功后，SecurityContextHolder 所持有的 SecurityContext 就不是空的了，且包含有认证成功的 Authentication 对象，待请求结束后我们就会将 SecurityContext 存在 session 中，等到下次请求的时候就可以从 session 中获取到该 SecurityContext 并把它赋予给 SecurityContextHolder 了，由于 SecurityContextHolder 已经持有认证过的 Authentication 对象了，所以下次访问的时候也就不再需要进行登录认证了。