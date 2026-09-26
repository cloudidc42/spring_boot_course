# Part 35: OAuth2 & OpenID Connect
## ขั้นตอนที่ 1001-1040

> **ระดับ:** ระดับโลก (World-Class)  
> **เวลาเรียน:** 6-7 ชั่วโมง  
> **เป้าหมาย:** Implement OAuth2 server และ social login

---

## ขั้นตอนที่ 1001: OAuth2 Concepts

```
OAuth2 Roles:
  Resource Owner = User
  Client         = Your app
  Authorization Server = Google/GitHub/your own
  Resource Server = API ที่ protect ด้วย token

OAuth2 Flows:
  Authorization Code  → Web apps (most secure)
  Client Credentials  → Machine-to-machine
  Device Code         → Smart TV, CLI tools
  PKCE + Auth Code    → Mobile/SPA apps (recommended)

Tokens:
  Access Token  = Short-lived (15min-1hr), used to call APIs
  Refresh Token = Long-lived, used to get new access token
  ID Token      = JWT with user info (OIDC specific)

OpenID Connect (OIDC):
  OAuth2 + identity layer
  Adds ID token + /userinfo endpoint
  Used for SSO (Single Sign-On)
```

---

## ขั้นตอนที่ 1002: Social Login with Spring Security OAuth2

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

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
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
          
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            scope: user:email,read:user
          
          facebook:
            client-id: ${FACEBOOK_CLIENT_ID}
            client-secret: ${FACEBOOK_CLIENT_SECRET}
            scope: email,public_profile
```

---

## ขั้นตอนที่ 1003: OAuth2 Security Configuration

```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class OAuth2SecurityConfig {
    
    private final OAuth2UserService customOAuth2UserService;
    private final OAuth2AuthenticationSuccessHandler successHandler;
    private final OAuth2AuthenticationFailureHandler failureHandler;
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**", "/oauth2/**").permitAll()
                .anyRequest().authenticated()
            )
            
            // JWT filter
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            
            // OAuth2 login
            .oauth2Login(oauth2 -> oauth2
                .authorizationEndpoint(e -> e.baseUri("/oauth2/authorize"))
                .redirectionEndpoint(e -> e.baseUri("/login/oauth2/code/*"))
                .userInfoEndpoint(ui -> ui.userService(customOAuth2UserService))
                .successHandler(successHandler)
                .failureHandler(failureHandler)
            )
            
            .build();
    }
}
```

---

## ขั้นตอนที่ 1004: Custom OAuth2 User Service

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class CustomOAuth2UserService extends DefaultOAuth2UserService {
    
    private final UserRepository userRepository;
    
    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
        OAuth2User oAuth2User = super.loadUser(userRequest);
        
        String provider = userRequest.getClientRegistration().getRegistrationId();
        
        OAuth2UserInfo userInfo = OAuth2UserInfoFactory.create(provider, oAuth2User.getAttributes());
        
        if (!StringUtils.hasText(userInfo.getEmail())) {
            throw new OAuth2AuthenticationException("Email not found from OAuth2 provider");
        }
        
        User user = userRepository.findByEmail(userInfo.getEmail())
            .map(existing -> updateExistingUser(existing, userInfo))
            .orElseGet(() -> registerNewUser(userInfo, provider));
        
        return UserPrincipal.create(user, oAuth2User.getAttributes());
    }
    
    private User registerNewUser(OAuth2UserInfo userInfo, String provider) {
        log.info("Registering new OAuth2 user: {} via {}", userInfo.getEmail(), provider);
        
        return userRepository.save(User.builder()
            .email(userInfo.getEmail())
            .name(userInfo.getName())
            .imageUrl(userInfo.getImageUrl())
            .provider(AuthProvider.valueOf(provider.toUpperCase()))
            .providerId(userInfo.getId())
            .emailVerified(true)
            .roles(Set.of(Role.USER))
            .build());
    }
    
    private User updateExistingUser(User user, OAuth2UserInfo userInfo) {
        user.setName(userInfo.getName());
        user.setImageUrl(userInfo.getImageUrl());
        return userRepository.save(user);
    }
}

// User info abstraction
public interface OAuth2UserInfo {
    String getId();
    String getName();
    String getEmail();
    String getImageUrl();
}

public class GoogleOAuth2UserInfo implements OAuth2UserInfo {
    private final Map<String, Object> attributes;
    
    public GoogleOAuth2UserInfo(Map<String, Object> attributes) {
        this.attributes = attributes;
    }
    
    @Override public String getId() { return (String) attributes.get("sub"); }
    @Override public String getName() { return (String) attributes.get("name"); }
    @Override public String getEmail() { return (String) attributes.get("email"); }
    @Override public String getImageUrl() { return (String) attributes.get("picture"); }
}

public class GithubOAuth2UserInfo implements OAuth2UserInfo {
    private final Map<String, Object> attributes;
    
    @Override public String getId() { return String.valueOf(attributes.get("id")); }
    @Override public String getName() { return (String) attributes.get("login"); }
    @Override public String getEmail() { return (String) attributes.get("email"); }
    @Override public String getImageUrl() { return (String) attributes.get("avatar_url"); }
}

public class OAuth2UserInfoFactory {
    public static OAuth2UserInfo create(String provider, Map<String, Object> attrs) {
        return switch (provider.toLowerCase()) {
            case "google" -> new GoogleOAuth2UserInfo(attrs);
            case "github" -> new GithubOAuth2UserInfo(attrs);
            default -> throw new OAuth2AuthenticationException("Unsupported provider: " + provider);
        };
    }
}
```

---

## ขั้นตอนที่ 1005: OAuth2 Success Handler

```java
@Component
@RequiredArgsConstructor
public class OAuth2AuthenticationSuccessHandler extends SimpleUrlAuthenticationSuccessHandler {
    
    private final JwtService jwtService;
    private final AppProperties appProperties;
    
    @Override
    public void onAuthenticationSuccess(
        HttpServletRequest request,
        HttpServletResponse response,
        Authentication authentication
    ) throws IOException {
        
        UserPrincipal userPrincipal = (UserPrincipal) authentication.getPrincipal();
        
        String accessToken = jwtService.generateToken(userPrincipal);
        String refreshToken = jwtService.generateRefreshToken(userPrincipal);
        
        String redirectUri = appProperties.getOauth2().getAuthorizedRedirectUri();
        
        // Redirect to frontend with tokens
        String targetUrl = UriComponentsBuilder.fromUriString(redirectUri)
            .queryParam("accessToken", accessToken)
            .queryParam("refreshToken", refreshToken)
            .build().toUriString();
        
        getRedirectStrategy().sendRedirect(request, response, targetUrl);
    }
}

@Component
public class OAuth2AuthenticationFailureHandler extends SimpleUrlAuthenticationFailureHandler {
    
    @Override
    public void onAuthenticationFailure(
        HttpServletRequest request,
        HttpServletResponse response,
        AuthenticationException exception
    ) throws IOException {
        
        String redirectUri = "/login?error=" + URLEncoder.encode(exception.getMessage(), StandardCharsets.UTF_8);
        getRedirectStrategy().sendRedirect(request, response, redirectUri);
    }
}
```

---

## ขั้นตอนที่ 1006: Authorization Server (Spring Authorization Server)

```xml
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-oauth2-authorization-server</artifactId>
</dependency>
```

```java
@Configuration
public class AuthorizationServerConfig {
    
    @Bean
    @Order(1)
    public SecurityFilterChain authorizationServerSecurityFilterChain(HttpSecurity http) throws Exception {
        OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http);
        
        http
            .getConfigurer(OAuth2AuthorizationServerConfigurer.class)
            .oidc(Customizer.withDefaults());  // Enable OIDC
        
        return http
            .exceptionHandling(e -> e.defaultAuthenticationEntryPointFor(
                new LoginUrlAuthenticationEntryPoint("/login"),
                new MediaTypeRequestMatcher(MediaType.TEXT_HTML)
            ))
            .oauth2ResourceServer(r -> r.jwt(Customizer.withDefaults()))
            .build();
    }
    
    @Bean
    public RegisteredClientRepository registeredClientRepository(JdbcTemplate jdbcTemplate) {
        // Store clients in database
        JdbcRegisteredClientRepository repository = new JdbcRegisteredClientRepository(jdbcTemplate);
        
        // Register a client
        RegisteredClient client = RegisteredClient.withId(UUID.randomUUID().toString())
            .clientId("my-app")
            .clientSecret("{bcrypt}" + new BCryptPasswordEncoder().encode("secret"))
            .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
            .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
            .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
            .authorizationGrantType(AuthorizationGrantType.CLIENT_CREDENTIALS)
            .redirectUri("http://localhost:3000/callback")
            .scope(OidcScopes.OPENID)
            .scope(OidcScopes.PROFILE)
            .scope("read")
            .scope("write")
            .tokenSettings(TokenSettings.builder()
                .accessTokenTimeToLive(Duration.ofHours(1))
                .refreshTokenTimeToLive(Duration.ofDays(30))
                .build())
            .build();
        
        repository.save(client);
        return repository;
    }
    
    @Bean
    public JWKSource<SecurityContext> jwkSource() {
        RSAKey rsaKey = generateRsa();
        JWKSet jwkSet = new JWKSet(rsaKey);
        return (selector, context) -> selector.select(jwkSet);
    }
    
    private static RSAKey generateRsa() {
        KeyPair keyPair = generateRsaKey();
        return new RSAKey.Builder((RSAPublicKey) keyPair.getPublic())
            .privateKey(keyPair.getPrivate())
            .keyID(UUID.randomUUID().toString())
            .build();
    }
    
    private static KeyPair generateRsaKey() {
        try {
            KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("RSA");
            keyPairGenerator.initialize(2048);
            return keyPairGenerator.generateKeyPair();
        } catch (Exception ex) {
            throw new IllegalStateException(ex);
        }
    }
}
```

---

## ขั้นตอนที่ 1007-1040: Resource Server

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class ResourceServerConfig {
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/v1/products/**").hasAuthority("SCOPE_read")
                .requestMatchers(HttpMethod.POST, "/api/v1/products/**").hasAuthority("SCOPE_write")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(rs -> rs
                .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtConverter()))
            )
            .build();
    }
    
    @Bean
    public JwtAuthenticationConverter jwtConverter() {
        JwtGrantedAuthoritiesConverter grantedAuthoritiesConverter = new JwtGrantedAuthoritiesConverter();
        grantedAuthoritiesConverter.setAuthoritiesClaimName("roles");
        grantedAuthoritiesConverter.setAuthorityPrefix("ROLE_");
        
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(grantedAuthoritiesConverter);
        return converter;
    }
}

// OAuth2 Login Flow Summary:
// 1. User clicks "Login with Google"
// 2. Browser redirects to /oauth2/authorize/google
// 3. Spring redirects to Google OAuth2 page
// 4. User logs in on Google
// 5. Google redirects back with auth code
// 6. Spring exchanges code for tokens
// 7. CustomOAuth2UserService loads/creates user
// 8. SuccessHandler generates JWT and redirects to frontend
// 9. Frontend stores JWT and uses for future requests
```

---

*[← Part 34: CQRS](./part-34-cqrs.md) | [Part 36: Kubernetes →](./part-36-kubernetes.md)*
