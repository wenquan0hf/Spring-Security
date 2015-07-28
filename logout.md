# 退出登录 logout

要实现退出登录的功能我们需要在 http 元素下定义 logout 元素，这样 Spring Security 将自动为我们添加用于处理退出登录的过滤器 LogoutFilter 到 FilterChain。当我们指定了 http 元素的 auto-config 属性为 true 时 logout 定义是会自动配置的，此时我们默认退出登录的 URL 为 “/j\_spring\_security\_logout”，可以通过 logout 元素的 logout-url 属性来改变退出登录的默认地址。

```
   <security:logout logout-url="/logout.do"/>
```

此外，我们还可以给 logout 指定如下属性：

<table>
<tr>
<td><strong>属性名</strong></td>
<td><strong>作用</strong></td>
</tr>
<tr>
<td>invalidate-session</td>
<td>表示是否要在退出登录后让当前 session 失效，默认为 true。</td>
</tr>
<tr>
<td>delete-cookies</td>
<td>指定退出登录后需要删除的 cookie 名称，多个 cookie 之间以逗号分隔。</td>
</tr>
<tr>
<td>logout-success-url</td>
<td>指定成功退出登录后要重定向的 URL。需要注意的是对应的 URL 应当是不需要登录就可以访问的。</td>
</tr>
<tr>
<td>success-handler-ref</td>
<td>指定用来处理成功退出登录的 LogoutSuccessHandler 的引用。</td>
</tr>
</table>











 