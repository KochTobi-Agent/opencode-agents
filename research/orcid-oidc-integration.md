# ORCID as an OpenID Connect Provider

## Overview

ORCID (Open Researcher and Contributor ID) provides a standardized way to authenticate researchers using OpenID Connect (OIDC), built on top of OAuth 2.0. ORCID is a persistent, unique identifier for researchers that enables seamless integration across research systems, reducing the need for multiple credentials while providing secure authentication and authorized access to researcher profiles.

## 1. ORCID OpenID Connect Endpoints and Configuration

### Production Environment

- **Issuer**: `https://orcid.org`
- **Authorization Endpoint**: `https://orcid.org/oauth/authorize`
- **Token Endpoint**: `https://orcid.org/oauth/token`
- **UserInfo Endpoint**: `https://orcid.org/oauth/userinfo`
- **JWKS URI**: `https://orcid.org/oauth/jwks`
- **Discovery Endpoint**: `https://orcid.org/.well-known/openid-configuration`

### Sandbox Environment

- **Issuer**: `https://sandbox.orcid.org`
- **Authorization Endpoint**: `https://sandbox.orcid.org/oauth/authorize`
- **Token Endpoint**: `https://sandbox.orcid.org/oauth/token`
- **UserInfo Endpoint**: `https://sandbox.orcid.org/oauth/userinfo`
- **JWKS URI**: `https://sandbox.orcid.org/oauth/jwks`
- **Discovery Endpoint**: `https://sandbox.orcid.org/.well-known/openid-configuration`

### OpenID Connect Configuration

ORCID supports the following OpenID Connect features:

```json
{
  "issuer": "https://sandbox.orcid.org",
  "authorization_endpoint": "https://sandbox.orcid.org/oauth/authorize",
  "token_endpoint": "https://sandbox.orcid.org/oauth/token",
  "userinfo_endpoint": "https://sandbox.orcid.org/oauth/userinfo",
  "jwks_uri": "https://sandbox.orcid.org/oauth/jwks",
  "scopes_supported": ["openid"],
  "response_types_supported": ["code", "id_token", "id_token token"],
  "grant_types_supported": ["authorization_code", "implicit", "refresh_token"],
  "token_endpoint_auth_methods_supported": ["client_secret_post"],
  "token_endpoint_auth_signing_alg_values_supported": ["RS256"],
  "id_token_signing_alg_values_supported": ["RS256"],
  "claims_supported": ["family_name", "given_name", "name", "auth_time", "iss", "sub"],
  "subject_types_supported": ["public"],
  "claims_parameter_supported": false
}
```

## 2. ORCID Sandbox Environment for Testing

### Overview

The sandbox testing server (`https://sandbox.orcid.org`) is a complete copy of the ORCID Registry software containing only testing data. It provides a safe environment to:

- Test integration without affecting production data
- Develop and test API implementations
- Train and demonstrate ORCID integration
- Obtain Member API credentials (free access for testing)

### Key Features

- **Data Independence**: Sandbox data is separate from production and not backed up
- **Free Access**: Anyone can request Member API credentials for sandbox testing
- **Realistic Environment**: Behaves identically to production except for email limitations

### Creating Test Accounts

1. Visit `https://sandbox.orcid.org/register`
2. Create a test account with a Mailinator email address
3. Use @mailinator.com addresses for email verification
4. Access Developer Tools at `https://sandbox.orcid.org/developer-tools`

### Important Limitations

- **Email Handling**: Only sends emails to @mailinator.com addresses
- **Data Reliability**: Not backed up; may be deleted without notice
- **Performance**: Less reliable than production servers
- **Data Isolation**: Separate database from production registry

## 3. How to Register an ORCID Application

### Step 1: Create an ORCID Account

1. Register at `https://orcid.org/register` (production) or `https://sandbox.orcid.org/register` (sandbox)
2. Verify your email address (required for developer access)
3. Sign in to your ORCID record

### Step 2: Register Application

**Public API Credentials:**

1. Sign in to your ORCID record
2. Click your name in the top right corner
3. Select "Developer Tools"
4. Read and agree to ORCID Public APIs Terms of Service
5. Click "Register for ORCID Public API Credentials"

**Member API Credentials:**

1. Contact ORCID through their registration form
2. Complete the application details
3. Provide organization information
4. ORCID team will review and issue credentials

### Step 3: Complete Application Details Form

Provide the following information:

- **Name**: Application name (displayed to users)
- **Application URL**: Website where users learn about your application
- **Application Description**: How you'll use the ORCID iD (shown on OAuth screen)
- **Redirect URI(s)**: Return URLs after authorization (HTTPS only for production)

### Step 4: Register Redirect URIs

**Critical Requirements:**

```
✓ HTTPS URIs only in production
✓ Exact domain matching (including subdomains)
✓ Register specific paths when possible
✓ Subdomains must be registered separately
✓ Wildcard domains NOT supported
```

**Redirect URI Matching Rules:**

- Registered: `https://example.com` → Works for `https://example.com/callback`
- Registered: `https://example.com/oauth/callback` → Works only for exact match
- Registered: `https://app.example.com` → Does NOT work for `https://example.com`

### Step 5: Save and Generate Credentials

1. Complete the application form
2. Click "Save my application and generate my client ID and secret"
3. Receive:
   - Client ID (public identifier)
   - Client Secret (keep confidential)

## 4. ORCID Scopes and User Information Available

### OpenID Connect Scopes

ORCID supports the following scopes:

#### Core Scope

- **`openid`**: Required for OpenID Connect; enables ID token and userinfo endpoint access

#### ORCID Specific Scopes (with openid)

- **`/authenticate`**: Authenticate user (read-only, alternative to openid)
- **`/read-limited`**: Read limited-access ORCID record data
- **`/read-public`**: Read public ORCID record data
- **`/activities/read-limited`**: Read limited-access activities
- **`/activities/read-public`**: Read public activities
- **`/person/read-limited`**: Read limited-access biographical information
- **`/person/read-public`**: Read public biographical information
- **`/activities/update`**: Write/update activities (Member API only)
- **`/person/update`**: Write/update biographical information (Member API only)

### Scope Combinations

```
# OpenID Connect Authentication Only
openid

# OpenID + Read Limited (authenticated + limited data access)
openid /read-limited

# OpenID + Public Read
openid /read-public

# Member API: OpenID + Update Permissions
openid /activities/update /person/update

# Simplified: OpenID + All Activities
openid /activities/read-limited
```

### User Information Claims

The ID Token and UserInfo endpoint provide the following claims:

**Basic Claims:**

```json
{
  "sub": "0000-0001-2345-6789",          // ORCID iD (subject)
  "iss": "https://orcid.org",            // Issuer
  "aud": "APP-ABC123XYZ789",             // Audience (your client ID)
  "auth_time": 1505987862,               // Authentication time (unix timestamp)
  "iat": 1505987863,                     // Issued at
  "exp": 1505988463                      // Expiration time
}
```

**User Information Claims:**

```json
{
  "given_name": "Sofia",
  "family_name": "Garcia",
  "name": "Sofia Garcia"
}
```

**Advanced Claims (Member API with openid scope):**

```json
{
  "amr": ["password"],                   // Authentication method (shows if 2FA used)
  "emails": ["sofia@example.com"],       // Email addresses (limited visibility)
  "email_verified": true
}
```

### Data Visibility and Permissions

- **Public Scope**: Only publicly visible ORCID record data
- **Limited Scope**: User's limited-access data (requires explicit permission)
- **Private Scope**: Cannot be accessed via API; limited visibility only
- **Email**: Only accessible if user has made it public in their ORCID record

## 5. Differences Between ORCID OAuth2 and OpenID Connect

### OAuth 2.0 (Standard)

**Purpose**: Authorization and access control
- Obtain access tokens for API calls
- Request specific scopes (read, write permissions)
- Access ORCID record data through REST API

**Key Features:**
```
Authorization Endpoint: /oauth/authorize
Token Endpoint: /oauth/token
No ID Token by default
No UserInfo endpoint
Scopes are ORCID-specific (e.g., /read-limited, /activities/update)
Token types: Bearer tokens, Refresh tokens
```

**Response Example:**
```json
{
  "access_token": "f5af9f51-07e6-4332-8f1a-c0c11c1e3728",
  "token_type": "bearer",
  "refresh_token": "f725f747-3a65-49f6-a231-3e8944ce464d",
  "expires_in": 631138518,
  "scope": "/read-limited",
  "name": "Sofia Garcia",
  "orcid": "0000-0001-2345-6789"
}
```

### OpenID Connect (Built on OAuth 2.0)

**Purpose**: Authentication and user information
- Authenticate users with signed ID tokens
- Exchange information about authenticated users
- Standardized identity layer

**Key Features:**
```
Includes all OAuth 2.0 features PLUS:
ID Token: JWT containing user identity claims
UserInfo Endpoint: Returns authenticated user information
OpenID Connect scope required
Signed tokens (RS256 algorithm)
Discovery mechanism (.well-known/openid-configuration)
JWKS endpoint for signature verification
```

**Response Example:**
```json
{
  "access_token": "f5af9f51-07e6-4332-8f1a-c0c11c1e3728",
  "token_type": "bearer",
  "id_token": "eyJraWQiOiJrZXk...",
  "refresh_token": "f725f747-3a65-49f6-a231-3e8944ce464d",
  "expires_in": 631138518,
  "scope": "openid /read-limited"
}
```

### Key Differences Summary

| Feature | OAuth 2.0 | OpenID Connect |
|---------|-----------|-----------------|
| **Primary Use** | Authorization (access control) | Authentication (identity) |
| **ID Token** | No | Yes (signed JWT) |
| **UserInfo Endpoint** | No | Yes |
| **Signature Verification** | No | Yes (RS256) |
| **JWKS Endpoint** | No | Yes |
| **Discovery** | No | Yes |
| **User Info Format** | Custom response | Standard claims |
| **Scope Format** | ORCID-specific | Standard "openid" + ORCID scopes |
| **Use Case** | API access | Single sign-on, session management |

### Implementation Implications

**When to Use OAuth 2.0:**
- Need to read/write ORCID record data via API
- Member API integrations with update permissions
- Longer-lived access tokens (20 years default)
- Direct API interactions

**When to Use OpenID Connect:**
- User authentication and sign-in
- Session management
- Stateless authentication
- Integration with standard OpenID Connect libraries (Spring Security, etc.)
- User information verification

**Recommended:** Use OpenID Connect for authentication + specific ORCID scopes for data access
```
Scope: openid /read-limited
Response includes: ID token (authentication) + access token (data access)
```

## 6. Complete Spring Boot Configuration for ORCID OIDC

### POM Dependencies

```xml
<dependencies>
    <!-- Spring Boot Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Spring Security + OAuth2 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-client</artifactId>
    </dependency>

    <!-- OIDC Support (includes OAuth2 Client) -->
    <dependency>
        <groupId>org.springframework.security</groupId>
        <artifactId>spring-security-oauth2-core</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.security</groupId>
        <artifactId>spring-security-oauth2-client</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.security</groupId>
        <artifactId>spring-security-oauth2-jose</artifactId>
    </dependency>

    <!-- JWT Support -->
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.12.3</version>
    </dependency>

    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.12.3</version>
        <scope>runtime</scope>
    </dependency>

    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId>
        <version>0.12.3</version>
        <scope>runtime</scope>
    </dependency>

    <!-- Logging -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-logging</artifactId>
    </dependency>
</dependencies>
```

### Application Properties (application.yml)

```yaml
spring:
  application:
    name: orcid-oidc-integration
  
  security:
    oauth2:
      client:
        registration:
          orcid:
            # Client credentials (obtain from ORCID)
            client-id: ${ORCID_CLIENT_ID}
            client-secret: ${ORCID_CLIENT_SECRET}
            
            # OIDC Configuration
            provider: orcid
            client-name: ORCID
            client-authentication-method: client_secret_post
            authorization-grant-type: authorization_code
            redirect-uri: "http://localhost:8080/login/oauth2/code/orcid"
            
            # Request scopes
            scope:
              - openid
              - /read-limited
        
        provider:
          orcid:
            # Production endpoints
            issuer-uri: https://orcid.org
            authorization-uri: https://orcid.org/oauth/authorize
            token-uri: https://orcid.org/oauth/token
            user-info-uri: https://orcid.org/oauth/userinfo
            jwks-uri: https://orcid.org/oauth/jwks
            user-name-attribute: sub
            
            # OR for Sandbox environment:
            # issuer-uri: https://sandbox.orcid.org
            # authorization-uri: https://sandbox.orcid.org/oauth/authorize
            # token-uri: https://sandbox.orcid.org/oauth/token
            # user-info-uri: https://sandbox.orcid.org/oauth/userinfo
            # jwks-uri: https://sandbox.orcid.org/oauth/jwks

server:
  port: 8080
  servlet:
    session:
      cookie:
        secure: true
        http-only: true
        same-site: lax
```

### Security Configuration

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.oauth2.client.CommonOAuth2Provider;
import org.springframework.security.oauth2.client.registration.ClientRegistration;
import org.springframework.security.oauth2.client.registration.InMemoryClientRegistrationRepository;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.CorsConfigurationSource;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;

@Configuration
@EnableWebSecurity
public class OrcidSecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
                .antMatchers("/", "/public/**").permitAll()
                .anyRequest().authenticated()
            .and()
            .oauth2Login()
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard", true)
                .userInfoEndpoint()
                    .userService(new OrcidOAuth2UserService())
            .and()
            .and()
            .logout()
                .logoutSuccessUrl("/")
                .permitAll()
            .and()
            .csrf()
                .disable()
            .cors();
        
        return http.build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOrigins(
            List.of(
                "http://localhost:3000",
                "http://localhost:8080",
                "https://yourdomain.com"
            )
        );
        configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
        configuration.setAllowCredentials(true);
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);
        return source;
    }
}
```

### OAuth2 User Service for ORCID

```java
import org.springframework.security.oauth2.client.userinfo.DefaultOAuth2UserService;
import org.springframework.security.oauth2.client.userinfo.OAuth2UserRequest;
import org.springframework.security.oauth2.core.OAuth2AuthenticationException;
import org.springframework.security.oauth2.core.user.DefaultOAuth2User;
import org.springframework.security.oauth2.core.user.OAuth2User;
import org.springframework.stereotype.Service;
import java.util.HashMap;
import java.util.Map;

@Service
public class OrcidOAuth2UserService extends DefaultOAuth2UserService {

    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) 
            throws OAuth2AuthenticationException {
        
        OAuth2User user = super.loadUser(userRequest);
        
        // Extract ORCID-specific information
        Map<String, Object> attributes = new HashMap<>(user.getAttributes());
        
        String orcidId = (String) attributes.get("sub");
        String givenName = (String) attributes.get("given_name");
        String familyName = (String) attributes.get("family_name");
        String name = (String) attributes.get("name");
        
        // Create enhanced user with ORCID information
        Map<String, Object> enhancedAttributes = new HashMap<>(attributes);
        enhancedAttributes.put("orcid_id", orcidId);
        enhancedAttributes.put("given_name", givenName);
        enhancedAttributes.put("family_name", familyName);
        
        return new DefaultOAuth2User(
            user.getAuthorities(),
            enhancedAttributes,
            "sub"
        );
    }
}
```

### Controller Example

```java
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.core.user.OAuth2User;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.servlet.view.RedirectView;

@RestController
@RequestMapping("/api")
public class OrcidController {

    @GetMapping("/user")
    public OrcidUserInfo getCurrentUser(
            @AuthenticationPrincipal OAuth2User principal) {
        
        if (principal == null) {
            return null;
        }
        
        String orcidId = (String) principal.getAttribute("sub");
        String givenName = (String) principal.getAttribute("given_name");
        String familyName = (String) principal.getAttribute("family_name");
        
        return new OrcidUserInfo(orcidId, givenName, familyName);
    }

    @PostMapping("/logout")
    public RedirectView logout() {
        return new RedirectView("/");
    }
}

@Controller
@RequestMapping("/")
public class HomeController {

    @GetMapping
    public String home(Model model, 
            @AuthenticationPrincipal OAuth2User principal) {
        if (principal != null) {
            String orcidId = (String) principal.getAttribute("sub");
            model.addAttribute("orcidId", orcidId);
            model.addAttribute("name", principal.getAttribute("name"));
        }
        return "home";
    }

    @GetMapping("/dashboard")
    public String dashboard(Model model,
            @AuthenticationPrincipal OAuth2User principal) {
        if (principal != null) {
            model.addAttribute("user", principal);
        }
        return "dashboard";
    }
}
```

### POJO for User Information

```java
public class OrcidUserInfo {
    private String orcidId;
    private String givenName;
    private String familyName;
    private String fullName;

    public OrcidUserInfo(String orcidId, String givenName, String familyName) {
        this.orcidId = orcidId;
        this.givenName = givenName;
        this.familyName = familyName;
        this.fullName = givenName + " " + familyName;
    }

    // Getters and setters
    public String getOrcidId() { return orcidId; }
    public String getGivenName() { return givenName; }
    public String getFamilyName() { return familyName; }
    public String getFullName() { return fullName; }
}
```

## 7. Code Examples for ORCID Integration

### Example 1: Basic Spring Boot OAuth2 Login

```java
// HTML Template (login.html)
<!DOCTYPE html>
<html>
<head>
    <title>ORCID Login</title>
</head>
<body>
    <h1>Sign in with ORCID</h1>
    <a href="/oauth2/authorization/orcid" class="btn btn-primary">
        Login with ORCID
    </a>
</body>
</html>
```

### Example 2: JWT Token Verification

```java
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.JwtException;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

@Service
public class OrcidJwtValidator {
    
    @Value("${spring.security.oauth2.client.provider.orcid.jwks-uri}")
    private String jwksUri;
    
    private RestTemplate restTemplate = new RestTemplate();
    
    public Claims validateToken(String token) throws JwtException {
        // Fetch JWKS from ORCID
        Map<String, Object> jwks = restTemplate.getForObject(jwksUri, Map.class);
        
        // Parse and validate JWT
        Claims claims = Jwts.parserBuilder()
            // Note: In production, fetch public keys from JWKS endpoint
            .build()
            .parseClaimsJws(token)
            .getBody();
        
        // Validate issuer
        String issuer = claims.getIssuer();
        if (!issuer.equals("https://orcid.org")) {
            throw new JwtException("Invalid issuer");
        }
        
        return claims;
    }
}
```

### Example 3: Accessing ORCID UserInfo Endpoint

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.oauth2.client.OAuth2AuthorizedClient;
import org.springframework.security.oauth2.client.OAuth2AuthorizedClientService;
import org.springframework.security.oauth2.client.authentication.OAuth2AuthenticationToken;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

@Service
public class OrcidUserInfoService {
    
    @Autowired
    private OAuth2AuthorizedClientService clientService;
    
    @Autowired
    private RestTemplate restTemplate;
    
    public Map<String, Object> getUserInfo(OAuth2AuthenticationToken auth) {
        OAuth2AuthorizedClient client = clientService.loadAuthorizedClient(
            "orcid",
            auth.getName()
        );
        
        if (client == null) {
            throw new IllegalStateException("ORCID client not found");
        }
        
        String accessToken = client.getAccessToken().getTokenValue();
        
        // Call userinfo endpoint
        HttpHeaders headers = new HttpHeaders();
        headers.setBearerAuth(accessToken);
        HttpEntity<String> entity = new HttpEntity<>(headers);
        
        ResponseEntity<Map> response = restTemplate.exchange(
            "https://orcid.org/oauth/userinfo",
            HttpMethod.GET,
            entity,
            Map.class
        );
        
        return response.getBody();
    }
}
```

### Example 4: Reading ORCID Record Data

```java
@Service
public class OrcidRecordService {
    
    private static final String ORCID_API_BASE = "https://pub.orcid.org/v3.0";
    
    @Autowired
    private RestTemplate restTemplate;
    
    public Map<String, Object> getProfile(String orcidId, String accessToken) {
        HttpHeaders headers = new HttpHeaders();
        headers.setBearerAuth(accessToken);
        headers.setAccept(List.of(MediaType.APPLICATION_JSON));
        
        HttpEntity<String> entity = new HttpEntity<>(headers);
        
        String url = ORCID_API_BASE + "/" + orcidId;
        
        ResponseEntity<Map> response = restTemplate.exchange(
            url,
            HttpMethod.GET,
            entity,
            Map.class
        );
        
        return response.getBody();
    }
    
    public Map<String, Object> getWorks(String orcidId, String accessToken) {
        String url = ORCID_API_BASE + "/" + orcidId + "/works";
        
        HttpHeaders headers = new HttpHeaders();
        headers.setBearerAuth(accessToken);
        
        HttpEntity<String> entity = new HttpEntity<>(headers);
        
        ResponseEntity<Map> response = restTemplate.exchange(
            url,
            HttpMethod.GET,
            entity,
            Map.class
        );
        
        return response.getBody();
    }
}
```

### Example 5: OAuth2 Token Refresh

```java
@Service
public class OrcidTokenRefreshService {
    
    @Autowired
    private OAuth2AuthorizedClientService clientService;
    
    public void refreshToken(OAuth2AuthenticationToken auth) {
        OAuth2AuthorizedClient client = clientService.loadAuthorizedClient(
            "orcid",
            auth.getName()
        );
        
        if (client != null && client.getRefreshToken() != null) {
            // Spring Security automatically handles token refresh
            // via OAuth2AuthorizationCodeGrantFilter
            clientService.saveAuthorizedClient(client, auth);
        }
    }
}
```

## 8. Best Practices and Common Issues with ORCID

### Best Practices

#### 1. Authentication Flow

```
✓ Use OpenID Connect for authentication
✓ Combine openid scope with specific ORCID scopes
✓ Verify ID token signature before trusting claims
✓ Store ORCID iD, not access tokens, as user identifier
✓ Use HTTPS redirect URIs in production
```

#### 2. Scope Management

```
✓ Request only necessary scopes
✓ Explain why each scope is needed to users
✓ Use /read-limited for accessing private research data
✓ Use /read-public only for public profiles
✓ Document scope requirements
```

#### 3. Data Handling

```
✓ Cache ORCID iD permanently
✓ Cache access tokens in secure session/database
✓ Refresh tokens before expiration
✓ Handle user profile visibility settings gracefully
✓ Don't assume email is always available
```

#### 4. Error Handling

```
✓ Handle token expiration gracefully
✓ Retry failed API calls with exponential backoff
✓ Log ORCID API errors for debugging
✓ Display user-friendly error messages
✓ Monitor API rate limits (10k requests/day)
```

#### 5. Security

```
✓ Never expose client secret in frontend
✓ Use HTTPS for all communications
✓ Validate redirect URIs strictly
✓ Sign and verify JWT tokens
✓ Use secure session cookies
✓ Implement CSRF protection
✓ Sanitize user input before display
```

#### 6. User Experience

```
✓ Offer ORCID as primary sign-in method
✓ Show ORCID iD in user profile
✓ Display ORCID logo correctly (per brand guidelines)
✓ Link to public ORCID profile (https://orcid.org/{orcid-id})
✓ Provide clear logout mechanism
✓ Allow account linking without re-authentication
```

### Common Issues and Solutions

#### Issue 1: "Invalid redirect URI" Error

**Symptoms:**
- Users see error after ORCID authentication
- Redirect fails with "redirect_uri_mismatch" error

**Solutions:**
```
1. Check exact domain matching (subdomains must match exactly)
2. Ensure HTTPS in production (HTTP only in development)
3. Include full path in redirect URI registration
4. No trailing slashes - both must match exactly
5. Update registration if path changes
```

**Example Matching:**
```
Registered: https://api.example.com/oauth/callback
✓ Works: https://api.example.com/oauth/callback
✗ Fails: https://api.example.com/oauth/callback/ (trailing slash)
✗ Fails: https://api.example.com/auth/callback (different path)
✗ Fails: https://example.com/oauth/callback (different subdomain)
```

#### Issue 2: Email Not Available

**Symptoms:**
- User profile information incomplete
- Email claim missing from JWT

**Solutions:**
```
1. Don't require email from ORCID
2. Prompt user to provide email manually
3. Check user's ORCID privacy settings (may have email private)
4. Use ORCID iD as unique identifier instead
5. Validate email through separate flow if needed
```

**Code Example:**
```java
String email = (String) claims.get("email");
if (email == null || email.isEmpty()) {
    // Fallback: request email from user
    // Don't assume it's available from ORCID
}
```

#### Issue 3: Token Expiration

**Symptoms:**
- Access token no longer valid after time period
- Refresh token expired

**Solutions:**
```
1. Implement token refresh mechanism
2. Check token expiration before API calls
3. Refresh tokens are long-lived (20 years)
4. Store refresh token securely
5. Re-authenticate if refresh fails
```

**Code Example:**
```java
if (accessToken.getExpiresAt().isBefore(LocalDateTime.now())) {
    refreshAccessToken();
}
```

#### Issue 4: Sandbox vs Production

**Symptoms:**
- Works in sandbox, fails in production
- Different endpoints or credentials

**Solutions:**
```
1. Use environment variables for endpoints
2. Separate configuration for sandbox/production
3. Test thoroughly in sandbox first
4. Update credentials when moving to production
5. Handle both environments in CI/CD
```

**Configuration:**
```yaml
orcid:
  environment: ${ORCID_ENV:sandbox}
  baseUrl: ${ORCID_BASE_URL:https://sandbox.orcid.org}
  clientId: ${ORCID_CLIENT_ID}
  clientSecret: ${ORCID_CLIENT_SECRET}
```

#### Issue 5: Rate Limiting

**Symptoms:**
- HTTP 429 (Too Many Requests) responses
- Inconsistent API availability

**Solutions:**
```
1. Implement request rate limiting (10k/day limit)
2. Cache responses when possible
3. Batch requests efficiently
4. Implement exponential backoff for retries
5. Monitor usage patterns
```

**Example:**
```java
@Retryable(maxAttempts = 3, backoff = @Backoff(delay = 1000))
public Map<String, Object> getProfile(String orcidId) {
    // API call with automatic retry on rate limit
}
```

#### Issue 6: Scope Validation in Sandbox

**Symptoms:**
- Scopes work in sandbox but rejected in production
- Insufficient permissions error

**Solutions:**
```
1. Sandbox allows testing all scopes for free
2. Production requires Member API for write scopes
3. Some scopes only available to ORCID members
4. Verify membership status for production
5. Request proper scopes based on API type
```

**Scope Availability:**
```
Public API:
  ✓ /authenticate (read-only)
  ✓ /read-public
  ✓ openid

Member API:
  ✓ All public API scopes
  ✓ /read-limited
  ✓ /activities/update
  ✓ /person/update
```

#### Issue 7: JWT Signature Verification

**Symptoms:**
- "Invalid signature" errors
- Unable to verify token authenticity

**Solutions:**
```
1. Fetch public keys from ORCID JWKS endpoint
2. Verify signature before trusting claims
3. Check token issuer matches expected value
4. Validate audience (client ID) in token
5. Check token expiration (exp claim)
```

**Code Example:**
```java
// Fetch JWKS from ORCID
RestTemplate rest = new RestTemplate();
Map jwks = rest.getForObject(
    "https://orcid.org/oauth/jwks", 
    Map.class
);

// Use JwtProcessorBuilder with JWKS
JwtProcessor<SecurityContext> processor = 
    new DefaultJwtProcessorBuilder<SecurityContext>()
        .withJWKSetSource(new ImmutableJWKSet(jwks))
        .build();
```

#### Issue 8: Account Linking

**Symptoms:**
- Users unable to link existing account with ORCID
- Duplicate accounts after ORCID sign-in

**Solutions:**
```
1. Check if ORCID iD already linked before creating account
2. Offer linking to existing account
3. Don't require email match for linking
4. Store ORCID iD as permanent account identifier
5. Prevent duplicate accounts
```

**Recommended Flow:**
```
1. User clicks "Sign in with ORCID"
2. Authenticate and receive ORCID iD
3. Check if ORCID iD exists in database
4. If exists: Link session and sign in
5. If not exists: Offer to link or create new account
```

### Monitoring and Logging

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class OrcidAuditService {
    private static final Logger logger = LoggerFactory.getLogger(OrcidAuditService.class);
    
    public void logOrcidAuthentication(String orcidId, String status) {
        logger.info("ORCID Authentication - ORCID ID: {}, Status: {}", 
            orcidId, status);
    }
    
    public void logOrcidApiCall(String endpoint, int statusCode, long duration) {
        logger.debug("ORCID API Call - Endpoint: {}, Status: {}, Duration: {}ms",
            endpoint, statusCode, duration);
    }
    
    public void logOrcidError(String orcidId, String error, Exception e) {
        logger.error("ORCID Error - ORCID ID: {}, Error: {}", 
            orcidId, error, e);
    }
}
```

## Additional Resources

- **Official Documentation**: https://info.orcid.org/documentation/
- **API Users Group**: https://groups.google.com/g/orcid-api-users
- **OpenID Connect Examples**: https://github.com/ORCID/orcid-openid-examples
- **Sandbox Testing Server**: https://sandbox.orcid.org
- **ORCID GitHub**: https://github.com/ORCID
- **OpenID Connect Spec**: https://openid.net/specs/openid-connect-core-1_0.html
- **OAuth 2.0 Spec**: https://tools.ietf.org/html/rfc6749
