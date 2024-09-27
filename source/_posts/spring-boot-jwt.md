---
title: Spring Boot 实现 JWT 认证
date: 2024-09-27 01:36:44
tags:
- Spring Boot
- Security
- JWT
categories:
- 技术实践
---

JWT 是 JSON Web Token 的缩写，用于实现无状态的认证系统。无状态指后端不需要存储用户 Session，前端与后端之间只通过**令牌（Token）**进行通信。无状态的好处是易于扩展，适合分布式系统。而且，相比另一种常用的无状态认证方案分布式 Session，JWT 更契合 RESTful API 的设计理念。

JWT 的令牌（Token）是由后端生成的一串字符串，前端发起 HTTP 请求时携带令牌一起发送给后端，后端通过解析令牌来验证用户的身份。整个过程，后端不需要存储令牌，也不需要维护 Session，所以 JWT 非常适合用于分布式系统。

JWT 认证的基本流程如下：

1. 用户调用登录接口，后端生成令牌并返回给前端。
2. 前端将令牌存储在本地（如 localStorage 或 cookie）。
3. 前端发起其他请求时，将令牌添加到请求头或请求体中。
4. 后端解析令牌，验证用户身份和权限。

前端携带 Token，较为常用的是 Bearer 身份认证，具体做法是在 HTTP 请求头中添加 `Authorization` 字段，值为 `Bearer <token>`，Bearer 和 token 之间是一个空格。

```yaml
Authorization: Bearer <token>
```

因此，如果想要在 Spring Boot 中实现 JWT 认证，需要实现以下功能：

- 生成令牌
- 解析令牌
- 验证令牌

## 生成、解析、验证

### 令牌格式

JWT 需要遵循 JWT 标准，生成的 Token 字符串具备固定的格式：

- 令牌头（Header）：包含令牌的类型（JWT）和使用的签名算法，默认使用 HMAC SHA-256 签名算法。
- 载荷（Payload）：有关用户和令牌的信息，是 Token 的主体部分。
- 签名（Signature）：用 Header 中的签名算法生成的摘要码，用于验证令牌的完整性。

一个典型的 JWT Token 为 `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c`，用 `.` 可以分为三部分，分别对应 Header、Payload 和 Signature。在 [jwt.io](https://jwt.io) 网站上可以查看解析后的内容。

![image-20240926173147117](https://img.prochase.top/bkimg/2024/09/f8261bdfeab5d00a6d4bb3c5ec819646.png)

令牌头（Header）和载荷（Payload）两部分都是 JSON 对象，使用特殊的 Base64Url 编码。Base64Url 编码是对 Base64 编码的改进，将 URL 中存在特殊意义的 `+` 和 `/` 替换为 `-` 和 `_`，从而在 URL 中使用。

签名部分则是对 Header 和 Payload 两部分进行签名生成的摘要码，`singature = HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), secret)`。secret 是后端使用的密钥，用于验证令牌的完整性，不能泄露，需要妥善保管。

在 Payload 中，默认包含的字段有：

- iss：issuer，签发人
- exp：expiration time，过期时间
- sub：subject，主题
- aud：audience，受众
- nbf：not before，生效时间
- iat：issued at，签发时间
- jti：JWT ID，令牌 ID

不过，并不需要提供以上所有信息，一个典型的 Payload 可能只包含以下内容：

- sub：主题，如用户 ID
- exp：过期时间，可以取 Token 生成时间的一个小时后
- iat：签发时间，取 Token 生成时间

同时，还可以根据需求，增加自定义字段，但需要注意，添加的字段越多，Token 的体积就越大，在传输过程中就需要占用更多的网络带宽，也会影响性能。

在安全性方面，JWT 存在一下问题：

- Base64Url 编码无加密，Header 和 Payload 部分是公开的，**不应该包含敏感信息**。
- **必须配合 HTTPS 使用**，否则存在被篡改的安全隐患。
- 基于令牌的认证方式无法主动撤销，只能等到令牌过期。不过可以通过黑名单的方式来实现令牌的撤销。

### 生成 JWT 令牌

生成令牌时，可以按照 JWT 规范手动实现，也可以使用现成工具库。在 Java 中，可以使用 `jjwt` 库，也可以使用 `com.auth0:java-jwt` 库。本文使用 `jjwt` 库作为例子。

```java
private static final String YOUR_SECRET = "your-secret-key";
private static final SecretKey SECRET_KEY = getSigningKey();

public String buildToken(String username, long expirationMills,
                         Map<String, Object> extraClaims) {
    return Jwts.builder()
            .claims()
            .subject(username)
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + expirationMills))
            .add(extraClaims)
            .and()
            .signWith(SECRET_KEY, Jwts.SIG.HS256)
            .compact();
}

private static SecretKey getSigningKey() {
    return Keys.hmacShaKeyFor(YOUR_SECRET.getBytes(StandardCharsets.UTF_8));
}
```

参数中的 `extraClaims` 是自定义的字段，可以添加一些额外的信息，但不能添加敏感信息。

`YOUR_SECRET` 字符串，是用于生成签名的密钥，使用 HMAC SHA-256 算法时，要求长度至少 32 字节（256 位）。

SecretKey 是对密钥的封装，考虑到构建成本，可以复用对象。生成 SecretKey 时，可以从配置文件中获取密钥字符串，也可以用代码生成。

```java
private SecretKey generateSecurityKey() {
    return Jwts.SIG.HS256.key().build();
}
```

### 解析 JWT 令牌

`jjwt` 库也提供了解析令牌的工具类，可以直接使用。

```java
private static final SecretKey SECRET_KEY = ...;

public Claims parseToken(String token) {
    return Jwts.parser()
            .verifyWith(SECRET_KEY)
            .build()
            .parseSignedClaims(token)
            .getBody();
}
```

从 Claims 对象中可以获取到令牌中的所有信息，包括自定义字段。

解析操作中，也包含了验证 Token 完整性的步骤，如果 Token 被篡改，会抛出 `JwtException`异常。

### 验证 JWT 令牌

验证令牌时，主要检查令牌是否过期。

```java
public boolean isTokenValid(String token) {
    Claims claims = parseToken(token);
    Date now = new Date();
    return !claims.getIssuedAt().after(now)
            && !claims.getExpiration().before(now);
}
```

## 与 Spring Security 集成

Spring Security 没有内置 JWT 认证的功能，需要自己实现。

### Spring Security 内部结构

Spring Security 主要基于 Filter 来实现认证和授权。Filter 是 Servlet 规范中的概念，在请求（HTTP Request）进入 Servlet 容器时，会经过一列长长的 Filter 链，链中每一个 Filter 都会对请求进行处理，至到最后到达 Servlet。在 Servlet 返回响应（HTTP Response）时，也会经过相同的 Filter 链，只不过顺序与请求时相反。Filter 链就像一个洋葱，请求数据从外向内穿过洋葱，而响应数据从内向外穿过洋葱。

![img](https://img.prochase.top/bkimg/2024/09/c48d490369949578992638249d9fa7b0.png)

除了 Filter，Spring Security 还有两个重要组件，AuthenticationManager 和 AuthenticationProvider。前者用于提供统一的认证功能，后者用于提供用户信息来源。一个 AuthenticationManager 可以包含多个 AuthenticationProvider，每个 AuthenticationProvider 对应一种用户来源，最常用的是 DaoAuthenticationProvider，用于从数据库中查询用户信息。

![image-20240926214101367](https://img.prochase.top/bkimg/2024/09/e6998ab9640fd77e1e2336a1091010de.png)

DaoAuthenticationProvider 会调用 UserDetailsService 接口，根据用户名获取用户信息。当系统使用数据库保管用户信息时，需要实现 UserDetailsService 接口，从数据库中查询用户信息，转换为 UserDetails 对象。

```java
public interface UserDetailsService {
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

`loadUserByUsername` 方法的返回类型是 UserDetails 接口，这是 Spring Security 定义的类型，包含了需要的用户信息。

```java
public interface UserDetails {
    Collection<? extends GrantedAuthority> getAuthorities();
    String getPassword();
    String getUsername();
    boolean isAccountNonExpired();
    boolean isAccountNonLocked();
    boolean isCredentialsNonExpired();
    boolean isEnabled();
}
```

UserDetails 接口的关键方法是 `getPassword` 和 `getUsername`，用于获取系统的登录凭证。`getAuthorities` 方法获取权限信息，如果不涉及到权限管理，返回空集合即可。`isAccountNonExpired`、`isAccountNonLocked`、`isCredentialsNonExpired`、`isEnabled` 方法获取用户的状态信息，默认返回 true，如果不需要更细粒度的控制，可以不用实现。

Spring Security 提供了一个 UserDetails 的实现类 `org.springframework.security.core.userdetails.User` 和 GrantedAuthority 接口的实现类 `org.springframework.security.core.SimpleGrantedAuthority`，可以直接使用。

### 实现 JWT 认证

为了更好地与 Spring Security 集成，我们应该将 JWT 认证的逻辑放在一个 Filter 中，并复用 Spring Security 的 AuthenticationManager 和 AuthenticationProvider。

首先，需要提供 JWT Token 的生成、解析、验证功能，这部分代码与前文一致，封装在 JwtTokenService 中。

其次，定义 PasswordEncoder、AuthenticationManager、UserDetailsService，前两者可以直接使用系统提供的实现类，UserDetailsService 需要自己实现。

```java
@EnableWebSecurity() // 启用 WebSecurityConfiguration
@Configuration
@RequiredArgsConstructor
public class SecurityConfig {

    private final UserMapper userMapper;

    /**
     * 启用 BCrypt 哈希算法处理密码，避免明文存储密码
     */
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    /**
     * 提供 UserDetailsService 接口的实现
     */
    @Bean
    public UserDetailsService userDetailsService() {
        return username -> userMapper.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("User not found"));
    }

    /**
     * 提供 AuthenticationManager 接口的实现
     */
    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

这里使用 Lambda 表达式实现 UserDetailsService，直接返回 UserMapper 返回值的前提是返回值是 UserDetails 接口的实现类。

`PasswordEncoder` 接口用于处理密码。为了保证安全性，不能直接存储明文密码，需要用**密码学哈希算法**进行单向映射。登录时对用户输入的密码进行相同操作，再进行比较。这样即使数据库泄露，黑客也无法知道用户的原始密码，也就无法用泄露的账号和密码登录系统，也无法根据用户习惯用相同密码尝试登录其他应用。

使用 DaoAuthenticationProvider 的`authenticate` 方法进行身份认证时，会自动调用 PasswordEncoder 对明文密码编码后再匹配。因此，如果使用 Spring Security 提供的认证机制，不需要手动调用 PasswordEncoder，系统会自动处理。但**注册用户时，必须用 PasswordEncoder 对明文密码进行编码**。

`BCryptPasswordEncoder` 是 Spring Security 提供的一种 PasswordEncoder 实现类，使用 BCrypt 哈希算法，这是安全性较高的算法，可以有效防止彩虹表攻击。

接着实现 JwtTokenFilter，内部调用 JwtTokenService，实现 JWT 认证逻辑。

```java
public class JwtTokenFilter extends OncePerRequestFilter {

    private final JwtTokenService jwtTokenService;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    @Nonnull HttpServletResponse response,
                                    @Nonnull FilterChain filterChain) throws ServletException, IOException {
        String authorization = request.getHeader(HttpHeaders.AUTHORIZATION);

        if (authorization == null || !authorization.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        // "Bearer ".length() == 7
        String token = authorization.substring(7);
        String username = jwtTokenService.extractUsername(token);

        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails user = userDetailsService.loadUserByUsername(username);
            if (jwtTokenService.isTokenValid(token, user)) {
                // 创建一个新的认证令牌，并将其设置为当前的安全上下文
                UsernamePasswordAuthenticationToken authToken =
                        new UsernamePasswordAuthenticationToken(
                                user,
                                null,
                                user.getAuthorities()
                        );
                // 为认证令牌绑定 Request 信息
                authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authToken);
            } else {
                log.warn("Invalid JWT token of user: {}", username);
            }
        }
        filterChain.doFilter(request, response);
        // 在 SecurityContextHolderFilter 中自动清除，不需要手动清除
        // SecurityContextHolder.getContext().setAuthentication(null);
    }
}
```

在上述代码中，没有调用 AuthenticationManager 来认证用户，只是实现了 Token -> UserDetails 的转换：

1. 校验 Token 是否有效
2. 从 Token 中获取用户名，用 UserDetailsService 获取对应的 UserDetails，构建为认证信息（Authentication）
3. 将认证信息保存进安全上下文。默认以线程变量方式存储在 ThreadLocal 对象中。

即使 Token 校验没通过，或者根据用户名查不到用户信息，或者根本就没提供 Token，JwtTokenFilter 也不会抛出异常，只是不会设置认证信息。

真正负责认证的是 AuthorizationFilter 类，这是 Spring Security 内置的 Filter，会根据认证信息进行认证。从安全上下文获取认证信息（Authentication），并调用 AuthenticationManager 进行身份认证。认证时会结合 URL 判断，如果 URL 需要身份认证，但无法从安全上下文获取对应 Authentication，就判定为没通过认证，抛出 AuthenticationException 异常。通过复用 AuthorizationFilter，而不是自己实现相关逻辑，可以更好地与 Spring Security 集成，复用其强大的认证机制。

我们还需要提供一个登录接口，根据用户的登录凭证，比如用户名密码，生成 Token 并返回。

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api/auth")
public class AuthController {

    private final AuthenticationManager authenticationManager;
    private final JwtTokenService jwtTokenService;


    @PostMapping("/login")
    public ResponseEntity<String> login(@RequestBody LoginRequest request) {
        Authentication authentication = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(
                        request.getUsername(),
                        request.getPassword()
                )
        );

        String token = jwtTokenService.buildToken(authentication.getName());
        return ResponseEntity.ok(token);
    }
}
```

有了上述类，就可以组装 JWT 认证的逻辑。在 Spring Security 中，最为核心的配置就是 SecurityFilterChain 类型的定义。通过 SecurityFilterChain，可以开启和关闭相关 Filter，可以指定哪些 URL 需要认证，哪些 URL 不需要认证，以及认证失败时的处理方式。

在实现 JWT 认证时，需要调整一下配置：

- 禁用 CSRF 保护。JWT 认证基于 Token，不需要 CSRF 保护。
- 禁用 Session。
- 配置 Cors，支持跨域。需要 注册一个 CorsFilter 类型的 Bean 才会生效。
- 配置 AuthenticationEntryPoint，指定认证失败时的处理方式。
- 注册 JwtTokenFilter，实现 Token -> UserDetails 的转换。
- 配置路径的认证规则，哪些路径需要认证，哪些路径不需要认证。

相关代码如下：

```java
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http,
                                                   JwtTokenFilter jwtTokenFilter,
                                                   DelegatedAuthenticationEntryPoint entryPoint) throws Exception {
        return http
                .cors(Customizer.withDefaults()) // 配合注册 CorsFilter Bean 才会生效
                .csrf(AbstractHttpConfigurer::disable) // 禁用 CSRF
                .sessionManagement(manager ->
                        manager.sessionCreationPolicy(SessionCreationPolicy.STATELESS)) // 无状态 Session
                .authorizeHttpRequests(auth -> {
                    auth.requestMatchers("/api/auth/**").permitAll(); // 开放登录接口
                    auth.anyRequest().authenticated(); // 其他接口需要认证
                })
                .exceptionHandling(exceptionHanding ->
                        exceptionHanding.authenticationEntryPoint(entryPoint)) // 认证失败的处理方式
                .addFilterBefore(jwtTokenFilter, UsernamePasswordAuthenticationFilter.class) // 注册 JwtTokenFilter
                .build();
    }

    @Bean
    public JwtTokenFilter jwtTokenFilter(@Qualifier("handlerExceptionResolver")
                                         HandlerExceptionResolver exceptionResolver) {
        return new JwtTokenFilter(jwtTokenService, userDetailsService, exceptionResolver);
    }

    // 用于支持 CORS
    @Bean
    public CorsFilter corsFilter() {
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowCredentials(true);
        config.addAllowedOrigin("*");
        config.addAllowedHeader("*");
        config.addAllowedMethod("*");
        source.registerCorsConfiguration("/**", config);
        return new CorsFilter(source);
    }
```

AuthenticationEntryPoint 涉及到错误处理，我们稍后再介绍。

基于目前的配置，就可以实现 JWT 认证。访问 login 之外的接口，如果没有提供 Token，会返回 401 错误码。通过在请求头添加 Token 后，就可以正常访问。对应的 Filter 链包含 13 个 Filter，从外向内依次是：

```bash
DisableEncodeUrlFilter (1/13)
WebAsyncManagerIntegrationFilter (2/13)
SecurityContextHolderFilter (3/13)
HeaderWriterFilter (4/13)
CorsFilter (5/13)
LogoutFilter (6/13)
JwtTokenFilter (7/13)
RequestCacheAwareFilter (8/13)
SecurityContextHolderAwareRequestFilter (9/13)
AnonymousAuthenticationFilter (10/13)
SessionManagementFilter (11/13)
ExceptionTranslationFilter (12/13)
AuthorizationFilter (13/13)
```

## 扩展功能

### 错误处理

上面我们提到，实际负责认证的是 AuthorizationFilter，认证失败时，会抛出 AuthenticationException 异常。这个异常无法由 Sping Boot 的 ExceptionHandler 处理，因为通用异常处理发生在 DispatcherServlet 中，这是一个 Servlet，位于上面洋葱图的最内圈，无法处理外圈 Filter 中的异常。因此，基于 ExceptionHandler 的机制，只能处理 Interceptor 和 Controller 中的异常，不能处理 Filter 中的异常。

Spring Security 在 AuthorizationFilter 之前注册了一个 ExceptionTranslationFilter，用来捕获 AuthorizationFilter 抛出的异常。当捕获到 AuthenticationException 异常时，会调用 AuthenticationEntryPoint 接口处理。默认的 AuthenticationEntryPoint 实现是 BasicAuthenticationEntryPoint，只会返回 401 错误码，如果要返回统一的异常相应，需要自定义 AuthenticationEntryPoint。

```java
@Component
public class DelegatedAuthenticationEntryPoint implements AuthenticationEntryPoint, InitializingBean {

    @Resource(name = "handlerExceptionResolver")
    private HandlerExceptionResolver exceptionResolver;

    @Override
    public void afterPropertiesSet() throws Exception {
        Assert.notNull(this.exceptionResolver, "exceptionResolver must be specified");
    }

    @Override
    public void commence(HttpServletRequest request,
                         HttpServletResponse response,
                         AuthenticationException authException)
            throws IOException, ServletException {
        log.debug("handle AuthenticationException with HandlerExceptionResolver, reason: {}",
                authException.getMessage());
        exceptionResolver.resolveException(request, response, null, authException);
    }
}
```

上述代码定义了一个 AuthenticationEntryPoint，不直接处理异常，而是委托给 HandlerExceptionResolver 处理。这样做的好处是可以直接复用 ExceptionHandler 中的异常处理机制，有利于统一代码。

对应的 ExceptionHandler 处理：

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(AuthenticationException.class)
    public ProblemDetail handleAuthenticationException(AuthenticationException exception,
                                                       HttpServletRequest request,
                                                       HttpServletResponse response) {
        log.debug("occur AuthenticationException: ", exception);
        log.warn("AuthenticationException in path {}: {}", request.getRequestURI(), exception.getMessage());
        response.setHeader(HttpHeaders.WWW_AUTHENTICATE, "Bearer");
        ProblemDetail errorDetail =
                ProblemDetail.forStatusAndDetail(HttpStatus.UNAUTHORIZED, exception.getMessage());
        errorDetail.setProperty("description", "Full authentication is required to access this resource");
        return errorDetail;
    }
}
```

### 为 UserDetails 添加缓存

在 JwtTokenFilter 中，每次请求都会调用 UserDetailsService 获取 UserDetails，相当于一次数据库查询。JwtTokenFilter 在每次请求时都会执行，如果系统用户量较多，频繁调用 UserDetailsService 会影响性能。可以考虑将 UserDetails 缓存起来，比如使用 Redis 缓存。

Spring Boot 有很多集成 Redis 的方案，最简单的是直接使用 Spirng Cache，需要引入 `spring-boot-starter-data-redis` 库。

使用 Redis Cache，需要配置序列化方案：

```java
@EnableCaching
@Configuration
public class CacheConfig {

    @Bean
    public RedisCacheConfiguration cacheConfiguration() {
        return RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(60)) // 默认缓存 60 分钟
                .disableCachingNullValues() // 不缓存 null
                .computePrefixWith(cacheName -> "lu:" + cacheName + ":") // 添加 lu: 前缀，并用单冒号替换调默认双冒号
                .serializeValuesWith(RedisSerializationContext.SerializationPair
                        .fromSerializer(new GenericJackson2JsonRedisSerializer())); // 使用 JSON 序列化
    }
}
```

配置好之后，就可以直接通过 `@Cacheable` 注解在 UserDetailsService 中使用缓存：

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class CustomCachingUserDetailsService implements UserDetailsService {

    private final UserMapper userMapper;
    private final RoleMapper roleMapper;

    @Override
    @Cacheable(value = "users", key = "#username") // 定义一个名为 users 的缓存，以参数中的 username 作为 key
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userMapper.findByUsername(username);
        if (user == null) {
            log.info("User {} not found", username);
            throw new UsernameNotFoundException("User '" + username + "' not found");
        }
        user.setRoles(roleMapper.listRolesByUserId(user.getId()));
        return user.asSecurityUser();
    }
}
```

有一个容易出错的地方，JSON 序列化不支持 Spring Security 提供的实现类 `org.springframework.security.core.userdetails.User` 和 `org.springframework.security.core.SimpleGrantedAuthority`，需要使用自定义的 UserDetails 实现和 GrantedAuthority 实现。

用上述 CustomCachingUserDetailsService 替换掉原来的 UserDetailsService，就可以实现 UserDetails 的缓存。JwtTokenFilter 每次处理请求，只有缓存无法命中时，才会调用 UserDetailsService 查询数据库。

### 禁用令牌

由于后端不会存储 Token，只有 Token 过期后才会失效，无法主动让一个 Token 失效。不过可以借助黑名单功能，实现类似的效果。

具体思路是维护一个黑名单，记录需要失效的用户，在进行 Token 认证时，查询用户是否在黑名单中。简单起见，可以借助 Redis 实现黑名单功能。

在 JwtTokenService 中，添加一个方法，用于将 Token 添加到黑名单中：

```java
public void blacklistAccessToken(String token) {
    if (!StringUtils.hasText(token)) {
        return;
    }
    String username = extractUsername(token);
    long ttl = extractExpiration(token).getTime() - System.currentTimeMillis();
    if (ttl > 0) {
        log.info("Access token blacklisted for user: {}", username);
        redisTemplate.opsForValue().set("lu:blacklist:" + username, token, ttl, TimeUnit.MILLISECONDS);
    }
}
```

在检验 Token 时，添加对黑名单的检查：

```java
public boolean isTokenValid(String token) {
    Claims claims = extractClaims(token);
    Date now = new Date();
    return !claims.getIssuedAt().after(now)
            && !claims.getExpiration().before(now)
            && !isTokenBlacklisted(token);
}

private boolean isTokenBlacklisted(String token) {
    String username = extractUsername(token);
    String blacklistedToken = redisTemplate.opsForValue().get("lu:blacklist:" + username);
    return token.equals(blacklistedToken);
}
```

当调用 `blacklistAccessToken` 后，相关 Token 无法通过校验，达到失效的效果。基于这个功能，可以实现登出 logout 功能。

```java
@PostMapping("/logout")
public void logout(@RequestHeader(HttpHeaders.AUTHORIZATION) String authHeader) {
    // "Bearer ".length() == 7
    String token = authHeader.substring(7);
    authenticationService.logout(token, command.getRefreshToken());
}
```

直接从请求头获取 Token，因此 logout 接口需要认证。

### 密钥轮换

在 JwtTokenService 中，无论采用配置文件提供 JWT 加密密钥，还是直接生成随机密钥，都存在一个问题：密钥固定不变，一旦泄露，就无法保证安全性。

解决办法是实现密钥轮换。定期更换 JWT 密钥，比如每 24 小时更换一次，将密钥泄露的影响降到最低。

实现密钥轮换最简单的办法是利用定时任务定期更换。值得注意的是，密钥轮换时，需要确保新旧密钥都能用于解密，因此需要保存旧的密钥。

```java
/**
 * 管理 JWT 使用的 SecretKey，提供密钥轮转功能，提高安全性
 */
@Slf4j
@Component
public class RotatingSecretKeyManager implements InitializingBean {

    private static final int MAX_KEYS = 2;
    private final Deque<SecretKey> keys = new ConcurrentLinkedDeque<>();

    @Override
    public void afterPropertiesSet() throws Exception {
        // 有必要预热
        rotateKeys();
    }

    @Scheduled(cron = "${security.jwt.key.rotation.cron:0 0 0 * * ?}")
    public void rotateKeys() {
        log.info("Rotating JWT signing keys");
        keys.offerFirst(generateSecurityKey());
        while (keys.size() > MAX_KEYS) {
            keys.pollLast();
        }
        log.info("JWT signing keys rotated. Current number of active keys: {}", keys.size());
//        jwtMetrics.incrementKeyRotationCount();
    }

    private SecretKey generateSecurityKey() {
        return Jwts.SIG.HS256.key().build();
    }

    public SecretKey getCurrentKey() {
        if (keys.isEmpty()) {
            rotateKeys();
        }
        return keys.peek();
    }

    public Iterable<SecretKey> secretKeys() {
        if (keys.isEmpty()) {
            rotateKeys();
        }
        return keys;
    }

}
```

这里运用了双端队列保管 SecretKey，每次轮换时，将新密钥添加到队列头部。这样，队头总是最新的密钥，队尾总是最旧的密钥。

修改 JwtTokenService 中生成 Token 的方法，从 RotatingSecretKeyManager 获取 SecretKey：

```java
public String buildToken(UserDetails userDetails, long expirationMills, Map<String, Object> extraClaims) {
    return Jwts.builder()
            .claims()
            .subject(userDetails.getUsername())
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + expirationMills))
            .add(extraClaims)
            .and()
            .signWith(keyManager.getCurrentKey(), Jwts.SIG.HS256)
            .compact();
}

private Claims extractClaims(String token) {
    JwtException exception = null;
    // 密钥会自动切换，token 对应的密钥可能被换掉了
    for (SecretKey secretKey : keyManager.secretKeys()) {
        try {
            return Jwts.parser()
                    .verifyWith(secretKey)
                    .build()
                    .parseSignedClaims(token)
                    .getPayload();
        } catch (JwtException e) {
            exception = e;
        }
    }
    assert exception != null;
    throw exception;
}
```

## 总结

本文介绍了如何在 Spring Boot 中实现 JWT 认证，并介绍了如何扩展 JWT 认证功能，包括错误处理、用户信息缓存、令牌失效、密钥轮换。重点是通过 JwtTokenFilter 将 JWT 令牌与 Spring Security 的认证功能结合起来，直接在 Spring Security 的 Filter 链中完成认证。

相关代码已上传到 GitHub，[xioshe/less-url](https://github.com/xioshe/less-url)，这是一个基于 Spring Boot 3 实现的短链服务，其中包含了 JWT 认证。

## 参考资料

[1] [JSON Web Token 入门教程 - 阮一峰的网络日志](https://www.ruanyifeng.com/blog/2018/07/json_web_token-tutorial.html)

[2] [Get Started with JSON Web Tokens](https://auth0.com/learn/json-web-tokens)

[3] [JWT authentication in Spring Boot 3 with Spring Security 6](https://blog.tericcabrel.com/jwt-authentication-springboot-spring-security/)
