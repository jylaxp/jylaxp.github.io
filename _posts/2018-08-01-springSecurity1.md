---
layout: post
title: Spring Security 过滤链和过滤器(一)
category: spring
---

### Spring Security 的过滤链如下

1. SecurityContextPersistenceFilter
2. WebAsyncManagerIntegrationFilter
3. HeaderWriterFilter
4. CsrfFilter
5. LogoutFilter
6. UsernamePasswordAuthenticationFilter
7. DefaultLoginPageGeneratingFilter
8. BasicAuthenticationFilter
9. RequestCacheAwareFilter
10. SecurityContextHolderAwareRequestFilter
11. AnonymousAuthenticationFilter
12. SessionManagementFilter
13. ExceptionTranslationFilter
14. FilterSecurityInterceptor

### 1. SecurityContextPersistenceFilter
该过滤器位于在过滤链第一个位置，主要功能是从session中加载SecurityContex，如果session中不存在SecurityContex，则创建新的SecurityContex,然后执行过滤链中后续的过滤器。在后续过滤器返回后，保存SecurityContex到session中。它其实是在为后续的过滤器准备安全上下文环境和后续的扫尾工作。
```java
public class SecurityContextPersistenceFilter extends GenericFilterBean {

	static final String FILTER_APPLIED = "__spring_security_scpf_applied";

    /** SecurityContex存储接口 */
	private SecurityContextRepository repo;


	public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {
		HttpServletRequest request = (HttpServletRequest) req;
		HttpServletResponse response = (HttpServletResponse) res;

        /** 保证统一个请求只有一个SecurityContext。如果已经创建过SecurityContext了，则直接执行后续过滤器。 */
		if (request.getAttribute(FILTER_APPLIED) != null) {
			// ensure that filter is only applied once per request
			chain.doFilter(request, response);
			return;
		}

		final boolean debug = logger.isDebugEnabled();

        /** 设置创建SecurityContext标识 */
		request.setAttribute(FILTER_APPLIED, Boolean.TRUE);

		if (forceEagerSessionCreation) {
			HttpSession session = request.getSession();

			if (debug && session.isNew()) {
				logger.debug("Eagerly created session: " + session.getId());
			}
		}

		HttpRequestResponseHolder holder = new HttpRequestResponseHolder(request,
				response);
		
		/** 从SecurityContex存储接口加载安全上下文 */
		SecurityContext contextBeforeChainExecution = repo.loadContext(holder);

		try {
		    /** 将安全上下文设置到SecurityContextHolder，后续过滤链可以通过SecurityContextHolder获取到安全上下文 */
			SecurityContextHolder.setContext(contextBeforeChainExecution);

            /** 执行后续过滤链 */
			chain.doFilter(holder.getRequest(), holder.getResponse());
		}
		finally {
			SecurityContext contextAfterChainExecution = SecurityContextHolder
					.getContext();
			// Crucial removal of SecurityContextHolder contents - do this before anything
			// else.
			SecurityContextHolder.clearContext();
			
			/** 保存安全上下文 */
			repo.saveContext(contextAfterChainExecution, holder.getRequest(),
					holder.getResponse());
					
			/** 移除标识 */
			request.removeAttribute(FILTER_APPLIED);

			if (debug) {
				logger.debug("SecurityContextHolder now cleared, as request processing completed");
			}
		}
	}
}
```
执行流程很简单明了
- 判断是否存在SecurityContex，如果存在，则执行过滤链后返回；
- 如果不存在SecurityContex，则设置创建标识，调用SecurityContextRepository接口加载SecurityContex；
- 将SecurityContex放在SecurityContextHolder中，后续过滤器可以通过SecurityContextHolder获取SecurityContex；
- 执行过滤链；
- 保存SecurityContex到Session中

这里的SecurityContextRepository是个接口，Spring Security提供了两个实现
- HttpSessionSecurityContextRepository
- NullSecurityContextRepository
默认使用的是HttpSessionSecurityContextRepository，将安全上下文存放在Session中。既然是接口，那么我们也可以实现该接口，将安全上下文存在我们指定的地方。



### 2. WebAsyncManagerIntegrationFilter
该过滤器扩展自OncePerRequestFilter，所以它的核心功能代码只会执行一次。WebAsyncManagerIntegrationFilter通过WebAsyncManager为SecurityContext和Spring Web提供整合功能。

### 3. HeaderWriterFilter
该过滤器扩展自OncePerRequestFilter，所以它的核心功能代码只会执行一次。该过滤器的功能是在请求返回之给Response添加HttpHeader，可用于开启浏览器保护机制，例如X-Frame-Options, X-XSS-Protection 和 X-Content-Type-Options。

### 4. CsrfFilter
该过滤器扩展自OncePerRequestFilter，所以它的核心功能代码只会执行一次。该过滤器的功能是在请求中添加csrfToken来防止csrf攻击。这个过滤器会放过GET, HEAD, TRACE, OPTIONS操作的验证，但会在这些请求中添加csrfToke。
```java
public final class CsrfFilter extends OncePerRequestFilter {
	public static final RequestMatcher DEFAULT_CSRF_MATCHER = new DefaultRequiresCsrfMatcher();

	private final CsrfTokenRepository tokenRepository;
	private RequestMatcher requireCsrfProtectionMatcher = DEFAULT_CSRF_MATCHER;
	private AccessDeniedHandler accessDeniedHandler = new AccessDeniedHandlerImpl();

	@Override
	protected void doFilterInternal(HttpServletRequest request,
			HttpServletResponse response, FilterChain filterChain)
					throws ServletException, IOException {
		request.setAttribute(HttpServletResponse.class.getName(), response);

        /** 从CsrfTokenRepository中加载csrfToken */
		CsrfToken csrfToken = this.tokenRepository.loadToken(request);
		final boolean missingToken = csrfToken == null;
		if (missingToken) {
		    /** csrfToken不在，则生成新的csrfToken，并保存到CsrfTokenRepository */
			csrfToken = this.tokenRepository.generateToken(request);
			this.tokenRepository.saveToken(csrfToken, request, response);
		}
		request.setAttribute(CsrfToken.class.getName(), csrfToken);
		request.setAttribute(csrfToken.getParameterName(), csrfToken);

        /** 判断是否忽略该请求的csrf验证 */
		if (!this.requireCsrfProtectionMatcher.matches(request)) {
		    /** GET, HEAD, TRACE, OPTIONS 不验证 */
			filterChain.doFilter(request, response);
			return;
		}

        /** 获取浏览器提交过来的token */
		String actualToken = request.getHeader(csrfToken.getHeaderName());
		if (actualToken == null) {
			actualToken = request.getParameter(csrfToken.getParameterName());
		}
		
		/** 与服务端的token对比 */
		if (!csrfToken.getToken().equals(actualToken)) {
			if (this.logger.isDebugEnabled()) {
				this.logger.debug("Invalid CSRF token found for "
						+ UrlUtils.buildFullRequestUrl(request));
			}
			
			/** token不一致，则交于accessDeniedHandler处理 */
			if (missingToken) {
				this.accessDeniedHandler.handle(request, response,
						new MissingCsrfTokenException(actualToken));
			}
			else {
				this.accessDeniedHandler.handle(request, response,
						new InvalidCsrfTokenException(csrfToken, actualToken));
			}
			return;
		}

		filterChain.doFilter(request, response);
	}


    /** 默认的RequestMatcher */
	private static final class DefaultRequiresCsrfMatcher implements RequestMatcher {
		private final HashSet<String> allowedMethods = new HashSet<String>(
				Arrays.asList("GET", "HEAD", "TRACE", "OPTIONS"));

		@Override
		public boolean matches(HttpServletRequest request) {
		    /** GET, HEAD, TRACE, OPTIONS 请求匹配失败，也就是忽略csrfToken的对比 */
			return !this.allowedMethods.contains(request.getMethod());
		}
	}
}
```
核心逻辑：
- 从CsrfTokenRepository中获取csrfToken，如果csrfToken不存在则生成新的csrfToken并保存。这CsrfTokenRepository接口我们可以扩展。
- 使用RequestMatcher匹配本次请求，判断是否需要对比csrfToken，如果不需要处理，则执行后续的过滤器。RequestMatcher接口，我们可以根据自己的需求进行自己的实现；默认的RequestMatcher是DefaultRequiresCsrfMatcher，该Matcher的匹配结果是处理除GET, HEAD, TRACE, OPTIONS之外的请求；
- 从请求获取浏览器提交过来的token，与服务端的token对比，如果一样，就放过请求，不一样，则交于accessDeniedHandler处理本次请求。我们可以实现自己的AccessDeniedHandler进行替换Spring Security默认提供的AccessDeniedHandlerImpl。

### 5. LogoutFilter
LogoutFilter处理用户注销登录。默认拦截的方法是POST， 路径是/logout。
```java
public class LogoutFilter extends GenericFilterBean {

	private RequestMatcher logoutRequestMatcher;
	private final LogoutHandler handler;
	private final LogoutSuccessHandler logoutSuccessHandler;

	
	public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {
		HttpServletRequest request = (HttpServletRequest) req;
		HttpServletResponse response = (HttpServletResponse) res;

        /** 判断是否需要注销登录 */
		if (requiresLogout(request, response)) {
			Authentication auth = SecurityContextHolder.getContext().getAuthentication();

            /** 注销登录处理器 */
			this.handler.logout(request, response, auth);

            /** 跳转到注销成功页 */
			logoutSuccessHandler.onLogoutSuccess(request, response, auth);
			return;
		}

		chain.doFilter(request, response);
	}

	 /** 判断本次请求是否注销登录请求，这里是protected方法，给子类留了扩展的机会 */
	protected boolean requiresLogout(HttpServletRequest request,
			HttpServletResponse response) {
		return logoutRequestMatcher.matches(request);
	}
}
```
- 判断是否请求注销登录页面，如果不算，则执行后续过滤器；
- 否则，执行注销逻辑。默认的LogoutHandler包含两个handler:CsrfLogoutHandler和SecurityContextLogoutHandle。其中CsrfLogoutHandler销毁CsrfToken，SecurityContextLogoutHandle无效掉Session，销毁授权信息，清楚SeecurityContext;
- 跳转到注销成功页面

### 6. UsernamePasswordAuthenticationFilter  
该过滤器扩展自AbstractAuthenticationProcessingFilter，完成基于用户名和密码的身份认证功能。AbstractAuthenticationProcessingFilter是基于HTTP认证请求的抽象处理器，它规范了身份认证流程，把具体的认证逻辑留给不同的认证方式扩展。我们看一下这个身份认证的流程。  
```java
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {

		HttpServletRequest request = (HttpServletRequest) req;
		HttpServletResponse response = (HttpServletResponse) res;

        /** 判断是否是身份认证请求 */
		if (!requiresAuthentication(request, response)) {
		    /** 如果不是身份认证请求，则执行后续过滤链 */
			chain.doFilter(request, response);

			return;
		}

		if (logger.isDebugEnabled()) {
			logger.debug("Request is to process authentication");
		}

		Authentication authResult;

		try {
		    /** 进行身份认证，获取认证结果。 attemptAuthentication方法留给子类实现 */
			authResult = attemptAuthentication(request, response);
			if (authResult == null) {
				// return immediately as subclass has indicated that it hasn't completed
				// authentication
				return;
			}
			sessionStrategy.onAuthentication(authResult, request, response);
		}
		catch (InternalAuthenticationServiceException failed) {
			logger.error(
					"An internal error occurred while trying to authenticate the user.",
					failed);
			/** 身份认证失败 */
			unsuccessfulAuthentication(request, response, failed);

			return;
		}
		catch (AuthenticationException failed) {
			// Authentication failed
			/** 身份认证失败 */
			unsuccessfulAuthentication(request, response, failed);

			return;
		}

		// Authentication success
		if (continueChainBeforeSuccessfulAuthentication) {
			chain.doFilter(request, response);
		}
        
        /** 身份认证成功 */
		successfulAuthentication(request, response, chain, authResult);
	}
```

总体流程:
- 判断请求是否是身份认证请求，如果不是，执行后续的过滤器，如果是，走后面流程；
- 调用attemptAuthentication方法完成身份认证。attemptAuthentication方法抽象方法，由子类根据不同的认证方式来完成具体的认证实现，将身份认证的细节从整体流程中剥离出来；
- 最后是是身份认证结果的处理。如果认证失败，将控制权交个AuthenticationFailureHandler处理；如果认证成功，控制权交给AuthenticationSuccessHandler处理；  

身份认证的整体流程很简单，但是，每一步都是都是扩展点。
- AuthenticationManager：身份认证管理器，用来完成认证。它只有一个方法```Authentication authenticate(Authentication authentication)```，该方法的出入参都是```Authentication```;
- Authentication 对用户身份认证信息的抽象，用来提供用户信息和权限，我们可以实现该接口，添加我们自己系统需要使用的用户信息。用户身份认证成功之后， ```Authentication```会存放在安全上下文```SecurityContextHolder```中；
- RequestMatcher 改匹配器用来判断请求是否是用户身份认证请求。```UsernamePasswordAuthenticationFilter```默认的匹配器的规则是:POST方法和请求的URL为/login，如果满足这两个条件，则拦截该请求，从请求中提取用户身份认证信息进行认证，否则放过该请求。我们可以实现自己的```RequestMatcher```注入到该过滤器中，改变匹配的规则；
- AuthenticationSuccessHandler 用户身份认证成功后会调用此接口，改接口的实现类可以完成目标地址跳转的操作，当然，你也可以有自己的实现；
- AuthenticationFailureHandler 用户身份认证失败会调用此接口，处理认证失败的情况；

接下来我们来看看UsernamePasswordAuthenticationFilter对AbstractAuthenticationProcessingFilter的扩展。
```java
    /** 默认的请求参数名称是username和password  */
	public static final String SPRING_SECURITY_FORM_USERNAME_KEY = "username";
	public static final String SPRING_SECURITY_FORM_PASSWORD_KEY = "password";

    /** 设置默认的参数名，这里可以通过setter方法设置这两个参数，对默认值进行覆盖 */
	private String usernameParameter = SPRING_SECURITY_FORM_USERNAME_KEY;
	private String passwordParameter = SPRING_SECURITY_FORM_PASSWORD_KEY;
	
	public UsernamePasswordAuthenticationFilter() {
	    /** 设置RequestMatcher，默认是 URL:/login, POST方法 */
		super(new AntPathRequestMatcher("/login", "POST"));
	}
	
    public Authentication attemptAuthentication(HttpServletRequest request,
			HttpServletResponse response) throws AuthenticationException {
			/** 校验是否是POST请求 */
		if (postOnly && !request.getMethod().equals("POST")) {
			throw new AuthenticationServiceException(
					"Authentication method not supported: " + request.getMethod());
		}

        /** 从请求中获取用户名和密码 */
		String username = obtainUsername(request);
		String password = obtainPassword(request);

		if (username == null) {
			username = "";
		}

		if (password == null) {
			password = "";
		}

		username = username.trim();

        /** 构造Authentication对象，用于身份认证 */
		UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(
				username, password);

		// Allow subclasses to set the "details" property
		setDetails(request, authRequest);

        /** 使用AuthenticationManager进行身份认证 */
		return this.getAuthenticationManager().authenticate(authRequest);
	}	
```
```UsernamePasswordAuthenticationFilter``` 是基于用户密码的身份认证拦截器。它拦截POST请求/login路径的请求，从请求中提取用户名和密码(参数名称默认的是username和password)封装成UsernamePasswordAuthenticationToken对象，然后调用AuthenticationManager进行身份认证。


### 7. DefaultLoginPageGeneratingFilter  
顾名思义，这个过滤器是用来生成默认的登录页面的。只有当没有配置登录的URL时，该过滤器才会插入到过滤链中。


### 8. BasicAuthenticationFilter 
该过滤器扩展自OncePerRequestFilter，所以它的核心功能代码只会执行一次。这个过滤器提供HTTP基本认证。

### 9. RequestCacheAwareFilter 
该过滤器用于恢复被打断的请求。它是用来处理这么一种场景：一个用户在未登录的情况下直接访问一个受保护的资源，系统会抛出AuthenticationException异常，ExceptionTranslationFilter捕获异常，并缓存该请求后，将该请求重定向到登录页，用户登录后需要恢复到原来要访问的URL，这个过滤器就是用来恢复之前的请求的。

### 10. SecurityContextHolderAwareRequestFilter
该过滤器用来包装请求，实现servlet api一些接口方法。

### 11. AnonymousAuthenticationFilter
该过滤器检查SecurityContextHolder中是否有Authentication信息，如果没有则创建一个AnonymousAuthenticationToken

### 12. SessionManagementFilter
检测从请求开始时，让用户是否通过身份认证，如果通过身份认证，则调用```SessionAuthenticationStrategy``` 执行与会话相关的活动，例如：会话固化保护或者会话并发控制。Session这块儿，后面单独拿出来研究，这里不深入。

### 13. ExceptionTranslationFilter
该过滤器处理由```AbstractSecurityInterceptor```抛出的```AccessDeniedException```和```AuthenticationException```异常。该处理器虽然没有做任何安全相关的事情，但是它捕获拦截器抛出的异常，将异常转化为合适的HTTP响应，是连接java异常和HTTP响应的桥梁。该处理器如果捕获的是```AuthenticationException```异常，则调用```AuthenticationEntryPoint```接口处理这个身份认证失败异常，这里我们可以实现```AuthenticationEntryPoint```接口，来处理身份认证失败的情况。过滤器如果捕获的是```AccessDeniedException```异常，会先判断是否是匿名用户，如果是匿名用户则调用```AuthenticationEntryPoint```接口，要求用户进行身份认证。如果不是匿名用户，过滤器则委托```AccessDeniedHandler```处理该异常，默认是交给```AccessDeniedHandlerImpl```处理。
```java
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {
		HttpServletRequest request = (HttpServletRequest) req;
		HttpServletResponse response = (HttpServletResponse) res;

		try {
		    /** 执行过滤链 */
			chain.doFilter(request, response);

			logger.debug("Chain processed normally");
		}
		catch (IOException ex) {
			throw ex;
		}
		catch (Exception ex) {
		    /** 捕获过滤链抛出的异常 */
			// Try to extract a SpringSecurityException from the stacktrace
			/** 展开异常 */
			Throwable[] causeChain = throwableAnalyzer.determineCauseChain(ex);
			
			/** 将异常转换成AuthenticationException异常，结果如果不为null，则说明抛出的是AuthenticationException异常 */
			RuntimeException ase = (AuthenticationException) throwableAnalyzer
					.getFirstThrowableOfType(AuthenticationException.class, causeChain);

			if (ase == null) {
			/** 说明ase不是AuthenticationException， 则再转换成AccessDeniedException异常，不为null则是AccessDeniedException */
				ase = (AccessDeniedException) throwableAnalyzer.getFirstThrowableOfType(
						AccessDeniedException.class, causeChain);
			}

			if (ase != null) {
			    /** 如果是上面的两种异常，则处理异常 */
				handleSpringSecurityException(request, response, chain, ase);
			}
			else {
			    /** 否则，抛出异常 */
				// Rethrow ServletExceptions and RuntimeExceptions as-is
				if (ex instanceof ServletException) {
					throw (ServletException) ex;
				}
				else if (ex instanceof RuntimeException) {
					throw (RuntimeException) ex;
				}

				// Wrap other Exceptions. This shouldn't actually happen
				// as we've already covered all the possibilities for doFilter
				throw new RuntimeException(ex);
			}
		}
	}
	
    private void handleSpringSecurityException(HttpServletRequest request,
			HttpServletResponse response, FilterChain chain, RuntimeException exception)
			throws IOException, ServletException {
		if (exception instanceof AuthenticationException) {
			logger.debug(
					"Authentication exception occurred; redirecting to authentication entry point",
					exception);
            /** 是AuthenticationException异常，要求进行身份认证 */
			sendStartAuthentication(request, response, chain,
					(AuthenticationException) exception);
		}
		else if (exception instanceof AccessDeniedException) {
		    /** AccessDeniedException异常，判断是否是匿名用户，是匿名用户，要求进行身份认证 */
			Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
			if (authenticationTrustResolver.isAnonymous(authentication) || authenticationTrustResolver.isRememberMe(authentication)) {
				logger.debug(
						"Access is denied (user is " + (authenticationTrustResolver.isAnonymous(authentication) ? "anonymous" : "not fully authenticated") + "); redirecting to authentication entry point",
						exception);

				sendStartAuthentication(
						request,
						response,
						chain,
						new InsufficientAuthenticationException(
								"Full authentication is required to access this resource"));
			}
			else {
			    /** 不是匿名用户，委托给accessDeniedHandler处理 */
				logger.debug(
						"Access is denied (user is not anonymous); delegating to AccessDeniedHandler",
						exception);

				accessDeniedHandler.handle(request, response,
						(AccessDeniedException) exception);
			}
		}
	}

	protected void sendStartAuthentication(HttpServletRequest request,
			HttpServletResponse response, FilterChain chain,
			AuthenticationException reason) throws ServletException, IOException {
		// SEC-112: Clear the SecurityContextHolder's Authentication, as the
		// existing Authentication is no longer considered valid
		SecurityContextHolder.getContext().setAuthentication(null);
		requestCache.saveRequest(request, response);
		logger.debug("Calling Authentication entry point.");
		authenticationEntryPoint.commence(request, response, reason);
	}
```


### 14. FilterSecurityInterceptor
这个过滤器是核心安全过滤器，它扩展自```AbstractSecurityInterceptor```，主要完成鉴权功能。

```FilterSecurityInterceptor``` 关键代码
```java
	public void doFilter(ServletRequest request, ServletResponse response,
			FilterChain chain) throws IOException, ServletException {
		FilterInvocation fi = new FilterInvocation(request, response, chain);
		invoke(fi);
	}
	
	public void invoke(FilterInvocation fi) throws IOException, ServletException {
		if ((fi.getRequest() != null)
				&& (fi.getRequest().getAttribute(FILTER_APPLIED) != null)
				&& observeOncePerRequest) {
			// filter already applied to this request and user wants us to observe
			// once-per-request handling, so don't re-do security checking
			fi.getChain().doFilter(fi.getRequest(), fi.getResponse());
		}
		else {
			// first time this request being called, so perform security checking
			if (fi.getRequest() != null) {
			    /** 请求第一次被该类型的过滤器处理，设置标识，后续不再处理 */
				fi.getRequest().setAttribute(FILTER_APPLIED, Boolean.TRUE);
			}

            /** 前置处理，用来鉴权 */
			InterceptorStatusToken token = super.beforeInvocation(fi);

			try {
			    /** 过滤链  */
				fi.getChain().doFilter(fi.getRequest(), fi.getResponse());
			}
			finally {
				super.finallyInvocation(token);
			}

            /** 后置 */
			super.afterInvocation(token, null);
		}
	}
```

```AbstractSecurityInterceptor``` 关键代码
```java
    protected InterceptorStatusToken beforeInvocation(Object object) {
		Assert.notNull(object, "Object was null");
		final boolean debug = logger.isDebugEnabled();

		if (!getSecureObjectClass().isAssignableFrom(object.getClass())) {
			throw new IllegalArgumentException(
					"Security invocation attempted for object "
							+ object.getClass().getName()
							+ " but AbstractSecurityInterceptor only configured to support secure objects of type: "
							+ getSecureObjectClass());
		}

        /** 获取元数据，这个数据会传递给访问决策器 */
		Collection<ConfigAttribute> attributes = this.obtainSecurityMetadataSource()
				.getAttributes(object);

		if (attributes == null || attributes.isEmpty()) {
			if (rejectPublicInvocations) {
				throw new IllegalArgumentException(
						"Secure object invocation "
								+ object
								+ " was denied as public invocations are not allowed via this interceptor. "
								+ "This indicates a configuration error because the "
								+ "rejectPublicInvocations property is set to 'true'");
			}

			if (debug) {
				logger.debug("Public object - authentication not attempted");
			}

			publishEvent(new PublicInvocationEvent(object));

			return null; // no further work post-invocation
		}

		if (debug) {
			logger.debug("Secure object: " + object + "; Attributes: " + attributes);
		}

		if (SecurityContextHolder.getContext().getAuthentication() == null) {
			credentialsNotFound(messages.getMessage(
					"AbstractSecurityInterceptor.authenticationNotFound",
					"An Authentication object was not found in the SecurityContext"),
					object, attributes);
		}

        /** 获取Authentication对象 */
		Authentication authenticated = authenticateIfRequired();

		// Attempt authorization
		try {
		    /** 鉴权，如果鉴权成功，则可以放过该请求 */
			this.accessDecisionManager.decide(authenticated, object, attributes);
		}
		catch (AccessDeniedException accessDeniedException) {
		    /** 鉴权失败 */
			publishEvent(new AuthorizationFailureEvent(object, attributes, authenticated,
					accessDeniedException));

			throw accessDeniedException;
		}

		if (debug) {
			logger.debug("Authorization successful");
		}

		if (publishAuthorizationSuccess) {
			publishEvent(new AuthorizedEvent(object, attributes, authenticated));
		}

		// Attempt to run as a different user
		Authentication runAs = this.runAsManager.buildRunAs(authenticated, object,
				attributes);

		if (runAs == null) {
			if (debug) {
				logger.debug("RunAsManager did not change Authentication object");
			}

			// no further work post-invocation
			return new InterceptorStatusToken(SecurityContextHolder.getContext(), false,
					attributes, object);
		}
		else {
			if (debug) {
				logger.debug("Switching to RunAs Authentication: " + runAs);
			}

			SecurityContext origCtx = SecurityContextHolder.getContext();
			SecurityContextHolder.setContext(SecurityContextHolder.createEmptyContext());
			SecurityContextHolder.getContext().setAuthentication(runAs);

			// need to revert to token.Authenticated post-invocation
			return new InterceptorStatusToken(origCtx, true, attributes, object);
		}
	}
	
	protected Object afterInvocation(InterceptorStatusToken token, Object returnedObject) {
		if (token == null) {
			// public object
			return returnedObject;
		}

		finallyInvocation(token); // continue to clean in this method for passivity

		if (afterInvocationManager != null) {
			// Attempt after invocation handling
			try {
				returnedObject = afterInvocationManager.decide(token.getSecurityContext()
						.getAuthentication(), token.getSecureObject(), token
						.getAttributes(), returnedObject);
			}
			catch (AccessDeniedException accessDeniedException) {
				AuthorizationFailureEvent event = new AuthorizationFailureEvent(
						token.getSecureObject(), token.getAttributes(), token
								.getSecurityContext().getAuthentication(),
						accessDeniedException);
				publishEvent(event);

				throw accessDeniedException;
			}
		}

		return returnedObject;
	}
	
	protected void finallyInvocation(InterceptorStatusToken token) {
		if (token != null && token.isContextHolderRefreshRequired()) {
			if (logger.isDebugEnabled()) {
				logger.debug("Reverting to original Authentication: "
						+ token.getSecurityContext().getAuthentication());
			}

			SecurityContextHolder.setContext(token.getSecurityContext());
		}
	}
```

这里只是过一遍整个过滤链，介绍每个过滤器的功能，具体如何使用Spring Security框架，我们后续在说。我后面会重点分析```AuthenticationManager```和```AccessDecisionManager```这个两个和我们框架使用者相关的代码，并给出一个示例。 

项目代码: [springSecurityExplore](https://github.com/manongcoding/springSecurityExplore) 

