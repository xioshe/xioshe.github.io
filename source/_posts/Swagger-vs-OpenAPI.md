---
title: API 文档管理：Swagger OpenAPI 傻傻分不清楚
date: 2024-09-21 19:36:05
tags:
- Spring Boot
- API
categories:
- 技术实践
---

API 文档是项目开发实践中极其重要的一个环节。借助接口文档，前端与后端，服务与服务，接口的结构和功能一目了然，交流成本显著降低。

市面上有很多 HTTP API 文档管理工具，但最便捷的应该是 Swagger。不但能管理接口，还提供了测试接口的功能；纯 Web 页面，无需打开客户端。就个人体验而言，Swagger 可以显著提高开发效率。

本文将按照从概念到实践的方式，讨论如何用 Swagger 来管理 Spring Boot 项目中的 HTTP API。

## Swagger？OpenAPI？

一直以来，关于 Swagger 的许多名词都让我困惑。Swagger 是什么？跟 Swagger UI 是什么关系？OpenAPI 又是什么？什么 Springfox Springdoc，这些又是什么？即使不明白，也不影响基本使用，偶尔会在扩展功能时有些问题，通过搜索引擎也很容易找到解决方案。但如果厘清了这些概念，就可以从更深层次理解 Swagger 的结构，从而减少使用时的困惑。

首先，Swagger 是一个专注 API 文档管理的项目，核心是 Swagger 规范。Swagger 规范定义了 HTTP API 的结构化表示形式，包括 API 路径、参数、响应结构、名称、备注等方方面面。简单来说，有了 Swagger 规范，我们就可以用 JSON 和 YAML 来表示 API。

Swagger 规范经历了两个大版本，1.0 和 2.0。到 3.0 时，改名为 OpenAPI 这个更标准化的名称。所以 OpenAPI 实际上就是新版的 Swagger 规范。我们可能会看到 OpenAPI 3.0 的写法，这并不意味着 OpenAPI 有三个版本，而是指 OpenAPI 的第一个版本就是 3.0。需要注意的是，OpenAPI 修改了一些内部名词，与 Swagger 2.0 不兼容。因此 OpenAPI 不支持使用 Swagger 2.0 的老项目。

基于接口定义规范，Swagger 项目提供了一系列工具用于辅助 API 文档管理。

最常用的是 Swagger UI，这是一个前端 Web 页面，用于展示 API 文档，只要数据符合 Swagger 规范，就能正确显示。此外，Swagger UI 还提供了接口测试的功能，在页面上就能调用接口，简单又便捷。我们可以遵循 OpenAPI 规范，直接在 Yaml 文件中定义 API，然后通过接口暴露给 Swagger UI，就能在使用 Swagger UI 显示 API 文档和测试 API。但这不仅繁琐，还容易出错。我们还有更好的选择。

另一个常用的 Swagger 工具是 Swagger Core，这是一个 Java jar 包，提供了 Swagger 规范对应的 Java 对象类，以及相应的注解类。在 Swagger 2.0 中常用的 `@Api` `@ApiModel` `@ApiOperation` 就出自这个包。同时，Swagger Core 也支持最新的 OpenAPI 规范。有了 Swagger Core，就能直接用注解定义 API，从注解中提取 API 文档信息，提供给 Swagger UI，而不是自己手动编写 Yaml 文件，显然更加简单。

Swagger Core 只提供了定义数据的功能，采集接口数据需要额外的库。在 Sping 生态中，可以使用 SpringFox 和 Springdoc。两者功能相近，都支持从根据注解采集 Swagger 接口定义数据，且都是独立开源项目，不隶属于 Swagger 和 Spring 的任何一方。两个库各有限制。 SpringFox 支持 Swagger 2 版本，但已经不再维护，不支持 OpenAPI，也不支持最新的 Spring Boot 3。Springdoc 只支持最新的 OpenAPI 规范，支持所有的 Spring Boot 版本。

![img](https://img.prochase.top/bkimg/2024/09/324517aa1899117f2946d02c2e92ff9c.png)

Swagger 专注于定义规范（OpenAPI、Swagger Core）和展示数据（Swagger UI），并不负责采集数据。可以通过 Springfox 或 Springdoc 来采集 Swagger 数据。选择的依据是 Spring Boot 的版本和 OpenAPI 的版本。

- 在 Spring Boot 3 项目中，只能选择 Springdoc，因此只能使用 OpenAPI 3.0。

- 在 Spring Boot 2.x/1.x 项目中，如果需要兼容 Swagger 2.0，只能选择 Springfox。

- 其他时��，Springdoc 是更好的选择。

如果不想用默认的 Swagger UI，GitHub 上有很多第三方 swagger-ui 项目。在 Chrome 浏览器商店，甚至还有一个 Swagger-UI 的扩展，扩展了更换主题的功能。

## 与 Spring Boot 3 共舞

现在进入实践部分，以 Spring Boot 3 为例，介绍如何使用 Springdoc 来管理 API 文档。

首先，引入 Springdoc 依赖。

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.6.0</version>
</dependency>
```

这个依赖会同时引入 `springdoc-openapi-starter-webmvc-ui` 和 `springdoc-openapi-starter-webmvc-api` 的依赖，前者封装了 Swagger UI，后者用于生成 OpenAPI 数据，内部引用 Swagger Core。

![Architecture](https://img.prochase.top/bkimg/2024/09/8813238f4da1ddcc296115ec5361d997.png)

通过引入 springdoc-openapi-starter-webmvc-ui 依赖，提供以下功能：

- 对 OpenAPI 注解的支持；
- 内嵌 Swagger UI 页面，默认地址是 `/swagger-ui.html`，同时将 META-INF/resources/webjars/swagger-ui/ 目录下的静态资源文件通过 `/swagger-ui/*` 对外开放；
- 提供 `/v3/api-docs` 接口，用于访问 OpenAPI 3.0 规范的 JSON 格式的 API 文档；
- 通过 `/v3/api-docs/swagger-config` 访问 Swagger UI 的配置。

如果项目使用了 Spring Security，则需要调整配置，不拦截 Swagger 相关的接口，否则 Swagger UI 页面无法正常工作。

```java
@EnableWebSecurity
@Configuration
public class SecurityConfig {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http.authorizeHttpRequests(auth -> {
                    auth.requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll();
                    auth.anyRequest().authenticated();
                })
                // ... 其他配置
                .build();
    }
}
```

此时启动项目，就可以通过 `http://localhost:8080/swagger-ui.html` 或 `http://localhost:8080/swagger-ui/index.html` 访问 Swagger UI 页面，查看 API 文档。

![image-20240921155805160](https://img.prochase.top/bkimg/2024/09/a180a1e1e3e7b9ed8cce965448091709.png)

Springdoc 提供了许多配置项，可以对 Swagger UI 进行配置。其中，`springdoc.swagger-ui.` 前缀的设置用于调整 Swagger UI，其他则是数据相关的配置。

```yaml
springdoc:
  swagger-ui:
    display-request-duration: true # 显示接口响应时间
    path: doc.html # 修改默认访问路径 swagger-ui.html 为 doc.html
    use-root-path: true # 使用根路径 / 访问 Swagger UI
  group-configs:
    - group: 'default' # 定义一个分组，名称为 default
      paths-to-match: '/**' # 匹配所有接口
```

启用 use-root-path 后，访问 Swagger UI 的地址为 `http://localhost:8080/`，这对于纯后端项目来说，非常方便。

更多配置项可以参考 [Springdoc 官方文档#5. Springdoc-openapi Properties](https://springdoc.org/#properties)。

### 多分组配置

在上面例子中，我们启用了一个默认分组，匹配所有接口。Springdoc 还支持对不同分组进行配置，不同分组的接口会显示在不同的 Swagger UI 页面上。

![image-20240921160112421](https://img.prochase.top/bkimg/2024/09/79809d94137a6a4661ebc486a0d4c2e4.png)

可以直接在配置文件中配置多个分组，也可以在代码中动态配置。

```java
@Bean
public GroupedOpenApi publicApi() {
    return GroupedOpenApi.builder()
        .group("public")
        .pathsToMatch("/public/**")
        .build();
}

@Bean
public GroupedOpenApi adminApi() {
    return GroupedOpenApi.builder()
        .group("admin")
        .pathsToMatch("/admin/**")
        .build();
}
```

添加了两个分组，`public` 和 `admin`，分别匹配 `/public/**` 和 `/admin/**` 路径。除了路径匹配，还可以通过 `tags` 配置分组名称，通过 `packagesToScan` 配置包扫描路径。

### 接口认证

在 OpenAPI 3.0 规范中，支持多种认证方式，包括 API Key、HTTP Basic、JWT、OAuth2 等，可以根据需要选择合适的认证方式。

比如，使用 JWT 认证，需要添加以下配置：

```java
@Configuration
@SecurityScheme(
        name = "bearerAuth",
        type = SecuritySchemeType.HTTP,
        scheme = "bearer",
        bearerFormat = "JWT"
)
@RequiredArgsConstructor
public class SwaggerSpringdocConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
                .addSecurityItem(new SecurityRequirement().addList("bearerAuth"));
    }
}

```

通过 `@SecurityScheme` 注解，定义了一个名为 `bearerAuth` 的认证组件，使用 JWT 认证。然后在全局配置中，通过 `addSecurityItem` 方法，启用 `bearerAuth` 作为全局认证组件。

全局启用 bearerAuth 认证组件后，想指定某个接口不启用认证，可以通过 `@SecurityRequirement` 注解，将接口从全局认证组件中排除。

如果没有启用全局认证，则需要通过 `@SecurityRequirement` 注解，在接口上指定认证组件。

```java
@GetMapping("/public")
@SecurityRequirement(name = "bearerAuth")
public String public() {
    return "public";
}
```

Springdoc 还支持更灵活的扩展方式，通过实现 `OpenApiCustomiser` 接口，可以对 OpenAPI 进行细粒度的调整。

```java
@Bean
public OpenApiCustomizer consumerTypeHeaderOpenAPICustomizer() {
    return openApi -> openApi.getPaths().values().stream()
            .flatMap(pathItem -> pathItem.readOperations().stream())
            .forEach(operation -> operation.addParametersItem(
                    new HeaderParameter().$ref("#/components/parameters/myConsumerTypeHeader")));
}
```

上述代码在所有接口中添加了一个名为 `myConsumerTypeHeader` 的请求头参数，`"#/components/parameters/myConsumerTypeHeader"` 是参数定义信息，可以用与上面 bearerAuth 相同方式定义。

## 第三方扩展

开源的 Swagger 扩展项目中，知名度比较高的是 [xiaoymin/knife4j](https://github.com/xiaoymin/knife4j)。这个项目最早是一个第三方 Swagger UI 项目，后面逐渐扩展了许多新功能。其中，比较有特色的是自定义文档，可以添加自己的 Markdown 文档，丰富接口文档内容。以及导出离线文档功能，可以将接口文档导出为 Markdown、HTML、Word、OpenAPI（JSON） 格式的文件。

Knife4j 内部使用了 Springdoc 和 SpringFox，既可以通过 Springdoc 使用 OpenAPI 风格注解，也可以通过 Springfox 使用 Swagger 2.0 风格注解，可以按需选择。

我在使用的时候，还是发现了一些问题。

首先，文档维护也不够及时，分类也很乱。实际使用时，解决问题的成本很高。

其次，UI 页面路径为 `/doc.html`，由 Spring Boot 默认的 Resource Handler 来处理，没有提供自定义选项。这在项目将根目录 `/` 作为一个接口路径时会出现问题，因为 Controller 中定义的 API 优先级高于默认的 Resource Handler，`/doc.html` 路径会被 `/{parameter}` 这个接口解析，导致 404 错误。针对这个问题，可以通过自定义 Resource Handler 的方式解决。将 swagger 页面和静态资源映射到 `/dev/` 路径下，避免与接口路径冲突。

```java
@Configuration
public class ResourceHandlerConfig implements WebMvcConfigurer {
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/dev/doc.html")
                .addResourceLocations("classpath:/META-INF/resources/");
        registry.addResourceHandler("/dev/webjars/**")
                .addResourceLocations("classpath:/META-INF/resources/webjars/");
    }
}
```

与之相对，Springdoc 实现时注册了新的 Resource Handler，将页面相关的静态资源映射到 `/swagger-ui/` 路径下，不会与接口路径冲突。

```java
// org.springdoc.webmvc.ui.SwaggerWebMvcConfigurer
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
    StringBuilder uiRootPath = new StringBuilder();
    if (swaggerPath.contains(DEFAULT_PATH_SEPARATOR))
        uiRootPath.append(swaggerPath, 0, swaggerPath.lastIndexOf(DEFAULT_PATH_SEPARATOR));
    if (actuatorProvider.isPresent() && actuatorProvider.get().isUseManagementPort())
        uiRootPath.append(actuatorProvider.get().getBasePath());

    registry.addResourceHandler(uiRootPath + SWAGGER_UI_PREFIX + "*/*" + SWAGGER_INITIALIZER_JS)
            .addResourceLocations(CLASSPATH_RESOURCE_LOCATION + DEFAULT_WEB_JARS_PREFIX_URL + DEFAULT_PATH_SEPARATOR)
            .setCachePeriod(0)
            .resourceChain(false)
            .addResolver(swaggerResourceResolver)
            .addTransformer(swaggerIndexTransformer);

    registry.addResourceHandler(uiRootPath + SWAGGER_UI_PREFIX + "*/**")
            .addResourceLocations(CLASSPATH_RESOURCE_LOCATION + DEFAULT_WEB_JARS_PREFIX_URL + DEFAULT_PATH_SEPARATOR)
            .resourceChain(false)
            .addResolver(swaggerResourceResolver)
            .addTransformer(swaggerIndexTransformer);
}
```

总的来看，Knife4j 的收益可能小于使用成本，需要慎重选择。个人建议是在确实需要其特有功能时才考虑使用 Knife4j，否则还是推荐使用 Springdoc。

## 总结

对于新项目，推荐使用 Springdoc 配合 OpenAPI 来管理 API 文档。

对于老项目，如果不想引入新的依赖，可以考虑使用 Swagger 2.0 的 Springfox 来管理 API 文档。

如果有扩展需求，可以考虑使用 Knife4j，但需要慎重考虑收益与成本是否匹配。

## 参考文档

[1] [Springdoc 官方文档](https://springdoc.org/)

[2] [Knife4j 官方文档](https://doc.xiaominfo.com/)

[3] [OpenAPI 官方文档](https://swagger.io/specification/)
