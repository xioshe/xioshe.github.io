---
title: 如何在 Spring Boot 接口中正确地序列化时间字段
date: 2024-11-21 17:17:46
tags:
- Spring Boot
- Date
categories:
- 技术实践
---

在 Java 项目中处理时间序列化从来就不是容易的事。一方面要面临多变的时间格式，年月日时分秒，毫秒，纳秒，周，还有讨厌的时区，稍不注意就可能收获一堆异常，另一方面，Java 又提供了 `Date` 和 `LocalDateTime` 两个版本的时间类型，二者分别对应着不同的序列化配置，光是弄清楚这些配置，就是一件麻烦事。但是在大多数时候，我们又不得不和这一堆配置打交道。

作为开始，让我们来了解一下可用的配置和相应的效果。

## 时间字符串配置

准备一个简单接口来展示效果。

```java
@Slf4j
@RestController
@RequestMapping("/boom")
public class BoomController {

    @Operation(summary = "boom")
    @GetMapping
    public BoomData getBoomData() {
        return new BoomData(Clock.systemDefaultZone());
    }

    @Operation(summary = "boom")
    @PostMapping
    public BoomData postBoomData(@RequestBody BoomData boomData) {
        log.info("boomData: {}", boomData);
        return boomData;
    }
}

@Data
@NoArgsConstructor
@AllArgsConstructor
public class BoomData {

    private Date date;
    private LocalDateTime localDateTime;
    private LocalDate localDate;
    private LocalTime localTime;

    public BoomData(Clock clock) {
        this.date = new Date(clock.millis());
        this.localDateTime = LocalDateTime.now(clock);
        this.localDate = LocalDate.now(clock);
        this.localTime = LocalTime.now(clock);
    }
}
```

上面涉及两种时间类型：

- `Date` 代表老版本日期类型，类似的还有 `Calendar`，陪着 Java 度过了漫长岁月，使用面极广。但相对而言，设计不太跟得上时代了，比如值可变导致线程不安全，月份从 0 开始有点反人类。
- `LocalDateTime` 代表 `java.time` 包的新版时间类型，JDK 8 中引入。新的时间类型解决老版本类型的设计缺陷，同时增加了丰富的 API 来提高易用性。

两种类型在记录的信息方面有一点区别：

- `Date` 的时间精度为毫秒，内部实际是一个 long 类型时间戳。此外还记录了时区信息，简单记录为 `Date = timestamp + timezone`。如果没有提供时区，默认使用系统时区。

- `LocalDateTime` 时间精度为纳秒，内部用 7 个整数来记录时间：

  - int year
  - short month
  - short day
  - byte hour
  - byte minute
  - byte second
  - int nano

  可以简单记录为 `LocalDateTime = year + month + day + hour + minute + second + nano`。（实际上应该是 `LocalDateTime = LocalDate + LocalTime`，`LocalDate = year + month + day`，`LocalTime = hour + minute + second + nano`。）

  LocalDateTime 没有时区信息，这也是类名中 Local 的含义，代表使用本地时区。如果需要时区信息，可以用 `ZonedDateTime` 类型，`ZonedDateTime = LocalDateTime + tz`。

了解了两个版本时间类型的区别，再看它们的序列化差异。

### JSON 序列化

调用 GET 接口，得到默认的序列化结果。

```json
{
  "date": "2024-10-10T21:07:08.781+08:00",
  "localDateTime": "2024-10-10T21:07:08.781283",
  "localDate": "2024-10-10",
  "localTime": "21:07:08.781263"
}
```

默认配置下，时间字段被序列化为时间字符串，但格式不尽相同。Spring Boot 使用 Jackson 进行 JSON 序列化，对不同的时间类型有不同的格式化规则：

- `Date` 默认按照 ISO 标准格式化
- `LocalDateTime` 也按照 ISO 标准处理，精确到微秒，少了时区
- `LocalDate` 和 `LocalTime` 与 LocalDateTime 处理方式相似

所谓 ISO 标准，指的是 **ISO 8601 标准**，一种专门处理日期时间格式的国际标准。将时间日期组合按 `yyyy-MM-dd'T'HH:mm:ss.SSSXXX` 格式处理，比如 `2024-10-10T21:07:08.781+08:00`，其中字母 T 为日期和时间分隔符，日期表示为年-月-日，时间表示为时:分:秒.毫秒。格式中的 `XXX` 指的是时区信息，对于东 8 区，表示为 `+08:00`。

默认情况下，调用 POST 接口，也需要保证 body 中的 JSON 串按照 ISO 8601 的格式处理时间字段，才能正常反序列化，否则 Spring Boot 会抛出异常。当然，时间格式的要求也没那么严格，可以省略时区、微秒、毫秒、秒，都能正常反序列化，但 T 不能省略，年月日时分不能省略。

在接口调用两端统一标准时，ISO 8601 表现不坏，但是，碰到国内互联网偏爱的 `yyyy-MM-dd HH:mm:ss` 格式，就会收获一个 `HttpMessageNotReadableException`，JVM 会提示你 `JSON parse error: Cannot deserialize value of type XXX ...`。

如果想要加入 `yyyy-MM-dd HH:mm:ss` 大家庭，最简单的办法是使用 `@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")`。@JsonFormat 注解用于指定时间类型的序列格式，对 Date 类型和 LocalDateTime 类型都有效。

```java
public class BoomData {

    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private Date date;
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime localDateTime;
    @JsonFormat(pattern = "yyyy-MM-dd")
    private LocalDate localDate;
    @JsonFormat(pattern = "HH:mm:ss")
    private LocalTime localTime;

}
```

此时能 GET 到满足格式的时间字符串

```json
{
  "date": "2024-11-20 15:15:57",
  "localDateTime": "2024-11-20 23:15:57",
  "localDate": "2024-11-20",
  "localTime": "23:15:57"
}
```

POST 请求也正常处理。

看样子 @JsonFormat 效果不坏。问题是，稍微有点繁琐，每个时间字段都要配置一遍。幸运的是，spring boot 支持全局设置 Jackson 时间序列化格式：

```yaml
spring:
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss # 全局时间格式
    time-zone: GMT+8 # 指定默认时区，除非时间字段已指定时区，否则 JSON 序列化时都会使用此时区
```

更加幸运的是，@JsonFormat 优先级比全局配置更高，让我们可以实现某些要求特殊格式的需求。

似乎只要组合 `spring.jackson.date-format` 和 @JsonFormat，我们就可以无所不能了。没有人比我更懂时间序列化！

可惜的是，`spring.jackson.date-format` 不支持新版时间类型。是的，在 2024 年，距离 `java.time` 包发布已经十年了，Spring 的序列化配置仍然不支持 LocalDateTime 类型。如果你要序列化 LocalDateTime 类型，最简单的办法就是使用 @JsonFormat。因为 @JsonFormat 是 Jackson 提供的注解。Spring 对此毫无作为。

发完牢骚，考虑如何全局配置 LocalDateTime 的格式化规则。方案有很多种，最简单的就是明确地告诉 Jackson，LocalDateTime 等类型按照某某格式序列化和反序列化。

```java
// 大概是 JacksonConfig 之类的类
@Bean
public Jackson2ObjectMapperBuilderCustomizer jackson2ObjectMapperBuilderCustomizer() {

    return builder -> {

        // formatter
        DateTimeFormatter dateFormatter = DateTimeFormatter.ofPattern("yyyy-MM-dd");
        DateTimeFormatter dateTimeFormatter =  DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
        DateTimeFormatter timeFormatter = DateTimeFormatter.ofPattern("HH:mm:ss");

        // deserializers
        builder.deserializers(new LocalDateDeserializer(dateFormatter));
        builder.deserializers(new LocalDateTimeDeserializer(dateTimeFormatter));
        builder.deserializers(new LocalTimeDeserializer(timeFormatter));

        // serializers
        builder.serializers(new LocalDateSerializer(dateFormatter));
        builder.serializers(new LocalDateTimeSerializer(dateTimeFormatter));
        builder.serializers(new LocalTimeSerializer(timeFormatter));
    };
}
```

上述代码为三种类型构建了不同的 `DateTimeFormatter`（java.time 包提供的格式化工具，线程安全），然后为每种类型绑定序列化器（Serializer）和反序列化器（Deserializer）。

现在 Local 系统的日期类型就跟 Date 表现一致了。

总结一下，在 JSON 序列化时：

- 如果使用了 Date 类型，可以用 `spring.jackson.date-format` 和 @JsonFormat 的组合来适应不同格式化要求
- 如果使用了 LocalDateTime 等类型，需要配置 Jackson，绑定序列化器和反序列化器，再结合 @JsonFormat 方能从心所欲

但此时还没结束，也并非结束的开始，只是开始的结束～

### 请求参数

除了 JSON 序列化，还有一种场景，也会涉及时间序列化。那就是请求参数中的时间字段，最常见的就是 Controller 方法中没有用 `@RequestBody` 标记的对象参数，比如 GET 请求，比如表单提交（`application/x-www-form-urlencoded`）的 POST 请求。

为了便于展示，在 BoomController 中添加一个新的接口方法。

```java
@GetMapping("query")
public BoomData queryBoomData(BoomData boomData) {
    log.info("boomData: {}", boomData);
    return boomData;
}
```

一个比较常用的 Query 接口的写法。试着调用一下。

```plaintext
GET http://localhost:8080/boom/query?localDateTime=2024-10-10T21:07:08.781283&date=2024-10-10T21:07:08.781+08:00
```

报错，field 'date': rejected value [2024-10-10T21:07:08.781+08:00]。

再试试

```plaintext
GET http://localhost:8080/boom/query?localDateTime=2024-10-10T21:07:08.781283&date=2024-10-10 21:07:08
```

还是报错，field 'date': rejected value [2024-10-10 21:07:08]。

什么格式才能不报错？

```plaintext
GET http://localhost:8080/boom/query?localDateTime=2024-10-10T21:07:08.781283&date=10/10/2024 21:07:08
```

没错，要用 `dd/MM/yyyy` 的格式。因为请求参数不归 JSON 序列化管，而是由 Spring MVC 处理。Spring MVC 默认的 Date 类型格式就是  `dd/MM/yyyy`。

要修改也简单，`@DateTimeFormat`，Spring 提供，专门处理时间参数格式化。

```json
@DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")
private Date date;
@DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")
private LocalDateTime localDateTime;
@DateTimeFormat(pattern = "yyyy-MM-dd")
private LocalDate localDate;
@DateTimeFormat(pattern = "yyyy-MM-dd")
private LocalTime localTime;
```

现在，正常处理请求 `http://localhost:8080/boom/query?localDateTime=2024-10-10 21:07:08&date=2024-10-10 21:07:08&localDate=2024-10-10&localTime=11:09:15`。

又到了寻找全局配置的时间了。

```yaml
spring:
  mvc:
    format:
      date: yyyy-MM-dd HH:mm:ss # 对 Date 和 LocalDate 类型有效，LocalDate 会忽略时间部分
      time: HH:mm:ss # 对 LocalTime 和 OffsetTime 有效
      date-time: yyyy-MM-dd HH:mm:ss # LocalDateTime, OffsetDateTime, and ZonedDateTime
```

按需选择即可。

总结一下，对于 GET 请求参数中的时间字段，和表单提交 POST 请求中的时间字段，可以通过 `spring.mvc.format.date/time/date-time` 来配置全局格式。

- 请求中只使用了 Date 类型，只需要配置 `spring.mvc.format.date`
- 如果使用了 `java.time` 包中的类型，需要根据类型选择不同配置项

对于不使用全局配置的场景，用 @DateTimeFormat 指定单独的时间格式。

## 一起来用时间戳吧

以上是使用时间字符串传递时间的情况，接下来，我们讨论一下用时间戳格式。

先理解一下有关时间戳的概念：

- GMT 时间，用来表示时区，比如 GMT+8，就是指东 8 区的时间。单独的 GMT 也可以看作 GMT+0，即 0 时区的时间，这个时区位于英国格林威治
- UTC 时间，与 GMT 是相同的概念，也用来表示时区，只不过 UTC 更精确一些。同样，UTC+8 可以表示东 8 区，单独 UTC 表示 0 时区

- Unix 纪元（Unix Epoch），一个特定的时间点，1970 年 1 月 1 日 00:00:00 UTC(+0)，也就是 0 时区中 1970 年元旦。这个时间点常用于计算机系统的时间起点，如同坐标轴上的 0。

指导了上述 3 个概念，时间戳的含义就容易解释了，**从 Unix 纪元开始经过的毫秒数**（或秒数，计算机常用毫秒）。把时间想象为一条长长的坐标轴，0 的位置是 Unix 纪元，在那之后，真实世界的每一毫秒，都对应时间轴上的一个点。

时间戳用整数表示，一个长整数，具备时间字符串一样的功能。因此，也可以用时间戳来传递时间信息。

如果我信誓旦旦地宣称时间戳优于时间字符串，肯定是十分主观的判断，但在接口中使用时间戳确实有一些亮晶晶的优点。

- 时区无关性，时间戳的值固定为 UTC+0 时区，无论位于哪个时区，同一时刻，同一时间戳。这样一来，就可以仅展示时考虑时区，其他时候都不需要考虑时区
- 体积小，一个 long 值足矣，比时间字符串更简短
- 兼容性好，不必考虑复杂的格式化规则

一些不可忽视的缺点：

- 可读性差，时间戳没有时间字符串直观，需要一些辅助转换工具，比如浏览器控制台
- 秒级时间戳和毫秒时间戳可能混淆，使用前要约定好

用 long 型时间戳也不需要考虑序列化问题，大多数平台都可以妥善处理 long 类型的序列化。但有些时候，在代码中用 Date 和 LocalDateTime 等明确的类型还是比 long 更方便。所以可能有这么一个需求：在代码中使用时间类型，在序列化时使用时间戳。也就是在 DTO 类中用 Date，在 JSON 字符串中用 long。

和使用时间字符串类型，这个需求也分为两种情况：

- JSON 序列化转换
- 请求参数转换

二者要分开处理。

### JSON 序列化中的时间戳

Spring 提供了一个配置项，控制 Jackson 在序列化时将时间类型处理为时间戳。

```properties
spring.jackson.serialization.write-dates-as-timestamps=true
```

此时，GET 请求中的 date 就会变成了 `"date": 1728572627475`，POST 时也能正确地识别时间戳。

但是，只有 Date 才有这种优渥的待遇，`java.time` 包的类型仍然面临自己动手丰衣足食的窘境。

开启 write-dates-as-timestamps 后，LocalDateTime 等类型会被序列化为整形数组（回忆一下 LocalDateTime 的简单公式）。

```json
{
  "date": 1728572627475,
  "localDateTime": [
    2024,
    10,
    10,
    23,
    3,
    47,
    475519000
  ],
  "localDate": [
    2024,
    10,
    10
  ],
  "localTime": [
    23,
    3,
    47,
    475564000
  ]
}
```

也不能说有问题，毕竟 LocalDateTime 精确到纳秒，直接转换为毫秒时间戳，会丢失精度。总之，要实现和谐转换，需要设置 Jackson。

```java
// 仍然是 JacksonConfig 之类的什么地方
@Bean
public Jackson2ObjectMapperBuilderCustomizer jackson2ObjectMapperBuilderCustomizer() {
    return builder -> {
        // deserializers
        builder.deserializers(new LocalDateDeserializer());
        builder.deserializers(new LocalDateTimeDeserializer());

        // serializers
        builder.serializers(new LocalDateSerializer());
        builder.serializers(new LocalDateTimeSerializer());
    };
}

public static class LocalDateTimeSerializer extends JsonSerializer<LocalDateTime> {

    /**
     * 如果没有重写 handledType() 方法，会报错
     * @return LocalDateTime.class
     */
    @Override
    public Class<LocalDateTime> handledType() {
        return LocalDateTime.class;
    }

    @Override
    public void serialize(LocalDateTime value, JsonGenerator gen, SerializerProvider serializers)
            throws IOException {
        if (value != null) {
            gen.writeNumber(value.atZone(ZoneId.systemDefault()).toInstant().toEpochMilli());
        }
    }
}

public static class LocalDateTimeDeserializer extends JsonDeserializer<LocalDateTime> {

    @Override
    public Class<?> handledType() {
        return LocalDateTime.class;
    }

    @Override
    public LocalDateTime deserialize(JsonParser parser, DeserializationContext deserializationContext) throws IOException {
        long timestamp = parser.getValueAsLong();
        return Instant.ofEpochMilli(timestamp).atZone(ZoneId.systemDefault()).toLocalDateTime();
    }
}

public static class LocalDateSerializer extends JsonSerializer<LocalDate> {

    @Override
    public Class<LocalDate> handledType() {
        return LocalDate.class;
    }

    @Override
    public void serialize(LocalDate value, JsonGenerator gen, SerializerProvider serializers)
            throws IOException {
        if (value != null) {
            gen.writeNumber(value.atStartOfDay(ZoneId.systemDefault()).toInstant().toEpochMilli());
        }
    }
}

public static class LocalDateDeserializer extends JsonDeserializer<LocalDate> {

    @Override
    public Class<?> handledType() {
        return LocalDate.class;
    }

    @Override
    public LocalDate deserialize(JsonParser parser, DeserializationContext deserializationContext) throws IOException {
        long timestamp = parser.getValueAsLong();
        return Instant.ofEpochMilli(timestamp).atZone(ZoneId.systemDefault()).toLocalDate();
    }
}
```

这里我们进行了一些生硬的强制措施，定义了一系列 Deserializer 和 Serializer，实现了 LocalDateTime 和 long 之间的序列化规则。

没有处理 LocalTime，因为单独的时间转换为时间戳不那么契合，时间戳有明确地年月日，这部分对于 LocalTime 显得多余，而且时间通常与时区有关，处理时要更谨慎一些。可以根据需求选择，如果明确需要使用时间戳来表示 LocalTime，可以采用类似的方法，注册 Deserializer 和 Serializer。

以上是在 JSON 序列化时将 Date、LocalDateTime 转化为时间戳需要的配置：

- 如果只使用 Date，使用 Spring 提供的配置项 `spring.jackson.serialization.write-dates-as-timestamps=true` 即可
- 如果使用了 LocalDateTime，需要进行额外的配置，明确地指定 Jackson 将 LocalDateTime 转换为时间戳

### 请求参数中的时间戳

在请求参数中使用时间戳复杂一些，因为不像时间字符串一样有现成的配置，需要手动实现转换规则。

可以利用 Converter 接口来解决这个问题。

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addConverter(new LongStringToDateConverter());
        registry.addConverter(new LongStringToLocalDateTimeConverter());
        registry.addConverter(new LongStringToLocalDateConverter());
        // registry.addConverter(new LongStringToLocalTimeConverter()); // 按需
    }

    private static class LongStringToDateConverter implements Converter<String, Date> {
        @Override
        public Date convert(String source) {
            try {
                long timestamp = Long.parseLong(source);
                return new Date(timestamp);
            } catch (NumberFormatException e) {
                return null;
            }
        }
    }

    private static class LongStringToLocalDateTimeConverter implements Converter<String, LocalDateTime> {

        @Override
        public LocalDateTime convert(String source) {
            try {
                long timestamp = Long.parseLong(source);
                return Instant.ofEpochMilli(timestamp).atZone(ZoneId.systemDefault()).toLocalDateTime();
            } catch (NumberFormatException e) {
                return null;
            }
        }
    }

    private static class LongStringToLocalDateConverter implements Converter<String, LocalDate> {

        @Override
        public LocalDate convert(String source) {
            try {
                long timestamp = Long.parseLong(source);
                return Instant.ofEpochMilli(timestamp).atZone(ZoneId.systemDefault()).toLocalDate();
            } catch (NumberFormatException e) {
                return null;
            }
        }
    }

    private static class LongStringToLocalTimeConverter implements Converter<String, LocalTime> {

        @Override
        public LocalTime convert(String source) {
            try {
                long timestamp = Long.parseLong(source);
                return Instant.ofEpochMilli(timestamp).atZone(ZoneId.systemDefault()).toLocalTime();
            } catch (NumberFormatException e) {
                return null;
            }
        }
    }

}
```

注意 Source 类型为 String 而不是 Long，因为 Spring MVC 会将所有的接口请求参数类型统一视为 String，然后调用 Converter 转换为其他类型。有许多内置的 Converter，比如转换为 long 类型时，就使用了内置的 StringToNumber 转换类。我们定义的 LongStringToDateConverter 与 StringToNumber 是平级的关系。

以上是在接口参数中将 Date、LocalDateTime 转化为时间戳需要的处理：很简单，注册 Converter 即可。

### Swagger UI 中的类型

使用 SwaggerUI 时，默认会使用 DTO 字段类型作为请求参数类型，也就是接收时间字符串。序列化时改为时间戳后，还需要在 Swagger UI 中统一。

Java 项目有两种集成 Swagger 的方式，Springdoc 和 Spring Fox。Springdoc 更新，对应的配置如下：

```java
@Bean
public OpenAPI customOpenAPI() {
    // 关键是要调用这个静态方法进行 replace
    SpringDocUtils.getConfig()
            .replaceWithClass(Date.class, Long.class)
            .replaceWithClass(LocalDateTime.class, Long.class)
            .replaceWithClass(LocalDate.class, Long.class);
    return new OpenAPI();
}
```

如果使用 Spring Fox，则需要使用另一种配置：

```java
@Bean
public Docket createRestApi() {
    return new Docket(DocumentationType.OAS_30)
            ...
            .build()
            // 重点是这句
            .directModelSubstitute(LocalDateTime.class, Long.class);
}
```

此时，在 Swagger UI 页面调试接口时，时间类型的参数就显示为整数了。

## The Only Neat Thing to Do

回顾一下，在 Spring Boot 接口中处理时间字段序列化，涉及两个场景：

- JSON 序列化
- GET 请求和表单提交请求中的参数

两种情况要分开设置。

在 Java 类型选择方面，Spring 对 Date 类型的支持比 LocalDateTime 好，有很多内置的配置，能省去很多麻烦。

如果要使用 LocalDateTime 等类型，在 JSON 序列化时要指定时间格式，在请求参数中也要指定时间格式。前者需要手动配置，后者可以使用 Spring 提供的配置项。

如果想要用时间戳传递数据，也需要分别设置，在 JSON 序列化时指定序列化器和反序列化器，在请求参数中绑定对应的 Converter 实现类。此外，统一 Swagger UI 的类型体验更佳。
