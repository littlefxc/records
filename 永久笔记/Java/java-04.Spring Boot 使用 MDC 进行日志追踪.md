---
title: Spring Boot 使用 MDC 进行日志追踪
status: Done
Tags:
  - MDC
  - 日志追踪
---

> 转载自 [Springboot使用MDC进行日志追踪_springboot日志追踪_繁华尽头满是殇的博客-CSDN博客](https://blog.csdn.net/qq_48922459/article/details/129001157)
>   
>与之不同的是本文配置的是过滤器而不是拦截器

## 前言

MDC（Mapped Diagnostic Context）是一个可以追踪程序上下文日志的东西，是springboot项目自带的`org.slf4j`包下的类，无需引入额外依赖即可使用。

## 一、为什么要跟踪日志

1. 假如我们需要分析用户 a 的请求日志，但是程序的访问量很大，还有 b、c、d 用户同时访问，那怎么确定哪一条日志是 a 用户请求的呢？这时候就需要使用`MDC`对用户 a 的日志进行跟踪。
2. 微服务调用，我们很难确定某条日志是从哪台机器请求的，这时候也可以使用`MDC`进行链路追踪。

## 二、MDC存储日志原理

`MDC` 使用 `ThreadLocal` 存储日志数据，所以它是线程安全的，它可以把同一个请求的日志都存储一个相同的值，每次打印日志的时候，会自动打印出当前日志的键值`value`，我们查询日志的时候就可以根据`value`来查询。

## 三、代码

### 1、封装MDC工具类

> 此工具类定义了一个`traceId`作为日志的`key`，使用`UUID`为不同的请求生成不同的`traceId`值，作为`value`

```java
import org.slf4j.MDC;  
  
import java.util.Map;  
import java.util.UUID;  
  
public class MdcUtil {  
    public static final String TRACE_ID = "traceId";  
  
    public static String generateTraceId() {  
        return UUID.randomUUID().toString().replace("-", "");  
    }  
  
    public static String getTraceId() {  
        return MDC.get(TRACE_ID);  
    }  
  
    public static void setTraceId(String traceId) {  
        MDC.put(TRACE_ID, traceId);  
    }  
  
    public static void setContextMap(Map<String, String> context) {  
        MDC.setContextMap(context);  
    }  
  
    public static void removeTraceId() {  
        MDC.remove(TRACE_ID);  
    }  
  
    public static void clear() {  
        MDC.clear();  
    }  
}
```

### 2、配置日志过滤器

下面是当请求进入时，为该请求生成一个traceId，如果有上层调用就用上层的ID，否则构建一个，上层ID就是微服务的时候，从消费者服务器请求过来时携带的traceId，这时应该沿用消费者服务器携带的ID作为，达到链路追踪的目的。

#### 2.1 HttpServletRequest 包装类

`javax.servlet.http.HttpServletRequest` 是无法修改 request 的 header 的，所以只能继承写个包装类，用Map类写个成员属性复制请求头。

```java
public class XssHttpServletRequestWrapper extends HttpServletRequestWrapper {  
    HttpServletRequest orgRequest;  
  
    private Map<String, String> headerMap = new HashMap<String, String>();  
  
    public XssHttpServletRequestWrapper(HttpServletRequest request) {  
        super(request);  
        orgRequest = request;  
    }
  
    @Override  
    public Map<String,String[]> getParameterMap() {  
        Map<String,String[]> map = new LinkedHashMap<>();  
        Map<String,String[]> parameters = super.getParameterMap();  
        for (String key : parameters.keySet()) {  
            String[] values = parameters.get(key);  
            for (int i = 0; i < values.length; i++) {  
                values[i] = xssEncode(values[i]);  
            }  
            map.put(key, values);  
        }  
        return map;  
    }  
  
    /**  
     * The default behavior of this method is to return getHeaderNames() on the     * wrapped request object.     */    @Override  
    public Enumeration<String> getHeaderNames() {  
        List<String> names = Collections.list(super.getHeaderNames());  
        names.addAll(headerMap.keySet());  
        return Collections.enumeration(names);  
    }  
  
    /**  
     * The default behavior of this method is to return getHeaders(String name)     * on the wrapped request object.     *     * @param name  
     */  
    @Override  
    public Enumeration<String> getHeaders(String name) {  
        List<String> values = Collections.list(super.getHeaders(name));  
        if (headerMap.containsKey(name)) {  
            values.add(headerMap.get(name));  
        }  
        return Collections.enumeration(values);  
    }  
  
  
  
    @Override  
    public String getHeader(String name) {  
        String value = super.getHeader(xssEncode(name));  
        if (StringUtils.isNotBlank(value)) {  
            value = xssEncode(value);  
        }  
        if (headerMap.containsKey(name)) {  
            value = headerMap.get(name);  
        }  
        return value;  
    }  
  
    public void addHeader(String name, String value) {  
        headerMap.put(name, value);  
    }  
  
    private String xssEncode(String input) {  
        return XssUtils.filter(input);  
    }  
  
    /**  
     * 获取最原始的request  
     */    public HttpServletRequest getOrgRequest() {  
        return orgRequest;  
    }  
  
    /**  
     * 获取最原始的request  
     */    public static HttpServletRequest getOrgRequest(HttpServletRequest request) {  
        if (request instanceof XssHttpServletRequestWrapper) {  
            return ((XssHttpServletRequestWrapper) request).getOrgRequest();  
        }  
  
        return request;  
    }  
  
}
```

#### 2.2 XssUtils

```java
public class XssUtils extends Whitelist {  
  
    /**  
     * XSS过滤  
     */  
    public static String filter(String html){  
        return Jsoup.clean(html, xssWhitelist());  
    }  
  
    /**  
     * XSS过滤白名单  
     */  
    private static Whitelist xssWhitelist(){  
        return new Whitelist()  
            //支持的标签  
            .addTags("a", "b", "blockquote", "br", "caption", "cite", "code", "col", "colgroup", "dd", "div", "dl",  
                    "dt", "em", "h1", "h2", "h3", "h4", "h5", "h6", "i", "img", "li", "ol", "p", "pre", "q", "small",  
                    "strike", "strong","sub", "sup", "table", "tbody", "td","tfoot", "th", "thead", "tr", "u","ul",  
                    "embed","object","param","span")  
  
            //支持的标签属性  
            .addAttributes("a", "href", "class", "style", "target", "rel", "nofollow")  
            .addAttributes("blockquote", "cite")  
            .addAttributes("code", "class", "style")  
            .addAttributes("col", "span", "width")  
            .addAttributes("colgroup", "span", "width")  
            .addAttributes("img", "align", "alt", "height", "src", "title", "width", "class", "style")  
            .addAttributes("ol", "start", "type")  
            .addAttributes("q", "cite")  
            .addAttributes("table", "summary", "width", "class", "style")  
            .addAttributes("tr", "abbr", "axis", "colspan", "rowspan", "width", "style")  
            .addAttributes("td", "abbr", "axis", "colspan", "rowspan", "width", "style")  
            .addAttributes("th", "abbr", "axis", "colspan", "rowspan", "scope","width", "style")  
            .addAttributes("ul", "type", "style")  
            .addAttributes("pre", "class", "style")  
            .addAttributes("div", "class", "id", "style")  
            .addAttributes("embed", "src", "wmode", "flashvars", "pluginspage", "allowFullScreen", "allowfullscreen",  
                "quality", "width", "height", "align", "allowScriptAccess", "allowscriptaccess", "allownetworking", "type")  
            .addAttributes("object", "type", "id", "name", "data", "width", "height", "style", "classid", "codebase")  
            .addAttributes("param", "name", "value")  
            .addAttributes("span", "class", "style")  
  
            //标签属性对应的协议  
            .addProtocols("a", "href", "ftp", "http", "https", "mailto")  
            .addProtocols("img", "src", "http", "https")  
            .addProtocols("blockquote", "cite", "http", "https")  
            .addProtocols("cite", "cite", "http", "https")  
            .addProtocols("q", "cite", "http", "https")  
            .addProtocols("embed", "src", "http", "https");  
    } 
  
}
```

#### 2.3 过滤器

1. 使用 HttpServletRequest 包装类包装原始 HttpServletRequest
2. 判断请求头是否包含 `traceId`，有则用 将其放入 MDC 工具类中，无则自我生成一个。
3. 将 `traceId` 填充到其它的 Http 工具类中。
4. 移除 `traceId`

```java
public class XssFilter implements Filter {  
  
   @Override  
   public void init(FilterConfig config) {  
   }  
  
   @Override  
   public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)  
            throws IOException, ServletException {  
      XssHttpServletRequestWrapper xssRequest = new XssHttpServletRequestWrapper(  
            (HttpServletRequest) request);  
      String traceId = xssRequest.getHeader(MdcUtil.TRACE_ID);  
      if (StringUtils.isBlank(traceId)) {  
         traceId = MdcUtil.generateTraceId();  
         xssRequest.addHeader(MdcUtil.TRACE_ID, traceId);  
      }  
      MdcUtil.setTraceId(traceId);  
      GlobalHeaders.INSTANCE.header(MdcUtil.TRACE_ID, traceId); 
      // 其他的Http工具类
      chain.doFilter(xssRequest, response);  
   }  
  
   @Override  
   public void destroy() {  
      MdcUtil.removeTraceId();  
      GlobalHeaders.INSTANCE.removeHeader(MdcUtil.TRACE_ID);  
      // 其他的Http工具类
   }  
  
}
```

#### 2.3 过滤器注册

配置过滤器为第一个过滤器

```java
@Bean  
public FilterRegistrationBean xssFilterRegistration() {  
    FilterRegistrationBean registration = new FilterRegistrationBean();  
    registration.setDispatcherTypes(DispatcherType.REQUEST);  
    registration.setFilter(new XssFilter());  
    registration.addUrlPatterns("/*");  
    registration.setName("xssFilter");  
    registration.setOrder(Integer.MAX_VALUE);  
    return registration;  
}
```


### 3、解决 traceId 的传递问题

#### 3.1 不同线程间的传递

需要重写线程池，在线程池启动新线程之前，复制当前线程的 traceId

##### 3.1.1 多线程日志追踪工具类

```java
/**  
 * 多线程日志追踪工具类  
 *  
 * @author fengxc 
 */
public class ThreadMdcUtil {  
    public static void setTraceIdIfAbsent() {  
        if (MdcUtil.getTraceId() == null) {  
            MdcUtil.setTraceId(MdcUtil.generateTraceId());  
        }  
    }  
  
    public static <T> Callable<T> wrap(final Callable<T> callable, final Map<String, String> context) {  
        return () -> {  
            if (context == null) {  
                MdcUtil.clear();  
            } else {  
                MdcUtil.setContextMap(context);  
            }  
            setTraceIdIfAbsent();  
            try {  
                return callable.call();  
            } finally {  
                MdcUtil.clear();  
            }  
        };  
    }  
  
    public static Runnable wrap(final Runnable runnable, final Map<String, String> context) {  
        return () -> {  
            if (context == null) {  
                MdcUtil.clear();  
            } else {  
                MdcUtil.setContextMap(context);  
            }  
            //设置traceId  
            setTraceIdIfAbsent();  
            try {  
                runnable.run();  
            } finally {  
                MdcUtil.clear();  
            }  
        };  
    }  
}
```

##### 3.1.2 线程池复制 traceId

```java
/**  
 * 日志追踪线程池配置  
 *  
 * @author fengxc 
 */
public class CustomThreadPoolTaskExecutor extends ThreadPoolTaskExecutor {  
  
    @Override  
    public void execute(@NotNull Runnable task) {  
        super.execute(ThreadMdcUtil.wrap(task, MDC.getCopyOfContextMap()));  
    }  
  
    @NotNull  
    @Override    public Future<?> submit(@NotNull Runnable task) {  
        return super.submit(ThreadMdcUtil.wrap(task, MDC.getCopyOfContextMap()));  
    }  
  
    @NotNull  
    @Override    public <T> Future<T> submit(@NotNull Callable<T> task) {  
        return super.submit(ThreadMdcUtil.wrap(task, MDC.getCopyOfContextMap()));  
    }  
}
```

##### 3.1.3 配置线程池

```java
@SpringBootConfiguration  
public class ThreadPoolConfig {  
    /**  
     * 这里定义一个日志追踪线程池，使用时直接注入ThreadPoolTaskExecutor，使用@Qualifier("getLogTraceExecutor")指定bean  
     */    @Bean  
    public CustomThreadPoolTaskExecutor getLogTraceExecutor() {  
       //创建了一个ThreadPoolTaskExecutor的子类，在每次提交线程的时候都会做一些子类配置的操作  
        CustomThreadPoolTaskExecutor executor =new CustomThreadPoolTaskExecutor();  
        // 设置线程名称前缀  
        executor.setThreadNamePrefix("LogTraceExecutor-");  
        executor.initialize();  
        return executor;  
    }  
  
    /**  
     * 重写默认线程池配置，@Async异步会使用这个线程池  
     */  
    @Bean  
    public Executor taskExecutor() {  
        //创建了一个ThreadPoolTaskExecutor的子类，在每次提交线程的时候都会做一些子类配置的操作  
        ThreadPoolTaskExecutor executor = new CustomThreadPoolTaskExecutor();  
        // 设置线程名称前缀  
        executor.setThreadNamePrefix("logTranceAsyncPool-");  
        executor.initialize();  
        return executor;  
    }  
}
```

### 4、配置logbook pattern

注意：`[%X{traceId}]`

```yml
logging:  
  pattern:  
    console: ${CONSOLE_LOG_PATTERN:%clr(%d{${LOG_DATEFORMAT_PATTERN:yyyy-MM-dd HH:mm:ss.SSS}}){faint} %clr(${LOG_LEVEL_PATTERN:-%5p}) %clr(${PID:- }){magenta} %clr(---){faint} %clr([%15.15t]){faint} [%X{traceId}] %clr(%-40.40logger{39}){cyan} %clr(:){faint} %m%n${LOG_EXCEPTION_CONVERSION_WORD:%wEx}}
```

## 三、实际效果

![[日志追踪效果.png]]