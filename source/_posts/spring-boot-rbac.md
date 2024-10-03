---
title: Spring Boot 实现权限控制
date: 2024-10-03 19:55:00
tags:
- Spring Boot
- Security
- RBAC
categories:
- 技术实践
---

上一篇文章 [Spring Boot 实现 JWT 认证](https://blog.prochase.top/2024/09/spring-boot-jwt/)，介绍了 Spring Boot 实现 JWT 认证的流程，本文将关注架构安全性的另一个重要概念——授权，也就是权限控制。

## RBAC 模型

权限控制有不同的模型，常用的一种是 RBAC。RBAC 是基于角色的访问控制（Role-Based Access Control）的缩写。

简单来讲，RBAC 模型大致结构为：

```plaintext
用户（User）-> 角色（Role）-> 权限（Permission）-> 资源（Resource）
```

用户拥有角色，角色被赋予权限，权限关联资源。同一个用户可能拥有多个角色，同一个角色也可能被赋予多个权限。访问资源需要一种或多种权限。用户访问资源时，对比用户拥有的权限和资源需要的权限。

Spring Security 支持 RBAC 模型，并做了一些简化，将角色和权限合并为 Authority。在资源端，角色是 Authority，隶属角色的权限也是 Authority，检查权限就是在需要的权限和用户拥有的 Authority 之间做对比。

具体实现上，Spring Security 的安全上下文保存了用户信息（`UserDetails`），用户信息中包含了用户拥有的权限（`GrantedAuthority`）。在 Spring Security 体系中，`UserDetailsService` 接口的 `loadUserByUsername` 方法用于从数据库获取 UserDetails 信息，也包括用户拥有的权限信息。

```java
public interface UserDetails extends Serializable {
    Collection<? extends GrantedAuthority> getAuthorities();
    // 省略其他方法
}

public interface UserDetailsService {
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

依托 GrantedAuthority 抽象，Spirng Security 的权限控制分为两部分：

- 用户权限管理，由 `UserDetailsService` 接口的 `loadUserByUsername` 方法实现。获取 UserDetails 时，实现 getAuthorities() 方法获取权限。至于权限从何而来，自己实现。通常存储在数据库中。
- 资源权限控制，为资源分配需要的权限。

访问资源时，Spring Security 会调用 `AuthorizationManager` 接口的 `check` 方法检查权限，对比需要的权限和用户拥有的权限。

## 实现用户权限管理

用户权限管理，实现权限-角色-用户的层级结构。通常是关系型数据的多对多关系表。

```java
class User {
    private Long id;
    private String username;
    private String password;
    private Set<Role> roles;
}

class Role {
    private Long id;
    private String name;
    private Set<Permission> permissions;
}

class Permission {
    private Long id;
    private String code;
}
```

总共需要三张实体表，外加两张关系表。

实现 `UserDetailsService` 接口的 `loadUserByUsername` 方法。

```java
@Service
@RequiredArgsConstructor
public class UserDetailsServiceImpl implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.selectByUsername(username);
        if (user == null) {
            throw new UsernameNotFoundException("User not found");
        }
        user.setAuthorities(getAuthorities());
        return user;
    }

    private Set<SimpleGrantedAuthority> getAuthorities(User user) {
        Set<String> authorities = new HashSet<>();
        for (Role role : user.getRoles()) {
            authorities.add("ROLE_" + role.getCode());
            for (Permission permission : role.getPermissions()) {
                authorities.add(permission.getCode());
            }
        }
        return authorities.stream().map(SimpleGrantedAuthority::new).collect(Collectors.toSet());
    }
}
```

Spring Security 处理角色时会自动加 `ROLE_` 前缀，在处理 Authority 时不会，所以定义角色时，角色名不加 ROLE_ 前缀。

## 实现资源权限控制

这部分讲述如何为资源指定需要的权限。

先理解什么是资源。资源可以有不同的粒度，比如方法、类、模块、系统等。但对后端应用而言，最直观的划分是 API，一个 API 就是一个资源，访问 API 需要权限。这也是 Spring Security 的默认粒度。

API 与 Controller 的方法存在一一对应的关系，因此，为 API 指定权限，也包括为 Controller 的方法指定权限。Spring Security 可以直接为 API 指定权限，也可以基于注解为 Controller 的方法指定权限。

### 直接为 API 指定权限

直接为 API 指定权限，通过在配置类定义 `SecurityFilterChain` 来指定。

```java
@EnableWebSecurity // 用于支持 SecurityFilterChain
@Configuration
public class SecurityConfig {
    @Bean
    SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
                .authorizeHttpRequests(auth -> {
                    auth.requestMatchers("/api/auth/**").permitAll()
                            .requestMatchers("/api/public/**").permitAll()
                            .requestMatchers("/api/admin/**").hasRole("ADMIN")
                            .requestMatchers("/api/users/edit").hasAuthority("user:edit")
                            .anyRequest().authenticated();
                })
                .build();
    }
}
```

对以 `/api/auth` 为前缀的 API 和以 `/api/public` 为前缀的 API，不进行权限检查。对以 `/api/admin` 为前缀的 API，需要用户具有 ADMIN 角色。对 `/api/users/edit` API，需要用户具有 user:edit 权限。其他 API 需要认证，不需要额外权限。

在 SecurityFilterChain 中，通常进行比较粗粒度的权限控制，比如以前缀来指定 API 权限。如果进行细粒度的权限控制，如果 API 很多，配置会非常繁琐，也不便于维护。此时，可以基于注解来指定 API 权限。

### 基于注解指定 API 权限

在 Controller 的方法上添加注解，可以间接地为 API 指定权限。

有多种注解支持在方法上控制权限，大致可以分为三类：

- Spring Security 内置注解，@PreAuthorize、@PostAuthorize、@PreFilter、@PostFilter
- 基于 JSR-250 规范的注解，@RolesAllowed、@PermitAll、@DenyAll
- Spring Security 的遗留注解 @Secured，文档介绍这是一个 Service 层注解。

要使用这些注解，需要在配置中开启支持。

```java
@EnableMethodSecurity(prePostEnabled = true, jsr250Enabled = true, securedEnabled = true)
@Configuration
public class SecurityConfig {
    // 省略其他配置
}
```

三类注解中，Spring Security 内置注解 prePost 的功能最强大，提供了基于表达式的权限定义和数据过滤的功能，推荐使用。

- PreAuthorize：指定方法需要的权限
- PostAuthorize：权限 + 数据过滤，不满足条件时抛出 AccessDeniedException 异常
- PreFilter：授权 + 列表过滤，不满足条件的数据会被过滤，不会抛出异常
- PostFilter：授权 + 列表过滤，不满足条件的数据会被过滤，不会抛出异常

使用 PreFilter 和 PostFilter 时，需要考虑数据过滤的性能问题。如果数据量很大，过滤会非常耗时，不如直接在 SQL 中限制过滤条件。

### 资源权限控制的原理

通过 `SecurityFilterChain` 配置的 API 权限，在 AuthorizationFilter 中检查。内部存在 `AuthorizationFilter -> RequestMatcherDelegatingAuthorizationManager` 的调用链。RequestMatcherDelegatingAuthorizationManager 类如其名，会根据 API 的路径来选择实际的 AuthorizationManager。

如果在 SecurityFilterChain 中没有指定 API 权限，只是开启了 authenticated 认证检查，则根据**路径匹配**到 AuthenticatedAuthorizationManager，调用链为 `AuthenticatedAuthorizationManager -> AuthenticationTrustResolverImpl`。AuthenticationTrustResolverImpl 只会检查 Authentication 是否存在用户 UserDetails，用户状态是否存在，不涉及权限检查的逻辑。

如果在 SecurityFilterChain 中用 `hasRole` `hasAuthority` 为 API 指定了权限，**路径匹配**到 AuthoritiesAuthorizationManager。这又构成了一条新的调用链 `AuthorityAuthorizationManager -> AuthoritiesAuthorizationManager`。最终，在 AuthoritiesAuthorizationManager 中，会调用 `isAuthorized` 方法，遍历所有注册的 GrantedAuthority，检查用户是否拥有权限。

```java
// AuthoritiesAuthorizationManager
private boolean isAuthorized(Authentication authentication, Collection<String> authorities) {
    for (GrantedAuthority grantedAuthority : getGrantedAuthorities(authentication)) {
        if (authorities.contains(grantedAuthority.getAuthority())) {
            return true;
        }
    }
    return false;
}
```

对于方法注解配置的权限，无法直接在 AuthorizationFilter 中处理。在 Filter-Servlet 洋葱圈中，Filter 只能从 Request 中获取信息，无法获取具体处理请求的方法信息。只有到了 Servlet 中，才有可能接触到方法信息。

Spring MVC 使用 DispatcherServlet 作为 Controller 和外部 Servlet 容器的桥梁，请求穿过重重 Filter 后，到达 DispatcherServlet 的刹那，才真正进入 Spring MVC 的世界。DispatcherServlet 内部，也有一个类似的洋葱圈，外层是重重叠叠的 Interceptor 拦截器，最内层才是 Controller 方法。

![Servlet Filter](https://img.prochase.top/bkimg/2024/10/b55bc6daa949a470cc2c1738c86a62c3.png)

在 Dispatcher 内部，能获取到 Controller 方法信息。因此，注解式的方法权限控制，都是用拦截器 Interceptor 实现。

对于注解 `@PreAuthorize("hasAuthority('user:edit')")`，调用链大致如下：

```plaintext
AuthorizationManagerBeforeMethodInterceptor -> PreAuthorizeAuthorizationManager -> ExpressionUtils
```

AuthorizationManagerBeforeMethodInterceptor 是前置拦截器，在 Controller 方法执行前，调用 AuthorizationManager 检查权限。如果是 @PostAuthorize 注解，则会使用 AuthorizationManagerAfterMethodInterceptor 后置拦截器。

PreAuthorizeAuthorizationManager 是具体的权限检查逻辑，与注解 @PreAuthorize 一一对应。调用 ExpressionUtils 的 evaluate 方法，解析 SpEL 表达式 `hasAuthority('user:edit')`，检查用户是否拥有权限。如果是 @PostAuthorize 注解，则会使用 PostAuthorizeAuthorizationManager。

解析表达式时，会使用 `MethodSecurityEvaluationContext` 提供的 Root 对象。最终执行 hasAuthority() 方法的，是 `MethodSecurityExpressionRoot` 类。

```java
// MethodSecurityExpressionRoot
private boolean hasAnyAuthorityName(String prefix, String... roles) {
    Set<String> roleSet = getAuthoritySet();
    for (String role : roles) {
        String defaultedRole = getRoleWithDefaultPrefix(prefix, role);
        if (roleSet.contains(defaultedRole)) {
            return true;
        }
    }
    return false;
}
```

getAuthoritySet() 方法会从 SecurityContext 中获取当前用户的权限，roles 参数则代表注解中指定的权限。

可以看到，不管实现方式如何，最终的权限检查逻辑，仍然是对比用户权限和注解权限。掌握这一点，在对 Spirng Security 进行扩展时就可以灵活变通。

## 自定义注解控制方法权限

基于注解的权限控制，除了 Spring Security 提供的注解，还可以使用自定义注解。自定义注解的优点在于实现权限控制的同时，还可以实现自动注册权限的功能。

```java
@RequirePermission(code = "user:get", name = "获取用户信息")
@GetMapping("/{id}")
public User getUserById(@PathVariable Long id) {
    return userRepository.selectByPrimaryKey(id);
}
```

1. 应用启动时，会自动注册 user:get 权限。
2. 调用 getUserById 方法时，也会像 @PreAuthorize 一样，检查用户是否拥有 user:get 权限。

要实现上述功能，首先需要一个自定义注解，比如 @RequirePermission。然后，基于这个注解实现如下功能：

- 应用启动时，扫描注解，注册权限。
- 调用方法时，检查权限。

### 基于注解自动注册权限

在应用启动时，扫描所有被 @RequirePermission 注解的方法，注册权限。将方法权限限制在 Controller 层是一个比较合适的粒度。

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface RequirePermission {
    String code();
    String name() default "";
    String description() default "";
}

@Component
@RequiredArgsConstructor
public class PermissionRegistrar implements ApplicationListener<ContextRefreshedEvent> {

    private final PermissionService permissionService;

    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        Map<String, Object> beans = event.getApplicationContext().getBeansWithAnnotation(Controller.class);
        for (Object bean : beans.values()) {
            registerPermissionsForBean(bean);
        }
    }

    private void registerPermissionsForBean(Object bean) {
        // 代理对象无法获取 method 注解，需要用原对象
        Class<?> clazz = AopUtils.getTargetClass(bean);
        for (Method method : clazz.getDeclaredMethods()) {
            RequirePermission annotation = method.getAnnotation(RequirePermission.class);
            if (annotation != null) {
                registerPermission(annotation);
            }
        }
        RequirePermission requirePermission = clazz.getDeclaredAnnotation(RequirePermission.class);
        if (requirePermission != null) {
            registerPermission(requirePermission);
        }
    }

    private void registerPermission(RequirePermission requirePermission) {
        String code = requirePermission.code();
        String name = StringUtils.defaultIfBlank(requirePermission.name(), code);
        String description = StringUtils.defaultIfBlank(requirePermission.description(), name);
        permissionService.createPermissionIfNotExists(code, name, description);
    }
}
```

AopUtils.getTargetClass 用于获取代理对象的原对象。因为无法直接从代理对象获取方法注解。此外，可以用 CommandLineRunner 结合 ApplicationContext 来替换 ApplicationListener。性能方面，可以先扫描，然后一次性注册，合并数据库写操作。

### 基于自定义注解检查权限

要为 @RequirePermission 实现类似 @PreAuthorize 的功能，最简单的办法是基于 AOP 实现，用切面来处理。优点是不会与框架耦合，只要有 Spring Boot 就行。缺点是侵入性太强，无法直接使用 Spring Security 提供的功能。

这里介绍两种用元注解为自定义注解添加权限检查功能的方法。

所谓元注解（meta-annotation），就是修饰注解的注解。比如想实现一个限制 ADMIN 角色的注解，可以这么写：

```java
@Target({ ElementType.METHOD, ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
@PreAuthorize("hasRole('ADMIN')")
public @interface IsAdmin {}
```

@PreAuthorize 是元注解，@IsAdmin 就是被修饰的注解，在 @PreAuthorize 修饰下，@IsAdmin 注解就具有了 @PreAuthorize 的功能。在方法上使用 @IsAdmin 注解，就可以实现权限检查。

```java
@GetMapping("/admin")
@IsAdmin
public String admin() {
    return "admin";
}
```

@IsAdmin 的功能比较简单，权限固定，如果要想实现 @IsUser 的功能，还得另起炉灶，提供一个新的注解。@RequirePermission 则不同，权限不固定，由属性值 code 决定。由于 Java 语言本身不支持在元注解获取被修饰注解的属性值，@PreAuthorize 无法直接获取 code 的值。想要 @RequirePermission 能检查权限，还得另想办法，解决元注解无法获取被修饰注解属性值的问题。

![image-20241002185840200](https://img.prochase.top/bkimg/2024/10/7ca49ac62c8512d50df65269bd66951e.png)

### 使用扩展 SpEL 表达式

一个简单的解决方案是使用 Spring Security 6.3 扩展的 SpEL 表达式，通过 `'{code}'` 获取注解的属性值。

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@PreAuthorize("hasAuthority('{code}')")
public @interface RequirePermission {
    String code();
    String name() default "";
    String description() default "";
}
```

执行 `hasAuthority('{code}')` 表达式时，能自动获取 RequirePermission.code() 的值，填充进 `{code}` 占位符中。

但要启用这种使用大括号的表达式，需要向 Spring 容器注册 PrePostTemplateDefaults 类型的 Bean。

```java
@EnableMethodSecurity(prePostEnabled = true)
@Configuration
public class SecurityConfig {

   @Bean
   static PrePostTemplateDefaults prePostTemplateDefaults() {
       return new PrePostTemplateDefaults();
   }
}
```

### 自己扩展 SpEL 表达式

在 Spring Security 6.3 之前，不支持大括号 `{code}` 获取注解属性的写法，无法直接将注解的属性传递给元注解。我们只能自己扩展 SpEL 表达式，绕点远路，先利用反射获取注解的属性值，再将属性值传给元注解。

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@PreAuthorize("hasAuthority(@permissionExpressionEvaluator.getPermission(#root.method))")
public @interface RequirePermission {
    String code();
    String name() default "";
    String description() default "";
}
```

在 @PreAuthorize 中，仍然使用了 `hasAuthority` 表达式，`@permissionExpressionEvaluator.getPermission()` 表示调用名为 permissionExpressionEvaluator Bean 的 `getPermission` 方法来获取权限，`#root.method` 表示获取被注解修饰方法的反射对象 Method。

PermissionExpressionEvaluator 是一个自定义的类，根据传入的反射对象 Method，利用反射获取方法上的注解信息，从而得到方法需要的权限，再从安全上下文 Authentication 中获取分配给当前用户的权限，两相比较，实现权限检查。

```java
@Component
public class PermissionExpressionEvaluator {

    private final Map<String, String> permissionCache = new ConcurrentHashMap<>(64);

    public String getPermission(Method method) {
        return permissionCache.computeIfAbsent(keyOf(method), k -> getPermissionCode(method));
    }

    private String getPermissionCode(Method method) {
        RequirePermission annotation = method.getAnnotation(RequirePermission.class);
        if (annotation != null) {
            return annotation.code();
        }
        // 方法注解优先于类注解
        annotation = method.getDeclaringClass().getAnnotation(RequirePermission.class);
        if (annotation != null) {
            return annotation.code();
        }
        return "";
    }

    private String keyOf(Method method) {
        Class<?> clazz = method.getDeclaringClass();
        String methodName = clazz.getName() + "." + method.getName();
        StringJoiner sj = new StringJoiner(",", methodName + "(", ")");
        for (Class<?> parameterType : method.getParameterTypes()) {
            sj.add(parameterType.getSimpleName());
        }
        return sj.toString();
    }

    public void clear() {
        this.permissionCache.clear();
    }
}
```

PermissionExpressionEvaluator 实现 getPermission 方法时，按照先方法注解再类注解的顺序获取权限，保证方法上的注解优先于类上的注解。这一点与 @PreAuthorize 的行为一致。同时在类和方法使用 @PreAuthorize 注解时，方法注解会覆盖类注解。此外，为了提高性能，PermissionExpressionEvaluator 还使用了 Map 来缓存方法的权限，避免每次都要靠反射获取。

上述实现需要使用 `#root.method` 获取方法反射。遗憾的是，Spring Security 的 SpEL 表达式不支持这种用法。Spring 体系大量使用 SpEL 表达式，但不同的模块会提供不同的 Context。**Context 不同，表达式可以获取的信息也不同**。比如 Spring Security 的 Context 中，可以使用 `hasRole`、`hasAuthority` 等方法，而 Web 模块的 Context 中，可以使用 `#request` 获取 HttpServletRequest 对象。Spring Security 解析注解的 SpEL 使用的 Context 为 `MethodSecurityEvaluationContext`，内部有一个 `MethodSecurityExpressionRoot` 类型的属性。Root 决定了 SpEL 表达式可以获取的信息。用表达式可以直接调用 Root 的方法，比如 `hasRole`、`hasAuthority`；用 `#root.property` 表达式可以访问 property 属性，比如 `#root.method`，就需要 Root 提供了名为 `method` 的属性。

现在的 MethodSecurityExpressionRoot 并没有 method 属性，但可以通过扩展 Root 来实现。我们可以继承 `MethodSecurityExpressionRoot`，添加 `method` 属性，再将新的 Root 传递给 Context。

具体代码如下：

```java
public class CustomMethodSecurityExpressionRoot extends SecurityExpressionRoot implements MethodSecurityExpressionOperations {

    private Object filterObject;

    private Object returnObject;

    private Object target;

    /**
     * 仅仅添加了 method 属性，其他都与 MethodSecurityExpressionRoot 保持一致
     */
    @Getter
    @Setter
    private Method method;

    public CustomMethodSecurityExpressionRoot(Authentication a) {
        super(a);
    }

    public CustomMethodSecurityExpressionRoot(Supplier<Authentication> authentication) {
        super(authentication);
    }

    @Override
    public void setFilterObject(Object filterObject) {
        this.filterObject = filterObject;
    }

    @Override
    public Object getFilterObject() {
        return this.filterObject;
    }

    @Override
    public void setReturnObject(Object returnObject) {
        this.returnObject = returnObject;
    }

    @Override
    public Object getReturnObject() {
        return this.returnObject;
    }

    void setThis(Object target) {
        this.target = target;
    }

    @Override
    public Object getThis() {
        return this.target;
    }
}

/**
 * 重写 MethodSecurityExpressionHandler，用于设置自定义 Root 对象
 */
public class CustomMethodSecurityExpressionHandler extends DefaultMethodSecurityExpressionHandler {

    /**
     * 重写以避免调用父类的 createSecurityExpressionRoot 方法
     */
    @Override
    public EvaluationContext createEvaluationContext(Supplier<Authentication> authentication, MethodInvocation mi) {
        /*
         createSecurityExpressionRoot 是 private 方法，由 invokespecial 指令调用，采用解析方式来确定方法版本。
         解析会直接根据外观类型来确定方法，因此如果不重写 createEvaluationContext 方法，
         就会直接调用 DefaultMethodSecurityExpressionHandler 的 createSecurityExpressionRoot 方法。
         */
        MethodSecurityExpressionOperations root = createSecurityExpressionRoot(authentication, mi);
        // 为了解决 MethodSecurityEvaluationContext 不可见问题
        CustomMethodSecurityEvaluationContext ctx = new CustomMethodSecurityEvaluationContext(root, mi,
                getParameterNameDiscoverer());
        ctx.setBeanResolver(getBeanResolver());
        return ctx;
    }

    @Override
    protected MethodSecurityExpressionOperations createSecurityExpressionRoot(
            Authentication authentication, MethodInvocation invocation) {
        return createSecurityExpressionRoot(() -> authentication, invocation);
    }

    private MethodSecurityExpressionOperations createSecurityExpressionRoot(Supplier<Authentication> authentication,
                                                                            MethodInvocation invocation) {
        CustomMethodSecurityExpressionRoot root = new CustomMethodSecurityExpressionRoot(authentication);
        root.setThis(invocation.getThis());
        root.setPermissionEvaluator(getPermissionEvaluator());
        root.setTrustResolver(getTrustResolver());
        root.setRoleHierarchy(getRoleHierarchy());
        root.setDefaultRolePrefix(getDefaultRolePrefix());
        root.setMethod(invocation.getMethod());
        return root;
    }
}

/**
 * MethodSecurityEvaluationContext 是 default 可见，为了在 CustomMethodSecurityExpressionHandler 中使用，复制过来
 */
public class CustomMethodSecurityEvaluationContext extends MethodBasedEvaluationContext {

    public CustomMethodSecurityEvaluationContext(MethodSecurityExpressionOperations root, MethodInvocation mi,
                                    ParameterNameDiscoverer parameterNameDiscoverer) {
        super(root, getSpecificMethod(mi), mi.getArguments(), parameterNameDiscoverer);
    }

    private static Method getSpecificMethod(MethodInvocation mi) {
        return AopUtils.getMostSpecificMethod(mi.getMethod(), AopProxyUtils.ultimateTargetClass(mi.getThis()));
    }
}
```

重写的类型比较多，主要逻辑如下：

- 由于 MethodSecurityExpressionRoot 是 private 可见，无法直接继承，所以 CustomMethodSecurityExpressionRoot 复制了 MethodSecurityExpressionRoot 的代码，添加了 method 属性。
- 为了使用自定义的 Root，还需要重写 DefaultMethodSecurityExpressionHandler 的 createSecurityExpressionRoot 方法，返回自定义的 Root。
- 仅仅重写 createSecurityExpressionRoot 方法还不够，由于框架直接调用 DefaultMethodSecurityExpressionHandler 的 createEvaluationContext 方法来获取 Context，而这个方法内部调用了 private 版的 createSecurityExpressionRoot，为了避免解析到父类，还需要重写 createEvaluationContext 方法。
- 重写 createEvaluationContext 方法时，由于默认的 MethodSecurityEvaluationContext 对外不可见，所以又复制了一个一模一样的 CustomMethodSecurityEvaluationContext 类。

总结下来，关键是为 Root 添加 method 属性，以及使 MethodSecurityExpressionHandler 使用自定义的 Root，其他都是为了解决可见性问题而复制了一堆代码。

现在，我们还需要将一切组合起来，就可以使用 @RequirePermission 注解来实现权限控制了。

```java
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
@Configuration
class SecurityConfig {

    @Bean
    MethodSecurityExpressionHandler expressionHandler() {
        // 用自定义的 Handler 替换默认的 DefaultMethodSecurityExpressionHandler
        return new CustomMethodSecurityExpressionHandler();
    }
}
```

此时，@PreAuthorize 等 Method Security 注解的表达式中，就可以使用 `#root.method` 就可以获取到方法反射对象。@RequirePermission 注解也就能正常工作了。

如果想要扩展其他功能，也可以采用类似的思路：

1. 自定义一个 Root，扩充属性或者添加方法。
2. 自定义 Handler，使用自定义的 Root。
3. 解决各种可见性问题。

## 总结

本文介绍了在 Spring Boot 中用 Spring Security 实现 RBAC 权限管理的方案，并提供了自定义注解来简化权限管理的思路。想要实现自定义注解，需要解决元注解无法获取被修饰注解属性值的问题。Spring Security 6.3 之后可以直接使用大括号表达式获取注解属性，而之前的版本需要自己扩展 SpEL 表达式来实现。

## 参考文章

[1] [Method Security :: Spring Security 文档](https://docs.spring.io/spring-security/reference/servlet/authorization/method-security.html)
