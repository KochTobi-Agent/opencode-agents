# Thymeleaf (v3)

## Overview

Thymeleaf is a modern server-side Java template engine for both web and standalone environments, capable of processing HTML, XML, JavaScript, CSS, and plain text. Its main goal is to bring elegant *natural templates* to development workflows — HTML that can be correctly displayed in browsers and also work as static prototypes, allowing for stronger collaboration between frontend and backend developers.

Thymeleaf is widely used with Spring Boot and Spring MVC applications, though it can be integrated into any JVM environment.

## Key Features

- **Natural Templating**: Templates remain valid HTML/XML using standard elements and attributes instead of custom syntax. Templates can be viewed as static prototypes in browsers without processing.
- **Rich Feature Set**: Internationalization (i18n), forms, fragments, layout management, conditionals, iteration, and more.
- **Extensibility**: Create custom dialects with your own processors and attributes.
- **Multiple Template Modes**: HTML, XML, TEXT, JAVASCRIPT, CSS, and RAW.
- **Spring Integration**: First-class support for Spring MVC and Spring Boot with Spring Expression Language (Spring EL).
- **No XML Required**: Unlike JSP, Thymeleaf 3+ supports plain HTML5 templates.

## Setup

### Maven Dependencies

**Standalone (non-Spring):**
```xml
<dependency>
    <groupId>org.thymeleaf</groupId>
    <artifactId>thymeleaf</artifactId>
    <version>3.1.2.RELEASE</version>
</dependency>
```

**Spring Boot:**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

**Spring MVC (manual configuration):**
```xml
<dependency>
    <groupId>org.thymeleaf</groupId>
    <artifactId>thymeleaf</artifactId>
    <version>3.1.2.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.thymeleaf</groupId>
    <artifactId>thymeleaf-spring5</artifactId>
    <version>3.1.2.RELEASE</version>
</dependency>
```

### Gradle Dependencies

```gradle
// Standalone
implementation 'org.thymeleaf:thymeleaf:3.1.2.RELEASE'

// Spring Boot
implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'
```

### Spring Boot Configuration (application.properties)

```properties
# Template location
spring.thymeleaf.prefix=classpath:/templates/
spring.thymeleaf.suffix=.html
spring.thymeleaf.mode=HTML

# Cache (disable for development)
spring.thymeleaf.cache=false

# Internationalization
spring.messages.basename=messages
```

### Standalone Configuration

```java
import org.thymeleaf.TemplateEngine;
import org.thymeleaf.templatemode.TemplateMode;
import org.thymeleaf.templateresolver.ClassLoaderTemplateClassResolver;

LoaderTemplateResolver templateResolver = new ClassLoaderTemplateResolver();
templateResolver.setTemplateMode(TemplateMode.HTML);
templateResolver.setPrefix("templates/");
templateResolver.setSuffix(".html");
templateResolver.setCacheable(false);

TemplateEngine templateEngine = new TemplateEngine();
templateEngine.setTemplateResolver(templateResolver);
```

## Standard/Dialects Usage

Thymeleaf uses **dialects** — collections of processors that define how templates are processed. The two main dialects are:

1. **Standard Dialect**: Basic Thymeleaf features with OGNL expression language
2. **SpringStandard Dialect**: Standard dialect adapted for Spring with Spring EL

Templates identify Thymeleaf usage via the `th` namespace:
```html
<html xmlns:th="http://www.thymeleaf.org">
```

### Expression Types

- `${...}` — Variable expressions (context variables/model attributes)
- `*{...}` — Selection expressions (selected object)
- `#{...}` — Message (i18n) expressions
- `@{...}` — Link (URL) expressions

## Code Examples

### Basic Template with Variable Substitution

```html
<!DOCTYPE html>
<html xmlns:th="http://www.w3.org/1999/xhtml">
<head>
    <title th:text="${pageTitle}">Default Title</title>
</head>
<body>
    <h1 th:text="${message}">Hello World!</h1>
    <p th:text="|Welcome, ${userName}!|">Welcome!</p>
</body>
</html>
```

### Processing a Template (Java)

```java
import org.thymeleaf.TemplateEngine;
import org.thymeleaf.context.Context;

Context context = new Context();
context.setVariable("pageTitle", "My Page");
context.setVariable("message", "Hello Thymeleaf!");
context.setVariable("userName", "John");

String html = templateEngine.process("template-name", context);
```

### Iteration (th:each)

```html
<ul>
    <li th:each="book : ${books}" th:text="${book.title}">Book Title</li>
</ul>
```

### Conditionals (th:if / th:unless)

```html
<div th:if="${user != null}">
    <p>Welcome, <span th:text="${user.name}">User</span>!</p>
</div>
<div th:unless="${user != null}">
    <p>Please log in.</p>
</div>
```

### URL Expressions (th:href, th:src)

```html
<a th:href="@{/user/profile}">Profile</a>
<a th:href="@{/order/details(id=${orderId})}">Order Details</a>
<img th:src="@{/images/logo.png}" />
<link th:href="@{/css/style.css}" rel="stylesheet" />
<script th:src="@{/js/app.js}"></script>
```

### Form Handling

```html
<form th:action="@{/save}" th:object="${user}" method="post">
    <input type="text" th:field="*{name}" placeholder="Name" />
    <input type="email" th:field="*{email}" placeholder="Email" />
    <button type="submit">Save</button>
</form>
```

### Internationalization (i18n)

```html
<!-- messages.properties -->
# welcome.message=Welcome to our app!

<p th:text="#{welcome.message}">Default welcome message</p>
```

### Fragments and Layouts

**Defining a fragment:**
```html
<!-- fragments.html -->
<footer th:fragment="footer" th:text="|© ${year} My App|">
    © 2024
</footer>
```

**Including a fragment:**
```html
<div th:replace="~{fragments :: footer}"></div>
<!-- Or with parameters -->
<div th:replace="~{fragments :: footer(year=2024)}"></div>
```

### Spring MVC Controller

```java
@Controller
public class UserController {
    
    @GetMapping("/users")
    public String listUsers(Model model) {
        List<User> users = userService.findAll();
        model.addAttribute("users", users);
        return "users";
    }
    
    @PostMapping("/save")
    public String saveUser(@ModelAttribute User user, Model model) {
        userService.save(user);
        return "redirect:/users";
    }
}
```

### Inline JavaScript

```html
<script th:inline="javascript">
    var userName = /*[[${user.name}]]*/ 'default';
    var items = /*[[${items}]]*/ [];
</script>
```

## Notes

- Thymeleaf 3 introduced significant performance improvements over version 2.x
- Templates are cached by default in production; disable with `spring.thymeleaf.cache=false` for development
- The `th:` prefix stands for Thymeleaf namespace and doesn't interfere with browser rendering
- For Spring Boot, templates go in `src/main/resources/templates/`
- Thymeleaf integrates well with validation frameworks (Bean Validation, Spring Validator)
- The Layout dialect (from ultraq.net.nz) provides powerful layout inheritance capabilities
