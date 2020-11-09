# servlet容器运行流程

1. Servlet容器启动会扫描，当前应用里面每一个jar包的
   	**ServletContainerInitializer**的实现
2. 提供**ServletContainerInitializer**的实现类；
   	必须绑定在，==**META-INF/services/javax.servlet.ServletContainerInitializer**==
      	文件的内容就是ServletContainerInitializer实现类的全类名；



总结：容器在启动应用的时候，会扫描当前应用每一个jar包里面
META-INF/services/javax.servlet.ServletContainerInitializer
指定的实现类，启动并运行这个实现类的方法(onStartup())；**传入感兴趣的类型**；

ServletContainerInitializer；
@HandlesTypes；



servletContext可以注册tomcat三大组件:

```java
//容器启动的时候会将@HandlesTypes指定的这个类型下面的子类（实现类，子接口等）传递过来；
//传入感兴趣的类型；
@HandlesTypes(value={HelloService.class})
public class MyServletContainerInitializer implements ServletContainerInitializer {

	/**
	 * 应用启动的时候，会运行onStartup方法；
	 * 
	 * Set<Class<?>> arg0：感兴趣的类型的所有后代类型；
	 * ServletContext arg1:代表当前Web应用的ServletContext；一个Web应用一个ServletContext；
	 * 
	 * 1）、使用ServletContext注册Web组件（Servlet、Filter、Listener）
	 * 2）、使用编码的方式，在项目启动的时候给ServletContext里面添加组件；
	 * 		必须在项目启动的时候来添加；
	 * 		1）、ServletContainerInitializer得到的ServletContext；
	 * 		2）、ServletContextListener得到的ServletContext；
	 */
	@Override
	public void onStartup(Set<Class<?>> arg0, ServletContext sc) throws ServletException {
		// TODO Auto-generated method stub
		System.out.println("感兴趣的类型：");
		for (Class<?> claz : arg0) {
			System.out.println(claz);
		}
		
		//注册组件  ServletRegistration  
		ServletRegistration.Dynamic servlet = sc.addServlet("userServlet", new UserServlet());
		//配置servlet的映射信息
		servlet.addMapping("/user");
		
		
		//注册Listener
		sc.addListener(UserListener.class);
		
		//注册Filter  FilterRegistration
		FilterRegistration.Dynamic filter = sc.addFilter("userFilter", UserFilter.class);
		//配置Filter的映射信息
		filter.addMappingForUrlPatterns(EnumSet.of(DispatcherType.REQUEST), true, "/*");
		
	}

}
```

 内置tomcat已经有这个jar包了, 在项目打包的时候让它不打包这个jar包避免冲突:

```xml
<dependency>
  		<groupId>javax.servlet</groupId>
  		<artifactId>servlet-api</artifactId>
  		<version>3.0-alpha-1</version>
  		<scope>provided</scope>
  	</dependency>
```



# 注解的方式整合springMVC

1. 实现初始化器(随便实现一个), 在servlet容器启动的时候加载完spring的配置类和webMVC配置类

   ![image-20201101233418124](img/image-20201101233418124.png)

   

```java
//web容器启动的时候创建对象；调用方法来初始化容器以前前端控制器
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

	//获取根容器的配置类；（Spring的配置文件）   父容器；
	@Override
	protected Class<?>[] getRootConfigClasses() {
		// TODO Auto-generated method stub
		return new Class<?>[]{RootConfig.class};
	}

	//获取web容器的配置类（SpringMVC配置文件）  子容器；
	@Override
	protected Class<?>[] getServletConfigClasses() {
		// TODO Auto-generated method stub
		return new Class<?>[]{AppConfig.class};
	}

	//获取DispatcherServlet的映射信息
	//  /：拦截所有请求, 较给springMVC处理（包括静态资源（xx.js,xx.png）），但是不包括*.jsp；
	//  /*：拦截所有请求；连*.jsp页面都拦截；jsp页面是tomcat的jsp引擎解析的；
	@Override
	protected String[] getServletMappings() {
		// TODO Auto-generated method stub
		return new String[]{"/"};
	}

}

```



webMVC配置类:

```java
//SpringMVC只扫描Controller；子容器
//useDefaultFilters=false 禁用默认的过滤规则；
@ComponentScan(value="com.atguigu",includeFilters={
		@Filter(type=FilterType.ANNOTATION,classes={Controller.class})
},useDefaultFilters=false)
@EnableWebMvc
public class AppConfig  extends WebMvcConfigurerAdapter  {

	//定制
	
	//视图解析器
	@Override
	public void configureViewResolvers(ViewResolverRegistry registry) {
		// TODO Auto-generated method stub
		//默认所有的页面都从 /WEB-INF/ xxx .jsp
		//registry.jsp();
		registry.jsp("/WEB-INF/views/", ".jsp");
	}
	
	//静态资源访问
	@Override
	public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
		// TODO Auto-generated method stub
		configurer.enable();
	}
	
	//拦截器
	@Override
	public void addInterceptors(InterceptorRegistry registry) {
		// TODO Auto-generated method stub
		//super.addInterceptors(registry);
		registry.addInterceptor(new MyFirstInterceptor()).addPathPatterns("/**");
	}

}

```



spring配置类:

```java
//Spring的容器不扫描controller;父容器
@ComponentScan(value="com.atguigu",excludeFilters={
		@Filter(type=FilterType.ANNOTATION,classes={Controller.class})
})
public class RootConfig {

}
```

---



#  springMVC的定制

1. @EnableWebMvc:开启SpringMVC定制配置功能；
   	<mvc:annotation-driven/>；

2. 配置组件（视图解析器、视图映射、静态资源映射、拦截器。。。）
   	extends WebMvcConfigurerAdapter(或其实现类)

   ```java
    * Copyright 2002-2016 the original author or authors.
   
   package org.springframework.web.servlet.config.annotation;
   
   import java.util.List;
   
   import org.springframework.format.FormatterRegistry;
   import org.springframework.http.converter.HttpMessageConverter;
   import org.springframework.validation.MessageCodesResolver;
   import org.springframework.validation.Validator;
   import org.springframework.web.method.support.HandlerMethodArgumentResolver;
   import org.springframework.web.method.support.HandlerMethodReturnValueHandler;
   import org.springframework.web.servlet.HandlerExceptionResolver;
   
   /**
    * An implementation of {@link WebMvcConfigurer} with empty methods allowing
    * subclasses to override only the methods they're interested in.
    *
    * @author Rossen Stoyanchev
    * @since 3.1
    */
   public abstract class WebMvcConfigurerAdapter implements WebMvcConfigurer {
   
   	/**
   	 * {@inheritDoc}
   	 * <p>This implementation is empty.
   	 */
   	@Override
   	public void configurePathMatch(PathMatchConfigurer configurer) {
   	}
   
   	/**
   	 * {@inheritDoc}
   	 * <p>This implementation is empty.
   	 */
   	@Override
   	public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
   	}
   
   	/**
   	 * {@inheritDoc}
   	 * <p>This implementation is empty.
   	 */
   	@Override
   	public void configureAsyncSupport(AsyncSupportConfigurer configurer) {
   	}
   
   	/**
   	 * {@inheritDoc}
   	 * <p>This implementation is empty.
   	 */
   	@Override
   	public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
   	}
   
   	/**
   	 * {@inheritDoc}
   	 * <p>This implementation is empty.
   	 */
   	@Override
   	public void addFormatters(FormatterRegistry registry) {
   	}
   
   	/**
   	 * {@inheritDoc}
   	 * <p>This implementation is empty.
   	 */
   	@Override
   	public void addInterceptors(InterceptorRegistry registry) {
   	}
   
   	/**
   	 * {@inheritDoc}
   	 * <p>This implementation is empty.
   	 */
   	@Override
   	public void addResourceHandlers(ResourceHandlerRegistry registry) {
   	}
   
   	/**
   	 * {@inheritDoc}
   	 * <p>This implementation is empty.
   	 */
   	@Override
   	public void addCorsMappings(CorsRegistry registry) {
   	}
   
   	/**
   	 * {@inheritDoc}
   	 * <p>This implementation is empty.
   	 */
   	@Override
   	public void addViewControllers(ViewControllerRegistry registry) {
   	}
   
   	/**
   	 * {@inheritDoc}
   	 * <p>This implementation is empty.
   	 */
   	@Override
   	public void configureViewResolvers(ViewResolverRegistry registry) {
   	}
   
   	/**
   	 * {@inheritDoc}
   	 * <p>This implementation is empty.
   	 */
   	@Override
   	public void addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
   	}
   
   	/**
   	 * {@inheritDoc}
   	 * <p>This implementation is empty.
   	 */
   	@Override
   	public void addReturnValueHandlers(List<HandlerMethodReturnValueHandler> returnValueHandlers) {
   	}
   
   	/**
   	 * {@inheritDoc}
   	 * <p>This implementation is empty.
   	 */
   	@Override
   	public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
   	}
   
   	/**
   	 * {@inheritDoc}
   	 * <p>This implementation is empty.
   	 */
   	@Override
   	public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
   	}
   
   	/**
   	 * {@inheritDoc}
   	 * <p>This implementation is empty.
   	 */
   	@Override
   	public void configureHandlerExceptionResolvers(List<HandlerExceptionResolver> exceptionResolvers) {
   	}
   
   	/**
   	 * {@inheritDoc}
   	 * <p>This implementation is empty.
   	 */
   	@Override
   	public void extendHandlerExceptionResolvers(List<HandlerExceptionResolver> exceptionResolvers) {
   	}
   
   	/**
   	 * {@inheritDoc}
   	 * <p>This implementation returns {@code null}.
   	 */
   	@Override
   	public Validator getValidator() {
   		return null;
   	}
   
   	/**
   	 * {@inheritDoc}
   	 * <p>This implementation returns {@code null}.
   	 */
   	@Override
   	public MessageCodesResolver getMessageCodesResolver() {
   		return null;
   	}
   
   }
   
   ```

   ---

   

   # Servlet异步请求

   ![image-20201102103630413](img/image-20201102103630413.png)

   1. 要支持异步的Servlet, 在Servlet进行映射的时候就要使Servlet支持异步==(**asyncSupported**):==

   ```java
   @WebServlet(value="/async",asyncSupported=true)
   public class HelloAsyncServlet extends HttpServlet {
   ```

   2. 开启异步模式:  

      **AsyncContext startAsync = req.startAsync();**   

      **//3、业务逻辑进行异步处理;开始异步处理**

      **startAsync.start(new Runnable() {**

      **@Override**
      			**public void run() {**
      				**try {**

      ​				==startAsync.complete();==

      ​              	//获取到异步上下文

      ​					AsyncContext asyncContext = req.getAsyncContext();

      ​					//4、获取响应

      ​					==ServletResponse response = asyncContext.getResponse();==

      ​					response.getWriter().write("hello async...");

      ​					**}**

      ​			**}**

      **}**

      ```java
      @Override
      	protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
      		//1、支持异步处理asyncSupported=true
      		//2、开启异步模式
      		System.out.println("主线程开始。。。"+Thread.currentThread()+"==>"+System.currentTimeMillis());
      		AsyncContext startAsync = req.startAsync();
      		
      		//3、业务逻辑进行异步处理;开始异步处理
      		startAsync.start(new Runnable() {
      			@Override
      			public void run() {
      				try {
      					System.out.println("副线程开始。。。"+Thread.currentThread()+"==>"+System.currentTimeMillis());
      					sayHello();
      					startAsync.complete();
      					//获取到异步上下文
      					AsyncContext asyncContext = req.getAsyncContext();
      					//4、获取响应
      					ServletResponse response = asyncContext.getResponse();
      					response.getWriter().write("hello async...");
      					System.out.println("副线程结束。。。"+Thread.currentThread()+"==>"+System.currentTimeMillis());
      				} catch (Exception e) {
      				}
      			}
      		});		
      		System.out.println("主线程结束。。。"+Thread.currentThread()+"==>"+System.currentTimeMillis());
      ```



# SpringMVC使用异步线程池

Controller返回值写成==**Callable**==类型或者==**DeferredResult**==类型

## 方式一:  Callable

```java
@ResponseBody
	@RequestMapping("/async01")
	public Callable<String> async01(){
        
		Callable<String> callable = new Callable<String>() {
			@Override
			public String call() throws Exception {
				Thread.sleep(2000);
				return "Callable<String> async01()";
			}
		};
		return callable;
	}
```

工作流程:

```java
/**
	 * 1、控制器返回Callable
	 * 2、Spring异步处理，将Callable 提交到 TaskExecutor 使用一个隔离的线程进行执行
	 * 3、DispatcherServlet和所有的Filter退出web容器的线程，但是response 保持打开状态；
	 * 4、Callable返回结果，SpringMVC将请求重新派发给容器，恢复之前的处理；
	 * 5、根据Callable返回的结果。SpringMVC继续进行视图渲染流程等（从收请求-视图渲染）。
	 * 
	 * preHandle.../springmvc-annotation/async01
		主线程开始...Thread[http-bio-8081-exec-3,5,main]==>1513932494700
		主线程结束...Thread[http-bio-8081-exec-3,5,main]==>1513932494700
		=========DispatcherServlet及所有的Filter退出线程============================
		
		================等待Callable执行==========
		副线程开始...Thread[MvcAsync1,5,main]==>1513932494707
		副线程开始...Thread[MvcAsync1,5,main]==>1513932496708
		================Callable执行完成==========
		
		================再次收到之前重发过来的请求========
		preHandle.../springmvc-annotation/async01
		postHandle...（Callable的之前的返回值就是目标方法的返回值）
		afterCompletion...
		
**/
```



## 方式二: DeferredResult

应用场景:

![image-20201102110813101](img/image-20201102110813101.png)

要给DeferredResult对象==**设置result**==才会结束业务线程, controller才会得到响应, ==**设置的对象就是controller的返回结果**==:

```java
@ResponseBody
	@RequestMapping("/createOrder")
	public DeferredResult<Object> createOrder(){
        //deferredResult消息中间件的模拟
		DeferredResult<Object> deferredResult = new DeferredResult<>((long)3000, "create fail...");
			//一般情况下其他模块会使用deferredResult对象来设置result, 消息中间件一收到消息就会通知controller继续执行
			deferredResult.setResult('订单号');
		
		return deferredResult;
	}
```





# 缓存

**概念**:

![image-20201102154728740](img/image-20201102154728740.png)





**原理**:

![image-20201103233023447](img/image-20201103233023447.png)







![image-20201103233826168](img/image-20201103233826168.png)





**缓存注解支持使用的spel表达式:**

![image-20201104000044494](img/image-20201104000044494.png)





## 用法:

1. 引入缓存依赖:

   ```xml
    		<dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-cache</artifactId>
           </dependency>
   ```

2. 开启基于注解的缓存方式:

   `@EnableCaching`

3. 使用缓存:

   ```java
    //当参数的id大于0的时候缓存才生效, 如果第一个参数的值等于2缓存不生效
       @Cacheable(/*cacheNames = {"emp"},*/ /*key = "#root.methodName+'['+#id+']'"*/ /*keyGenerator = "myKeyGenerator", *//*condition = "#id > 0", unless = "#a0 == 2"*/)
       public Employee getEmp(Integer id) {
           System.out.println("查询" + id + "号员工");
           Employee emp = employeeMappper.getEmpById(id);
           return emp;
       }
   ```

   

## 原理

```java
	/*
     * 将方法的运行结果进行缓存, 以后再要相同的数据, 直接从缓存中获取, 不用调用方法
     *
     * CacheManager管理多个Cache组件, 对缓存真正的CRUD操作在Cache组件中, 每一个缓存组件都有自己的唯一一个名字
     * 几个属性:
     *   cacheNames/value: 指定缓存组件的名字; 将方法的返回结果放在哪个缓存中, 是数组的方式, 可以指定多个缓存!
     *   key: 缓存数据使用的key, 可以用它来指定, 默认是使用方法参数的值
     *       编写SpEL: #id; 参数id的值   #a0  #p0  #root.args[0]都是表示方法的第一个参数的值
     *           getEmp[2]
     *   keyGenerator: key的生成器, 可以自己指定key的生成器的组件id
     *       key/keyGenerator: 二选一使用
     *   CacheManager: 指定缓存管理器, 或者cacheResolver指定获取解析器
     *
     *   condition: 指定符合条件下的情况下才缓存, 例如: condition="#id>0"
     *
     *   unless: 否定缓存, 当cunless指定的条件为true, 方法的返回值就不会被缓存, 可以获取到结果进行判断
     *       unless="#result==null"
     *
     *   sync: 是否异步? 异步模式的时候unless不支持
     * */


    /*
    * 原理:
    *   1. 自动配置原理:CacheAutoConfiguration
    *   2. 缓存的配置类:
    *   0 = "org.springframework.boot.autoconfigure.cache.GenericCacheConfiguration"
        1 = "org.springframework.boot.autoconfigure.cache.JCacheCacheConfiguration"
        2 = "org.springframework.boot.autoconfigure.cache.EhCacheCacheConfiguration"
        3 = "org.springframework.boot.autoconfigure.cache.HazelcastCacheConfiguration"
        4 = "org.springframework.boot.autoconfigure.cache.InfinispanCacheConfiguration"
        5 = "org.springframework.boot.autoconfigure.cache.CouchbaseCacheConfiguration"
        6 = "org.springframework.boot.autoconfigure.cache.RedisCacheConfiguration"
        7 = "org.springframework.boot.autoconfigure.cache.CaffeineCacheConfiguration"
        8 = "org.springframework.boot.autoconfigure.cache.SimpleCacheConfiguration"[默认开启]
        9 = "org.springframework.boot.autoconfigure.cache.NoOpCacheConfiguration"
            可以使用debug=true查看哪个缓存配置生效

         * 3. 那个默认配置类生效? SimpleCacheConfiguration
         * 4. 给容器中注册了一个CacheManager: ConcurrentMapCacheManager
         * 5. 可以获取和创建ConcurrentMapCache类型的缓存组件; 他的作用将数据保存在ConcurrentMap中
         *
         *  运行流程:
         *  @cacheable:
         *      1. 方法运行之前, 先去查询Cache(缓存组件), 按照cacheNames指定的名字获取;
         *          (CachManager先获取对应的缓存), 第一次获取缓存如果没有cache组件会自动创建.
         *      2. 去cache中查找缓存的内容, 使用一个key, 默认就是方法的参数;
         *              key是按照某种策略生成的, 默认是使用keyGenerator生成的, 默认使用SimpleKeyGenerator生成key
         *      3. 没有查到缓存就调用目标方法
         *      4. 目标方法返回的结果放进缓存中
         *
         *
         *      @Cacheable标注的方法执行之前先来检查缓存中有没有这个数据, 默认按照参数的值作为key去查询查询缓存
         *       如果没有就运行方法, 并将返回结果放进缓存
         *
         *  核心:
         *      1. 使用CacheManager[ConcurrentMapCacheManager]按照名字得到Cache[ConcurrentMapCache]组件
         *      2. key使用keyGenerator生成的, 默认是SimpleKeyGenerator
         *
         *
         *
         * */
```



## 键的生成话策略

实现KeyGenerator或者其子类, 并且添加到容器中:

```java
@Configuration
public class MyCacheConfig {
    @Bean(name = "myKeyGenerator")
    public KeyGenerator myKeyGenerator() {
        return new KeyGenerator() {
            /**
             * @param o 目标类
             * @param method 目标方法
             * @param objects 目标方法参数
             * @return 返回的值作为键
             */
            @Override
            public Object generate(Object o, Method method, Object... objects) {
                return method.getName() + "[" + Arrays.asList(objects).toString() + "]";
            }
        };

    }
}
```



## redis使用Jackson2JsonRedisSerializer序列化器进行缓存

```

```
