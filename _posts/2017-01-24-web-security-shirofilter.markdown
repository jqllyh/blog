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

In spring-shiro.xml, shiroFilter is created by ShiroFilterFactoryBean. This is a special spring bean, **FactoryBean**, which implement `getObject` interface. whenever a shiroFilter is needed, the FactoryBean's `getObject` method will be called, it will call `createInstance`.

### 2.1 The born of ShiroFilter

~~~java

    protected AbstractShiroFilter createInstance() throws Exception {

       
        SecurityManager securityManager = getSecurityManager();

        FilterChainManager manager = createFilterChainManager();

        //Expose the constructed FilterChainManager by first wrapping it in a
        // FilterChainResolver implementation. The AbstractShiroFilter implementations
        // do not know about FilterChainManagers - only resolvers:
        PathMatchingFilterChainResolver chainResolver = new PathMatchingFilterChainResolver();
        chainResolver.setFilterChainManager(manager);

        //Now create a concrete ShiroFilter instance and apply the acquired SecurityManager and built
        //FilterChainResolver.  It doesn't matter that the instance is an anonymous inner class
        //here - we're just using it because it is a concrete AbstractShiroFilter instance that accepts
        //injection of the SecurityManager and FilterChainResolver:
        return new SpringShiroFilter((WebSecurityManager) securityManager, chainResolver);
    }

~~~

From the `createInstance` function, we can see the Shiro function entrypoint *ShiroFilter* is an instance of SpringShiroFilter. As it's name suggested, it a web Filter, more exactly, it inherits from AbstractShiroFilter.
Main parameters for constructor are SecurityManager and PathMatchingFilterChainResolver. SecurityManager is the core of shiro, we need discuss it in another chapter. Here we just focus on ChainResolver.

### 2.2 PathMatchingFilterChainResolver

ChainResolver in an important part in the filter skeleton, it use ChainManager to manager the filter and filterChain, and get the corresponding filterChain according to URL. so let's first look at FilterChainManager.

#### 2.2.1 FilterChainManager
   The name is very clear, it manager filterChain.
~~~java
       protected FilterChainManager createFilterChainManager() {

        DefaultFilterChainManager manager = new DefaultFilterChainManager();
        Map<String, Filter> defaultFilters = manager.getFilters();
        //apply global settings if necessary:
        for (Filter filter : defaultFilters.values()) {
            applyGlobalPropertiesIfNecessary(filter);
        }

        //Apply the acquired and/or configured filters:
        Map<String, Filter> filters = getFilters();
        if (!CollectionUtils.isEmpty(filters)) {
            for (Map.Entry<String, Filter> entry : filters.entrySet()) {
                String name = entry.getKey();
                Filter filter = entry.getValue();
                applyGlobalPropertiesIfNecessary(filter);
                if (filter instanceof Nameable) {
                    ((Nameable) filter).setName(name);
                }
                //'init' argument is false, since Spring-configured filters should be initialized
                //in Spring (i.e. 'init-method=blah') or implement InitializingBean:
                manager.addFilter(name, filter, false);
            }
        }

        //build up the chains:
        Map<String, String> chains = getFilterChainDefinitionMap();
        if (!CollectionUtils.isEmpty(chains)) {
            for (Map.Entry<String, String> entry : chains.entrySet()) {
                String url = entry.getKey();
                String chainDefinition = entry.getValue();
                manager.createChain(url, chainDefinition);
            }
        }

        return manager;
    }
~~~
The main parts in the create procedure include following:

1. defaultFilters is the default Filter that came with shiro, that is *DefaultFilter*. current has 11 default filters, such as *anon, authc, authcBasic, etc*.

2. filters is the filters configured to shiroFilter Bean, the tag name is *<property name="filters">*.

3. chains is the filterChain configured to shiroFilterBean, the tag name is  *<property name="filterChainDefinitions">*.

    The chains  is instance of SimpleNamedFilterList, which implement NamedFilterList. the chain has backingList, which is an instance of List<Filter>. The really interesting of the List is the method *proxy*
~~~java
    
    public FilterChain proxy(FilterChain orig) {
        return new ProxiedFilterChain(orig, this);
    }
    
    public class ProxiedFilterChain implements FilterChain {

    private static final Logger log = LoggerFactory.getLogger(ProxiedFilterChain.class);

    private FilterChain orig;
    private List<Filter> filters;
    private int index = 0;

    public ProxiedFilterChain(FilterChain orig, List<Filter> filters) {
        if (orig == null) {
            throw new NullPointerException("original FilterChain cannot be null.");
        }
        this.orig = orig;
        this.filters = filters;
        this.index = 0;
    }

    public void doFilter(ServletRequest request, ServletResponse response) throws IOException, ServletException {
        if (this.filters == null || this.filters.size() == this.index) {
            //we've reached the end of the wrapped chain, so invoke the original one:
            if (log.isTraceEnabled()) {
                log.trace("Invoking original filter chain.");
            }
            this.orig.doFilter(request, response);
        } else {
            if (log.isTraceEnabled()) {
                log.trace("Invoking wrapped filter at index [" + this.index + "]");
            }
            this.filters.get(this.index++).doFilter(request, response, this);
        }
    }
}
~~~

From the code above, you can deduce that NamedFilterList::Proxy is translate filterList to **really** FilterChain.

Finally, it comes the important part PatternMatcher.  
         
#### 2.2.2 PatternMatcher

From the PatternMatcher source code, we can easy deduce what this class do.

~~~java
    
     public FilterChain getChain(ServletRequest request, ServletResponse response, FilterChain originalChain) {
        FilterChainManager filterChainManager = getFilterChainManager();
        if (!filterChainManager.hasChains()) {
            return null;
        }

        String requestURI = getPathWithinApplication(request);

        //the 'chain names' in this implementation are actually path patterns defined by the user.  We just use them
        //as the chain name for the FilterChainManager's requirements
        for (String pathPattern : filterChainManager.getChainNames()) {

            // If the path does match, then pass on to the subclass implementation for specific checks:
            if (pathMatches(pathPattern, requestURI)) {
                if (log.isTraceEnabled()) {
                    log.trace("Matched path pattern [" + pathPattern + "] for requestURI [" + requestURI + "].  " +
                            "Utilizing corresponding filter chain...");
                }
                return filterChainManager.proxy(originalChain, pathPattern);
            }
        }

        return null;
    }

    protected boolean pathMatches(String pattern, String path) {
        PatternMatcher pathMatcher = getPathMatcher();
        return pathMatcher.matches(pattern, path);
    }
~~~

PathMatchingFilterChainResolver implement FilterChainResolver, the only one interface needed is getChain.


### 2.3 What's the ShiroFilter really do
ShiroFilter is a subclass of AbstractShiroFilter. and it get SecurityManager and FilterChainResolver from bean configuration. so, let as see the internal of AbstractShiroFilter.

 ~~~java
     protected void doFilterInternal(ServletRequest servletRequest, ServletResponse servletResponse, final FilterChain chain)
            throws ServletException, IOException {

        Throwable t = null;

        try {
            final ServletRequest request = prepareServletRequest(servletRequest, servletResponse, chain);
            final ServletResponse response = prepareServletResponse(request, servletResponse, chain);

            final Subject subject = createSubject(request, response);

            //noinspection unchecked
            subject.execute(new Callable() {
                public Object call() throws Exception {
                    updateSessionLastAccessTime(request, response);
                    executeChain(request, response, chain);
                    return null;
                }
            });
        } catch (ExecutionException ex) {
            t = ex.getCause();
        } catch (Throwable throwable) {
            t = throwable;
        }
        .......
    }

   
    protected FilterChain getExecutionChain(ServletRequest request, ServletResponse response, FilterChain origChain) {
        FilterChain chain = origChain;

        FilterChainResolver resolver = getFilterChainResolver();
        if (resolver == null) {
            log.debug("No FilterChainResolver configured.  Returning original FilterChain.");
            return origChain;
        }

        FilterChain resolved = resolver.getChain(request, response, origChain);
        if (resolved != null) {
            log.trace("Resolved a configured FilterChain for the current request.");
            chain = resolved;
        } else {
            log.trace("No FilterChain configured for the current request.  Using the default.");
        }

        return chain;
    }

    protected void executeChain(ServletRequest request, ServletResponse response, FilterChain origChain)
            throws IOException, ServletException {
        FilterChain chain = getExecutionChain(request, response, origChain);
        chain.doFilter(request, response);
    }
 ~~~
 
 1. when the ShiroFilter be invoked, its doFilterInternal will be called(see *OncePerRequestFilter*).
 2. Prepare httprequest and httpresponse, and create subject from the current session.
 2. Using resolver, get execute chain for the current request, attation please, the resolved FilterChain include origChain from Servlet Container-provided chain.
 3. call chain.doFilter to execute all the filter in the chain. the chain is an instance of ProxiedFilterChain.
 

### 2.3 Filter & FilterChain

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

- Filters can perform security checks.

- Compress the response stream.

- Create a different response.

What does doFilter() do ?

The doFilter() is called every time the container determines that the filter should be applied to a page.
It takes three arguments
 
ServletRequest
ServlerResponse
FilterChain

All the functionality that your filter supposed to do is implemented inside doFilter() method.

#### What is FilterChain ?

Your filters do not know anything about the other filters and servlet. FilterChain knows the order of the invocation of filters and driven by the filter elements you defined in the DD.




## 3 Out of Box Shiro Filter   

## UML sequence

```flow
st=>start: 开始
e=>end: 结束
op=>operation: 我的操作
cond=>condition: 确认？

st->op->cond
cond(yes)->e
cond(no)->op
```

```sequence
张三->李四: 嘿，小四儿, 写博客了没?
Note right of 李四: 李四愣了一下，说：
李四-->张三: 忙得吐血，哪有时间写。
```


    
  
