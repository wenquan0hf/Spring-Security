# 异常信息本地化

Spring Security 支持将展现给终端用户看的异常信息本地化，这些信息包括认证失败、访问被拒绝等。而对于展现给开发者看的异常信息和日志信息（如配置错误）则是不能够进行本地化的，它们是以英文硬编码在 Spring Security 的代码中的。在 Spring-Security-core-xxx.jar 包的 org.springframework.security 包下拥有一个以英文异常信息为基础的 messages.properties 文件，以及其它一些常用语言的异常信息对应的文件，如 messages\_zh\_CN.properties 文件。那么对于用户而言所需要做的就是在自己的 ApplicationContext 中定义如下这样一个 bean。

```
   <bean id="messageSource"
   class="org.springframework.context.support.ReloadableResourceBundleMessageSource">
      <property name="basename"
         value="classpath:org/springframework/security/messages" />
   </bean>
```

如果要自己定制 messages.properties 文件，或者需要新增本地化支持文件，则可以 copy Spring Security 提供的默认 messages.properties 文件，将其中的内容进行修改后再注入到上述 bean 中。比如我要定制一些中文的提示信息，那么我可以在 copy 一个 messages.properties 文件到类路径的 “com/xxx” 下，然后将其重命名为 messages\_zh\_CN.properties，并修改其中的提示信息。然后通过 basenames 属性注入到上述 bean 中，如：

```
   <bean id="messageSource"
   class="org.springframework.context.support.ReloadableResourceBundleMessageSource">
      <property name="basenames">
         <array>
            <!-- 将自定义的放在 Spring Security 内置的之前 -->
            <value>classpath:com/xxx/messages</value>
            <value>classpath:org/springframework/security/messages</value>
         </array>
      </property>
   </bean>
```

有一点需要注意的是将自定义的 messages.properties 文件路径定义在 Spring Security 内置的 message.properties 路径定义之前。