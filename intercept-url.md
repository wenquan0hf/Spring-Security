# intercept-url配置

## 指定拦截的 url

通过 pattern 指定当前 intercept-url 定义应当作用于哪些 url。

```
<security:intercept-url pattern="/**" access="ROLE_USER"/>
```

## 指定访问权限

可以通过 access 属性来指定 intercept-url 对应 URL 访问所应当具有的权限。access 的值是一个字符串，其可以直接是一个权限的定义，也可以是一个表达式。常用的类型有简单的角色名称定义，多个名称之间用逗号分隔，如：

```
<security:intercept-url pattern="/secure/**" access="ROLE_USER,ROLE_ADMIN"/>
```

在上述配置中就表示 secure 路径下的所有 URL 请求都应当具有 ROLE\_USER 或 ROLE\_ADMIN 权限。当 access 的值是以 “ROLE\_” 开头的则将会交由 RoleVoter 进行处理。
 
此外，其还可以是一个表达式，上述配置如果使用表达式来表示的话则应该是如下这个样子。

```
   <security:http use-expressions="true">
      <security:form-login />
      <security:logout />
      <security:intercept-url pattern="/secure/**" access="hasAnyRole('ROLE_USER','ROLE_ADMIN')"/>
   </security:http>
```

或者是使用 hasRole()表达式，然后中间以 or 连接，如：

```
   <security:intercept-url pattern="/secure/**" access="hasRole('ROLE_USER') or hasRole('ROLE_ADMIN')"/>
```

需要注意的是使用表达式时需要指定 http 元素的 use-expressions=”true”。更多关于使用表达式的内容将在后文介绍。当 intercept-url 的 access 属性使用表达式时默认将使用 WebExpressionVoter 进行处理。

此外，还可以指定三个比较特殊的属性值，默认情况下将使用 AuthenticatedVoter 来处理它们。IS\_AUTHENTICATED\_ANONYMOUSLY 表示用户不需要登录就可以访问；IS\_AUTHENTICATED\_REMEMBERED 表示用户需要是通过 Remember-Me 功能进行自动登录的才能访问；IS\_AUTHENTICATED\_FULLY 表示用户的认证类型应该是除前两者以外的，也就是用户需要是通过登录入口进行登录认证的才能访问。如我们通常会将登录地址设置为 IS\_AUTHENTICATED\_ANONYMOUSLY。

```
   <security:http>
      <security:form-login login-page="/login.jsp"/>
      <!-- 登录页面可以匿名访问 -->
      <security:intercept-url pattern="/login.jsp*" access="IS_AUTHENTICATED_ANONYMOUSLY"/>
      <security:intercept-url pattern="/**" access="ROLE_USER"/>
   </security:http>
```

## 指定访问协议
       
需求可以通过指定 intercept-url 的 requires-channel 属性来指定。requires-channel 支持三个值：http、https 和 any。any 表示 http 和 https 都可以访问。

```
   <security:http auto-config="true">
      <security:form-login/>
      <!-- 只能通过 https 访问 -->
      <security:intercept-url pattern="/admin/**" access="ROLE_ADMIN" requires-channel="https"/>
      <!-- 只能通过 http 访问 -->
      <security:intercept-url pattern="/**" access="ROLE_USER" requires-channel="http"/>
   </security:http>
```

需要注意的是当试图使用 http 请求限制了只能通过 https 访问的资源时会自动跳转到对应的 https 通道重新请求。如果所使用的 http 或者 https 协议不是监听在标准的端口上（http 默认是 80，https 默认是 443），则需要我们通过 port-mapping 元素定义好它们的对应关系。

```
   <security:http auto-config="true">
      <security:form-login/>
      <!-- 只能通过 https 访问 -->
      <security:intercept-url pattern="/admin/**" access="ROLE_ADMIN" requires-channel="https"/>
      <!-- 只能通过 http 访问 -->
      <security:intercept-url pattern="/**" access="ROLE_USER" requires-channel="http"/>
      <security:port-mappings>
         <security:port-mapping http="8899" https="9988"/>
      </security:port-mappings>
   </security:http>
```

## 指定请求方法

通常我们都会要求某些 URL 只能通过 POST 请求，某些 URL 只能通过 GET 请求。这些限制 Spring Security 也已经为我们实现了，通过指定 intercept-url 的 method 属性可以限制当前 intercept-url 适用的请求方式，默认为所有的方式都可以。
  
```
 <security:http auto-config="true">
      <security:form-login/>
      <!-- 只能通过 POST 访问 -->
      <security:intercept-url pattern="/post/**" method="POST"/>
      <!-- 只能通过 GET 访问 -->
      <security:intercept-url pattern="/**" access="ROLE_USER" method="GET"/>
   </security:http>
```

method 的可选值有 GET、POST、DELETE、PUT、HEAD、OPTIONS 和 TRACE。