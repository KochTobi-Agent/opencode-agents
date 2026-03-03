# Vaadin 25 + Spring Boot 4 — OIDC Integration Guide

## Overview

This guide covers end-to-end integration of OpenID Connect (OIDC) and OAuth2 login into a
**Vaadin 25** application running on **Spring Boot 4** / **Spring Security 7**.

### How the Stack Fits Together

Vaadin 25 is deliberately co-designed with Spring Boot 4 and Spring Framework 7. When you add
`vaadin-spring-boot-starter` together with `spring-boot-starter-oauth2-client`, Spring Boot's
auto-configuration wires up:

- A `ClientRegistrationRepository` from your `application.yaml` provider definitions.
- Spring Security's OAuth2 login filter chain.
- Vaadin's `NavigationAccessControl` (the `BeforeEnterListener` that enforces view-level
  security annotations).
- Vaadin's CSRF protection, which transparently excludes internal Vaadin push/UIDL requests
  from the standard Spring CSRF check.

You supply:

- A `@Configuration` class using `VaadinSecurityConfigurer` (the Vaadin 25 composable style).
- Provider credentials in `application.yaml`.
- Security annotations (`@AnonymousAllowed`, `@PermitAll`, `@RolesAllowed`) on each `@Route` view.

### Architecture: Request Flow

```
Browser
  │
  │  GET /dashboard  (unauthenticated)
  ▼
┌────────────────────────────────────────────────────────────┐
│              Spring Security Filter Chain                  │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  VaadinSecurityConfigurer                            │  │
│  │  ├─ Registers Vaadin CSRF token strategy             │  │
│  │  │    (excludes /vaadinServlet/**, UIDL endpoints)   │  │
│  │  ├─ Configures request cache (saves original URL)    │  │
│  │  ├─ Wires oauth2Login() redirect endpoint            │  │
│  │  └─ Sets session-ended logout redirect URI           │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  User not authenticated → redirect to:                     │
│    /oauth2/authorization/{registrationId}                  │
└────────────────────────────────────────────────────────────┘
  │
  │  Redirect to Identity Provider (Keycloak / Google / etc.)
  ▼
┌──────────────────────────┐
│   Identity Provider      │
│   (OIDC Authorization    │
│    Server)               │
│  User authenticates,     │
│  provider issues         │
│  authorization code      │
└──────────────────────────┘
  │
  │  Redirect back: /login/oauth2/code/{registrationId}?code=…
  ▼
┌────────────────────────────────────────────────────────────┐
│              Spring Security Filter Chain                  │
│  ├─ Exchanges code for tokens (token endpoint)             │
│  ├─ Fetches userinfo (userinfo endpoint)                   │
│  ├─ Populates SecurityContext with OidcUser / OAuth2User   │
│  └─ Restores original /dashboard URL from request cache    │
└────────────────────────────────────────────────────────────┘
  │
  │  GET /dashboard  (now authenticated)
  ▼
┌────────────────────────────────────────────────────────────┐
│           Vaadin Request Handling                          │
│                                                            │
│  NavigationAccessControl  (BeforeEnterListener)            │
│  │                                                         │
│  ├─ AnnotatedViewAccessChecker                             │
│  │    Reads @AnonymousAllowed / @PermitAll / @RolesAllowed │
│  │    / @DenyAll from DashboardView class                  │
│  │                                                         │
│  ├─ RoutePathAccessChecker  (optional, if enabled)         │
│  │    Maps route paths to HttpSecurity URL-pattern rules   │
│  │                                                         │
│  └─ Decision Resolver                                      │
│       ALLOW  →  render DashboardView                       │
│       DENY   →  redirect to login (or /error)              │
└────────────────────────────────────────────────────────────┘
```

---

## 1. Required Dependencies

### Maven — pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
           https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.example</groupId>
  <artifactId>vaadin-oidc-app</artifactId>
  <version>1.0-SNAPSHOT</version>
  <packaging>jar</packaging>

  <properties>
    <java.version>21</java.version>
    <vaadin.version>25.0.0</vaadin.version>
  </properties>

  <!-- Spring Boot 4 parent — pulls in Spring Security 7 -->
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>4.0.0</version>
  </parent>

  <!-- Vaadin BOM — aligns all vaadin-* artifact versions -->
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
    <!-- Vaadin Flow + Spring Boot auto-configuration -->
    <dependency>
      <groupId>com.vaadin</groupId>
      <artifactId>vaadin-spring-boot-starter</artifactId>
      <!-- version managed by vaadin-bom -->
    </dependency>

    <!-- Spring Security 7 (pulled in transitively, but explicit for clarity) -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-security</artifactId>
    </dependency>

    <!-- OAuth2 / OIDC client support -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-oauth2-client</artifactId>
    </dependency>

    <!-- Test dependencies -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.springframework.security</groupId>
      <artifactId>spring-security-test</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <!-- Vaadin Maven plugin: bundles frontend assets at build time -->
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

### Gradle — build.gradle.kts

```kotlin
plugins {
    id("org.springframework.boot") version "4.0.0"
    id("io.spring.dependency-management") version "1.1.7"
    id("com.vaadin") version "25.0.0"           // Vaadin Gradle plugin
    java
}

group = "com.example"
version = "1.0-SNAPSHOT"

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

// Import the Vaadin BOM so all vaadin-* versions are aligned
dependencyManagement {
    imports {
        mavenBom("com.vaadin:vaadin-bom:25.0.0")
    }
}

dependencies {
    // Vaadin Flow + Spring Boot auto-configuration
    implementation("com.vaadin:vaadin-spring-boot-starter")

    // Spring Security 7
    implementation("org.springframework.boot:spring-boot-starter-security")

    // OAuth2 / OIDC client support
    implementation("org.springframework.boot:spring-boot-starter-oauth2-client")

    // Testing
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.security:spring-security-test")
}
```

> **Note — Spring Boot 4 ships Spring Security 7.**  
> `WebSecurityConfigurerAdapter` was removed in Security 6 and is entirely absent in Security 7.
> All security must be declared via `SecurityFilterChain` beans. Vaadin 25 wraps this with
> `VaadinSecurityConfigurer` (composable) or `VaadinWebSecurity` (inheritance, still supported).
> For new projects, use the composable style shown below.

---

## 2. Security Configuration Class

Use `VaadinSecurityConfigurer.vaadin()` as a composable `SecurityConfigurer` with Spring's
`HttpSecurity.with(...)`. This is the **recommended pattern for Vaadin 25** — it avoids
inheritance and aligns with modern Spring Security 6/7 conventions.

```java
package com.example.security;

import com.vaadin.flow.spring.security.VaadinAwareSecurityContextHolderStrategyConfiguration;
import com.vaadin.flow.spring.security.VaadinSecurityConfigurer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;

/**
 * Security configuration for Vaadin 25 + Spring Boot 4 with OIDC login.
 *
 * Key points:
 *  - @Import(VaadinAwareSecurityContextHolderStrategyConfiguration.class) is REQUIRED when
 *    using the composable VaadinSecurityConfigurer. It ensures the Spring Security context
 *    is correctly propagated across Vaadin's asynchronous server push events.
 *  - Do NOT extend VaadinWebSecurity here — that is the old inheritance-based approach.
 *  - oauth2LoginPage() takes two arguments:
 *      (1) the OAuth2 authorization redirect URL (Spring's internal endpoint)
 *      (2) the URI to redirect to after the user's session ends (post-logout)
 */
@Configuration
@EnableWebSecurity
@Import(VaadinAwareSecurityContextHolderStrategyConfiguration.class)
public class SecurityConfiguration {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http.with(VaadinSecurityConfigurer.vaadin(), configurer -> {
            configurer.oauth2LoginPage(
                "/oauth2/authorization/keycloak", // (1) redirect unauthenticated users here
                "{baseUrl}/session-ended"         // (2) redirect here after logout
            );
        });
        return http.build();
    }
}
```

### What `VaadinSecurityConfigurer` Does for You

When `http.with(VaadinSecurityConfigurer.vaadin(), ...)` is applied, Vaadin automatically
configures all of the following — you do **not** need to set these manually:

| Concern | What Vaadin Configures |
|---|---|
| CSRF | Excludes Vaadin's internal UIDL/push endpoints from Spring's CSRF check; keeps CSRF enabled for all other requests |
| Request cache | Saves the original URL before the OAuth2 redirect, restores it after successful login |
| `NavigationAccessControl` | Registers the `BeforeEnterListener` that evaluates `@AnonymousAllowed`, `@PermitAll`, `@RolesAllowed` |
| Unauthenticated redirect | Sends unauthenticated Vaadin navigation attempts to the configured login endpoint |
| Session ended page | Registers the `/session-ended` route (or your custom URI) as the post-logout landing page |

### Multi-Provider Variant

If you register multiple OAuth2 providers (e.g., Keycloak + GitHub), you need a custom login
page to let the user pick a provider. Set `loginPage()` to your Vaadin login view's route:

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http.with(VaadinSecurityConfigurer.vaadin(), configurer -> {
        // Point to a Vaadin view annotated with @AnonymousAllowed
        // that renders provider-selection buttons
        configurer.loginPage("/login");
    });
    // oauth2Login is still needed so Spring processes the callback
    http.oauth2Login(oauth2 -> oauth2
        .loginPage("/login")
        .defaultSuccessUrl("/dashboard", true)
    );
    return http.build();
}
```

---

## 3. Provider Configuration (application.yaml)

### GitHub — OAuth2 (NOT full OIDC)

> **⚠️ Limitation:** GitHub does **not** support the `openid` scope. It does not issue ID
> tokens and has no OIDC discovery endpoint. The authenticated principal will be an `OAuth2User`,
> **not** an `OidcUser`. There is no `sub` claim from an ID token — use the `id` attribute
> (GitHub's numeric user ID) as a stable identifier. See §6 for how to read this.

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          github:
            client-id: "${GITHUB_CLIENT_ID}"
            client-secret: "${GITHUB_CLIENT_SECRET}"
            # openid is NOT a valid GitHub scope — omit it
            scope: read:user, user:email
            # provider defaults (auth-uri, token-uri, userinfo-uri) are
            # inferred automatically from the 'github' registration key
```

---

### Keycloak — Self-Hosted Full OIDC

> **Recommended:** Use `issuer-uri`. Spring Boot will call
> `{issuer-uri}/.well-known/openid-configuration` at startup to discover all endpoints
> automatically. The authenticated principal is an `OidcUser`.

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
            # Auto-discovers: authorization-uri, token-uri, user-info-uri, jwk-set-uri
            issuer-uri: "https://auth.example.com/realms/my-realm"
            # 'preferred_username' is Keycloak's human-readable username claim
            user-name-attribute: preferred_username
```

> **Manual endpoint alternative** (use when `issuer-uri` discovery is blocked, e.g. in
> internal networks where the app cannot reach the Keycloak host at startup):

```yaml
        provider:
          keycloak:
            authorization-uri: "https://auth.example.com/realms/my-realm/protocol/openid-connect/auth"
            token-uri:         "https://auth.example.com/realms/my-realm/protocol/openid-connect/token"
            user-info-uri:     "https://auth.example.com/realms/my-realm/protocol/openid-connect/userinfo"
            jwk-set-uri:       "https://auth.example.com/realms/my-realm/protocol/openid-connect/certs"
            user-name-attribute: preferred_username
```

---

### ORCID — Academic OIDC (Manual Endpoints + `client_secret_post`)

> **⚠️ ORCID specifics:**
> - No standard OIDC discovery (`.well-known/openid-configuration` is not at the typical path),
>   so all endpoints must be specified manually.
> - ORCID requires credentials to be sent in the POST **body**, not the `Authorization` header.
>   This means `client-authentication-method: client_secret_post` is **mandatory**.
> - The `sub` claim contains the ORCID iD (e.g., `0000-0002-1825-0097`).
> - For sandbox testing replace `orcid.org` with `sandbox.orcid.org`.

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
            # CRITICAL: ORCID requires credentials in the POST body, not Basic auth header
            client-authentication-method: client_secret_post
        provider:
          orcid:
            authorization-uri: "https://orcid.org/oauth/authorize"
            token-uri:         "https://orcid.org/oauth/token"
            user-info-uri:     "https://orcid.org/oauth/userinfo"
            jwk-set-uri:       "https://orcid.org/oauth/jwks"
            # 'sub' is the ORCID iD (e.g. 0000-0002-1825-0097)
            user-name-attribute: sub
```

---

### Google — Full OIDC (Auto-Configured)

> Spring Boot has built-in defaults for Google. All endpoints are discovered automatically.
> The authenticated principal is an `OidcUser` with `email`, `name`, `picture`, and `sub` claims.

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: "${GOOGLE_CLIENT_ID}"
            client-secret: "${GOOGLE_CLIENT_SECRET}"
            # 'openid' is required to get an OidcUser (ID token)
            scope: openid, profile, email
            # All Google endpoints are inferred from the 'google' registration key
```

---

### Complete Multi-Provider Example

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

          google:
            client-id: "${GOOGLE_CLIENT_ID}"
            client-secret: "${GOOGLE_CLIENT_SECRET}"
            scope: openid, profile, email

          keycloak:
            client-id: "${KEYCLOAK_CLIENT_ID}"
            client-secret: "${KEYCLOAK_CLIENT_SECRET}"
            scope: openid, profile, email, roles
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            client-authentication-method: client_secret_basic

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
            token-uri:         "https://orcid.org/oauth/token"
            user-info-uri:     "https://orcid.org/oauth/userinfo"
            jwk-set-uri:       "https://orcid.org/oauth/jwks"
            user-name-attribute: sub
```

---

## 4. Securing Vaadin Views

Security annotations are placed directly on `@Route`-annotated view classes.
`NavigationAccessControl` evaluates them as a `BeforeEnterListener` before the view renders.

### Annotation Reference

| Annotation | Import Package | Behaviour |
|---|---|---|
| `@AnonymousAllowed` | `com.vaadin.flow.server.auth` | Any visitor — authenticated or not — may access the view |
| `@PermitAll` | `jakarta.annotation.security` | Any **authenticated** user may access the view |
| `@RolesAllowed({"ROLE_X"})` | `jakarta.annotation.security` | Only users holding the specified role(s) may access the view |
| `@DenyAll` | `jakarta.annotation.security` | No one can access the view via Vaadin navigation (explicit lockout) |

> **Default-deny:** A view class with **no annotation** is treated as `@DenyAll`.
> This is intentional — every view must have an explicit access decision.

### View Examples

```java
package com.example.views;

import com.vaadin.flow.component.html.H1;
import com.vaadin.flow.component.login.LoginOverlay;
import com.vaadin.flow.component.orderedlayout.VerticalLayout;
import com.vaadin.flow.router.BeforeEnterEvent;
import com.vaadin.flow.router.BeforeEnterObserver;
import com.vaadin.flow.router.PageTitle;
import com.vaadin.flow.router.Route;
// ✅ Vaadin-specific annotation — from com.vaadin.flow.server.auth
import com.vaadin.flow.server.auth.AnonymousAllowed;
// ✅ JSR-250 annotations — from jakarta.annotation.security (NOT javax.annotation.security)
import jakarta.annotation.security.PermitAll;
import jakarta.annotation.security.RolesAllowed;

// --- Public login redirect page: anyone may visit ---
@Route("login")
@PageTitle("Sign In")
@AnonymousAllowed   // REQUIRED — without this, unauthenticated users loop forever
public class LoginRedirectView extends VerticalLayout {
    public LoginRedirectView() {
        // Redirect browser to OAuth2 authorization endpoint
        getElement().executeJs(
            "window.location.href = '/oauth2/authorization/keycloak';"
        );
    }
}

// --- Open landing page ---
@Route("")
@PageTitle("Welcome")
@AnonymousAllowed
public class HomeView extends VerticalLayout {
    public HomeView() {
        add(new H1("Welcome! Please log in."));
    }
}

// --- Any authenticated user ---
@Route("dashboard")
@PageTitle("Dashboard")
@PermitAll
public class DashboardView extends VerticalLayout {
    public DashboardView() {
        add(new H1("Dashboard"));
    }
}

// --- Admins only ---
@Route("admin")
@PageTitle("Admin Panel")
@RolesAllowed("ADMIN")   // matches GrantedAuthority "ROLE_ADMIN" or "ADMIN" depending on config
public class AdminView extends VerticalLayout {
    public AdminView() {
        add(new H1("Admin Panel"));
    }
}

// --- No one can navigate here via Vaadin router ---
@Route("internal")
@jakarta.annotation.security.DenyAll
public class InternalView extends VerticalLayout { }
```

### NavigationAccessControl — Custom Configuration

By default, `VaadinSecurityConfigurer` activates `AnnotatedViewAccessChecker`. You can
customize the checker pipeline via a `NavigationAccessControlConfigurer` bean:

```java
package com.example.security;

import com.vaadin.flow.server.auth.NavigationAccessControlConfigurer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class NavigationAccessConfig {

    /**
     * The @Bean method MUST be static when it is declared in the same @Configuration
     * class as VaadinSecurityConfigurer to avoid a circular dependency.
     */
    @Bean
    static NavigationAccessControlConfigurer navigationAccessControlConfigurer() {
        return new NavigationAccessControlConfigurer()
            // Enable annotation-based checking (this is the default — usually not needed)
            // .withAnnotatedViewAccessChecker()

            // Also enable URL-pattern-based checking (useful for gradual migration)
            .withRoutePathAccessChecker();
    }
}
```

---

## 5. Accessing the Authenticated User

### In a Vaadin Component (UI thread)

Vaadin components run on the UI thread where the `SecurityContext` is always available.
Use `SecurityContextHolder` directly:

```java
package com.example.views;

import com.vaadin.flow.component.html.Paragraph;
import com.vaadin.flow.component.orderedlayout.VerticalLayout;
import com.vaadin.flow.router.Route;
import jakarta.annotation.security.PermitAll;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.oauth2.core.oidc.user.OidcUser;
import org.springframework.security.oauth2.core.user.OAuth2User;

@Route("profile")
@PermitAll
public class ProfileView extends VerticalLayout {

    public ProfileView() {
        Authentication authentication = SecurityContextHolder
            .getContext()
            .getAuthentication();

        if (authentication == null || !authentication.isAuthenticated()) {
            add(new Paragraph("Not authenticated."));
            return;
        }

        Object principal = authentication.getPrincipal();

        if (principal instanceof OidcUser oidcUser) {
            // Full OIDC user (Keycloak, Google, ORCID with openid scope)
            add(new Paragraph("Name:    " + oidcUser.getFullName()));
            add(new Paragraph("Email:   " + oidcUser.getEmail()));
            add(new Paragraph("Subject: " + oidcUser.getSubject()));
            // ID token is also available:
            // oidcUser.getIdToken().getTokenValue()

        } else if (principal instanceof OAuth2User oauth2User) {
            // Plain OAuth2 user (GitHub — no OidcUser, no ID token)
            // GitHub attributes: login, name, email, id (numeric), avatar_url
            add(new Paragraph("Login: " + oauth2User.getAttribute("login")));
            add(new Paragraph("Name:  " + oauth2User.getAttribute("name")));
            // Use the numeric 'id' as a stable identifier (GitHub has no 'sub' claim)
            Integer githubId = oauth2User.getAttribute("id");
            add(new Paragraph("GitHub ID: " + githubId));
        }
    }
}
```

### In a Spring Bean (Service / Repository layer)

Use `@AuthenticationPrincipal` in Spring MVC controllers, or inject `SecurityContext` via
a helper in non-web beans:

```java
package com.example.service;

import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.core.oidc.user.OidcUser;
import org.springframework.security.oauth2.core.user.OAuth2User;
import org.springframework.stereotype.Component;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @AuthenticationPrincipal works in @RestController / @Controller methods.
 * It does NOT work as a parameter injected into a Vaadin view constructor —
 * Vaadin instantiates views itself, not Spring MVC.
 */
@RestController
public class UserProfileController {

    // OIDC providers (Keycloak, Google, ORCID)
    @GetMapping("/api/me/oidc")
    public String oidcProfile(@AuthenticationPrincipal OidcUser user) {
        return "Hello, " + user.getFullName() + " <" + user.getEmail() + ">";
    }

    // OAuth2-only providers (GitHub)
    @GetMapping("/api/me/oauth2")
    public String oauth2Profile(@AuthenticationPrincipal OAuth2User user) {
        return "Hello, " + user.getAttribute("login");
    }
}
```

### Via Vaadin's `AuthenticationContext` Helper (Recommended for Vaadin views)

Vaadin provides `AuthenticationContext` as a convenience Spring bean for use inside views
and layouts. It wraps `SecurityContextHolder` in a Vaadin-aware way:

```java
package com.example.views;

import com.vaadin.flow.component.applayout.AppLayout;
import com.vaadin.flow.component.button.Button;
import com.vaadin.flow.component.html.Span;
import com.vaadin.flow.component.orderedlayout.HorizontalLayout;
import com.vaadin.flow.server.auth.AuthenticationContext;
import org.springframework.security.oauth2.core.oidc.user.OidcUser;

public class MainLayout extends AppLayout {

    private final transient AuthenticationContext authContext;

    public MainLayout(AuthenticationContext authContext) {
        this.authContext = authContext;

        authContext.getAuthenticatedUser(OidcUser.class).ifPresent(user -> {
            Span welcome = new Span("Welcome, " + user.getFullName());
            Button logout = new Button("Sign out", e -> authContext.logout());
            addToNavbar(new HorizontalLayout(welcome, logout));
        });
    }
}
```

### VaadinSession Caveats

- **Never cache** `OidcUser` or `OAuth2User` in a `VaadinSession` attribute. The session can
  outlive the `SecurityContext` during async push operations, leading to stale user data.
- **Always read** the principal from `SecurityContextHolder` at the moment you need it.
- **`@PushStateNavigation`** routes pass through `NavigationAccessControl` on each navigation
  event; the security context is re-checked on every view change.

---

## 6. Logout Configuration

### OIDC Back-Channel Logout (Spring Security 7)

Spring Security 7 provides first-class OIDC back-channel logout support. When the Identity
Provider (e.g., Keycloak) terminates a session server-to-server, it sends a logout token to
your app's back-channel endpoint, and Spring Security invalidates the corresponding session.

```java
package com.example.security;

import com.vaadin.flow.spring.security.VaadinAwareSecurityContextHolderStrategyConfiguration;
import com.vaadin.flow.spring.security.VaadinSecurityConfigurer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.oauth2.client.oidc.web.server.logout.OidcBackChannelLogoutHandler;
import org.springframework.security.oauth2.client.oidc.web.logout.OidcSessionRegistry;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
@Import(VaadinAwareSecurityContextHolderStrategyConfiguration.class)
public class SecurityConfiguration {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http.with(VaadinSecurityConfigurer.vaadin(), configurer -> {
            configurer.oauth2LoginPage(
                "/oauth2/authorization/keycloak",
                "{baseUrl}/session-ended"
            );
        });

        // Enable OIDC back-channel logout
        // Default endpoint: POST /logout/connect/back-channel/{registrationId}
        // Register this URL in your Keycloak client's "Backchannel Logout URL" field.
        http.oidcLogout(oidc -> oidc
            .backChannel(Customizer.withDefaults())
        );

        return http.build();
    }

    /**
     * Optional: customize the back-channel logout handler.
     * Needed if you use Spring Session with a non-default cookie name.
     */
    @Bean
    public OidcBackChannelLogoutHandler oidcLogoutHandler(OidcSessionRegistry sessionRegistry) {
        OidcBackChannelLogoutHandler handler = new OidcBackChannelLogoutHandler(sessionRegistry);
        // Override if you use Spring Session Redis and renamed the session cookie:
        // handler.setSessionCookieName("SESSION");
        return handler;
    }
}
```

### Keycloak Back-Channel Logout Setup

In your Keycloak realm → Clients → `my-client` → Settings:

```
Backchannel Logout URL:  https://your-app.example.com/logout/connect/back-channel/keycloak
Backchannel Logout Session Required: ON
```

### Front-Channel (Browser-Initiated) Logout

When a user clicks "Sign out" in the Vaadin UI, use `AuthenticationContext.logout()`:

```java
Button logoutButton = new Button("Sign out", e -> authContext.logout());
```

This triggers Spring Security's logout handler, which:
1. Invalidates the HTTP session.
2. Clears the `SecurityContext`.
3. Redirects the browser to the logout URL configured in `oauth2LoginPage()` — i.e., the
   OIDC provider's logout endpoint if configured, or `{baseUrl}/session-ended` by default.

### Session-Ended View

Create a simple Vaadin view for the post-logout landing page:

```java
package com.example.views;

import com.vaadin.flow.component.html.H2;
import com.vaadin.flow.component.html.Paragraph;
import com.vaadin.flow.component.orderedlayout.VerticalLayout;
import com.vaadin.flow.router.PageTitle;
import com.vaadin.flow.router.Route;
import com.vaadin.flow.server.auth.AnonymousAllowed;

@Route("session-ended")
@PageTitle("Session Ended")
@AnonymousAllowed   // user is logged out — must be anonymous
public class SessionEndedView extends VerticalLayout {

    public SessionEndedView() {
        add(
            new H2("You have been signed out."),
            new Paragraph("Your session has ended.")
        );
    }
}
```

---

## 7. Common Pitfalls

### ❌ Pitfall 1 — `javax` vs `jakarta` for `@RolesAllowed`

Vaadin 25 targets **Jakarta EE 11**. All `javax.*` packages have been replaced by `jakarta.*`.
Using the wrong import **silently fails** — Vaadin's `AnnotatedViewAccessChecker` will not
find the annotation and will apply default-deny.

```java
// ❌ WRONG — javax is Jakarta EE 8 / Spring Boot 2 era; not recognized by Vaadin 25
import javax.annotation.security.RolesAllowed;

// ✅ CORRECT — jakarta is Jakarta EE 9+ / Spring Boot 3+ / Vaadin 24+
import jakarta.annotation.security.RolesAllowed;

// NOTE: @AnonymousAllowed is Vaadin-specific and is always:
import com.vaadin.flow.server.auth.AnonymousAllowed;
// (it has no javax/jakarta equivalent)
```

---

### ❌ Pitfall 2 — Using `VaadinWebSecurity` (Inheritance) vs `VaadinSecurityConfigurer` (Composable)

Both approaches work in Vaadin 25, but they should **not be mixed**:

```java
// ❌ WRONG — mixing inheritance and composable in one class
@Configuration
public class SecurityConfig extends VaadinWebSecurity {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        super.configure(http);
        // Don't also call http.with(VaadinSecurityConfigurer.vaadin(), ...)
        // This applies Vaadin's security defaults twice → unpredictable behaviour
        http.with(VaadinSecurityConfigurer.vaadin(), c -> { ... }); // ❌
    }
}

// ✅ CORRECT — choose ONE approach
// Option A: VaadinWebSecurity (inheritance, classic)
@Configuration
public class SecurityConfig extends VaadinWebSecurity {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        super.configure(http);
        setLoginView(http, LoginView.class);
    }
}

// Option B: VaadinSecurityConfigurer (composable, Vaadin 25 recommended)
@Configuration
@Import(VaadinAwareSecurityContextHolderStrategyConfiguration.class)
public class SecurityConfig {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http.with(VaadinSecurityConfigurer.vaadin(), c ->
            c.oauth2LoginPage("/oauth2/authorization/keycloak", "{baseUrl}/session-ended")
        );
        return http.build();
    }
}
```

---

### ❌ Pitfall 3 — GitHub Does NOT Support `openid` Scope

GitHub implements OAuth2 but **not** OpenID Connect. If you add `openid` to the scope:

```yaml
# ❌ WRONG — GitHub will reject this or silently ignore 'openid'
scope: openid, read:user, user:email
```

Spring Security expects an ID token when `openid` scope is requested. Since GitHub never
returns one, the token exchange will either fail or produce an `OAuth2User` that Spring tries
to parse as `OidcUser`, causing a `NullPointerException` or `IllegalArgumentException`.

```yaml
# ✅ CORRECT — omit openid entirely for GitHub
scope: read:user, user:email
```

**Stable identifier workaround:** GitHub's `login` attribute (username) can change. Use the
numeric `id` attribute as a stable, immutable identifier instead of `sub`:

```java
OAuth2User user = (OAuth2User) authentication.getPrincipal();
Integer stableGithubId = user.getAttribute("id");  // e.g., 1234567
String login = user.getAttribute("login");         // e.g., "octocat"
```

---

### ❌ Pitfall 4 — `issuer-uri` vs Manual Endpoints for Keycloak

```yaml
# ✅ PREFERRED — auto-discovery via issuer-uri
provider:
  keycloak:
    issuer-uri: "https://auth.example.com/realms/my-realm"

# ⚠️  USE ONLY when issuer-uri discovery is blocked at startup time
# (e.g., Keycloak is not reachable from the app host during boot).
# You must keep these manually in sync with Keycloak's actual endpoints.
provider:
  keycloak:
    authorization-uri: "https://auth.example.com/realms/my-realm/protocol/openid-connect/auth"
    token-uri:         "https://auth.example.com/realms/my-realm/protocol/openid-connect/token"
    user-info-uri:     "https://auth.example.com/realms/my-realm/protocol/openid-connect/userinfo"
    jwk-set-uri:       "https://auth.example.com/realms/my-realm/protocol/openid-connect/certs"
```

With `issuer-uri`, Spring Boot validates the discovery document at startup. If Keycloak is not
reachable (e.g., in Docker Compose before Keycloak is healthy), the app will **fail to start**.
Solutions: use Docker health-checks, or temporarily use manual endpoints during development.

---

### ❌ Pitfall 5 — CSRF and Vaadin (Do NOT Disable It)

A common mistake when integrating OAuth2 is to blindly disable CSRF:

```java
// ❌ WRONG — disabling CSRF breaks Vaadin's form-based interactions
http.csrf(csrf -> csrf.disable());
```

`VaadinSecurityConfigurer` already handles CSRF correctly: it registers a Vaadin-aware CSRF
token repository that **excludes** Vaadin's internal UIDL and push endpoints while keeping
CSRF active for all regular HTTP endpoints. Never disable CSRF globally in a Vaadin app.

---

### ❌ Pitfall 6 — Missing `@AnonymousAllowed` on Login View → Infinite Redirect Loop

If your login/redirect view does not have `@AnonymousAllowed`, this happens:

```
User (unauthenticated) → GET /dashboard
  → NavigationAccessControl: DENY (no annotation) → redirect to /login
  → NavigationAccessControl: DENY (no @AnonymousAllowed on /login) → redirect to /login
  → ... infinite loop (browser: ERR_TOO_MANY_REDIRECTS)
```

```java
// ❌ WRONG — missing annotation causes infinite redirect
@Route("login")
public class LoginRedirectView extends VerticalLayout { ... }

// ✅ CORRECT — always annotate your login/session-ended views
@Route("login")
@AnonymousAllowed
public class LoginRedirectView extends VerticalLayout { ... }

@Route("session-ended")
@AnonymousAllowed
public class SessionEndedView extends VerticalLayout { ... }
```

---

### ❌ Pitfall 7 — Missing `@Import(VaadinAwareSecurityContextHolderStrategyConfiguration.class)`

When using the composable `VaadinSecurityConfigurer` (not extending `VaadinWebSecurity`), you
**must** import `VaadinAwareSecurityContextHolderStrategyConfiguration`. Without it, the Spring
Security context is not propagated correctly to Vaadin's server push threads, causing
`SecurityContextHolder.getContext().getAuthentication()` to return `null` inside push callbacks.

```java
// ✅ Always include this when using VaadinSecurityConfigurer directly
@Configuration
@Import(VaadinAwareSecurityContextHolderStrategyConfiguration.class)
public class SecurityConfiguration { ... }
```

---

## 8. Testing

### Required Test Dependencies

These are already included if you added `spring-boot-starter-test` and `spring-security-test`
in §1. Verify they are present:

```xml
<!-- Maven -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-test</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.springframework.security</groupId>
  <artifactId>spring-security-test</artifactId>
  <scope>test</scope>
</dependency>
```

```kotlin
// Gradle
testImplementation("org.springframework.boot:spring-boot-starter-test")
testImplementation("org.springframework.security:spring-security-test")
```

---

### `@WithMockUser` — Form-Login / Role-Based Tests

Use `@WithMockUser` for any test that only needs to simulate "a logged-in user with roles".
It populates the `SecurityContext` with a `UsernamePasswordAuthenticationToken` before
the test method runs.

```java
package com.example;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@SpringBootTest
@AutoConfigureMockMvc
class SecurityIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void unauthenticatedUserIsRedirectedToLogin() throws Exception {
        mockMvc.perform(get("/dashboard"))
            .andExpect(status().is3xxRedirection());
    }

    @Test
    @WithMockUser(username = "alice", roles = {"USER"})
    void authenticatedUserCanAccessDashboard() throws Exception {
        // /dashboard is @PermitAll — any authenticated user passes
        mockMvc.perform(get("/dashboard"))
            .andExpect(status().isOk());
    }

    @Test
    @WithMockUser(username = "bob", roles = {"USER"})
    void nonAdminCannotAccessAdminPanel() throws Exception {
        // /admin is @RolesAllowed("ADMIN")
        mockMvc.perform(get("/admin"))
            .andExpect(status().isForbidden());
    }

    @Test
    @WithMockUser(username = "carol", roles = {"USER", "ADMIN"})
    void adminCanAccessAdminPanel() throws Exception {
        mockMvc.perform(get("/admin"))
            .andExpect(status().isOk());
    }
}
```

---

### `@WithMockUser` on a Test Class (Applied to All Methods)

```java
@SpringBootTest
@AutoConfigureMockMvc
@WithMockUser(username = "testuser", roles = {"USER"})
class DashboardViewTests {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void dashboardLoads() throws Exception {
        mockMvc.perform(get("/dashboard"))
            .andExpect(status().isOk());
    }
}
```

---

### `@WithOidcLogin` — OIDC Principal Tests (Spring Security Test)

For tests that need a real `OidcUser` principal (e.g., to verify that your view correctly
reads `oidcUser.getEmail()`), use `@WithOidcLogin` from `spring-security-test`:

```java
package com.example;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.oidcLogin;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@SpringBootTest
@AutoConfigureMockMvc
class OidcLoginTests {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void oidcUserCanAccessProfile() throws Exception {
        mockMvc.perform(
            get("/profile")
                .with(oidcLogin()
                    .idToken(token -> token
                        .claim("sub", "user-abc-123")
                        .claim("email", "alice@example.com")
                        .claim("name", "Alice Example")
                    )
                )
        ).andExpect(status().isOk());
    }

    @Test
    void oidcUserWithAdminScopeCanAccessAdmin() throws Exception {
        mockMvc.perform(
            get("/admin")
                .with(oidcLogin()
                    .authorities(
                        new org.springframework.security.core.authority
                            .SimpleGrantedAuthority("ROLE_ADMIN")
                    )
                )
        ).andExpect(status().isOk());
    }
}
```

---

### Vaadin View Unit Testing with `SpringNavigationAccessControl`

For unit-testing Vaadin views in isolation (without a full HTTP stack), wire up
`NavigationAccessControl` manually via a `VaadinServiceInitListener`:

```java
package com.example;

import com.vaadin.flow.spring.security.SpringNavigationAccessControl;
import com.vaadin.flow.server.VaadinServiceInitListener;
import com.example.views.LoginRedirectView;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * Test-only configuration — wires NavigationAccessControl into the Vaadin service
 * without a full Spring Security filter chain. Use in @SpringBootTest slices that
 * do not start the full HTTP context.
 */
@Configuration
class TestSecurityConfig {

    @Bean
    VaadinServiceInitListener setupViewSecurity() {
        SpringNavigationAccessControl accessControl = new SpringNavigationAccessControl();
        accessControl.setLoginView(LoginRedirectView.class);
        return event -> event.getSource().addUIInitListener(uiEvent ->
            uiEvent.getUI().addBeforeEnterListener(accessControl)
        );
    }
}
```

---

## Quick-Reference Checklist

Use this checklist when setting up a new Vaadin 25 + Spring Boot 4 OIDC project:

- [ ] `vaadin-bom:25.0.0` in `<dependencyManagement>` (Maven) or `dependencyManagement {}` (Gradle)
- [ ] `vaadin-spring-boot-starter` (no version — managed by BOM)
- [ ] `spring-boot-starter-security`
- [ ] `spring-boot-starter-oauth2-client`
- [ ] `@Configuration` class uses `VaadinSecurityConfigurer.vaadin()` (NOT extending `VaadinWebSecurity`)
- [ ] `@Import(VaadinAwareSecurityContextHolderStrategyConfiguration.class)` on the config class
- [ ] `oauth2LoginPage(authorizationUrl, sessionEndedUrl)` called inside the `with()` lambda
- [ ] All `@RolesAllowed` / `@PermitAll` / `@DenyAll` imports are from `jakarta.annotation.security`
- [ ] `@AnonymousAllowed` (Vaadin-specific) on login view AND session-ended view
- [ ] No `csrf(csrf -> csrf.disable())` — VaadinSecurityConfigurer handles CSRF
- [ ] GitHub scope does NOT include `openid`
- [ ] ORCID uses `client-authentication-method: client_secret_post`
- [ ] Keycloak uses `issuer-uri` (preferred) or manual endpoints (fallback)
- [ ] Back-channel logout endpoint registered in Keycloak if `oidcLogout()` is configured
- [ ] `spring-security-test` on test classpath for `@WithMockUser` / `oidcLogin()`

---

## References

- [Vaadin 25 Security Docs](https://vaadin.com/docs/latest/flow/security/enabling-security)
- [Vaadin VaadinSecurityConfigurer Guide](https://vaadin.com/docs/latest/flow/security/vaadin-security-configurer)
- [Vaadin OAuth2 Integration](https://vaadin.com/docs/latest/flow/integrations/spring/oauth2)
- [Vaadin NavigationAccessControl](https://vaadin.com/docs/latest/flow/security/advanced-topics/navigation-access-control)
- [Spring Boot 4 Security Reference](https://docs.spring.io/spring-boot/4.0/-SNAPSHOT/reference/web/spring-security)
- [Spring Security 7 OAuth2 Login](https://docs.spring.io/spring-security/reference/7.0/servlet/oauth2/index)
- [Spring Security 7 OIDC Logout](https://docs.spring.io/spring-security/reference/7.0/-SNAPSHOT/servlet/oauth2/login/logout)
- [Spring Security Test Support](https://docs.spring.io/spring-security/reference/servlet/test/index.html)
- [ORCID OAuth2 / OIDC Docs](https://info.orcid.org/documentation/api-tutorials/api-tutorial-get-and-authenticated-orcid-id/)
- [Keycloak OIDC Back-Channel Logout](https://www.keycloak.org/docs/latest/securing_apps/#_backchannel_logout)
