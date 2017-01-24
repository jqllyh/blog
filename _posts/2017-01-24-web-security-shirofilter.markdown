#Filter mechanism in shiro

In our web project, we use shiro as the security skeleton. In order to deeply understand the internal of shiro, we shortly analyse the shiro filter.
    
## 1. Configuration for shiro

### 1.1 In web.xml

~~~java
	<!-- Shiro配置 -->
	<!-- 配置Shiro过滤器,先让Shiro过滤系统接收到的请求 -->
	<!-- 这里filter-name必须对应applicationContext.xml中定义的<bean id="shiroFilter"/> -->
	<!-- 使用[/*]匹配所有请求,保证所有的可控请求都经过Shiro的过滤 -->
	<!-- 通常会将此filter-mapping放置到最前面(即其他filter-mapping前面),以保证它是过滤器链中第一个起作用的 -->
	<filter>
		<filter-name>shiroFilter</filter-name>
		<filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
		<init-param>
			<!-- 该值缺省为false,表示生命周期由SpringApplicationContext管理,设置为true则表示由ServletContainer管理 -->
			<param-name>targetFilterLifecycle</param-name>
			<param-value>true</param-value>
		</init-param>
	</filter>
	<filter-mapping>
		<filter-name>shiroFilter</filter-name>
		<url-pattern>/*</url-pattern>
	</filter-mapping>
~~~
As the comment in xml said, shiroFilter should be placed in the front of all the filter-mapping. we may wonder what's the function of *DelegatingFilterProxy*, I haven't got the source code for the class. In some blog said, the function is that finding the filter name "shiroFilter" in the spring applicationContext, and delegating all filter operation to it.

### 1.2 In spring-shiro.xml

~~~java
	<bean id="shiroFilter" class="org.apache.shiro.spring.web.ShiroFilterFactoryBean">
		<!-- shiro的核心安全接口 -->
		<property name="securityManager" ref="securityManager" />
		<!-- 要求登录时的链接 -->
		<property name="loginUrl" value="/login.html" />
		<!-- 登陆成功后要跳转的连接 -->
		<property name="successUrl" value="/index.html" />
		<!-- 未授权时要跳转的连接 -->
		<property name="unauthorizedUrl" value="/denied.jsp" />
		
		<!-- Shiro连接约束配置,即过滤链的定义 -->
		<!-- 此处可配合我的这篇文章来理解各个过滤连的作用http://blog.csdn.net/jadyer/article/details/12172839 -->
		<!-- 下面value值的第一个'/'代表的路径是相对于HttpServletRequest.getContextPath()的值来的 -->
		<!-- anon：它对应的过滤器里面是空的,什么都没做,这里.do和.jsp后面的*表示参数,比方说login.jsp?main这种 -->
		<!-- authc：该过滤器下的页面必须验证后才能访问,它是Shiro内置的一个拦截器org.apache.shiro.web.filter.authc.FormAuthenticationFilter -->
		<property name="filterChainDefinitions">
			<value>
				/resources/** = anon
				/login.html = authc
				/logout.html = logout
				/register.html = anon
				/captcha.html = anon
				/denied.jsp = anon
				/view/error/** = anon
				/*/withoutAuth/** = anon
				/api/** = authc, checkPermission[abcd,1234]
				/** = authc
			</value>
		</property>
		<!-- 自定义过滤器 -->
		<property name="filters">
			<map>
				<entry key="rememberMeFilter" value-ref="rememberMeFilter"/>
				<entry key="kickout" value-ref="kickoutSessionFilter" />
				<entry key="checkPermission">
				    <bean class="com.webside.shiro.filter.PermissionFilter" />
				</entry>
				<entry key="authc">  
                   <bean class="org.apache.shiro.web.filter.authc.PassThruAuthenticationFilter" />  
                </entry>
			</map>
		</property>
	</bean>

~~~
Configuration for shiro is relative complex. above lines just list the part relative to filter. in the following section, we focus on the shiroFilter bean, mainly on the members of shiroFilter, and their relations.  

## 2 Start Point from SpringShiroFilter

In spring-shiro.xml, shiroFilter is created by ShiroFilterFactoryBean. This is a special spring bean, **FactoryBean**, which implement `getObject` interface. whenever a shiroFilter is needed, the FactoryBean's getObject method will be called.


### 2.1 PathMatchingFilterChainResolver

- FilterChainManager
   Create Procedure
         
- PatternMatcher
   Create Procedure
   
### 2.2 FilterChain

### 2.3 Filter 

The point is that each filter stays "in front" and "behind" each servlet it is mapped to. So if you have a filter around a servlet, you'll have:

~~~java
void doFilter(..) { 
    // do stuff before servlet gets called

    // invoke the servlet, or any other filters mapped to the target servlet
    chain.doFilter(..);

    // do stuff after the servlet finishes
}
~~~
You also have the option not to call chain.doFilter(..) in which case the servlet will never be called. This is useful for security purposes - for example you can check whether there's a user logged-in.

another good post that explain filter and filter chain, as follow:

#### What are Filters ?

Filters are used to intercept and process requests before they are sent to servlets(in case of request) .

OR

Filters are used to intercept and process a response before they are sent back to client by a servlet.

![filter-filterchain](images/filter.png)

Why they are used ?

-Filters can perform security checks.

-Compress the response stream.

-Create a different response.

What does doFilter() do ?

The doFilter() is called every time the container determines that the filter should be applied to a page.
It takes three arguments

->ServletRequest

->ServlerResponse

->FilterChain

All the functionality that your filter supposed to do is implemented inside doFilter() method.

#### What is FilterChain ?

Your filters do not know anything about the other filters and servlet. FilterChain knows the order of the invocation of filters and driven by the filter elements you defined in the DD.




## 3 Out of Box Shiro Filter   

    
  



##Web.xml中的配置
```java


	
```

```java
	<!-- 配置shiro的过滤器工厂类，这里bean的id shiroFilter要和我们在web.xml中配置的shior过滤器名称一致<filter-name>shiroFilter</filter-name> -->
	<!-- Shiro主过滤器本身功能十分强大,其强大之处就在于它支持任何基于URL路径表达式的、自定义的过滤器的执行 -->
	<!-- Web应用中,Shiro可控制的Web请求必须经过Shiro主过滤器的拦截,Shiro对基于Spring的Web应用提供了完美的支持 -->
	<bean id="shiroFilter" class="org.apache.shiro.spring.web.ShiroFilterFactoryBean">
		<!-- shiro的核心安全接口 -->
		<property name="securityManager" ref="securityManager" />
```
