#  Spring MVC 



## Spring MVC 框架有什么用



Spring Web MVC 框架提供“模型-视图-控制器”( Model-View-Controller )架构和随时可用的组件，用于开发灵活且松散耦合的Web应用程序。

MVC 模式有助于分离应用程序的不同方面，如输入逻辑，业务逻辑和 UI 逻辑，同时在所有这些元素之间提供松散耦合。





## 介绍下 Spring MVC 的核心组件



Spring MVC 一共有九大核心组件，分别是：

- MultipartResolver
- LocaleResolver
- ThemeResolver
- HandlerMapping
- HandlerAdapter
- HandlerExceptionResolver
- RequestToViewNameTranslator
- ViewResolver
- FlashMapManager



## 描述一下 DispatcherServlet 的工作流程



DispatcherServlet 的工作流程可以用一幅图来说明：

![DispatcherServlet 的工作流程](Spring MVC.assets/15300766829012.jpg)



**但是 Spring MVC 的流程真的一定是酱紫么**

答案当然不是。对于目前主流的架构，前后端已经进行分离了，所以 Spring MVC 只负责 **M**odel 和 **C**ontroller 两块，而将 **V**iew 移交给了前端。所以，在上图中的步骤 ⑤ 和 ⑥ 两步，已经不在需要。

那么变成什么样了呢？在步骤 ③ 中，如果 Handler(Controller) 执行完后，如果判断方法有 `@ResponseBody` 注解，则直接将结果写回给用户( 浏览器 )。

但是 HTTP 是不支持返回 Java POJO 对象的，所以需要将结果使用 [HttpMessageConverter](http://svip.iocoder.cn/Spring-MVC/HandlerAdapter-5-HttpMessageConverter/) 进行转换后，才能返回。例如说，大家所熟悉的 [FastJsonHttpMessageConverter](https://github.com/alibaba/fastjson/wiki/在-Spring-中集成-Fastjson) ，将 POJO 转换成 JSON 字符串返回。



## WebApplicationContext

WebApplicationContext 是实现ApplicationContext接口的子类，专门为 WEB 应用准备的。

- 它允许从相对于 Web 根目录的路径中**加载配置文件**，**完成初始化 Spring MVC 组件的工作**。
- 从 WebApplicationContext 中，可以获取 ServletContext 引用，整个 Web 应用上下文对象将作为属性放置在 ServletContext 中，以便 Web 应用环境可以访问 Spring 上下文。





## Spring MVC 拦截器

**HandlerInterceptor是Spring MVC 框架的一部分，位于DispatcherServlet和我们的Controller之间。**我们可以在请求到达控制器之前以及视图渲染之前和之后拦截请求。

`org.springframework.web.servlet.HandlerInterceptor` ，拦截器接口。代码如下：

```java
// HandlerInterceptor.java

/**
 * 拦截处理器，在 {@link HandlerAdapter#handle(HttpServletRequest, HttpServletResponse, Object)} 执行之前
 */
default boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
		throws Exception {
	return true;
}

/**
 * 拦截处理器，在 {@link HandlerAdapter#handle(HttpServletRequest, HttpServletResponse, Object)} 执行成功之后
 */
default void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler,
		@Nullable ModelAndView modelAndView) throws Exception {
}

/**
 * 拦截处理器，在 {@link HandlerAdapter#handle(HttpServletRequest, HttpServletResponse, Object)} 执行完之后，无论成功还是失败
 *
 * 并且，只有该处理器 {@link #preHandle(HttpServletRequest, HttpServletResponse, Object)} 执行成功之后，才会被执行
 */
default void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler,
		@Nullable Exception ex) throws Exception {
}
```



一共有三个方法，分别为：

- `#preHandle(...)` 方法，调用 Controller 方法之**前**执行。
- `#postHandle(...)` 方法，调用 Controller 方法之**后**执行。
- `#afterCompletion(...)` 方法，处理完 Controller 方法返回结果之**后**执行。
  - 例如，页面渲染后。
  - **当然，要注意，无论调用 Controller 方法是否成功，都会执行**。





## Spring MVC 的拦截器可以做哪些事情

- 记录访问日志。
- 记录异常日志。
- CSRF防护
- 需要登陆的请求操作，拦截未登陆的用户。
- …







## Spring MVC 的拦截器和 Filter 过滤器有什么差别

让我们看一个图表，显示*Filter*和*HandlerInterceptor*在请求/响应流中的位置：

![1706545716048](Spring MVC.assets/1706545716048.png)

**过滤器在请求到达 DispatcherServlet 之前拦截请求，这使得它们非常适合粗粒度任务，**例如：

- 验证
- 日志记录和审计
- 图像和数据压缩
- 我们希望与 Spring MVC 解耦的任何功能

**另一方面，HandlerIntercepor拦截DispatcherServlet和我们的Controller之间的请求。**这是在 Spring MVC 框架内完成的，提供对*Handler*和*ModelAndView*对象的访问。这减少了重复并允许更细粒度的功能，例如：

- 处理横切问题，例如应用程序日志记录

- **详细的授权检查**（例如获取request的属性值）

- CSRF防护

  >
  >将CSRF（Cross-Site Request Forgery）防护放在`HandlerInterceptor`（Spring MVC的拦截器）中的主要优势在于它可以更好地与Spring MVC框架集成，并且对于Web应用的请求生命周期具有更细粒度的控制。以下是一些原因：
  >
  >1. **框架集成：** `HandlerInterceptor`是Spring MVC框架提供的一部分，它与Spring MVC的生命周期和组件紧密集成。这使得CSRF防护可以更容易地与其他框架特性（如控制器、处理器方法等）结合使用。
  >2. **请求生命周期控制：** `HandlerInterceptor`提供了在请求处理的不同阶段执行代码的能力，包括在处理器方法执行前、执行后，以及在视图渲染之前。这允许在请求的不同阶段进行CSRF防护的检查。
  >3. **与业务逻辑的集成：** CSRF防护通常涉及到在请求中包含令牌，验证令牌等操作。将这些逻辑集成到`HandlerInterceptor`中可以更容易地与具体的业务逻辑和控制器方法集成，而无需额外的层次结构。
  >4. **更高级别的抽象：** `HandlerInterceptor`提供了更高级别的抽象，可以直接访问Spring MVC中的请求、响应和其他相关对象。这使得CSRF防护的实现更加直观和方便。
  >5. **精准的路径匹配：** 拦截器可以通过配置指定要拦截的路径模式，使得CSRF防护只在特定的请求路径下生效，从而提高了精准度。
  >
  >当然，最终选择在何处实现CSRF防护取决于应用的需求和架构。在一些情况下，特别是当需要在整个Servlet容器范围内进行一般性的请求处理时，使用过滤器可能更合适。然而，对于Spring MVC应用，利用`HandlerInterceptor`的特性通常会更加方便。

- 操作 Spring 上下文或模型

**容器不同**：拦截器构建在 Spring MVC 体系中；Filter 构建在 Servlet 容器之上。

**使用便利性不同**：拦截器提供了三个方法，分别在不同的时机执行；过滤器仅提供一个方法，当然也能实现拦截器的执行时机的效果，就是麻烦一些。

我们会发现，拓展性好的框架，都会提供相应的拦截器或过滤器机制，方便的我们做一些拓展。例如：

- Dubbo 的 Filter 机制。
- Spring Cloud Gateway 的 Filter 机制。
- Struts2 的拦截器机制。

