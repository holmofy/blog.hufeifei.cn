---
title: SpringBoot3的新特性-RFC7807-ProblemDetail
date: 2023-5-28
categories: J2EE
tags: 
- SpringBoot
- WebFlux
---

默认情况下，SpringBoot提供了[DefaultErrorAttributes](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/web/servlet/error/DefaultErrorAttributes.html)类，该类实现了[ErrorAttributes](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/web/servlet/error/ErrorAttributes.html)接口，以在发生未处理的错误时生成错误响应。在默认错误的情况下，系统会生成一个 JSON 响应结构，我们可以更仔细地检查它：
```json
{
    "timestamp": "2023-04-01T00:00:00.000+00:00",
    "status": 500,
    "error": "Internal Server Error",
    "path": "/api/example"
}
```

虽然此错误响应包含一些关键属性，但它可能无助于查问题。幸运的是，我们可以通过在 Spring WebFlux 应用程序中创建ErrorAttributes接口的自定义实现来修改此默认行为。

从 Spring Framework 6 开始提供了[ProblemDetail](https://docs.spring.io/spring-framework/docs/6.0.7/reference/html/web.html#mvc-ann-rest-exceptions)来支持[RFC7807规范](https://www.rfc-editor.org/rfc/rfc7807.html)的表示。ProblemDetail包括一些定义错误详细信息的标准属性，还有一个扩展详细信息以进行自定义的选项。下面列出了支持的属性：

* type (string) – 标识问题类型的 URI 引用
* title (string) – 问题类型的简短摘要
* status (number) – HTTP 状态码
* detail (string) – 应该包含异常的详细信息。
* instance (string) – 一个 URI 引用，用于识别问题的具体原因。例如，它可以指导致问题的属性。

除了上面提到的标准属性外，ProblemDetail还包含一个Map<String, Object> 以添加自定义参数以提供有关问题的更多详细信息。示例错误响应结构如下：

```json
{
  "type": "https://example.com/probs/email-invalid",
  "title": "Invalid email address",
  "detail": "The email address 'john.doe' is invalid.",
  "status": 400,
  "timestamp": "2023-04-07T12:34:56.789Z",
  "errors": [
    {
      "code": "123",
      "message": "Error message",
      "reference": "https//error/details#123"
    }
  ]
}
```

Spring Framework 还提供了一个名为[ErrorResponseException](https://docs.spring.io/spring-framework/docs/current/javadoc-api//org/springframework/web/ErrorResponseException.html)的基本实现。此异常封装了一个ProblemDetail对象，该对象生成有关发生的错误的附加信息。我们可以扩展这个异常来自定义和添加属性。

## 在SpringBoot中启用ProblemDetail

SpringBoot默认并没有开启ProblemDetail功能，需要通过以下方式任意一种方式开启：

1、yaml配置文件

```yaml
# webmvc
spring:
  mvc:
    problemdetails:
      enabled: true
# webflux
spring:
  webflux:
    problemdetails:
      enabled: true
```

2、添加ResponseEntityExceptionHandler

```java
@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {
    //...
}
```

