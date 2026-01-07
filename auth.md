# OAuth 2.1 인증 서버

> Hexagonal Architecture 기반 SSO(Single Sign-On) 인프라 구현

`Java 21` `Spring Boot 4` `Spring Authorization Server` `OAuth 2.1` `OIDC` `PostgreSQL` `Testcontainers`

## 1. 프로젝트 개요

### 배경

개인 도메인의 여러 서비스에서 공통으로 사용할 인증 시스템이 필요했습니다. 최신 OAuth 2.1 스펙을 준수하면서 PKCE, Client Credentials, Device Authorization Flow 등 다양한 인증 시나리오를 지원하는 Authorization Server를 직접 구현해보는 것이 목표였습니다.

### 핵심 목표

- **OAuth 2.1 준수**: PKCE 필수화, Device Flow 등 최신 스펙 구현
- **Hexagonal Architecture**: 도메인 로직의 완전한 격리와 테스트 용이성
- **RFC 표준 준수**: RFC 5321(Email), RFC 9457(Problem Details), RFC 7636(PKCE)
- **DDD 적용**: Value Object, Aggregate, Factory Method 패턴 활용

코드베이스: [github.com/kontang5/auth](https://github.com/kontang5/auth)

## 2. 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│                        External Clients                         │
│              Web Apps · Mobile Apps · CLI · IoT Devices         │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Adapter (In) - Web                         │
│     AuthController · DeviceController · ConsentController       │
│              GlobalExceptionHandler (RFC 9457)                  │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Application Layer                           │
│      UserService · RegisterUserUseCase · GetUserUseCase         │
│                    Port In/Out Interfaces                       │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                       Domain Layer                              │
│          User (Aggregate) · Email (Value Object) · Role         │
│            No Spring Dependencies · Pure Business Logic         │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Adapter (Out) - Persistence                  │
│      UserRepositoryAdapter · JPA Entity · Spring Data JPA       │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Infrastructure                             │
│          PostgreSQL · OAuth2 Tables · Flyway Migrations         │
└─────────────────────────────────────────────────────────────────┘
```

## 3. 주요 구현 내용

### 3.1 Domain-Driven Design: User Aggregate

도메인 레이어는 Spring/JPA 의존성이 전혀 없는 순수 Java로 구현됩니다.

```java
// User.java - Aggregate Root
public class User {

    private final Long id;
    private final String publicId;
    private final Email email;
    private final String passwordHash;
    private final Role role;
    private final boolean enabled;

    // Factory Method - 새 사용자 생성
    public static User create(Email email, String passwordHash, Role role) {
        Objects.requireNonNull(email, "Email must not be null");
        Objects.requireNonNull(role, "Role must not be null");
        validatePasswordHash(passwordHash);

        Instant now = Instant.now();
        String publicId = UlidCreator.getUlid().toString();

        return new User(null, publicId, email, passwordHash, role, true, now, now);
    }

    // Reconstitution - 영속성에서 복원
    public static User reconstitute(Long id, String publicId, Email email,
            String passwordHash, Role role, boolean enabled,
            Instant createdAt, Instant updatedAt) {
        return new User(id, publicId, email, passwordHash, role, enabled, createdAt, updatedAt);
    }

    // 상태 변경 메서드 - Immutable
    public User disable() {
        return new User(id, publicId, email, passwordHash, role, false, createdAt, Instant.now());
    }
}
```

| 설계 결정               | 이유                          |
|---------------------|-----------------------------|
| ULID Public ID      | 내부 DB ID 노출 방지, URL-safe, 정렬 가능 |
| Factory Method      | 불변식(Invariant) 강제, 생성 로직 캡슐화   |
| Reconstitute 분리     | 영속성 복원 시 검증 스킵, 성능 최적화        |
| Immutable 상태 변경     | Thread-safe, 부수효과 없음         |

### 3.2 Value Object: RFC 5321 Email 검증

이메일 검증을 단순 정규식이 아닌 RFC 5321 스펙에 맞게 구현합니다.

```java
// Email.java - Value Object
public final class Email {

    private static final int MAX_TOTAL_LENGTH = 254;
    private static final int MAX_LOCAL_PART_LENGTH = 64;

    public static Email of(String email) {
        if (email == null || email.isBlank()) {
            throw new InvalidEmailException("Email cannot be null or blank");
        }

        String normalized = email.strip().toLowerCase();

        if (normalized.length() > MAX_TOTAL_LENGTH) {
            throw new InvalidEmailException("Email exceeds maximum length");
        }

        String localPart = normalized.substring(0, normalized.indexOf('@'));

        if (localPart.length() > MAX_LOCAL_PART_LENGTH) {
            throw new InvalidEmailException("Local part exceeds maximum length");
        }

        if (normalized.contains("..")) {
            throw new InvalidEmailException("Email cannot contain consecutive dots");
        }

        if (!EMAIL_PATTERN.matcher(normalized).matches()) {
            throw new InvalidEmailException("Invalid email format");
        }

        return new Email(normalized);
    }
}
```

| 검증 규칙           | RFC 5321 근거                |
|-----------------|---------------------------|
| 최대 254자         | 전체 이메일 주소 길이 제한            |
| Local Part 64자  | @ 앞부분 길이 제한                |
| 연속 점(..) 금지     | 메일 라우팅 문제 방지               |
| 소문자 정규화         | 대소문자 구분 없는 비교를 위한 표준화      |

### 3.3 OAuth 2.1 Authorization Server

Spring Authorization Server를 기반으로 PKCE, Client Credentials, Device Flow를 구현합니다.

```java
// SecurityConfig.java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) {
    http
        .oauth2AuthorizationServer(authorizationServer -> authorizationServer
            .oidc(Customizer.withDefaults())
            .deviceAuthorizationEndpoint(device -> device
                .verificationUri("/device")
            )
            .deviceVerificationEndpoint(device -> device
                .consentPage("/oauth2/consent")
            )
        )
        .authorizeHttpRequests(authorize -> authorize
            .requestMatchers("/api/users/**").hasRole("ADMIN")
            .requestMatchers("/oauth2/device_authorization").permitAll()
            .anyRequest().authenticated()
        )
        .formLogin(form -> form.loginPage("/login").permitAll());

    return http.build();
}
```

| Grant Type           | Endpoint                       | 용도                 |
|----------------------|--------------------------------|--------------------|
| Authorization Code + PKCE | `/oauth2/authorize`            | 웹/모바일 앱 인증         |
| Client Credentials   | `/oauth2/token`                | 서비스 간 인증 (M2M)      |
| Device Authorization | `/oauth2/device_authorization` | CLI, IoT 디바이스 인증   |

### 3.4 RSA Key Management

JWT 서명을 위한 RSA 키 쌍을 파일에서 로드하거나 런타임에 생성합니다.

```java
// AuthorizationServerConfig.java
@Bean
public JWKSource<SecurityContext> jwkSource() {
    KeyPair keyPair = loadOrGenerateKeyPair();
    RSAPublicKey publicKey = (RSAPublicKey) keyPair.getPublic();
    RSAPrivateKey privateKey = (RSAPrivateKey) keyPair.getPrivate();

    RSAKey rsaKey = new RSAKey.Builder(publicKey)
            .privateKey(privateKey)
            .keyID(keyId)
            .build();

    JWKSet jwkSet = new JWKSet(rsaKey);
    return new ImmutableJWKSet<>(jwkSet);
}

private KeyPair loadOrGenerateKeyPair() {
    if (privateKeyPath.isBlank() || publicKeyPath.isBlank()) {
        log.warn("RSA key paths not configured. Generating runtime keys.");
        return generateRsaKey();  // 개발 환경용 임시 키
    }
    // PEM 파일에서 PKCS8/X509 형식 키 로드
    RSAPrivateKey privateKey = loadPrivateKey(privateKeyPath);
    RSAPublicKey publicKey = loadPublicKey(publicKeyPath);
    return new KeyPair(publicKey, privateKey);
}
```

> 프로덕션에서는 환경변수로 키 경로를 주입, 개발 환경에서는 런타임 생성

### 3.5 RFC 9457 Problem Details 예외 처리

클라이언트에는 일반적인 메시지를, 로그에는 상세 정보를 기록하는 3계층 예외 전략입니다.

```java
// GlobalExceptionHandler.java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(DuplicateEmailException.class)
    public ProblemDetail handleDuplicateEmail(DuplicateEmailException ex) {
        log.warn("Duplicate email registration attempt: {}", ex.getMessage());
        // 클라이언트에는 일반 메시지만 반환 (보안)
        return ProblemDetail.forStatusAndDetail(
            HttpStatus.CONFLICT, "Email already registered");
    }

    @ExceptionHandler(DomainException.class)
    public ProblemDetail handleDomainException(DomainException ex) {
        log.warn("Domain validation failed: {}", ex.getMessage());
        return ProblemDetail.forStatusAndDetail(
            HttpStatus.BAD_REQUEST, "Invalid request data");
    }
}
```

```
예외 흐름:
Domain Layer          Application Layer         Presentation Layer
     │                       │                         │
InvalidEmailException ──▶ 전파 ──────────────▶ ProblemDetail (400)
                                               "Invalid request data"
                                               (상세 메시지는 로그에만)
```

### 3.6 Hexagonal Architecture: Port & Adapter

도메인 로직과 인프라를 완전히 분리합니다.

```java
// Port (Out) - 도메인이 정의하는 인터페이스
public interface UserRepository {
    User save(User user);
    Optional<User> findByEmail(Email email);
    boolean existsByEmail(Email email);
}

// Adapter (Out) - 인프라가 구현
@Repository
public class UserRepositoryAdapter implements UserRepository {

    private final JpaUserRepository jpaRepository;

    @Override
    public User save(User user) {
        UserEntity entity = toEntity(user);
        UserEntity saved = jpaRepository.save(entity);
        return toDomain(saved);
    }

    @Override
    public Optional<User> findByEmail(Email email) {
        return jpaRepository.findByEmail(email.value())
                .map(this::toDomain);
    }
}
```

| 레이어          | 의존성                   | 역할                   |
|--------------|-------------------------|----------------------|
| Domain       | 없음 (Pure Java)          | 비즈니스 규칙, 불변식         |
| Application  | Domain                   | Use Case 오케스트레이션     |
| Adapter In   | Application, Domain       | HTTP → Use Case 변환   |
| Adapter Out  | Application, Domain, JPA  | Domain → DB 영속화      |

## 4. 테스트 전략

Testcontainers로 실제 PostgreSQL을 사용한 통합 테스트를 수행합니다.

```java
// OAuth2IntegrationTest.java
@SpringBootTest
@AutoConfigureMockMvc
@Import(TestcontainersConfiguration.class)
class OAuth2IntegrationTest {

    @Test
    @DisplayName("Client credentials grant returns access token")
    void clientCredentialsGrant() throws Exception {
        String credentials = Base64.getEncoder()
                .encodeToString((CLIENT_ID + ":" + CLIENT_SECRET).getBytes());

        mockMvc.perform(post("/oauth2/token")
                        .header("Authorization", "Basic " + credentials)
                        .contentType(MediaType.APPLICATION_FORM_URLENCODED)
                        .param("grant_type", "client_credentials")
                        .param("scope", "service.read"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.access_token").exists())
                .andExpect(jsonPath("$.token_type").value("Bearer"));
    }

    @Test
    @DisplayName("Device authorization endpoint returns device and user codes")
    void deviceAuthorizationEndpoint() throws Exception {
        mockMvc.perform(post("/oauth2/device_authorization")
                        .param("client_id", DEVICE_CLIENT_ID)
                        .param("client_secret", "device-secret"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.device_code").exists())
                .andExpect(jsonPath("$.user_code").exists())
                .andExpect(jsonPath("$.verification_uri").exists());
    }
}
```

## 5. API 엔드포인트

| Method | Endpoint                          | 인증       | 설명                  |
|--------|-----------------------------------|----------|---------------------|
| GET    | `/.well-known/openid-configuration` | Public   | OIDC Discovery      |
| GET    | `/oauth2/jwks`                    | Public   | JSON Web Key Set    |
| GET    | `/oauth2/authorize`               | User     | Authorization Code  |
| POST   | `/oauth2/token`                   | Client   | Token Exchange      |
| POST   | `/oauth2/device_authorization`    | Public   | Device Flow 시작      |
| GET    | `/device`                         | User     | Device Code 인증 UI   |
| POST   | `/api/users`                      | ADMIN    | 사용자 등록              |
| GET    | `/api/users/me`                   | User     | 현재 사용자 조회           |
