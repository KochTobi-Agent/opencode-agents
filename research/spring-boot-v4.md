# Spring Boot (v4)

## Overview

Spring Boot 4 is a major release that requires **Spring Framework 7** and a minimum of **Java 17** (Java 21+ recommended for virtual threads and native image support). It continues the opinionated, auto-configuration-driven philosophy of Boot 3, but ships with Spring Security 7, Jackson 3 as the default JSON library, first-class GraalVM native image support, and a number of removed/replaced APIs that accumulated as `@Deprecated(forRemoval=true)` during the Boot 3.x lifecycle.

**Key version anchors:**
| Component | Version |
|---|---|
| Spring Boot | 4.0.x (GA), 4.0.3-SNAPSHOT |
| Spring Framework | 7.x |
| Spring Security | 7.x |
| Jackson | 3.x (Jackson 2 auto-config deprecated → removed) |
| Minimum Java | 17 |
| Recommended Java | 21+ |

---

## Dependencies

### Maven – Parent POM (recommended)

```xml
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>4.0.3-SNAPSHOT</version>
</parent>
```

### Core Starters

```xml
<!-- Web (Spring MVC + embedded Tomcat) -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- Spring Security (auto-wired form login, HTTP Basic) -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- OAuth2 / OIDC Login (client-side) -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>

<!-- OAuth2 Resource Server (JWT / opaque token) -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

### Gradle (Kotlin DSL)

```kotlin
plugins {
    id("org.springframework.boot") version "4.0.3-SNAPSHOT"
    id("io.spring.dependency-management") version "1.1.7"
    kotlin("jvm") version "2.1.x"
    kotlin("plugin.spring") version "2.1.x"
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-oauth2-client")
}
```

### Migration Helper (Boot 3 → 4)

Add temporarily to detect renamed/removed properties at startup:

```xml
<!-- Maven -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-properties-migrator</artifactId>
  <scope>runtime</scope>
</dependency>
```

```groovy
// Gradle
runtimeOnly("org.springframework.boot:spring-boot-properties-migrator")
```

> **Remove this dependency** once migration is complete — it has a non-trivial startup cost.

---

## OAuth2 / OIDC Configuration

### How Boot 4 Auto-Configuration Works

When `spring-boot-starter-oauth2-client` is on the classpath and at least one `spring.security.oauth2.client.registration.*` is defined, Boot 4 auto-configures:

- A `ClientRegistrationRepository` bean (in-memory by default).
- An `OAuth2AuthorizedClientRepository` (session-backed for servlets, or `ServerOAuth2AuthorizedClientRepository` for reactive).
- A `SecurityFilterChain` that redirects unauthenticated users to the OAuth2/OIDC provider login page — **unless** you define your own `SecurityFilterChain` bean, which disables the Boot default.

### Built-in (Common) Providers

Boot 4 ships with pre-configured defaults for:
`google`, `github`, `facebook`, `okta`

If the registration key matches a known provider name, Boot infers endpoints automatically:

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          github:
            client-id: "${GITHUB_CLIENT_ID}"
            client-secret: "${GITHUB_CLIENT_SECRET}"
          google:
            client-id: "${GOOGLE_CLIENT_ID}"
            client-secret: "${GOOGLE_CLIENT_SECRET}"
```

### GitHub (OAuth2 — not OIDC)

GitHub uses OAuth2 but **not** OpenID Connect (no `openid` scope, no ID token).

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          github:
            client-id: "${GITHUB_CLIENT_ID}"
            client-secret: "${GITHUB_CLIENT_SECRET}"
            scope: read:user, user:email
            # provider defaults are inferred automatically
```

> The authenticated principal will be an `OAuth2User` (not `OidcUser`).
> `user-name-attribute` defaults to `login` for the GitHub provider.

### Google (OIDC)

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: "${GOOGLE_CLIENT_ID}"
            client-secret: "${GOOGLE_CLIENT_SECRET}"
            scope: openid, profile, email
            # Boot infers all Google OIDC endpoints automatically
```

Manual (properties-style) with explicit provider:

```properties
spring.security.oauth2.client.registration.google.client-id=${GOOGLE_CLIENT_ID}
spring.security.oauth2.client.registration.google.client-secret=${GOOGLE_CLIENT_SECRET}
spring.security.oauth2.client.registration.google.scope=openid,profile,email
spring.security.oauth2.client.registration.google.client-authentication-method=client_secret_basic
spring.security.oauth2.client.registration.google.authorization-grant-type=authorization_code
spring.security.oauth2.client.registration.google.redirect-uri={baseUrl}/login/oauth2/code/{registrationId}
```

### Keycloak (OIDC — custom provider)

Keycloak supports OpenID Connect discovery. Configure via `issuer-uri` and Boot will call `{issuer-uri}/.well-known/openid-configuration` to discover all endpoints.

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          keycloak:
            client-id: "${KEYCLOAK_CLIENT_ID}"
            client-secret: "${KEYCLOAK_CLIENT_SECRET}"
            scope: openid, profile, email, roles
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            client-authentication-method: client_secret_basic
        provider:
          keycloak:
            issuer-uri: "https://auth.example.com/realms/my-realm"
            # Discovered automatically from /.well-known/openid-configuration:
            # authorization-uri, token-uri, user-info-uri, jwk-set-uri
            user-name-attribute: preferred_username
```

For Keycloak JWT roles in the token, you can override `user-name-attribute`:

```yaml
            user-name-attribute: sub   # use subject claim as principal name
```

### ORCID (OAuth2 + partial OIDC)

ORCID is an OAuth2 provider with its own user-info endpoint. It is **not** a full OIDC provider by default (no `.well-known/openid-configuration` on the standard path), so endpoints must be specified manually.

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          orcid:
            client-id: "${ORCID_CLIENT_ID}"
            client-secret: "${ORCID_CLIENT_SECRET}"
            scope: openid, /authenticate
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            client-name: "ORCID"
            client-authentication-method: client_secret_post
        provider:
          orcid:
            authorization-uri: "https://orcid.org/oauth/authorize"
            token-uri: "https://orcid.org/oauth/token"
            user-info-uri: "https://orcid.org/oauth/userinfo"
            user-name-attribute: sub
            jwk-set-uri: "https://orcid.org/oauth/jwks"
```

> For the ORCID **sandbox** environment replace `orcid.org` with `sandbox.orcid.org`.

### Full Multi-Provider YAML Example

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          # --- GitHub (OAuth2) ---
          github:
            client-id: "${GITHUB_CLIENT_ID}"
            client-secret: "${GITHUB_CLIENT_SECRET}"
            scope: read:user, user:email

          # --- Google (OIDC, auto-discovered) ---
          google:
            client-id: "${GOOGLE_CLIENT_ID}"
            client-secret: "${GOOGLE_CLIENT_SECRET}"
            scope: openid, profile, email

          # --- Keycloak (OIDC, auto-discovered) ---
          keycloak:
            client-id: "${KEYCLOAK_CLIENT_ID}"
            client-secret: "${KEYCLOAK_CLIENT_SECRET}"
            scope: openid, profile, email
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            client-authentication-method: client_secret_basic

          # --- ORCID (OAuth2 + partial OIDC, manual endpoints) ---
          orcid:
            client-id: "${ORCID_CLIENT_ID}"
            client-secret: "${ORCID_CLIENT_SECRET}"
            scope: openid, /authenticate
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            client-name: "ORCID"
            client-authentication-method: client_secret_post

        provider:
          keycloak:
            issuer-uri: "https://auth.example.com/realms/my-realm"
            user-name-attribute: preferred_username

          orcid:
            authorization-uri: "https://orcid.org/oauth/authorize"
            token-uri: "https://orcid.org/oauth/token"
            user-info-uri: "https://orcid.org/oauth/userinfo"
            jwk-set-uri: "https://orcid.org/oauth/jwks"
            user-name-attribute: sub
```

---

## Security Configuration

### Spring Security 7 — SecurityFilterChain (Servlet)

The `WebSecurityConfigurerAdapter` was **removed** in Spring Security 6 and is **not present** in Security 7. All configuration must use `SecurityFilterChain` beans.

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authorize -> authorize
                .requestMatchers("/", "/login", "/error", "/webjars/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(Customizer.withDefaults())   // enables OIDC/OAuth2 login
            .oauth2Client(Customizer.withDefaults())  // enables token management for API calls
            .logout(logout -> logout
                .logoutSuccessUrl("/")
                .invalidateHttpSession(true)
                .clearAuthentication(true)
            );
        return http.build();
    }
}
```

### Custom OAuth2 Login Page

```java
http
    .oauth2Login(oauth2 -> oauth2
        .loginPage("/login")           // custom login page
        .defaultSuccessUrl("/dashboard", true)
        .failureUrl("/login?error")
    );
```

### Manually Registering Clients (No Properties File)

```java
import org.springframework.security.oauth2.client.registration.ClientRegistration;
import org.springframework.security.oauth2.client.registration.ClientRegistrationRepository;
import org.springframework.security.oauth2.client.registration.InMemoryClientRegistrationRepository;
import org.springframework.security.oauth2.core.AuthorizationGrantType;
import org.springframework.security.oauth2.core.ClientAuthenticationMethod;
import org.springframework.security.oauth2.core.oidc.IdTokenClaimNames;

@Bean
public ClientRegistrationRepository clientRegistrationRepository() {
    return new InMemoryClientRegistrationRepository(
        googleClientRegistration(),
        keycloakClientRegistration()
    );
}

private ClientRegistration googleClientRegistration() {
    // Use CommonOAuth2Provider for built-in providers
    return CommonOAuth2Provider.GOOGLE.getBuilder("google")
        .clientId("google-client-id")
        .clientSecret("google-client-secret")
        .build();
}

private ClientRegistration keycloakClientRegistration() {
    return ClientRegistration.withRegistrationId("keycloak")
        .clientId("my-app")
        .clientSecret("my-secret")
        .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
        .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
        .redirectUri("{baseUrl}/login/oauth2/code/{registrationId}")
        .scope("openid", "profile", "email")
        .authorizationUri("https://auth.example.com/realms/my-realm/protocol/openid-connect/auth")
        .tokenUri("https://auth.example.com/realms/my-realm/protocol/openid-connect/token")
        .userInfoUri("https://auth.example.com/realms/my-realm/protocol/openid-connect/userinfo")
        .userNameAttributeName("preferred_username")
        .jwkSetUri("https://auth.example.com/realms/my-realm/protocol/openid-connect/certs")
        .clientName("Keycloak")
        .build();
}
```

### Full Override (Disabling Boot Auto-Configuration)

If you declare *both* a `SecurityFilterChain` **and** a `ClientRegistrationRepository` bean, Boot's OAuth2 auto-configuration backs off entirely:

```java
@Configuration
public class OAuth2LoginConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .oauth2Login(Customizer.withDefaults());
        return http.build();
    }

    @Bean
    public ClientRegistrationRepository clientRegistrationRepository() {
        return new InMemoryClientRegistrationRepository(/* registrations */);
    }

    @Bean
    public OAuth2AuthorizedClientService authorizedClientService(
            ClientRegistrationRepository clientRegistrationRepository) {
        return new InMemoryOAuth2AuthorizedClientService(clientRegistrationRepository);
    }

    @Bean
    public OAuth2AuthorizedClientRepository authorizedClientRepository(
            OAuth2AuthorizedClientService authorizedClientService) {
        return new AuthenticatedPrincipalOAuth2AuthorizedClientRepository(authorizedClientService);
    }
}
```

### Accessing the Authenticated Principal

```java
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.core.oidc.user.OidcUser;
import org.springframework.security.oauth2.core.user.OAuth2User;

// For OIDC providers (Google, Keycloak, ORCID with openid scope)
@GetMapping("/profile")
public String profile(@AuthenticationPrincipal OidcUser user, Model model) {
    model.addAttribute("name",  user.getFullName());
    model.addAttribute("email", user.getEmail());
    model.addAttribute("sub",   user.getSubject());
    return "profile";
}

// For plain OAuth2 providers (GitHub)
@GetMapping("/profile")
public String profile(@AuthenticationPrincipal OAuth2User user, Model model) {
    model.addAttribute("name",  user.getAttribute("name"));
    model.addAttribute("login", user.getAttribute("login"));
    return "profile";
}
```

### OIDC Back-Channel Logout (Spring Security 7)

Spring Security 7 adds first-class support for [OIDC Back-Channel Logout](https://openid.net/specs/openid-connect-backchannel-1_0.html):

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .oauth2Login(Customizer.withDefaults())
        .oidcLogout(oidc -> oidc
            .backChannel(Customizer.withDefaults())
            // Default endpoint: POST /logout/connect/back-channel/{registrationId}
        );
    return http.build();
}

// Optional: publish as a bean to customize session cookie name or internal URI
@Bean
OidcBackChannelLogoutHandler oidcLogoutHandler(OidcSessionRegistry sessionRegistry) {
    OidcBackChannelLogoutHandler handler = new OidcBackChannelLogoutHandler(sessionRegistry);
    handler.setSessionCookieName("SESSION");   // override if using Spring Session
    // handler.setLogoutUri("http://localhost:8080/logout/connect/back-channel/{registrationId}");
    return handler;
}
```

### OAuth2 + OAuth2 Client Combined

```java
http
    .oauth2Login(Customizer.withDefaults())   // login via provider
    .oauth2Client(Customizer.withDefaults()); // manage tokens for downstream API calls
```

---

## Auto-Configuration Changes (Boot 3 → Boot 4)

### Removed / Changed Auto-Configurations

| Boot 3 | Boot 4 Status |
|---|---|
| `Jackson2AutoConfiguration` | **Deprecated → to be removed in 4.3.0**; use Jackson 3 auto-config |
| `Jackson2EndpointAutoConfiguration` (Actuator) | **Deprecated for removal** as of 4.0.0 |
| `WebSecurityConfigurerAdapter` (Security) | **Removed** (was removed in Security 6; not present in Security 7) |
| `spring.security.oauth2.client.*` property namespace | **Unchanged** — same namespace, same keys |
| `OutputCaptureRule` (JUnit 4 test rule) | **Deprecated for removal** 4.0.0 — use JUnit 5 `OutputCaptureExtension` |

### Jackson 2 → Jackson 3 Migration

Boot 4 defaults to Jackson 3. Jackson 2 support is provided via a compatibility layer but deprecated:

- `Jackson2Properties.getLocale()` — deprecated, replaced by Jackson 3 equivalent.
- `Jackson2Properties.getDatatype()` — deprecated.
- `Jackson2AutoConfiguration` — marked `@Deprecated(since="4.0.0", forRemoval=true)`.

To use Jackson 3 explicitly, simply do **nothing** — it is the default. If you need Jackson 2 for legacy compatibility, include `jackson-databind` 2.x explicitly, but expect deprecation warnings.

### Java Version Requirements

Boot 4 recognises the following `JavaVersion` enum constants (from `org.springframework.boot.system.JavaVersion`):

`SEVENTEEN`, `EIGHTEEN`, `NINETEEN`, `TWENTY`, `TWENTY_ONE`, `TWENTY_TWO`, `TWENTY_THREE`, `TWENTY_FOUR`, `TWENTY_FIVE`, `TWENTY_SIX`

- **Minimum**: Java 17 (`SEVENTEEN`)
- **Recommended**: Java 21 (`TWENTY_ONE`) — required for virtual threads and Project Loom features.

### Property Migration

Use `spring-boot-properties-migrator` (runtime scope) to detect renamed/removed properties. It prints warnings at startup for each deprecated key with its replacement.

Common renames from Boot 3.x:
- Properties under `spring.security.oauth2.*` are **stable** — no changes.
- `spring.jackson.*` properties may have Jackson-3-era equivalents.

---

## Migration Notes (Boot 3 → Boot 4)

### Step-by-Step Upgrade Checklist

1. **Upgrade Java** to 17 minimum (21 recommended).
2. **Update the `spring-boot-starter-parent`** version to `4.0.x`.
3. **Add `spring-boot-properties-migrator`** (runtime scope) temporarily — remove after resolving all warnings.
4. **Replace `WebSecurityConfigurerAdapter`** — already mandatory since Boot 3; if you skipped this, do it now. Use `SecurityFilterChain` beans.
5. **Migrate Jackson 2 → Jackson 3** if you rely on `Jackson2ObjectMapperBuilder` or `Jackson2AutoConfiguration`. Check for removed `getLocale()` / `getDatatype()` usages.
6. **Switch JUnit 4 → JUnit 5** — `OutputCaptureRule` and other JUnit 4 test rules are removed.
7. **Review `@Deprecated(forRemoval=true)` usages** from Boot 3 — all are expected to be removed in Boot 4.x or 4.3.0.

### Security Configuration Migration

```java
// ❌ Boot 2 / early Boot 3 style (REMOVED in Security 7)
@Configuration
public class OldSecurity extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.oauth2Login();
    }
}

// ✅ Boot 3 / Boot 4 style (SecurityFilterChain bean)
@Configuration
@EnableWebSecurity
public class NewSecurity {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .oauth2Login(Customizer.withDefaults());
        return http.build();
    }
}
```

### OAuth2 / OIDC — No Breaking Changes

The `spring.security.oauth2.client.*` property namespace and the `spring-boot-starter-oauth2-client` starter are **fully compatible** between Boot 3 and Boot 4. No property renames or starter renames are required for standard OAuth2/OIDC login setups.

### Native Image (GraalVM)

Boot 4 with Spring Framework 7 ships improved native image hints. For OAuth2 workloads:
- No special configuration needed — Spring Security and OAuth2 client are natively supported.
- Use `spring-boot-starter-parent` 4.x; the build plugin handles AOT processing.

### Reactive (WebFlux) Counterparts

For reactive apps, replace:
- `HttpSecurity` → `ServerHttpSecurity`
- `SecurityFilterChain` → `SecurityWebFilterChain`
- `OAuth2AuthorizedClientRepository` → `ServerOAuth2AuthorizedClientRepository`
- `OidcBackChannelLogoutHandler` → `OidcBackChannelServerLogoutHandler`

```java
@Bean
public SecurityWebFilterChain springSecurityFilterChain(ServerHttpSecurity http) {
    http
        .authorizeExchange(ex -> ex.anyExchange().authenticated())
        .oauth2Login(Customizer.withDefaults())
        .oidcLogout(oidc -> oidc.backChannel(Customizer.withDefaults()));
    return http.build();
}
```

---

## Key APIs

| API | Purpose |
|---|---|
| `SecurityFilterChain` | Core security configuration unit (replaces `WebSecurityConfigurerAdapter`) |
| `HttpSecurity.oauth2Login(Customizer)` | Enable OAuth2/OIDC login flow |
| `HttpSecurity.oauth2Client(Customizer)` | Enable OAuth2 authorized client management |
| `HttpSecurity.oidcLogout(Customizer)` | Enable OIDC back-channel / front-channel logout |
| `ClientRegistrationRepository` | Stores `ClientRegistration` definitions (providers) |
| `InMemoryClientRegistrationRepository` | Default in-memory implementation |
| `CommonOAuth2Provider` | Enum with built-in Google, GitHub, Facebook, Okta defaults |
| `OAuth2AuthorizedClientService` | Manages per-user authorized client tokens |
| `OAuth2AuthorizedClientRepository` | HTTP-request-scoped access to authorized clients |
| `OidcBackChannelLogoutHandler` | Handles provider-initiated back-channel logout |
| `OidcSessionRegistry` | Tracks active OIDC sessions for back-channel logout |
| `@AuthenticationPrincipal OidcUser` | Inject OIDC principal (has ID token, claims) |
| `@AuthenticationPrincipal OAuth2User` | Inject OAuth2 principal (attributes from userinfo) |

---

## Notes

- **`issuer-uri` vs. manual endpoints**: Always prefer `issuer-uri` for OIDC-compliant providers (Keycloak, Google, Okta). Boot will perform the `.well-known/openid-configuration` discovery call at startup. For providers without OIDC discovery (GitHub, ORCID), specify endpoints manually.
- **`client-authentication-method`**: Defaults to `client_secret_basic` (HTTP Basic with `Authorization` header). Use `client_secret_post` for providers that require credentials in the POST body (e.g., ORCID).
- **`user-name-attribute`**: Determines which claim is used as `Principal.getName()`. Defaults differ per provider: `sub` (OIDC standard), `login` (GitHub), `preferred_username` (Keycloak).
- **PKCE**: Supported natively for public clients. Set `client-authentication-method: none` and Boot/Security will enable PKCE automatically for `authorization_code` grant.
- **Session vs. JWT**: `spring-boot-starter-oauth2-client` is a *client* — it stores tokens in the HTTP session (or Redis via Spring Session). It is **not** a resource server. For JWT validation of incoming requests, also add `spring-boot-starter-oauth2-resource-server`.
- **Boot 4 snapshot coordinates**: As of early 2026, the GA artifact is `4.0.x`; `4.0.3-SNAPSHOT` refers to the rolling snapshot build. Check https://repo.spring.io/snapshot for availability.

---

## References

- [Spring Boot 4.0 Reference Documentation](https://docs.spring.io/spring-boot/4.0/-SNAPSHOT/reference/)
- [Spring Boot 4.0 Security — OAuth2 Login](https://docs.spring.io/spring-boot/4.0/-SNAPSHOT/reference/web/spring-security)
- [Spring Security 7 Reference — OAuth2 Login](https://docs.spring.io/spring-security/reference/7.0/servlet/oauth2/index)
- [Spring Security 7 Reference — OIDC Logout](https://docs.spring.io/spring-security/reference/7.0/-SNAPSHOT/servlet/oauth2/login/logout)
- [Spring Security 7 Javadoc](https://docs.spring.io/spring-security/reference/7.0/api/)
- [Spring Boot Properties Migrator](https://docs.spring.io/spring-boot/4.0/-SNAPSHOT/appendix/deprecated-application-properties/index)
