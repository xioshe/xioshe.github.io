---
title: 如何封装接口返回结构？—— SpringBoot 实践
date: 2024-07-08 21:44:28
tags:
- SpringBoot
- API
categories:
- 技术实践
---

本文将讨论 API 接口返回结构的封装思路，并给出 SpringBoot 框架下的实践。

## 为什么要统一接口的返回结构？

调用 API 接口已经成了日常开发工作的一环，无论从事前端开发还是后端开发，或多或少会与 API 接口打交道。前后端分离、后端微服务化、SaaS，这些耳熟能详的名词，都涉及到了 API 调用。由于普遍使用 API，业界发展出了一些 API 规范，比如 RESTful、QraphQL。这些规范统一了接口风格，降低了接口的使用成本，目前已经成了主流。但这些接口规范都没有明确规定是否需要返回统一的结构，选择权在于开发者。

衡量接口是否需要返回相同的结构，可以从优劣两方面分析。

统一接口返回结构具有以下优势：

- 降低心智负担
- 降低前端开发难度
- 提高代码可维护性

统一的结构代表统一的模式，能显著降低心智负担。我们的大脑是一台懒惰的机器，它善于分析信息的差异，从而利用差异来处理信息，但不善于处理混乱无序的信息。RESTful 风格就包涵了统一模式的思想——从资源的角度看待数据，复用 HTTP 方法来表示对数据的操作。在这个统一模式下，拿到一组全新的接口，序员们也能快速分辨出各个接口大概的功能，从而提高工作效率。另一方面，统一的模式也能避免序员在开发接口时过度纠结于方法命名。与之类似，统一的响应结构也照顾了懒惰的大脑，使从接口响应中提取关键信息变得更加容易。一个从没使用过的 API，序员在拿到响应数据时，也能快速判断请求是否成功，推断出大致的失败原因。这就是统一模式带来的遍历。

所有前端开发者都不希望拿到风格迥异的 API。风格统一的接口更利于前端代码的封装和复用。现代工程化前端通常会使用 HTTP 客户端工具包来请求接口，比如 axios，并进行一定程度的封装。封装的一个方向是异常处理，根据接口的返回结果判断是否出现异常，进而采取统一的异常处理流程，不必在每次请求时单独处理。试想一下，有些接口用 status 属性表示异常状态，另一些则用 code 属性，甚至还有些接口使用 HTTP 状态码。这时候前端如何兼容所有接口就成了一个极大的挑战，没人会喜欢做这样的工作。

需要修改接口返回内容时，统一的结构能避免不少麻烦。比如需要调整错误码，基于统一的结构的代码可以集中处理，不必逐一检查每个接口。

**那么，代价呢？**

统一的接口返回结构主要有三个方面的弊端：

- **降低了接口的灵活性**：统一也意味着约束，开发者不能随意改变接口的结构，不得不戴着镣铐起舞。
- **增加了开发成本**：开发者需要编写更多的代码，来保证不管异常与否接口都能返回一致的结构。不过这个问题可以通过框架层面的封装来避免。
- **降低了代码可读性**：额外的处理逻辑意味着更高的代码阅读成本。这个问题也可以通过封装来避免。

RPC 似乎是一个特例，统一返回结构弊大于利。对于 RPC 接口而言，优势在于灵活的返回值结构和更高的性能。固定的返回结构会失去灵活性，更复杂的响应结构会影响性能。

## 封装时需要注意的细节

在封装接口返回结构的时候，有几个不得不考虑的细节。这些问题没有统一答案，我仅提出自己的观点。

### 是否应该复用 HTTP 错误码？

RESTful 接口规范提倡复用 HTTP 协议的 Status Code 作为接口状态，比如 4xx 代表客户端异常，5xx 代表服务端异常，优点在于可以统一处理一些通用异常类型。我最早是从[**《凤凰架构》**](https://icyfenix.cn/architect-perspective/general-architecture/api-style/rest.html)中看到这种说法，当时十分认同。但是，在具体实践中，我发现 SpringBoot 或者说 SpringMVC 修改 HTTP 状态码的代码比较繁琐，在接口发生异常时也很难统一处理。

后来我又看到另一种处理思路——明确区分 HTTP 状态码和业务状态码，凯撒的归凯撒，上帝的归上帝。HTTP 状态码代表的是技术层面的细节，而业务状态码代表了业务细节。如果一个属性既能表示技术又能表示业务，就是一种严重的耦合，这不利于代码的扩展。一种合适的做法是**将 HTTP 状态码和业务状态码分开**，由技术框架处理 HTTP Status Code，而开发者控制业务层面的状态码。

```json
200 OK
{
  "code": 404,
  "msg": "not found",
  "data": {
      ...
  }
}
```

### 接口是否应该返回单个字符串？

这属于接口风格层面的内容。建议接口统一返回 kv 形式的返回值，也就是对象或者 Map。优点在于风格统一，对前端比较友好，处理响应时不用考虑返回值是单字符串还是对象两种不同的情况。

### 是否封装没有返回值的接口？

返回 `void` 的接口对应的 HTTP 响应没有 ResponseBody，只能通过 HTTP 状态码判断接口是否正常。封装接口返回结构时，如果已经决定区分 HTTP 状态码和业务状态码，为了正确识别业务异常，需要对 `void` 接口的返回值进行包装，**即使不需要返回数据，也要返回业务状态码**。此时，可能出现下面这种情况，个人觉得可以接受。

```json
{
  "code": 200,
  "msg": "ok",
  "data": null
}
```

## 如何在 SpringBoot 中返回统一的接口结构？

目前常见的接口返回结构封装风格是  `code`、`msg`、`data` 三种属性，命名可能有区别，但内容相差无几。

- `code` 代表业务状态码，一般为**数字**。
- `msg` 是对状态码的简要描述，有时候状态码相同描述不同，可能需要考虑国际化的问题。
- `data` 代表接口返回值。**不建议用空对象代表 null**，不要把错误隐藏在盒子里面。

```json
{
  "code": 200,
  "msg": "ok",
  "data": {
      "id": 1,
      "name": "x12"
  }
}
```

在 Java 中，封装一个具有相关字段的类即可。为了便于管理错误类型，可以用枚举类或者静态常量类集中维护 code 和 msg。

```java
public enum ResultCode {

    SUCCESS(1, "success"),
    PARAM_INVALID(1001, "invalid parameter"),
    USER_NOT_EXIST(2004, "user not exist");

    private final Integer code;
    private final String message;

    ResultCode(Integer code, String message) {
        this.code = code;
        this.message = message;
    }

    public Integer getCode() {
        return code;
    }

    public String getMessage() {
        return message;
    }

}
```

**不建议使用接口取代静态常量类**，虽然很方便，但会破坏接口语义。

```java
@Data
public class Result implements Serializable {

    private final Integer code;
    private final String msg;
    private final Object data;

    public Result(ResultCode resultCode, Object data) {
        this.code = resultCode.getCode();
        this.msg = resultCode.getMessage();
        this.data = data;
    }

    public static Result success(Object data) {
        return new Result(ResultCode.SUCCESS, data);
    }

    public static Result failure(ResultCode resultCode) {
        return new Result(resultCode, null);
    }
}
```

**不建议将 `data` 的类型设计为泛型**。构建完 `Result` 后就直接发送出去了，不存在进一步的处理，类型参数没有任何意义。还会产生 `Result<Void>` 这种丑陋的东西。

使用时，需要在 Controller 类中利用 try-catch 分别包装正常结果和异常结果。

```java
@GetMapping
public Result something() {
    try {
        return Result.success(Map.of("word", somethingService.mayThrowException("any args")));
    } catch (Exception e) {
        log.error(e.getMessage(), e);
        return Result.failure(5000, e.getMessage());
    }
}
```

上述代码展示了一个简单的封装，通常这种简单封装会存在一些问题。

- 存在大量重复代码。

- 构建 Result 的代码增加了 Controller 的复杂性，降低了可读性。
- 有时需要将 Result 类型下降到 Service 层中，比如要在 Service 处理异常。这会导致 Service 层对 Controller 层的依赖，加深了代码耦合。
- 枚举类型的 ResultCode 不易扩展。

为了解决这些问题，我们需要更深层次的封装。

## 如何做的更好？

针对上述问题，有两个调整方向：

- 自动包装 Controller 方法返回值
- 自动包装异常

### 如何自动包装 Controller 的方法返回值？

自动包装方法返回值，代表不需要显式地在 Controller 层中构建 `Result` 对象，而是由框架将返回的对象转换为 `Result`。例如上面接口可以简化为下面的样子。

```java
// SomethingController
@WrappedResponse
@GetMapping
public Map<String, String> wrapSomething() {
    return Map.of("word", somethingService.mayThrowException("any args"));
}
```

要实现这个功能，可以使用 SpringMVC 提供的 `ResponseBodyAdvice` 接口。

`ResponseBodyAdvice` 作用于 SpringMVC 的请求处理流程，可以修改被 `@ResponseBody` 注解标记的 Controller 方法的返回值。该接口在返回值写入 `HttpServletResponse` 之前被调用。

![spring mvc doDispatch](https://img.prochase.top/bkimg/2024/07/bff7cb08e1dacd4ddbd71ad59d79e60d.png)

我们可以利用 `ResponseBodyAdvice` 来实现自动包装返回值。

上图第 7 步，对应 `ResponseBodyAdvice` 的处理流程。我们可以在此将方法返回值包装为 `Result` 对象。

上图第 8 步，将对象序列化并写入 `HttpServletResponse`。

`ResponseBodyAdvice` 接口定义了两个方法：

- `supports` 返回 true 才会执行 `beforeBodyWrite` 方法。
- `beforeBodyWrite` 将 Controller 方法的返回值作为参数传入，并返回新的对象。SpringMVC 会将新的返回值将作为结果写入 `HttpServletResponse` 中。

两个方法都有一个 `MethodParameter` 类型的参数 `returnType`。这是一个很有用的参数，代表 Controller 方法返回值的反射，在上面的例子中就是 `Map<String, String>` 的反射。通过 `returnType` 可以获取方法的反射（Method），进而获取方法名、方法参数列表、方法注解等信息，对应上面的例子就是 `wrapSomething`的参数和注解；还可以获取到方法所在类的反射，如此一来，就可以获得 `SomethingController` 类的各种信息。

为了控制自动包装的粒度，我们使用了 `@WrappedResponse` 注解。只有用该注解标记的方法才会被自动包装。通过 `returnType` 可以判断方法是否被注解标记。

```java
@ControllerAdvice
public class WrappedResponseAdvice implements ResponseBodyAdvice<Object> {

    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        // 忽略 Result 类型的方法
        var returnClazz = returnType.getParameterType();
        if (Result.class.isAssignableFrom(returnClazz)) {
            return false;
        }

        return returnType.hasMethodAnnotation(WrappedResponse.class)
               || returnType.getContainingClass().isAnnotationPresent(WrappedResponse.class);
    }

    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType, Class<? extends HttpMessageConverter<?>> selectedConverterType, ServerHttpRequest request, ServerHttpResponse response) {
        // 将 body 包装为 Result
        return Result.success(body);
    }
}
```

在 `supports` 方法中，先排除了返回 `Result` 类型的方法，然后检查方法或者方法所在类有没有被 `@WrappedResponse` 注解标记。

在 `beforeBodyWrite` 方法中，直接包装 `body`。注意，如果在上图第 4 步执行 Controller 的方法时抛出了异常，`DispatcherServlet` 会捕获并处理异常，不会继续执行第 5-8 步，因此这里不涉及异常处理的代码。关于 `DispatcherServlet` 类如何处理异常，下一小节会深入探讨。

在 `WrappedResponseAdvice` 类中，需要使用 `@ControllerAdvice` 标记。因为 **`ResponseBodyAdvice` 必须配合 `@ControllerAdvice` 一起使用**。Spring 容器会通过 `@ControllerAdvice` 注解来扫描并注册所有 `ResponseBodyAdvice` 对象。

![init ResponseBodyAdvice](https://img.prochase.top/bkimg/2024/07/0346b84bcb3a040f838a7f2aaceb1cfe.png)

`RequestMappingHandlerAdapter` 类会从容器中获取所有被`@ControllerAdvice` 标记的 bean（即使没有实现 `ResponseBodyAdvice` 接口），然后将 bean 传递给 `RequestResponseBodyMethodProcessor` 类。

`RequestResponseBodyMethodProcessor` 参与了请求处理流程，从所有的 `ControllerAdvice` 中筛选出 `ResponseBodyAdvice` 接口的实现类，然后调用 `beforeBodyWrite` 处理返回值。

### 如何自动处理异常？

现在让我们来把异常也包装成 `Result`。

SpringMVC 提供了默认的异常处理流程，会收集异常类型，以 JSON 的形式返回。默认的返回值结构如下：

```json
{
  "timestamp": "2024-07-08T12:42:33.873+00:00",
  "status": 500,
  "error": "Internal Server Error",
  "path": "/foo/oops"
}
```

这与我们定义的结构不一致，需要进行调整。

控制 SpringMVC 的异常返回结构可以使用 `@RestControllerAdvice` 和 `ExceptionHandler` 两个注解。

```java
@RestControllerAdvice
public class ExceptionHandlers {

    @ExceptionHandler(AutoWrappedException.class)
    public Result handleAutoWrappedException(AutoWrappedException e) {
        return Result.failure(e.getCode(), e.getMessage(), e.getData());
    }
}
```

上述代码中，我们定义了一个 `ExceptionHandler` 用于处理 `AutoWrappedException` 类型的异常。所有 `AutoWrappedException` 都会调用 `handleAutoWrappedException` 方法处理。我们可以定义多个 `ExceptionHandler`，为不同异常类型定义不同的处理方式。

这里我们定义一个最基础的异常，获取所有的异常，统一返回 `Result`。

```java
@ExceptionHandler(Throwable.class)
public Result handleException(Throwable e) {
    var msg = e.getMessage();
    if (msg == null) {
        msg = "未处理异常";
    }
    return Result.failure(5000, msg);
}
```

现在，同一个异常的返回结构为：

```json
{
  "code": 5000,
  "msg": "oops",
  "data": null
}
```

这基本满足要求。

**SpringMVC 异常处理的主要流程是什么样的？**

异常处理过程涉及到了三个关键类和两个注解。

![exception handler](https://img.prochase.top/bkimg/2024/07/adcd1ef301e34ed33bce7ab477004c89.png)

其中，两个注解为：

- `@ExceptionHandler` 注解标记的方法会被视作异常处理方法，与一个具体的异常类型绑定。
- `@RestControllerAdvice` 内部用 `@ControllerAdvice` 标记，这与上一小节 `ResponseBodyAdvice` 的初始化流程一致。`RestControllerAdvice` 多了一个 `@ResponseBody` 注解，这与在 `RestController` 一致，旨在提示 SpringMVC，这个方法的返回值不走视图渲染流程，而是直接序列化为 JSON 再写入请求的 `HttpServletResponse`。

三个关键类为：

- **`DispatcherServlet`** 是第一个关键类。SpringMVC 在 `DispatcherServlet` 类中统一处理请求处理流程中的异常。`DispatcherServlet` 类维护了一个 `HandlerExceptionResolver` 列表，执行初始化方法时会从 Spring 容器中获取 `HandlerExceptionResolver` 类型 bean。

- **`ExceptionHandlerExceptionResolver`** 是第二个关键类。SpringMVC 在 `WebMvcConfigurationSupport` 中声明 `ExceptionHandlerExceptionResolver` 。这个关键类在初始化时，会按如下顺序处理：
  1. 从容器中获取 `@ControllerAdvice` 注解标记的 bean，为每个 advice 创建一个 `ExceptionHandlerMethodResolver`。

  2. 用 Map 记录了 ControllerAdviceBean → ExceptionHandlerMethodResolver 的映射关系。

- **`ExceptionHandlerMethodResolver`** 是第三个关键类。在这个类内部，处理上一步中关联的 advice。
  1. 通过反射获取到 advice 中所有被 `@ExceptionHandler` 标记的方法，以及注解中指定的异常类型。
  2. 用一个 Map 维护 exception type → Method 的映射关系。这里的 Map 使用了 `LinkedHashMap`，用于维持异常处理的顺序。

处理异常时，会按照如下顺序处理。

1. 由 `DispatcherServlet` 统一处理，先从 resolvers 中筛选出一个合适的 `ExceptionHandlerMethodResolver`。

2. `ExceptionHandlerMethodResolver` 遍历自己的 Map，根据 Controller 类型找到合适的 `ExceptionHandlerMethodResolver`。

3. `ExceptionHandlerMethodResolver` 遍历自己的 Map，根据异常类型找到合适的 Method，然后通过反射处理异常。

**那么，问题是什么呢？**

上述封装仅仅提供了基础功能，与其他框架共同工作会存在一些问题。

- 需要为单一接口提供禁用异常处理的选项，否则接口返回值没有包装，异常却被包装了。这种不一致对接口调用者而言无疑很麻烦。
- Swagger 无法识别包装之后的结构，只能获取包装之前的结构。需要额外处理这个问题。
- 默认会将异常的 message 返回给前端。这种粗糙的做法存在安全问题，需要更细粒度的控制选项。

## 还可以做的更好吗？

为了将上面的简单玩具改造成生产可用的工具，还需要进一步完善。

### 从默认不包装到默认包装

默认处理所有接口的返回值，不再需要在 Controller 或方法上添加 `WrappedResponse` 注解。这将带来一些新的挑战。

- 需要为单一接口提供禁用选项。这对于一些要求返回结构的第三方 API 很有用。
- 需要包级别的禁用选项。
- 需要根据 url 禁用。配置 url 时支持通配符。
- 需要根据方法返回值类型禁用。
- 需要忽略指定路径，否则会影响 Spring Boot Actuator 的接口和 Swagger 的接口。

### 更灵活的异常处理

不需要开发者自己去注册 `ExceptionHandler`，这需要我们基于 SpringMVC 异常处理机制设计一个封装方案。

- 开发者可以为任何异常指定 code 和 msg，包括自定义的异常和来自依赖的异常。

为 code 和 msg 提供更灵活的设置方式。

- 可选择是否将异常信息写入 msg。

- 支持为断言 assert 抛出的异常指定 code 和 msg 内容。
- 支持为 Hibernate Validator 抛出的异常指定 code 和 msg 内容。

### 错误信息国际化

利用 Spring 的国际化功能，让 msg 的内容支持国际化。

有两种实现国际化的方式，可以采用 code -> i18n 的关联方式，也可以采用 msg -> i18n 的关联方式。建议使用 msg，因为可以为同一个 code 提供不同的文本。

具体做法是定义 code 和 msg 时，msg 为字符串的 message id，然后根据 message id 去国际化文件中查找并替换。这需要考虑查找的性能问题。

## 说了这么多，有没有现有的工具包？

GitHub 上有一个 [Graceful Response](https://github.com/feiniaojin/graceful-response) 项目，实现了上述功能。这个项目文档齐全，值得尝试。

实际上，这篇的思路就来自这个框架。在此之前，我只知道一些封装的方式，但直到了解这个框架之后，才第一次有了全局上的认识。

本文按照设计 -> 实践 -> 优化的顺序，介绍了如何在 SpringBoot 中统一封装接口返回结构。相关代码已经上传 [GitHub](https://github.com/xioshe/spring-boot-api-response)。
