# spring MVC cors跨域实现源码解析

名词解释：跨域资源共享（Cross-Origin Resource Sharing）

简单说就是只要协议、IP、http方法任意一个不同就是跨域。

spring MVC自4.2开始添加了跨域的支持。

跨域具体的定义请移步https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS查看

## 使用案例

spring mvc中跨域使用有3种方式：

在web.xml中配置CorsFilter
```xml
<filter>
  <filter-name>cors</filter-name>
  <filter-class>org.springframework.web.filter.CorsFilter</filter-class>
</filter>
<filter-mapping>
  <filter-name>cors</filter-name>
  <url-pattern>/*</url-pattern>
</filter-mapping>
```

在xml中配置
```xml
// 简单配置，未配置的均使用默认值，就是全面放开
<mvc:cors>  
    <mvc:mapping path="/**" />  
</mvc:cors> 

// 这是一个全量配置
<mvc:cors>  
    <mvc:mapping path="/api/**"  
        allowed-origins="http://domain1.com, http://domain2.com"  
        allowed-methods="GET, PUT"  
        allowed-headers="header1, header2, header3"  
        exposed-headers="header1, header2" allow-credentials="false"  
        max-age="123" />  
  
    <mvc:mapping path="/resources/**"  
        allowed-origins="http://domain1.com" />  
</mvc:cors>  
```

使用注解
```java
@CrossOrigin(maxAge = 3600)  
@RestController  
@RequestMapping("/account")  
public class AccountController {  
  
    @CrossOrigin("http://domain2.com")  
    @RequestMapping("/{id}")  
    public Account retrieve(@PathVariable Long id) {  
        // ...  
    }  
}  
```

## 涉及概念
* CorsConfiguration 具体封装跨域配置信息的pojo

* CorsConfigurationSource request与跨域配置信息映射的容器

* CorsProcessor 具体进行跨域操作的类

* 诺干跨域配置信息初始化类

* 诺干跨域使用的Adapter


### 涉及的java类:
* 封装信息的pojo

    CorsConfiguration


* 存储request与跨域配置信息的容器

    CorsConfigurationSource、UrlBasedCorsConfigurationSource


* 具体处理类

    CorsProcessor、DefaultCorsProcessor


* CorsUtils


* 实现OncePerRequestFilter接口的Adapter

    CorsFilter


* 校验request是否cors，并封装对应的Adapter

    AbstractHandlerMapping、包括内部类PreFlightHandler、CorsInterceptor


* 读取CrossOrigin注解信息

    AbstractHandlerMethodMapping、RequestMappingHandlerMapping 

* 从xml文件中读取跨域配置信息

    CorsBeanDefinitionParser


* 跨域注册辅助类

    MvcNamespaceUtils


## debug分析

要看懂代码我们需要先了解下封装跨域信息的pojo--CorsConfiguration

这边是一个非常简单的pojo，除了跨域对应的几个属性，就只有combine、checkOrigin、checkHttpMethod、checkHeaders。

属性都是多值组合使用的。
```java
	
	public static final String ALL = "*";
	// 允许的请求源
	private List<String> allowedOrigins;
	// 允许的http方法
	private List<String> allowedMethods;
	// 允许的请求头
	private List<String> allowedHeaders;
	// 返回的响应头
	private List<String> exposedHeaders;
	// 是否允许携带cookies
	private Boolean allowCredentials;
	// 预请求的存活有效期
	private Long maxAge;
```

combine是将跨域信息进行合并

3个check方法分别是核对request中的信息是否包含在允许范围内

### 配置初始化

在系统启动时通过CorsBeanDefinitionParser解析配置文件；

加载RequestMappingHandlerMapping时，通过InitializingBean的afterProperties的钩子调用initCorsConfiguration初始化注解信息；

#### 配置文件初始化
在CorsBeanDefinitionParser类的parse方法中打一个断点。

方法中打一个断点

通过代码可以看到这边解析<mvc:cors>中的定义信息。

跨域信息的配置可以以path为单位定义多个映射关系。

解析时如果没有定义则使用默认设置

```java
// CorsBeanDefinitionParser
if (mappings.isEmpty()) {
	// 最简配置时的默认设置
	CorsConfiguration config = new CorsConfiguration();
	config.setAllowedOrigins(DEFAULT_ALLOWED_ORIGINS);
	config.setAllowedMethods(DEFAULT_ALLOWED_METHODS);
	config.setAllowedHeaders(DEFAULT_ALLOWED_HEADERS);
	config.setAllowCredentials(DEFAULT_ALLOW_CREDENTIALS);
	config.setMaxAge(DEFAULT_MAX_AGE);
	corsConfigurations.put("/**", config);
}else {
	// 单个mapping的处理
	for (Element mapping : mappings) {
		CorsConfiguration config = new CorsConfiguration();
		if (mapping.hasAttribute("allowed-origins")) {
			String[] allowedOrigins = StringUtils.tokenizeToStringArray(mapping.getAttribute("allowed-origins"), ",");
			config.setAllowedOrigins(Arrays.asList(allowedOrigins));
		}
		// ...
	}
```

解析完成后，通过MvcNamespaceUtils.registerCorsConfiguratoions注册

这边走的是spring bean容器管理的统一流程，现在转化为BeanDefinition然后再实例化。

```java
// MvcNamespaceUtils
	public static RuntimeBeanReference registerCorsConfigurations(Map<String, CorsConfiguration> corsConfigurations, ParserContext parserContext, Object source) {
		if (!parserContext.getRegistry().containsBeanDefinition(CORS_CONFIGURATION_BEAN_NAME)) {
			RootBeanDefinition corsConfigurationsDef = new RootBeanDefinition(LinkedHashMap.class);
			corsConfigurationsDef.setSource(source);
			corsConfigurationsDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
			if (corsConfigurations != null) {
				corsConfigurationsDef.getConstructorArgumentValues().addIndexedArgumentValue(0, corsConfigurations);
			}
			parserContext.getReaderContext().getRegistry().registerBeanDefinition(CORS_CONFIGURATION_BEAN_NAME, corsConfigurationsDef);
			parserContext.registerComponent(new BeanComponentDefinition(corsConfigurationsDef, CORS_CONFIGURATION_BEAN_NAME));
		}
		else if (corsConfigurations != null) {
			BeanDefinition corsConfigurationsDef = parserContext.getRegistry().getBeanDefinition(CORS_CONFIGURATION_BEAN_NAME);
			corsConfigurationsDef.getConstructorArgumentValues().addIndexedArgumentValue(0, corsConfigurations);
		}
		return new RuntimeBeanReference(CORS_CONFIGURATION_BEAN_NAME);
	}
```

#### 注解初始化

在RequestMappingHandlerMapping的initCorsConfiguration中扫描使用CrossOrigin注解的方法，并提取信息。

```java
//
	@Override
	protected CorsConfiguration initCorsConfiguration(Object handler, Method method, RequestMappingInfo mappingInfo) {
		HandlerMethod handlerMethod = createHandlerMethod(handler, method);
		CrossOrigin typeAnnotation = AnnotatedElementUtils.findMergedAnnotation(handlerMethod.getBeanType(), CrossOrigin.class);
		CrossOrigin methodAnnotation = AnnotatedElementUtils.findMergedAnnotation(method, CrossOrigin.class);

		if (typeAnnotation == null && methodAnnotation == null) {
			return null;
		}

		CorsConfiguration config = new CorsConfiguration();
		updateCorsConfig(config, typeAnnotation);
		updateCorsConfig(config, methodAnnotation);

		// ... 设置默认值
		return config;
	}	
```

### 跨域请求处理

HandlerMapping在正常处理完查找处理器后，在AbstractHandlerMapping.getHandler中校验是否是跨域请求,如果是分两种进行处理：

* 如果是预请求，将处理器替换为内部类PreFlightHandler

* 如果是正常请求，添加CorsInterceptor拦截器

拿到处理器后，通过请求头是否包含Origin判断是否跨域,如果是跨域,通过UrlBasedCorsConfigurationSource获取跨域配置信息，并委托getCorsHandlerExecutionChain处理

UrlBasedCorsConfigurationSource是CorsConfigurationSource的实现，从类名就可以猜出这边request与CorsConfiguration的映射是基于url的。getCorsConfiguration中提取request中的url后，逐一验证配置是否匹配url。

```java
	// UrlBasedCorsConfigurationSource
	public CorsConfiguration getCorsConfiguration(HttpServletRequest request) {
		String lookupPath = this.urlPathHelper.getLookupPathForRequest(request);
		for(Map.Entry<String, CorsConfiguration> entry : this.corsConfigurations.entrySet()) {
			if (this.pathMatcher.match(entry.getKey(), lookupPath)) {
				return entry.getValue();
			}
		}
		return null;
	}

	// AbstractHandlerMapping
	public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		Object handler = getHandlerInternal(request);
		// ...

		HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
		if (CorsUtils.isCorsRequest(request)) {
			CorsConfiguration globalConfig = this.corsConfigSource.getCorsConfiguration(request);
			CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
			CorsConfiguration config = (globalConfig != null ? globalConfig.combine(handlerConfig) : handlerConfig);
			executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
		}
		return executionChain;
	}
	// HttpHeaders
	public static final String ORIGIN = "Origin";

	// CorsUtils
	public static boolean isCorsRequest(HttpServletRequest request) {
		return (request.getHeader(HttpHeaders.ORIGIN) != null);
	}
```

通过请求头的http方法是否options判断是否预请求，如果是使用PreFlightRequest替换处理器；如果是普通请求，添加一个拦截器CorsInterceptor。

PreFlightRequest是CorsProcessor对于HttpRequestHandler的一个适配器。这样HandlerAdapter直接使用HttpRequestHandlerAdapter处理。

CorsInterceptor 是CorsProcessor对于HnalderInterceptorAdapter的适配器。

```java
	// AbstractHandlerMapping
	protected HandlerExecutionChain getCorsHandlerExecutionChain(HttpServletRequest request,
			HandlerExecutionChain chain, CorsConfiguration config) {

		if (CorsUtils.isPreFlightRequest(request)) {
			HandlerInterceptor[] interceptors = chain.getInterceptors();
			chain = new HandlerExecutionChain(new PreFlightHandler(config), interceptors);
		}
		else {
			chain.addInterceptor(new CorsInterceptor(config));
		}
		return chain;
	}


	private class PreFlightHandler implements HttpRequestHandler {

		private final CorsConfiguration config;

		public PreFlightHandler(CorsConfiguration config) {
			this.config = config;
		}

		@Override
		public void handleRequest(HttpServletRequest request, HttpServletResponse response)
				throws IOException {

			corsProcessor.processRequest(this.config, request, response);
		}
	}


	private class CorsInterceptor extends HandlerInterceptorAdapter {

		private final CorsConfiguration config;

		public CorsInterceptor(CorsConfiguration config) {
			this.config = config;
		}

		@Override
		public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
				Object handler) throws Exception {

			return corsProcessor.processRequest(this.config, request, response);
		}
	}

	// CorsUtils
	public static boolean isPreFlightRequest(HttpServletRequest request) {
		return (isCorsRequest(request) && request.getMethod().equals(HttpMethod.OPTIONS.name()) &&
				request.getHeader(HttpHeaders.ACCESS_CONTROL_REQUEST_METHOD) != null);
	}

```


