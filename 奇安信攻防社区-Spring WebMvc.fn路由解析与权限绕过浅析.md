0x00 前言
=======

 `WebMvc.fn` 是 Spring Web MVC 的一部分，它提供了一种轻量级的函数式编程模型，允许开发者使用函数来定义路由和处理 HTTP 请求。这种模型是注解编程模型的替代方案。

 在 `WebMvc.fn` 中，HTTP 请求由 `HandlerFunction` 处理，这是一个接收 `ServerRequest` 并返回 `ServerResponse` 的函数。一般可以通过RouterFunctions.route()方法进行处理。例如下面的例子：

```Java
@Configuration
public class PersonHandlerConfiguration {

    @Bean
    public RouterFunction<ServerResponse> person() {
        return route().GET("/person", request -> {
            return ServerResponse.status(HttpStatus.OK).body("Hello World") ;
        }).build() ;
    }
}
```

 启动SpringApplication成功访问并解析相应的路由：

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-2e26fafe452c8704b7d5565dd9303e394a4b7178.png)

0x01 WebMvc.fn路由解析过程
====================

 在SpringMvc中，DispatcherServlet是前端控制器设计模式的实现,提供Spring Web MVC的集中访问点,而且负责职责的分派。同样的WebMvc.fn也是经过DispatcherServlet进行处理的。以2.5.8版本springboot为例，当请求`/person`接口时，看看具体的路由解析过程。

 当Spring MVC接收到请求时，Servlet容器会调用DispatcherServlet的service方法（方法的实现在其父类FrameworkServlet中定义）。根据不同的请求方法，会调用对应的processRequest方法，例如GET请求会调用doGet方法。然后在doDispatch方法进行处理，这里会顺序循环调用HandlerMapping的getHandler方法找到合适的HandlerMapping供后续解析处理（WebMvc.fn使用的是RouterFunctionMapping）：

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-0ed85ed1ce711133bbbbae2e984f9bb5747c9858.png)

 RouterFunctionMapping会根据传入的 `HttpServletRequest` 找到相应的处理器函数`HandlerFunction`。主要是在org.springframework.web.servlet.function.support.RouterFunctionMapping#getHandlerInternal方法进行处理：

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-511f815d1d8db3baa3ea5e646b2cf84a12314862.png)

 首先检查 `this.routerFunction` 是否不为 `null`。它定义了应用程序的路由规则，如果 `routerFunction` 存在，会使用 `HttpServletRequest` 创建一个 `ServerRequest` 对象：

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-e1a1d0506e37ae5475f1b0987fc1bd8c82fa6a74.png)

 然后使用 routerFunction.route(request) 来路由请求。这个方法尝试匹配 ServerRequest 与 RouterFunction 中定义的路由规则。如果找到匹配的路由，则返回一个包含 HandlerFunction 的 Optional 对象：

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-8a7525145277e0459f61d9b24119db7a358b2894.png)

 这里会调用org.springframework.web.servlet.function.RouterFunctionBuilder#route方法进行进一步的处理，这里遍历一系列的路由函数routerFunctions的route方法，尝试找到第一个与当前请求匹配的路由，并返回相应的处理器函数`HandlerFunction`：

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-bddf72ca073a8bd78d181ba8f80c5e24d41b4d6a.png)

 RequestPredicates里定义了请求的规则，包括请求参数、请求方式等。上述案例代码会调用DefaultRouterFunction#route方法进行处理，主要是通过 `this.predicate.test(request)` 来检查当前路由函数的`Predicate`是否匹配传入的 `ServerRequest` 对象：

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-9b34c22c9cdd0f574020c959134205c0cfd2f5b1.png)

 在test方法中，主要做了两个判断。一个是判断当前请求的方法是否在对应路由函数的方法集合中：

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-d9ffc3d9534f0888c04574424a262c0d1ca1db95.png)

 另一个是通过PathPattern的matchAndExtract方法进行请求路径的匹配：

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-24e07422d011d5f894b17a1ee4b0eba906c605e3.png)

 两者均匹配成功时，会得到具体的处理器函数`HandlerFunction`，此时即可执行对应的逻辑来进一步解析请求。

 上面的案例是只注册了一个RouterFunction，如果容器中有多个RouterFunction类型的bean时，DifferentComposedRouterFunction会逐层包装，将所有RouterFunction组合在一个对象中。然后调用DifferentComposedRouterFunction#route方法，在递归调用过程中，最终会调用到配置类中定义的BuiltRouterFunction#route重复上述的过程来获取目标Handler：

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-3a75bd748899ab74e9b75b8554f308f9be4cbef7.png)

 以上是WebMvc.fn大致的路由解析过程。

1.1 静态资源解析
----------

 除此之外，RouterFunctions可以通过PathResourceLookupFunction在响应式路由中处理静态资源请求。将请求的路径映射到文件系统中的资源，如图片、样式表或 JavaScript 文件等。

 解析时主要会调用ResourcesRouterFunction#route方法进行处理，实际调用的是org.springframework.web.servlet.function.PathResourceLookupFunction#apply方法：

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-a1ac9ed46ebb6dfee0038589e930bcafd00b2de6.png)

 首先会使用 this.pattern.matches(pathContainer) 检查请求路径是否与定义的路由模式匹配。如果不匹配，返回一个空的 Optional。如果路径匹配，使用 this.pattern.extractPathWithinPattern(pathContainer)提取模式内的路径部分:

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-a3d5f113106db4fe0dd80561942d49365ba40513.png)

 然后会调用 this.processPath(pathContainer.value()) 方法对路径进行进一步处理，若处理后的路径包含`%`，则进行URL解码：

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-12bec5354bfddbf52fc843e27c0ba58eab6e1862.png)

 processPath方法的主要作用是确保路径以正斜杠（`/`）开始，并且移除路径中不必要的部分：

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-880b1a1026f3c7f3ea7dae72079ee9676680f8ee.png)

 判断解码后的路径是否合法，若没问题则通过this.location.createRelative(path) 创建一个 Resource 对象，这个对象代表了请求的资源。然后检查创建的资源是否可读，并且是否位于定义的资源位置下。如果是，返回一个包含该资源的 Optional。否则，返回空：

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-bd97da321227a6654a1a4e9fe6d00e0c5da31fa4.png)

 首先是isInvalidPath方法，这个方法关注安全性，确保路径不会导致不安全的文件访问或路径遍历攻击：

- 检查路径字符串 `path` 是否包含 `WEB-INF` 或 `META-INF`，这些目录正常情况下不应该被外部可访问
- 使用 ResourceUtils.isUrl(relativePath) 检查相对路径是否是一个有效的 URL（类似classpath:的调用）。如果是，或者路径以 url: 开头，表明这是一个指向外部资源的路径。也是不合理的
- 检查路径中是否包含 `..`，这可能是尝试进行路径遍历的尝试。通过 `StringUtils.cleanPath(path)` 处理路径后会再次检查是否包含 `../`

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-fdc6044301e11761a7401e9baaf3c7ef3015d3e5.png)

 isResourceUnderLocation方法，核心是校验请求的资源是否位于定义的资源位置下，并且对对请求内容再次URL解码并检查是否包含`../`危险字符：

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-cf36edbc6d7677351172a8fc2b80af7f3f4d77ea.png)

 相比常见的RequestMappingHandlerMapping解析过程，RouterFunctionMapping会更简洁一些。下面看看在基于请求Path的鉴权场景下有什么潜在权限绕过的风险。

0x02 RouterFunctionMapping权限绕过浅析
================================

 根据前面的分析可知，获取请求路径的方式主要是`request.requestPath().pathWithinApplication`。在某些低版本还可能用到`request.pathContainer`,然后通过`PathContainer.parsePath`进行处理：

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-e2e2aa15e040555d0b14f307beeb942b6db95a79.png)

 最后在实际路径匹配时都会调用相应解析器的匹配方法进行处理，这点跟RequestMappingHandlerMapping是类似的。以PathPatternParser解析器为例，其会调用removeSemicolonContent移除URL路径中分号，同时进行r匹配时，会对路径中的URL编码进行解码操作。并且还会识别请求path尾部额外的/。也就是说，从路径匹配的角度，常规RequestMappingHandlerMapping解析时的一些权限绕过思路在RouterFunctionMapping也是适用的（`../`在请求中会被识别为非法请求，无需关注），以请求`/person`接口为例,可以看到基本的变形后路径仍可以正常解析并获取对应的资源：

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-dc21625ca1455eeb3de7ae9c41a72fa6f3f6c846.png)

 除此之外，前面还提到了，RouterFunctions可以通过PathResourceLookupFunction在响应式路由中处理静态资源请求。并且通过不同的Resolver还可以支持`file://`协议的使用。

 下面是一个实际的案例，在特定的场景下，可能需要对日志文件的访问进行严格控制，以确保只有授权的用户才能查看或下载日志文件。实际业务场景中通过`RouterFunction`来快速定义路由，并且通过filter的方式**对特定路径下特定后缀的文件进行了权限的控制**，可以看到下面通过file协议把tomcat的日志目录映射到了web路径中，可以通过`/log`接口就行访问：

```Java
@Configuration
public class LogRouter {

    ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();

    @Bean
    RouterFunction staticResourceLocator(){
        return RouterFunctions.resources("/log/**", resolver.getResource("file:///usr/local/tomcat/logs/"));
    }

}
```

 基于前面的分析，除了在RouterFunction配置的路径映射，还会对实际请求的资源路径做一些处理，在org.springframework.web.servlet.function.PathResourceLookupFunction#apply方法中，原始请求路径中包含`%`时，会进行一次URL解码：

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-b1379c80011e0c974ee5071c95808ae753d7535d.png)

 后续就是基础的安全检查，然后调用createRelative创建对应的resource，跟进对应的实现，主要是org.springframework.core.io.FileUrlResource:

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-124b981550a9c1da5dbbd7a41d41e49e6e2cc6ec.png)

 比较可惜的是，这里将path中的`#`替换成了对应的URL编码，此时就没法利用类似`file:///usr/local/tomcat/logs/catalina.yyyy-MM-dd.log#png`的方法来绕过类似`*.log`的后缀匹配了：

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-2695efa7b3e53e09b1ec14f111dbd25824bb2b01.png)

 除了相关的安全检查外，还有一个可读性的检查，主要是通过checkReadable方法实现的，如果是file协议的话，会获取实际的File对象，然后通过canRead方法进行判断：

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-f16ae4ad2f858b5150dab379e06bb47ddfe4cfba.png)

 继续跟进可以看到是通过org.springframework.util.ResourceUtils#getFile方法来获取协议中的文件路径的：

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-399eda939a24b7966431714bc31ee2a48e93d066.png)

 在对应的getFile方法中，实际上会通过URI对象的getSchemeSpecificPart方法再做一层解码：

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-39bfa74b195915dfea3c0ed208c605f5a0fc007e.png)

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-f34a954c6c7b54c7150dfde1091d32479f01ccf8.png)

 也就是说，若请求的文件路径是URL双重编码的，在实际判断时，也能通过对应的可读性判断，通过相关的安全检查也是没问题的（实际请求的内容没有类似`../`的穿越符，也没有读取配置资源以外的目录）。这里提供了一个思路，那能否通过URL双重编码的方式来请求实际的目录资源呢？

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-f7e4b072292153f1dcab3672a9c92329ef45746d.png)

 继续跟进对应的`HandlerFunction`解析，最终会通过org.springframework.core.io.UrlResource#getInputStream方法来获取对应的资源：

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-8b514db4fabd9dc8f8af48700ee5ce1019e998e7.png)

 在建立请求时可以看到，这里会再对请求路径解码一层，结合org.springframework.web.servlet.function.PathResourceLookupFunction#apply方法的解码，也就是说，在特定场景下，还以可以通过双重URL编码来绕过对应的权限控制：

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-7ef041df0068b3f2705b75da276317411493d3f4.png)

 映证对应的猜想，以读取catalina.out为例，可以看到通过双重URL编码的方式在非登陆状态下读取到了相关的文件内容：

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-5932be25603f69a4e64b88a6f219c0fb2176d5fe.png)

 除此以外，对于使用类似shiro/springsecurity等通用鉴权措施也是类似的，可以解析不同场景下解析差异，来达到绕过的效果。

0x03 其他
=======

 除了传统的Filter/intercept的鉴权方式以外，还可以通过使用路由函数生成器上的before、after或filter方法来进行处理，例如`before` 方法允许你在请求处理链的开始处插入逻辑。这通常用于执行一些前置操作，如认证、日志记录或请求预处理。

![image.png](https://shs3.b.qianxin.com/attack_forum/2024/09/attach-d9338d9f5cdd7c393894f1a2cd29fcf6d7d7915c.png)

 这类方法方便请求处理的不同阶段插入自定义逻辑。与此同时也绑定了对应的路由处理函数，避免了类似filter/interceptor基于path鉴权措施在特定场景下的绕过问题。此外，WebFlux也有WebFlux.fn的内容。具体实现也是类似的，在实际代码审计过程中可以关注下。