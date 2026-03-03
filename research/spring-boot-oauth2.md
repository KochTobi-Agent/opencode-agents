# Spring Boot OAuth2 Implementation (Latest)

## Overview

Spring Boot OAuth2 provides comprehensive support for OAuth 2.0 and OpenID Connect 1.0 authentication. The framework includes both server-side authorization server capabilities and client-side OAuth2 client authentication for integrating with external OAuth2 providers. Spring Authorization Server implements OAuth 2.1 and OpenID Connect 1.0 specifications, providing a secure foundation for building identity providers and OAuth2-enabled applications.

**Key Components:**
- Spring Security OAuth2 Client: For client-side authentication
- Spring Authorization Server: For building OAuth2/OIDC-compliant authorization servers
- OAuth2 Login: Social login integration with external providers
- OpenID Connect 1.0 support for modern authentication flows

## Latest Version & Setup Requirements

### Spring Boot Versions
- **Latest: Spring Boot 3.x** (recommended)
- **LTS: Spring Boot 2.7.x** (extended support)
- Spring Security: 6.x+ (for Spring Boot 3.x)

### Prerequisites
- Java 17+ (for Spring Boot 3.x)
- Java 11+ (for Spring Boot 2.7.x)
- Maven 3.6+ or Gradle 7.0+
- OAuth2 provider accounts (client ID and client secret)

## Maven Dependencies

### Spring Boot Starter OAuth2 Client
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

### Spring Boot Starter OAuth2 Authorization Server (for building auth servers)
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-authorization-server</artifactId>
</dependency>
```

## Gradle Dependencies

### OAuth2 Client
```gradle
implementation "org.springframework.boot:spring-boot-starter-oauth2-client"
```

### OAuth2 Authorization Server
```gradle
implementation "org.springframework.boot:spring-boot-starter-oauth2-authorization-server"
```

## Spring Security OAuth2 Client Configuration

### 1. Basic Configuration Structure

The OAuth2 client configuration is organized into two main sections:

- **registration**: Defines OAuth2 provider client registrations (client ID, secret, scopes)
- **provider**: Defines OAuth2 provider endpoints (authorization, token, user info URIs)

### 2. Application Properties Configuration

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          # Registration ID (must be unique)
          github-idp:
            provider: github
            client-id: your-github-client-id
            client-secret: your-github-client-secret
            # Optional: specific scopes for this provider
            scope: user,repo
        
        provider:
          # Custom provider configuration
          github:
            # Standard GitHub endpoints are auto-configured
            # but you can override if needed
            authorization-uri: https://github.com/login/oauth/authorize
            token-uri: https://github.com/login/oauth/access_token
            user-info-uri: https://api.github.com/user
```

### 3. Security Configuration (Java)

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.oauth2.client.CommonOAuth2Provider;
import org.springframework.security.oauth2.client.registration.ClientRegistration;
import org.springframework.security.oauth2.client.registration.InMemoryClientRegistrationRepository;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class OAuth2SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authorize -> authorize
                .requestMatchers("/", "/login", "/error").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2Login -> oauth2Login
                .loginPage("/login")
                .defaultSuccessUrl("/home")
            );
        
        return http.build();
    }
}
```

### 4. Configuration for Authorization Code Grant Flow

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          client-a:
            provider: spring
            client-id: client-a
            client-secret: secret
            authorization-grant-type: authorization_code
            redirect-uri: "http://127.0.0.1:8080/login/oauth2/code/client-a"
            scope: scope-a,scope-b
        
        provider:
          spring:
            issuer-uri: http://localhost:9000
```

### 5. OAuth2 Login Template Configuration

```java
import org.springframework.context.annotation.Bean;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;

@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/login")
                .defaultSuccessUrl("/authenticated", true)
                .failureUrl("/login?error=true")
            );
        
        return http.build();
    }
}
```

## Free OAuth2 Providers Configuration

### 1. GitHub OAuth2 Provider

**Provider Details:**
- **Provider ID**: github
- **Authorization URI**: https://github.com/login/oauth/authorize
- **Token URI**: https://github.com/login/oauth/access_token
- **User Info URI**: https://api.github.com/user
- **Pre-configured**: Yes (built-in support)

**Registration Steps:**
1. Go to GitHub Settings → Developer settings → OAuth Apps
2. Click "New OAuth App"
3. Fill in Application details
4. Get Client ID and Client Secret

**Configuration:**
```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          github:
            client-id: your-github-client-id
            client-secret: your-github-client-secret
            scope: user,repo
```

**Application Properties Example:**
```properties
spring.security.oauth2.client.registration.github.client-id=your-client-id
spring.security.oauth2.client.registration.github.client-secret=your-client-secret
```

### 2. ORCID OAuth2 Provider

**Provider Details:**
- **Organization**: Open Researcher and Contributor ID
- **Authorization URI**: https://orcid.org/oauth/authorize (sandbox: https://sandbox.orcid.org/oauth/authorize)
- **Token URI**: https://orcid.org/oauth/token (sandbox: https://sandbox.orcid.org/oauth/token)
- **User Info URI**: https://orcid.org/oauth/userinfo
- **Pre-configured**: No (custom configuration required)

**Registration Steps:**
1. Go to https://orcid.org/ (or sandbox for testing)
2. Create an ORCID account
3. Register an application at https://orcid.org/account/applications
4. Obtain Client ID and Client Secret

**Configuration:**
```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          orcid:
            provider: orcid
            client-id: your-orcid-client-id
            client-secret: your-orcid-client-secret
            authorization-grant-type: authorization_code
            redirect-uri: "http://localhost:8080/login/oauth2/code/orcid"
            scope: openid,profile,email
        
        provider:
          orcid:
            authorization-uri: https://orcid.org/oauth/authorize
            token-uri: https://orcid.org/oauth/token
            user-info-uri: https://orcid.org/oauth/userinfo
            user-name-attribute: sub
```

**Java Configuration for ORCID:**
```java
import org.springframework.context.annotation.Bean;
import org.springframework.security.oauth2.client.registration.ClientRegistration;
import org.springframework.security.oauth2.client.registration.ClientRegistrationRepository;
import org.springframework.security.oauth2.client.registration.InMemoryClientRegistrationRepository;
import org.springframework.security.oauth2.core.AuthorizationGrantType;

@Bean
public ClientRegistrationRepository clientRegistrationRepository() {
    ClientRegistration orcidRegistration = ClientRegistration.withRegistrationId("orcid")
        .clientId("your-orcid-client-id")
        .clientSecret("your-orcid-client-secret")
        .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
        .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
        .redirectUri("http://localhost:8080/login/oauth2/code/orcid")
        .scope("openid", "profile", "email")
        .authorizationUri("https://orcid.org/oauth/authorize")
        .tokenUri("https://orcid.org/oauth/token")
        .userInfoUri("https://orcid.org/oauth/userinfo")
        .userNameAttributeName("sub")
        .build();
    
    return new InMemoryClientRegistrationRepository(orcidRegistration);
}
```

### 3. Keycloak OAuth2/OIDC Provider

**Provider Details:**
- **Type**: Self-hosted OpenID Connect/OAuth2 server
- **License**: Open Source (Apache 2.0)
- **Authorization URI**: {keycloak-server}/auth/realms/{realm}/protocol/openid-connect/auth
- **Token URI**: {keycloak-server}/auth/realms/{realm}/protocol/openid-connect/token
- **User Info URI**: {keycloak-server}/auth/realms/{realm}/protocol/openid-connect/userinfo
- **Pre-configured**: Partial (custom Keycloak instance configuration required)

**Setup Steps:**

1. **Download and Run Keycloak:**
```bash
# Download Keycloak
wget https://github.com/keycloak/keycloak/releases/download/20.0.0/keycloak-20.0.0.tar.gz

# Extract and start
tar -xzf keycloak-20.0.0.tar.gz
cd keycloak-20.0.0/bin
./kc.sh start-dev
```

2. **Create Realm and Client in Keycloak Admin Console:**
   - Navigate to http://localhost:8080/admin
   - Create a new realm
   - Create a new client with Authorization Code flow
   - Set Valid Redirect URIs: http://localhost:8081/login/oauth2/code/keycloak
   - Get Client ID and Client Secret

**Configuration:**
```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          keycloak:
            provider: keycloak
            client-id: your-keycloak-client-id
            client-secret: your-keycloak-client-secret
            authorization-grant-type: authorization_code
            redirect-uri: "http://localhost:8081/login/oauth2/code/keycloak"
            scope: openid,profile,email
        
        provider:
          keycloak:
            issuer-uri: http://localhost:8080/auth/realms/master
            # Or use individual endpoints:
            # authorization-uri: http://localhost:8080/auth/realms/master/protocol/openid-connect/auth
            # token-uri: http://localhost:8080/auth/realms/master/protocol/openid-connect/token
            # user-info-uri: http://localhost:8080/auth/realms/master/protocol/openid-connect/userinfo
            user-name-attribute: preferred_username
```

**Java Configuration for Keycloak:**
```java
import org.springframework.context.annotation.Bean;
import org.springframework.security.oauth2.client.registration.ClientRegistration;
import org.springframework.security.oauth2.client.registration.ClientRegistrationRepository;
import org.springframework.security.oauth2.client.registration.InMemoryClientRegistrationRepository;
import org.springframework.security.oauth2.core.AuthorizationGrantType;
import org.springframework.security.oauth2.core.ClientAuthenticationMethod;

@Bean
public ClientRegistrationRepository clientRegistrationRepository() {
    ClientRegistration keycloakRegistration = ClientRegistration.withRegistrationId("keycloak")
        .clientId("your-keycloak-client-id")
        .clientSecret("your-keycloak-client-secret")
        .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
        .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
        .redirectUri("http://localhost:8081/login/oauth2/code/keycloak")
        .scope("openid", "profile", "email")
        .authorizationUri("http://localhost:8080/auth/realms/master/protocol/openid-connect/auth")
        .tokenUri("http://localhost:8080/auth/realms/master/protocol/openid-connect/token")
        .userInfoUri("http://localhost:8080/auth/realms/master/protocol/openid-connect/userinfo")
        .userNameAttributeName("preferred_username")
        .issuerUri("http://localhost:8080/auth/realms/master")
        .build();
    
    return new InMemoryClientRegistrationRepository(keycloakRegistration);
}
```

**Docker Setup for Keycloak:**
```bash
docker run -p 8080:8080 \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin \
  quay.io/keycloak/keycloak:latest \
  start-dev
```

## Code Examples for OAuth2 Configuration

### Example 1: Complete Spring Boot OAuth2 Application with GitHub

**pom.xml:**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-client</artifactId>
    </dependency>
</dependencies>
```

**application.yml:**
```yaml
spring:
  application:
    name: oauth2-demo
  security:
    oauth2:
      client:
        registration:
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            scope: user:email,read:user
```

**SecurityConfig.java:**
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/", "/error").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/login")
                .defaultSuccessUrl("/home", true)
            )
            .logout(logout -> logout
                .logoutSuccessUrl("/")
            );
        
        return http.build();
    }
}
```

**HomeController.java:**
```java
import org.springframework.security.oauth2.client.authentication.OAuth2AuthenticationToken;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class HomeController {

    @GetMapping("/home")
    public String home(OAuth2AuthenticationToken token, Model model) {
        model.addAttribute("name", token.getPrincipal().getAttribute("name"));
        model.addAttribute("email", token.getPrincipal().getAttribute("email"));
        model.addAttribute("avatar", token.getPrincipal().getAttribute("avatar_url"));
        return "home";
    }
}
```

### Example 2: Multi-Provider OAuth2 Configuration

**application.yml:**
```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            scope: user:email,read:user
          
          orcid:
            provider: orcid
            client-id: ${ORCID_CLIENT_ID}
            client-secret: ${ORCID_CLIENT_SECRET}
            authorization-grant-type: authorization_code
            redirect-uri: "http://localhost:8080/login/oauth2/code/orcid"
            scope: openid,profile,email
          
          keycloak:
            provider: keycloak
            client-id: ${KEYCLOAK_CLIENT_ID}
            client-secret: ${KEYCLOAK_CLIENT_SECRET}
            authorization-grant-type: authorization_code
            redirect-uri: "http://localhost:8080/login/oauth2/code/keycloak"
            scope: openid,profile,email
        
        provider:
          orcid:
            authorization-uri: https://orcid.org/oauth/authorize
            token-uri: https://orcid.org/oauth/token
            user-info-uri: https://orcid.org/oauth/userinfo
            user-name-attribute: sub
          
          keycloak:
            issuer-uri: ${KEYCLOAK_ISSUER_URI:http://localhost:8080/auth/realms/master}
            user-name-attribute: preferred_username
```

### Example 3: Custom OAuth2AuthenticationSuccessHandler

```java
import org.springframework.security.core.Authentication;
import org.springframework.security.oauth2.client.authentication.OAuth2AuthenticationToken;
import org.springframework.security.web.authentication.AuthenticationSuccessHandler;
import org.springframework.stereotype.Component;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

@Component
public class OAuth2AuthenticationSuccessHandler implements AuthenticationSuccessHandler {

    @Override
    public void onAuthenticationSuccess(
            HttpServletRequest request,
            HttpServletResponse response,
            Authentication authentication) throws IOException, ServletException {
        
        OAuth2AuthenticationToken token = (OAuth2AuthenticationToken) authentication;
        String provider = token.getAuthorizedClientRegistrationId();
        String username = token.getPrincipal().getAttribute("name");
        
        // Log successful authentication
        System.out.println("User " + username + " authenticated via " + provider);
        
        // Redirect based on provider
        if ("github".equals(provider)) {
            response.sendRedirect("/github-dashboard");
        } else if ("orcid".equals(provider)) {
            response.sendRedirect("/orcid-profile");
        } else if ("keycloak".equals(provider)) {
            response.sendRedirect("/keycloak-home");
        } else {
            response.sendRedirect("/home");
        }
    }
}
```

### Example 4: Authorization Server Configuration with OpenID Connect

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.oauth2.server.authorization.config.annotation.web.configuration.OAuth2AuthorizationServerConfiguration;
import org.springframework.security.oauth2.server.authorization.config.annotation.web.configurers.OAuth2AuthorizationServerConfigurer;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class AuthorizationServerConfig {

    @Bean
    public SecurityFilterChain authorizationServerSecurityFilterChain(HttpSecurity http) throws Exception {
        OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http);
        
        http.getConfigurer(OAuth2AuthorizationServerConfigurer.class)
            .oidc(Customizer.withDefaults());
        
        return http.build();
    }
}
```

## Key APIs and Components

### ClientRegistration
- Configuration class for an OAuth2 client registration
- Contains client ID, secret, provider endpoints, scopes, and grant types

### ClientRegistrationRepository
- Repository for managing ClientRegistration instances
- InMemoryClientRegistrationRepository: for static configuration
- JdbcClientRegistrationRepository: for database-backed configuration

### OAuth2AuthenticationToken
- Authentication token containing OAuth2-authenticated user information
- Accessible in controllers after successful authentication
- Provides access to principal attributes (name, email, etc.)

### OAuth2LoginAuthenticationFilter
- Filter that processes OAuth2 login requests
- Handles authorization code exchange for access tokens
- Automatic bearer token attachment for protected resource requests

### OAuth2AuthorizationServerConfigurer
- Configures OAuth2 authorization server endpoints
- Sets up token endpoint, authorization endpoint, OIDC endpoints
- Supports custom grant types and token generators

## Common Properties

```yaml
# Required properties
spring.security.oauth2.client.registration.{registrationId}.client-id
spring.security.oauth2.client.registration.{registrationId}.client-secret

# Optional properties
spring.security.oauth2.client.registration.{registrationId}.provider
spring.security.oauth2.client.registration.{registrationId}.scope
spring.security.oauth2.client.registration.{registrationId}.redirect-uri
spring.security.oauth2.client.registration.{registrationId}.authorization-grant-type

# Provider configuration
spring.security.oauth2.client.provider.{providerId}.authorization-uri
spring.security.oauth2.client.provider.{providerId}.token-uri
spring.security.oauth2.client.provider.{providerId}.user-info-uri
spring.security.oauth2.client.provider.{providerId}.issuer-uri
spring.security.oauth2.client.provider.{providerId}.user-name-attribute
```

## Environment Variables Setup

```bash
# GitHub
export GITHUB_CLIENT_ID="your-github-client-id"
export GITHUB_CLIENT_SECRET="your-github-client-secret"

# ORCID
export ORCID_CLIENT_ID="your-orcid-client-id"
export ORCID_CLIENT_SECRET="your-orcid-client-secret"

# Keycloak
export KEYCLOAK_CLIENT_ID="your-keycloak-client-id"
export KEYCLOAK_CLIENT_SECRET="your-keycloak-client-secret"
export KEYCLOAK_ISSUER_URI="http://localhost:8080/auth/realms/master"
```

## Security Best Practices

1. **Never hardcode secrets** - Use environment variables or external configuration
2. **HTTPS only** - Use HTTPS in production for all OAuth2 communications
3. **Validate scopes** - Request only necessary scopes
4. **Token expiration** - Handle access token refresh flows
5. **CORS configuration** - Properly configure CORS for front-end applications
6. **State parameter** - Spring Security automatically handles CSRF protection via state parameter
7. **Secure redirect URIs** - Register only trusted redirect URIs in provider applications

## Troubleshooting Common Issues

### Issue: "client_id not found"
- Verify environment variables or properties are correctly set
- Check that provider is correctly configured

### Issue: "Invalid redirect_uri"
- Ensure redirect URI matches exactly what's registered in the OAuth2 provider
- Default format: `{base-url}/login/oauth2/code/{registrationId}`

### Issue: "Invalid scope"
- Verify scopes are supported by the provider
- Different providers have different scope requirements

### Issue: "CORS errors when accessing user info"
- Spring Security handles this automatically
- Ensure RestTemplate or WebClient is properly configured if custom implementations are used

## Notes

- **Spring Authorization Server** became the official authorization server implementation in Spring Security 5.5+
- **Pre-configured providers**: GitHub, Google, Facebook, Okta (built-in endpoint configurations)
- **Custom providers** (like ORCID and Keycloak): Require explicit endpoint configuration
- **OpenID Connect support**: Available through Spring Authorization Server and compatible OIDC providers
- **Token refresh**: Automatically handled by Spring Security when refresh tokens are available
- **Spring Boot 3.0+**: Uses Spring Security 6.x with improved configuration model

## References

- Spring Authorization Server: https://github.com/spring-projects/spring-authorization-server
- Spring Security OAuth2: https://spring.io/projects/spring-security
- GitHub OAuth2: https://docs.github.com/en/apps/oauth-apps
- ORCID OAuth2: https://orcid.org/about/what-is-orcid/mission
- Keycloak: https://www.keycloak.org/
