# Vaadin Flow (v25)

## Overview

Vaadin Flow 25 is the latest major release of the Java-based full-stack web framework for building modern, server-driven web applications. It provides a component-based programming model where all UI logic runs on the JVM, with automatic client-server synchronization handled by the framework.

**Version Context:**
- **Vaadin 25.0.0-beta1** — current beta as of early 2026
- **Vaadin 24.8.x** — latest stable LTS branch
- Vaadin 25 targets **Spring Boot 4** / **Spring Framework 7** (Jakarta EE 11)
- Requires **Java 21+** (LTS baseline aligns with Spring Boot 4)

**Framework Type:** Server-side Java UI framework with optional Hilla (React/Lit) frontend bridge  
**Source Reputation:** High (official Vaadin documentation, benchmark score 86.6)

---

## What's New in Vaadin 25

### Major Changes from Vaadin 24

| Area | Vaadin 24 | Vaadin 25 |
|------|-----------|-----------|
| Spring Boot baseline | Spring Boot 3.x | Spring Boot 4.x |
| Spring Framework | Spring Framework 6.x | Spring Framework 7.x |
| Jakarta EE | Jakarta EE 10 | Jakarta EE 11 |
| Java baseline | Java 17+ | Java 21+ |
| Security configurer | `VaadinWebSecurity` (extend) | `VaadinSecurityConfigurer` (compose) |
| BOM artifact | `vaadin-bom` 24.x | `vaadin-bom` 25.x |

### Security API Evolution

Vaadin 25 introduces a **composable security model** alongside the existing inheritance-based approach:

- **Old approach (still supported):** Extend `VaadinWebSecurity` and override `configure(HttpSecurity)`
- **New approach (Vaadin 25+):** Use `VaadinSecurityConfigurer.vaadin()` as a composable `SecurityConfigurer` with Spring's `HttpSecurity.with()`

The composable style aligns with modern Spring Security 6+ patterns and avoids inheritance chains.

### Spring Boot Auto-Configuration

With `vaadin-spring-boot-starter` on the classpath and Spring Boot 4, the `SpringSecurityAutoConfiguration` automatically enables:
- Vaadin CSRF protection (internal requests excluded)
- Default request cache behaviour
- `NavigationAccessControl` (replaces the old `ViewAccessChecker`)
- Redirect of unauthenticated navigation to the configured login view

---

## Dependencies

### Maven (pom.xml) — Vaadin 25 + Spring Security + OAuth2

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>my-vaadin-app</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <properties>
        <java.version>21</java.version>
        <vaadin.version>25.0.0</vaadin.version>
    </properties>

    <!-- Spring Boot 4 parent -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>4.0.0</version>
    </parent>

    <!-- Vaadin BOM for version alignment -->
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>com.vaadin</groupId>
                <artifactId>vaadin-bom</artifactId>
                <version>${vaadin.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <!-- Core Vaadin + Spring Boot integration -->
        <dependency>
            <groupId>com.vaadin</groupId>
            <artifactId>vaadin-spring-boot-starter</artifactId>
        </dependency>

        <!-- Spring Security (required for Vaadin security helpers) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>

        <!-- OAuth2 / OpenID Connect (optional, for OIDC providers) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-oauth2-client</artifactId>
        </dependency>

        <!-- JPA / Data (optional) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>

        <!-- H2 for development (optional) -->
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>runtime</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- Vaadin Maven plugin: bundles frontend assets -->
            <plugin>
                <groupId>com.vaadin</groupId>
                <artifactId>vaadin-maven-plugin</artifactId>
                <version>${vaadin.version}</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>prepare-frontend</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

### Gradle (build.gradle.kts) — Vaadin 25 + Spring Security + OAuth2

```kotlin
plugins {
    id("org.springframework.boot") version "4.0.0"
    id("io.spring.dependency-management") version "1.1.7"
    id("com.vaadin") version "25.0.0"
    kotlin("jvm") version "2.1.0"
}

group = "com.example"
version = "1.0-SNAPSHOT"
java.sourceCompatibility = JavaVersion.VERSION_21

dependencyManagement {
    imports {
        mavenBom("com.vaadin:vaadin-bom:25.0.0")
    }
}

dependencies {
    // Core Vaadin + Spring Boot integration
    implementation("com.vaadin:vaadin-spring-boot-starter")

    // Spring Security
    implementation("org.springframework.boot:spring-boot-starter-security")

    // OAuth2 / OpenID Connect
    implementation("org.springframework.boot:spring-boot-starter-oauth2-client")

    // JPA / Data (optional)
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")

    // H2 for development
    runtimeOnly("com.h2database:h2")
}
```

---

## Security Configuration

### Approach 1 — VaadinWebSecurity (Inheritance, Classic Style)

Extend `VaadinWebSecurity` and override its `configure` methods. This is the long-standing pattern, still supported in Vaadin 25.

```java
import com.vaadin.flow.spring.security.VaadinWebSecurity;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityCustomizer;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;
import org.springframework.security.web.util.matcher.AntPathRequestMatcher;

@EnableWebSecurity
@Configuration
public class SecurityConfiguration extends VaadinWebSecurity {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // Permit static/public resources BEFORE calling super
        http.authorizeHttpRequests(auth ->
            auth.requestMatchers(new AntPathRequestMatcher("/public/**"))
                .permitAll()
        );

        // Delegates to VaadinWebSecurity for:
        //   - Vaadin CSRF exclusions
        //   - Default request cache
        //   - NavigationAccessControl wiring
        //   - Redirect unauthenticated requests
        super.configure(http);

        // Register login view with NavigationAccessControl
        setLoginView(http, LoginView.class);
    }

    @Override
    public void configure(WebSecurity web) throws Exception {
        // Additional web security customisations
        super.configure(web);
    }

    @Bean
    public UserDetailsService userDetailsService() {
        return new InMemoryUserDetailsManager(
            User.withUsername("user")
                .password("{noop}user")
                .roles("USER")
                .build(),
            User.withUsername("admin")
                .password("{noop}admin")
                .roles("USER", "ADMIN")
                .build()
        );
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### Approach 2 — VaadinSecurityConfigurer (Composable, Vaadin 25 Style)

Use `VaadinSecurityConfigurer.vaadin()` as a pluggable `SecurityConfigurer`. This is the **recommended pattern in Vaadin 25** and integrates cleanly with Spring Security 6/7's composable DSL.

```java
import com.vaadin.flow.spring.security.VaadinSecurityConfigurer;
import com.vaadin.flow.spring.security.VaadinAwareSecurityContextHolderStrategyConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@Import(VaadinAwareSecurityContextHolderStrategyConfiguration.class)
public class SecurityConfiguration {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http.with(VaadinSecurityConfigurer.vaadin(), configurer -> {
            // VaadinSecurityConfigurer automatically:
            //   - Configures Vaadin CSRF
            //   - Enables NavigationAccessControl
            //   - Sets up request cache
        });
        return http.build();
    }
}
```

### Approach 3 — OAuth2 / OpenID Connect Login

Combine `VaadinSecurityConfigurer` with Spring's OAuth2 client support for OIDC providers such as Keycloak, Google, GitHub, or Azure AD.

```java
import com.vaadin.flow.spring.security.VaadinSecurityConfigurer;
import com.vaadin.flow.spring.security.VaadinAwareSecurityContextHolderStrategyConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@Import(VaadinAwareSecurityContextHolderStrategyConfiguration.class)
public class OAuth2SecurityConfiguration {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http.with(VaadinSecurityConfigurer.vaadin(), configurer -> {
            configurer.oauth2LoginPage(
                "/oauth2/authorization/keycloak",  // (1) OAuth2 provider endpoint
                "{baseUrl}/session-ended"           // (2) Post-logout redirect URI
            );
        });
        return http.build();
    }
}
```

**application.yaml — Keycloak OIDC provider:**
```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          keycloak:
            provider: keycloak
            client-id: my-client-id
            client-secret: ${KEYCLOAK_CLIENT_SECRET}
            authorization-grant-type: authorization_code
            scope: openid,profile,email,roles
        provider:
          keycloak:
            issuer-uri: http://keycloak.local:8180/realms/my-realm
            user-name-attribute: preferred_username
```

**application.properties — alternative format:**
```properties
spring.security.oauth2.client.registration.keycloak.provider=keycloak
spring.security.oauth2.client.registration.keycloak.client-id=my-client-id
spring.security.oauth2.client.registration.keycloak.client-secret=${KEYCLOAK_CLIENT_SECRET}
spring.security.oauth2.client.registration.keycloak.authorization-grant-type=authorization_code
spring.security.oauth2.client.registration.keycloak.scope=openid,profile,email

spring.security.oauth2.client.provider.keycloak.issuer-uri=http://keycloak.local:8180/realms/my-realm
```

**Google OAuth2 (application.yaml):**
```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid,profile,email
```

---

## Key APIs

### VaadinWebSecurity

| Method | Description |
|--------|-------------|
| `configure(HttpSecurity http)` | Override to add custom HTTP security rules. Call `super.configure(http)` to apply Vaadin defaults. |
| `configure(WebSecurity web)` | Override for WebSecurity-level configuration (e.g., ignoring static resources). |
| `setLoginView(HttpSecurity, Class<?>)` | Registers a Vaadin view class as the login page for `NavigationAccessControl` redirects. |
| `setLoginView(HttpSecurity, String)` | Registers a plain URL path as the login page. |
| `setStatefulAuthentication(...)` | Configures stateful vs. stateless session handling. |

### VaadinSecurityConfigurer

Composable `SecurityConfigurer` used with `HttpSecurity.with(VaadinSecurityConfigurer.vaadin(), ...)`.

| Method | Description |
|--------|-------------|
| `VaadinSecurityConfigurer.vaadin()` | Static factory — creates the configurer instance. |
| `oauth2LoginPage(String loginPage, String logoutRedirectUri)` | Configures OAuth2/OIDC login endpoint and post-logout redirect. |
| `loginPage(String loginUrl)` | Sets a custom form-login URL. |

### NavigationAccessControl

`NavigationAccessControl` is the framework-level `BeforeEnterListener` that evaluates security annotations and route-path rules before each Vaadin navigation event.

| Component | Description |
|-----------|-------------|
| `NavigationAccessControl` | Central controller; runs registered `NavigationAccessChecker` instances in sequence. |
| `NavigationAccessChecker` | Interface for pluggable access-check strategies. |
| `AnnotatedViewAccessChecker` | Default checker; reads `@AnonymousAllowed`, `@PermitAll`, `@RolesAllowed`, `@DenyAll` annotations from the target view class. Delegates to `AccessAnnotationChecker`. |
| `RoutePathAccessChecker` | Checker that maps navigation route paths to Spring Security's URL-pattern-based `HttpSecurity` rules. Useful for migrating legacy Spring Security configurations. |
| `NavigationAccessControlConfigurer` | Spring `@Bean` for customising which checkers are active and how decisions are resolved. |
| `AccessAnnotationChecker` | Low-level helper that evaluates JSR-250 security annotations; can be extended for custom logic. |

### NavigationAccessControlConfigurer Bean

Declare this `@Bean` to customise the checker pipeline:

```java
@Bean
NavigationAccessControlConfigurer navigationAccessControlConfigurer() {
    return new NavigationAccessControlConfigurer()
            // Enable annotation-based checking (default, usually not needed explicitly)
            // .withAnnotatedViewAccessChecker()

            // Enable route-path-based checking (for URL-pattern rules)
            .withRoutePathAccessChecker()

            // Add a fully custom checker
            .withNavigationAccessChecker(new MyCustomChecker())

            // Filter the set of available checkers
            .withAvailableNavigationAccessCheckers(checker ->
                checker instanceof VotingNavigationAccessChecker
            )

            // Replace the default decision resolver
            .withDecisionResolver(new MyDecisionResolver());
}
```

> **Important:** If this bean is declared inside a class that also uses `VaadinSecurityConfigurer`, the method **must be `static`** to avoid circular dependency issues.

```java
class SecurityConfig extends VaadinWebSecurity {

    // static to avoid circular dependency with VaadinSecurityConfigurer
    @Bean
    static NavigationAccessControlConfigurer navigationAccessControlConfigurer() {
        return new NavigationAccessControlConfigurer()
            .withRoutePathAccessChecker()
            .withDecisionResolver(new CustomDecisionResolver());
    }
}
```

### RoutePathAccessChecker

Activating `RoutePathAccessChecker` allows centralized URL-pattern access rules (defined in `HttpSecurity.authorizeHttpRequests(...)`) to also govern Vaadin view navigation — enabling hybrid access control in apps migrated from older Spring MVC + Vaadin setups.

```java
@Bean
NavigationAccessControlConfigurer navigationAccessControlConfigurer() {
    return new NavigationAccessControlConfigurer()
            .withRoutePathAccessChecker();
}
```

---

## Security Annotations

Security annotations are placed on Vaadin view classes (`@Route`-annotated) and, for Hilla/browser-callable services, on service classes and methods.

| Annotation | Package | Behaviour |
|------------|---------|-----------|
| `@AnonymousAllowed` | `com.vaadin.flow.server.auth` | Any visitor (authenticated or not) may access the view. |
| `@PermitAll` | `jakarta.annotation.security` | Any **authenticated** user may access the view. |
| `@RolesAllowed({"ROLE_A"})` | `jakarta.annotation.security` | Only users with the specified role(s) may access the view. |
| `@DenyAll` | `jakarta.annotation.security` | No one may access the view (explicit lockout). |

**Default behaviour:** If **no annotation** is present on a view, `NavigationAccessControl` applies `@DenyAll` semantics — the view is inaccessible.

### Annotation Precedence (Method vs Class)

Method-level annotations override class-level annotations for Hilla browser-callable services:

```java
@BrowserCallable
@RolesAllowed("ADMIN")          // Class default: admin only
public class AdminService {

    @PermitAll                  // Method override: all authenticated users
    public List<String> getPublicData() { ... }

    public void adminOnlyAction() { ... }  // Inherits class @RolesAllowed
}
```

### Annotated Vaadin Views

```java
import com.vaadin.flow.router.Route;
import com.vaadin.flow.server.auth.AnonymousAllowed;
import jakarta.annotation.security.PermitAll;
import jakarta.annotation.security.RolesAllowed;

// Accessible by all (not logged-in users too)
@Route("login")
@AnonymousAllowed
public class LoginView extends VerticalLayout { ... }

// Any authenticated user
@Route("dashboard")
@PermitAll
public class DashboardView extends VerticalLayout { ... }

// Only users with ROLE_ADMIN
@Route("admin")
@RolesAllowed("ADMIN")
public class AdminView extends VerticalLayout { ... }

// Explicitly locked (no one can navigate here via Vaadin router)
@Route("internal")
@DenyAll
public class InternalView extends VerticalLayout { ... }
```

---

## Code Examples

### Full Spring Security Configuration (Form Login)

```java
package com.example.security;

import com.vaadin.flow.spring.security.VaadinWebSecurity;
import com.example.views.LoginView;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.util.matcher.AntPathRequestMatcher;

@EnableWebSecurity
@Configuration
public class SecurityConfiguration extends VaadinWebSecurity {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // Allow static resources before Vaadin catches everything
        http.authorizeHttpRequests(auth ->
            auth.requestMatchers(new AntPathRequestMatcher("/images/**")).permitAll()
        );

        super.configure(http);           // Apply Vaadin security defaults
        setLoginView(http, LoginView.class); // Wire login view to NavigationAccessControl
    }

    @Bean
    public UserDetailsService userDetailsService() {
        return new InMemoryUserDetailsManager(
            User.withUsername("user")
                .password(passwordEncoder().encode("secret"))
                .roles("USER")
                .build(),
            User.withUsername("admin")
                .password(passwordEncoder().encode("admin"))
                .roles("USER", "ADMIN")
                .build()
        );
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### Full OAuth2 / OIDC Configuration (Keycloak)

```java
package com.example.security;

import com.vaadin.flow.spring.security.VaadinSecurityConfigurer;
import com.vaadin.flow.spring.security.VaadinAwareSecurityContextHolderStrategyConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@Import(VaadinAwareSecurityContextHolderStrategyConfiguration.class)
public class OidcSecurityConfiguration {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http.with(VaadinSecurityConfigurer.vaadin(), configurer -> {
            configurer.oauth2LoginPage(
                "/oauth2/authorization/keycloak",  // Redirect unauthenticated users here
                "{baseUrl}/session-ended"           // After logout, redirect here
            );
        });
        return http.build();
    }
}
```

### Custom NavigationAccessChecker

```java
package com.example.security;

import com.vaadin.flow.server.auth.NavigationAccessChecker;
import com.vaadin.flow.server.auth.NavigationContext;
import com.vaadin.flow.server.auth.AccessCheckResult;

public class TenantAccessChecker implements NavigationAccessChecker {

    @Override
    public AccessCheckResult check(NavigationContext context) {
        // Custom logic: check tenant membership
        boolean allowed = TenantService.isMember(
            context.getPrincipal(), context.getNavigationTarget()
        );
        if (allowed) {
            return AccessCheckResult.allow();
        }
        return AccessCheckResult.deny("Tenant access denied");
    }
}
```

Register it via the configurer bean:

```java
@Bean
static NavigationAccessControlConfigurer navigationAccessControlConfigurer() {
    return new NavigationAccessControlConfigurer()
        .withNavigationAccessChecker(new TenantAccessChecker());
}
```

### SSO Kit Integration (Vaadin Commercial Add-on)

For production-grade SSO with back-channel logout support, Vaadin offers the commercial **SSO Kit** add-on:

```xml
<!-- pom.xml — requires Vaadin commercial subscription -->
<dependency>
    <groupId>com.vaadin</groupId>
    <artifactId>sso-kit-starter</artifactId>
</dependency>
```

```properties
# application.properties
spring.security.oauth2.client.provider.keycloak.issuer-uri=https://my-keycloak.io/realms/my-realm
spring.security.oauth2.client.registration.keycloak.client-id=my-client
spring.security.oauth2.client.registration.keycloak.client-secret=${KEYCLOAK_SECRET}
spring.security.oauth2.client.registration.keycloak.scope=profile,openid,email,roles

vaadin.sso.login-route=/oauth2/authorization/keycloak
vaadin.sso.back-channel-logout=true
```

### Vaadin UI Unit Testing with Security

```java
import com.vaadin.flow.spring.security.SpringNavigationAccessControl;
import com.vaadin.flow.server.VaadinServiceInitListener;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
class TestViewSecurityConfig {

    @Bean
    VaadinServiceInitListener setupViewSecurityScenario() {
        SpringNavigationAccessControl accessControl = new SpringNavigationAccessControl();
        accessControl.setLoginView(LoginView.class);
        return event -> event.getSource().addUIInitListener(uiEvent ->
            uiEvent.getUI().addBeforeEnterListener(accessControl)
        );
    }
}
```

---

## Spring Boot 4 Compatibility Notes

Vaadin 25 is designed for **Spring Boot 4 / Spring Framework 7**. Key implications:

| Topic | Detail |
|-------|--------|
| **Jakarta namespaces** | All `javax.*` imports replaced with `jakarta.*` (e.g., `jakarta.annotation.security.RolesAllowed`) |
| **Java 21 baseline** | Virtual threads, records, and sealed classes available by default |
| **Spring Security 7** | `HttpSecurity.with(configurer, ...)` API is the primary extension point; `WebSecurityConfigurerAdapter` fully removed |
| **Hibernate 7** | Updated ORM version; potential schema/query migration needed |
| **Servlet containers** | Tomcat 11 / Jetty 12 required |
| **Migration from 24** | Replace Vaadin 24 BOM version; update `spring-boot-starter-parent` version; rename `javax.*` imports; adopt `VaadinSecurityConfigurer` composable style |

### Migrating from Hilla/Hilla-React Starter

```xml
<!-- Before (Hilla/old) -->
<dependency>
    <groupId>dev.hilla</groupId>
    <artifactId>hilla-spring-boot-starter</artifactId>
</dependency>

<!-- After (Vaadin 25) -->
<dependency>
    <groupId>com.vaadin</groupId>
    <artifactId>vaadin-spring-boot-starter</artifactId>
</dependency>
```

---

## Security Architecture Summary

```
HTTP Request
    │
    ▼
Spring Security Filter Chain
    │
    ├── VaadinSecurityConfigurer (or VaadinWebSecurity)
    │       ├── CSRF protection (Vaadin internal requests excluded)
    │       ├── Request cache
    │       └── OAuth2 / Form login endpoint wiring
    │
    ▼
Vaadin Request Handling
    │
    ▼
NavigationAccessControl (BeforeEnterListener)
    │
    ├── AnnotatedViewAccessChecker
    │       └── Reads @AnonymousAllowed / @PermitAll / @RolesAllowed / @DenyAll
    │           on target view class
    │
    ├── RoutePathAccessChecker (optional)
    │       └── Maps route paths to Spring Security URL rules
    │
    └── Custom NavigationAccessChecker(s)
            └── Your domain-specific logic
    │
    ▼
Decision Resolver
    │
    ├── ALLOW → render view
    └── DENY  → redirect to login view (or error view)
```

---

## Notes

- **`@AnonymousAllowed`** is in `com.vaadin.flow.server.auth` package (Vaadin-specific), while `@PermitAll`, `@RolesAllowed`, and `@DenyAll` are from `jakarta.annotation.security` (JSR-250 standard).
- **Default-deny:** Any view class without a security annotation is completely inaccessible via Vaadin navigation. This is intentional to force explicit access decisions.
- **`VaadinAwareSecurityContextHolderStrategyConfiguration`** is required when using `VaadinSecurityConfigurer` (composable style) to ensure the Spring Security context is correctly propagated across Vaadin's asynchronous push events.
- **Circular dependency:** When declaring a `NavigationAccessControlConfigurer` bean inside a `VaadinWebSecurity` subclass that also uses `VaadinSecurityConfigurer`, the bean method **must be `static`**.
- **Vaadin SSO Kit** (commercial) adds back-channel logout, session management, and deeper OIDC integration beyond what the open-source `VaadinSecurityConfigurer.oauth2LoginPage(...)` provides.
- **`RoutePathAccessChecker`** is particularly useful when migrating apps that previously relied solely on Spring Security URL patterns, allowing gradual migration to annotation-based access control.
- For testing, `@WithMockUser` (Spring Security Test) works with Vaadin views in unit tests; for integration tests, use `SpringNavigationAccessControl` with a `VaadinServiceInitListener` bean.

## References

- Vaadin Security Docs: https://vaadin.com/docs/latest/flow/security/enabling-security
- Navigation Access Control: https://vaadin.com/docs/latest/flow/security/advanced-topics/navigation-access-control
- OAuth2 Integration: https://vaadin.com/docs/latest/flow/integrations/spring/oauth2
- SSO Kit: https://vaadin.com/docs/latest/tools/sso/getting-started
- Vaadin Flow GitHub: https://github.com/vaadin/flow
- Vaadin Docs GitHub: https://github.com/vaadin/docs
