# Vaadin OAuth2 Integration (v25)

## Overview

Vaadin Flow is a modern Java web framework that enables building type-safe web applications without JavaScript/HTML/CSS knowledge. Version 25.0.0 (beta) and v24.8.4 (stable) support OAuth2 authentication integration with Spring Security, allowing developers to leverage existing OAuth2 providers like Google, GitHub, Keycloak, and custom OIDC providers for free authentication without paid subscriptions.

## Latest Vaadin Version

- **Current Stable**: Vaadin Flow 24.8.4
- **Latest Beta**: Vaadin Flow 25.0.0-beta1
- **Framework Type**: Java-based web framework with server-side rendering
- **Source Reputation**: High (Benchmark Score: 86.6/100)
- **Code Snippets Available**: 9,453+

## Spring Security OAuth2 Integration with Vaadin

### Architecture Overview

Vaadin integrates with Spring Security OAuth2 through:
1. **Spring Boot OAuth2 Client Starter** - handles OAuth2 protocol
2. **VaadinSecurityConfigurer** - bridges Vaadin security with Spring Security
3. **VaadinWebSecurity** - base class for Vaadin-specific security configuration
4. **VaadinAwareSecurityContextHolderStrategyConfiguration** - manages security context in Vaadin sessions

### Key Integration Points

| Component | Purpose |
|-----------|---------|
| `VaadinSecurityConfigurer` | Configure OAuth2 login pages in Vaadin |
| `VaadinWebSecurity` | Extend for Vaadin-specific security setup |
| `oauth2LoginPage()` | Specify OAuth2 authorization endpoint |
| `@AnonymousAllowed` | Mark views accessible without authentication |
| `@RolesAllowed` | Control access by user roles |
| `VaadinSession` | Store user authentication state |

## Free Authentication Options in Vaadin

### 1. **Social OAuth2 Providers** (No subscription required)

These services offer free OAuth2 authentication tiers suitable for development and small applications:

#### Google OAuth2
- **Cost**: Free tier available
- **Scopes**: `openid`, `profile`, `email`
- **Requires**: Google Cloud Project setup

#### GitHub OAuth2
- **Cost**: Free tier available
- **Scopes**: `user:email`, `read:user`
- **Requires**: GitHub Developer application

#### Keycloak (Self-hosted)
- **Cost**: Open-source, self-hosted (free)
- **Scopes**: `openid`, `profile`, `email`
- **Advantage**: Full control over identity provider

### 2. **In-Memory Authentication** (Development)

For development/testing without external providers:
```java
@Bean
public UserDetailsService userDetailsService() {
    InMemoryUserDetailsManager manager = new InMemoryUserDetailsManager();
    manager.createUser(
        User.withUsername("user")
            .password(passwordEncoder().encode("password"))
            .roles("USER")
            .build()
    );
    return manager;
}
```

### 3. **Local Database Authentication** (Self-hosted)

Store user credentials in your own database with Spring Security.

## Code Examples for Vaadin + Spring Security OAuth2

### Example 1: Basic Vaadin Security Configuration with In-Memory Users

```java
package com.example;

import com.vaadin.flow.spring.security.VaadinWebSecurity;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

@SpringBootApplication
public class VaadinOAuth2Application {
    public static void main(String[] args) {
        SpringApplication.run(VaadinOAuth2Application.class, args);
    }
}

@Configuration
@EnableWebSecurity
class SecurityConfiguration extends VaadinWebSecurity {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // Secure all views except those marked with @AnonymousAllowed
        super.configure(http);

        // Set login view
        setLoginView(http, LoginView.class);
    }

    @Bean
    public UserDetailsService userDetailsService() {
        InMemoryUserDetailsManager manager = new InMemoryUserDetailsManager();

        manager.createUser(
            User.withUsername("user")
                .password(passwordEncoder().encode("user"))
                .roles("USER")
                .build()
        );

        manager.createUser(
            User.withUsername("admin")
                .password(passwordEncoder().encode("admin"))
                .roles("USER", "ADMIN")
                .build()
        );

        return manager;
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### Example 2: OAuth2 Login with Keycloak Provider

**Step 1: Add Dependencies** (pom.xml)
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>

<dependency>
    <groupId>com.vaadin</groupId>
    <artifactId>vaadin-spring-boot-starter</artifactId>
    <version>25.0.0</version>
</dependency>
```

**Step 2: Configure Security** (Java)
```java
import com.vaadin.flow.spring.security.VaadinAwareSecurityContextHolderStrategyConfiguration;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.context.annotation.Bean;

@Configuration
@EnableWebSecurity
@Import(VaadinAwareSecurityContextHolderStrategyConfiguration.class)
class OAuth2SecurityConfiguration {
    
    @Bean
    SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http.with(VaadinSecurityConfigurer.vaadin(), configurer -> {
            configurer.oauth2LoginPage(
                "/oauth2/authorization/keycloak",  // OAuth2 authorization endpoint
                "{baseUrl}/session-ended"           // Post-logout redirect
            );
        });
        return http.build();
    }
}
```

**Step 3: Configure OAuth2 Provider** (application.yaml)
```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          keycloak:
            provider: keycloak
            client-id: vaadin-app
            client-secret: your-client-secret
            authorization-grant-type: authorization_code
            scope: openid,profile,email
        provider:
          keycloak:
            issuer-uri: http://localhost:8180/realms/my-realm
            user-name-attribute: preferred_username
```

### Example 3: Google OAuth2 Login

**Configuration** (application.yaml)
```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: your-google-client-id.apps.googleusercontent.com
            client-secret: your-google-client-secret
            scope: openid,profile,email
        provider:
          google:
            issuer-uri: https://accounts.google.com
```

**Security Configuration** (Java)
```java
@Configuration
@EnableWebSecurity
@Import(VaadinAwareSecurityContextHolderStrategyConfiguration.class)
class GoogleOAuth2SecurityConfiguration {
    
    @Bean
    SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http.with(VaadinSecurityConfigurer.vaadin(), configurer -> {
            configurer.oauth2LoginPage(
                "/oauth2/authorization/google",
                "{baseUrl}/session-ended"
            );
        });
        return http.build();
    }
}
```

### Example 4: Protected Vaadin Views with Role-Based Access

```java
package com.example.views;

import com.vaadin.flow.component.button.Button;
import com.vaadin.flow.component.html.H1;
import com.vaadin.flow.component.orderedlayout.VerticalLayout;
import com.vaadin.flow.component.grid.Grid;
import com.vaadin.flow.router.Route;
import com.vaadin.flow.router.BeforeEnterEvent;
import com.vaadin.flow.router.BeforeEnterObserver;
import com.vaadin.flow.server.VaadinSession;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;

import javax.annotation.security.RolesAllowed;

// Public view - accessible to all
@Route("public")
public class PublicView extends VerticalLayout {
    public PublicView() {
        add(new H1("Welcome - Public View"));
    }
}

// Admin-only view
@Route("admin")
@RolesAllowed("ADMIN")
public class AdminView extends VerticalLayout implements BeforeEnterObserver {
    
    private Grid<String> grid = new Grid<>();

    public AdminView() {
        add(new H1("Admin Dashboard"));
        setupGrid();
    }

    private void setupGrid() {
        grid.addColumn(item -> item).setHeader("Admin Data");
        grid.setItems("Item 1", "Item 2", "Item 3");
        add(grid);
    }

    @Override
    public void beforeEnter(BeforeEnterEvent event) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        
        boolean isAdmin = auth.getAuthorities().stream()
            .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"));
        
        if (!isAdmin) {
            event.rerouteTo(PublicView.class);
        }
    }
}

// User profile view
@Route("profile")
public class ProfileView extends VerticalLayout {
    
    public ProfileView() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        String username = auth.getName();
        
        add(new H1("User Profile"));
        add(new Button("Welcome, " + username));
    }
}
```

### Example 5: Custom Authentication with Navigation Guards

```java
package com.example.security;

import com.vaadin.flow.component.UI;
import com.vaadin.flow.router.BeforeEnterEvent;
import com.vaadin.flow.router.BeforeEnterObserver;
import com.vaadin.flow.server.VaadinSession;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;

public class AuthenticationGuard implements BeforeEnterObserver {
    
    private final String requiredRole;

    public AuthenticationGuard(String requiredRole) {
        this.requiredRole = requiredRole;
    }

    @Override
    public void beforeEnter(BeforeEnterEvent event) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        
        // Check if user is authenticated
        if (auth == null || !auth.isAuthenticated()) {
            event.rerouteTo("/login");
            return;
        }

        // Check if user has required role
        boolean hasRole = auth.getAuthorities().stream()
            .anyMatch(a -> a.getAuthority().equals("ROLE_" + requiredRole));
        
        if (!hasRole) {
            event.rerouteTo("/access-denied");
        }
    }

    // Store user info in session for later access
    public static void storeUserInfo(Authentication auth) {
        VaadinSession session = VaadinSession.getCurrent();
        session.setAttribute("username", auth.getName());
        session.setAttribute("authorities", auth.getAuthorities());
    }

    // Retrieve user info from session
    public static String getUsername() {
        VaadinSession session = VaadinSession.getCurrent();
        Object username = session.getAttribute("username");
        return username != null ? (String) username : "Anonymous";
    }
}
```

## Best Practices for Free Subscription Users

### 1. **Provider Selection**

✅ **Recommended for Free Tier:**
- **Keycloak** (self-hosted) - Zero cost, full control
- **Google OAuth** - Generous free tier, fast setup
- **GitHub OAuth** - Perfect for developer apps
- **Microsoft Entra ID** - Free tier available

❌ **Avoid if on Budget:**
- Commercial identity platforms with paid-only plans
- Services charging per authentication

### 2. **Development Strategy**

```
Development Phase:
├── Use in-memory authentication (@InMemoryAuth)
├── Test with local Keycloak (Docker)
└── Validate security flows locally

Staging Phase:
├── Deploy to free tier (Heroku, Railway, Render)
├── Connect to free OAuth provider
└── Test end-to-end OAuth flow

Production Phase:
├── Use self-hosted Keycloak or
├── Upgrade to free tier of major provider
└── Monitor rate limits
```

### 3. **Secure Configuration Management**

```java
@Configuration
public class SecureOAuth2Config {
    
    // Use environment variables, NOT hardcoded secrets
    @Value("${spring.security.oauth2.client.registration.keycloak.client-secret}")
    private String clientSecret;
    
    // Use property files with .gitignore
    // application-secrets.yaml (NEVER commit)
    // application-secrets.yaml → .gitignore
    
    // Alternative: Use Spring Cloud Config Server
}
```

**application.yaml (committed):**
```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          keycloak:
            client-id: ${OAUTH_CLIENT_ID}
            client-secret: ${OAUTH_CLIENT_SECRET}
```

### 4. **Rate Limiting Considerations**

| Provider | Free Tier Limits | Notes |
|----------|-----------------|-------|
| **Google** | Unlimited OAuth2 token requests | No daily limit for basic OAuth |
| **GitHub** | 5,000 requests/hour (token endpoint) | Sufficient for most apps |
| **Keycloak** | Self-hosted (no limit) | Only limited by server resources |
| **Microsoft** | Unlimited in free tier | For non-production apps |

### 5. **Cost-Effective Architecture**

**Single-Sign-On Setup (Free):**
```
┌─────────────────┐
│  Vaadin App 1   │
├─────────────────┤
│  Vaadin App 2   │  ← All authenticate to
├─────────────────┤    one free provider
│  Vaadin App 3   │
└────────┬────────┘
         │
         v
    ┌─────────────────┐
    │  Keycloak       │  (Self-hosted, free)
    │  OR Google/GitHub
    └─────────────────┘
```

### 6. **Dependency Management**

**Minimal pom.xml for OAuth2:**
```xml
<dependencies>
    <!-- Vaadin -->
    <dependency>
        <groupId>com.vaadin</groupId>
        <artifactId>vaadin-spring-boot-starter</artifactId>
        <version>25.0.0</version>
    </dependency>

    <!-- Spring Security with OAuth2 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-client</artifactId>
    </dependency>

    <!-- Database (optional) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

### 7. **Testing Without Real Credentials**

```java
package com.example.security;

import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.security.test.context.support.WithMockUser;

@AutoConfigureMockMvc
public class SecurityTests {
    
    @WithMockUser(username = "testuser", roles = "USER")
    @Test
    void testProtectedView() {
        // Test secured views without OAuth provider
    }
    
    @WithMockUser(username = "admin", roles = "ADMIN")
    @Test
    void testAdminView() {
        // Test admin endpoints
    }
}
```

### 8. **Monitoring & Logging**

```yaml
logging:
  level:
    org.springframework.security: DEBUG
    org.springframework.security.oauth2: DEBUG
    com.vaadin: INFO
```

### 9. **Free Hosting + Free Auth Stack**

**Recommended Combination:**
```
Frontend:    Vaadin Flow (open-source)
             ↓
Backend:     Spring Boot (open-source)
             ↓
Database:    H2/PostgreSQL free tier
             ↓
Auth:        Keycloak (self-hosted) OR Google/GitHub OAuth
             ↓
Hosting:     Render / Railway / Heroku (free tier available)
```

### 10. **Common Pitfalls to Avoid**

❌ **Storing secrets in code:**
```java
// WRONG
client-secret: "my-secret-12345"
```

✅ **Using environment variables:**
```bash
export OAUTH_CLIENT_SECRET="my-secret-12345"
```

❌ **Hardcoding redirect URIs:**
```java
// WRONG
redirectUri: "http://localhost:8080/login/oauth2/code/google"
```

✅ **Using configurable URIs:**
```yaml
spring.security.oauth2.client.registration.google.redirect-uri: ${OAUTH_REDIRECT_URI}
```

## Integration Checklist for Free Users

- [ ] Select free OAuth2 provider (Google, GitHub, or Keycloak)
- [ ] Add Spring Security OAuth2 Client starter dependency
- [ ] Create SecurityConfiguration extending VaadinWebSecurity
- [ ] Configure OAuth2 provider in application.yaml
- [ ] Implement role-based access with @RolesAllowed
- [ ] Set up login and logout flows
- [ ] Store credentials in environment variables
- [ ] Test with @WithMockUser annotations
- [ ] Deploy to free hosting with free auth tier
- [ ] Monitor logs for authentication errors
- [ ] Document your OAuth2 flow for team

## References & Resources

- **Vaadin OAuth2 Documentation**: https://vaadin.com/docs/latest/flow/integrations/spring/oauth2
- **Spring Security OAuth2**: https://spring.io/projects/spring-security
- **Spring Authorization Server**: https://github.com/spring-projects/spring-authorization-server
- **Keycloak**: https://www.keycloak.org/ (self-hosted, open-source)
- **Vaadin Flow Examples**: GitHub vaadin/flow-examples

## Summary

Vaadin v25 with Spring Security OAuth2 provides a powerful, free solution for building secure web applications. By leveraging free OAuth2 providers (Google, GitHub) or self-hosted Keycloak, developers can implement enterprise-grade authentication without subscription costs. The framework's tight Spring integration, combined with role-based access control and modern security patterns, makes it ideal for both small projects and scalable enterprise applications.

**Key Takeaway**: For free users, the recommended stack is Vaadin + Spring Security + Keycloak (self-hosted) or Google OAuth2, which provides unlimited authentication at zero cost.
