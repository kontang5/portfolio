# Little Garden: 농장 재고·주문 관리 시스템

> 수기 장부와 감에 의존하던 농장 운영을 디지털화하는 백엔드 API

`Java 21` `Spring Boot 3.5` `Spring Security` `JWT` `PostgreSQL` `Flyway` `JPA` `JUnit 5`

## 1. 프로젝트 개요

### 배경

부모님께서 운영하시는 농가에서 재고 파악과 주문 관리를 수기 장부와 기억에 의존하고 계셨습니다. 어떤 작물이 얼마나 남았는지, 누구에게 얼마를 보내야 하는지 매번 확인하는 과정에서 누락과 혼선이 잦았고, 이를
시스템으로 해결하고자 직접 개발하게 되었습니다.

### 핵심 목표

- 실시간 재고 관리: 주문 생성 시 자동 재고 차감, 취소 시 복원
- 주문 상태 추적: PENDING → CONFIRMED → SHIPPED → COMPLETED Workflow
- 역할 기반 접근 제어: ADMIN / MANAGER / STAFF 권한 분리
- API 우선 설계: 프론트엔드 독립적인 HTTP API

코드베이스: [github.com/kontang5/littlegarden](https://github.com/kontang5/littlegarden)

### 현황

- HTTP API 엔드포인트 32개 (인증, 사용자, 상품, 주문, 역할 관리)
- API 문서: Swagger UI 제공 (`/swagger-ui`)
- 2025년 가을 고구마 수확 시즌, 프로토타입으로 주문 100여 건 처리
- 개선: 역할 기반 접근 제어, 주문 상태 머신 재설계 등

## 2. 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client (Web/Mobile)                     │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Spring Security Filter                     │
│              JWT authorize · authentication context, CORS       │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Controller Layer                        │
│         AuthController · UserController · OrderController       │
│              ProductController · RoleController                 │
│                            @PreAuthorize                        │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Service Layer                           │
│              비즈니스 로직 · 트랜잭션 관리 · 재고 연산                    │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Repository Layer                         │
│                 JPA Repository · Custom Queries                 │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                          PostgreSQL                             │
│                  Flyway 마이그레이션 · 감사 로그                      │
└─────────────────────────────────────────────────────────────────┘
```

## 3. 주요 구현 내용

### 3.1 JWT 기반 Stateless 인증

| 토큰 유형         | 만료 시간 | 용도              |
|---------------|-------|-----------------|
| Access Token  | 30분   | API 요청 인증       |
| Refresh Token | 6시간   | Access Token 갱신 |
| Setup Token   | 30분   | 신규 사용자 비밀번호 설정  |

```java
// JwtAuthenticationFilter.java
@Override
protected void doFilterInternal(HttpServletRequest request, ...) {
    String token = extractToken(request);
    if (token != null && jwtTokenProvider.validateToken(token)) {
        UserPrincipal principal = UserPrincipal.fromToken(
                jwtTokenProvider.getUserId(token),
                jwtTokenProvider.getUsername(token),
                jwtTokenProvider.getRole(token)
        );
        SecurityContextHolder.getContext().setAuthentication(
                new UsernamePasswordAuthenticationToken(principal, null, principal.getAuthorities())
        );
    }
    filterChain.doFilter(request, response);
}
```

> 서버 측 세션 저장 없이 토큰만으로 인증 상태를 유지

### 3.2 역할 기반 접근 제어 (RBAC)

| 역할      | 권한 범위                              |
|---------|------------------------------------|
| ADMIN   | 전체 시스템 관리, 사용자/역할 CRUD, 삭제된 데이터 복구 |
| MANAGER | 주문/상품 관리, 재고 조정, 삭제(soft delete)   |
| STAFF   | 주문 생성, 상품 조회, 본인 정보 수정             |

```java
// OrderController.java
@PostMapping
@PreAuthorize("hasAnyRole('ADMIN', 'MANAGER', 'STAFF')")
public ResponseEntity<OrderResponse> createOrder(...) {
}

@DeleteMapping("/{id}")
@PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")
public ResponseEntity<Void> deleteOrder(...) {
}
```

> URL 패턴 기반 보안과 메서드 레벨 `@PreAuthorize`를 병행하여 이중 방어

### 3.3 주문 상태 머신

```
PENDING ──────► CONFIRMED ──────► SHIPPED ──────► COMPLETED
    │               │                │
    │               │                │ (ADMIN/MANAGER + 사유 필수)
    ▼               ▼                ▼
CANCELED ◄─────────────────────────────
```

| 상태 전이              | 트리거 조건                 | 부가 동작                |
|--------------------|------------------------|----------------------|
| → CONFIRMED        | 재고 확인 완료               | -                    |
| → SHIPPED          | 직배송, 택배 (운송장 번호 필수)    | trackingNumber 검증    |
| → COMPLETED        | 배송 완료 확인               | completedAt 타임스탬프 기록 |
| → CANCELED         | 일반: PENDING/CONFIRMED만 | 재고 복원, canceledAt 기록 |
| SHIPPED → CANCELED | ADMIN/MANAGER + 사유 필수  | 재고 복원                |

```java
// OrderService.java - 상태 전이 검증
private static final Map<OrderStatus, Set<OrderStatus>> VALID_TRANSITIONS = Map.of(
                OrderStatus.PENDING, EnumSet.of(OrderStatus.CONFIRMED, OrderStatus.CANCELED),
                OrderStatus.CONFIRMED, EnumSet.of(OrderStatus.SHIPPED, OrderStatus.CANCELED),
                OrderStatus.SHIPPED, EnumSet.of(OrderStatus.COMPLETED),
                OrderStatus.COMPLETED, EnumSet.noneOf(OrderStatus.class),
                OrderStatus.CANCELED, EnumSet.noneOf(OrderStatus.class)
        );
```

### 3.4 원자적 재고 관리

```java
// ProductRepository.java
@Modifying
@Query("UPDATE Product p SET p.stock = p.stock - :quantity " +
        "WHERE p.id = :productId AND p.stock >= :quantity AND p.isActive = true")
int deductStock(@Param("productId") Long productId, @Param("quantity") int quantity);
```

| 시나리오  | 처리 방식                                          |
|-------|------------------------------------------------|
| 주문 생성 | `deductStock()` 단일 쿼리로 차감 (WHERE 조건으로 부족 시 실패) |
| 주문 취소 | `restoreStock()` 으로 재고 복원                      |
| 수량 변경 | 기존 수량과의 차이(delta) 계산 후 차감/복원                   |

> 단일 UPDATE 문으로 Race Condition 방지, 영향받은 row 수로 성공/실패 판단

### 3.5 Soft Delete와 감사 추적

```java
// SoftDeleteEntity.java
@MappedSuperclass
public abstract class SoftDeleteEntity extends BaseEntity {
    private boolean isActive = true;
    private Instant deletedAt;
    private Long deletedBy;
}

// BaseEntity.java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {
    @CreatedBy
    private Long createdBy;
    @CreatedDate
    private Instant createdAt;
    @LastModifiedBy
    private Long lastModifiedBy;
    @LastModifiedDate
    private Instant lastModifiedAt;
}
```

> 모든 엔티티에 생성/수정/삭제 이력이 자동 기록되어 문제 발생 시 추적 가능

### 3.6 일관된 예외 처리

```
ApplicationException (interface)
├── OrderException (abstract)
│   ├── OrderNotFoundException
│   ├── InsufficientStockException
│   ├── InvalidOrderStatusTransitionException
│   └── ...
├── ProductException (abstract)
├── UserException (abstract)
└── RoleException (abstract)
```

```java
// GlobalExceptionHandler.java
@ExceptionHandler(OrderNotFoundException.class)
public ProblemDetail handleOrderNotFound(OrderNotFoundException ex, WebRequest request) {
    log.warn("Order not found: {}", ex.getErrorMessage());  // 내부 로그용
    return problemDetailFactory.createProblemDetail(
            HttpStatus.NOT_FOUND,
            ex.getClientMessage(),  // 사용자 노출용: "주문을 찾을 수 없습니다"
            request
    );
}
```

| 구분            | 용도              | 예시                                 |
|---------------|-----------------|------------------------------------|
| errorMessage  | 내부 로깅/디버깅       | "Order ID 5 not found in database" |
| clientMessage | API 응답 (사용자 노출) | "주문을 찾을 수 없습니다"                    |

> RFC 7807 ProblemDetail 형식으로 일관된 에러 응답 제공

### 3.7 OpenAPI 명세 제공

```java
    @PostMapping("/login")
@Operation(
        summary = "User login",
        description = "Authenticate user with username and password. Returns access token (1 hour) and refresh token (7 days)."
)
@ApiResponses(value = {
        @ApiResponse(
                responseCode = "200",
                description = "Login successful",
                content = @Content(
                        mediaType = "application/json",
                        schema = @Schema(implementation = LoginResponse.class)
                )
        ),
        @ApiResponse(
                responseCode = "401",
                description = "Authentication failed - invalid credentials or user not active",
                content = @Content(
                        mediaType = "application/problem+json",
                        schema = @Schema(implementation = ProblemDetail.class)
                )
        )
})
```

## 4. 테스트 전략

### 테스트 피라미드

| 레이어         | 테스트 방식                | 주요 검증 항목               |
|-------------|-----------------------|------------------------|
| Unit        | Mockito + JUnit 5     | Validation, 비즈니스 로직    |
| Controller  | @WebMvcTest + MockMvc | 요청/응답 매핑, 권한 검증, 예외 처리 |
| Repository  | @DataJpaTest + H2     | 커스텀 쿼리, 필터링 로직         |
| Integration | @SpringBootTest       | 전체 플로우, 트랜잭션 롤백        |

### 테스트 커버리지 예시

```java
// OrderServiceTest.java
@Nested
@DisplayName("주문 상태 변경")
class UpdateOrderStatusTest {

    @Test
    @DisplayName("PENDING → CONFIRMED 성공")
    void updateStatus_PendingToConfirmed_Success() {
    }

    @Test
    @DisplayName("SHIPPED → CANCELED: STAFF는 불가")
    void updateStatus_ShippedToCanceled_StaffRole_Throws() {
    }

    @Test
    @DisplayName("COMPLETED → 어떤 상태로도 변경 불가")
    void updateStatus_CompletedToAny_Throws() {
    }
}
```

## 5. 설계 결정 기록

### 5.1 Spring Boot

- 비교 대상: Deno Fresh
- 선택 이유: 프레임워크가 아직 발전 중이고, 시간내에 서비스 제공을 위해서 익숙한 Spring Boot 선택

### 5.2 HTTP API

| 항목     | HATEOAS        | HTTP API (선택) |
|--------|----------------|---------------|
| 자기 서술성 | 응답에 액션 링크포함    | API문서 별도 제공   |
| 응답크기   | 링크 메타데이터로 장황해짐 | 필요한 데이터만 전달   |
| 클라이언트  | 링크 기반 동적 탐색    | UI가 흐름을 결정    |

> Swagger 문서와 기능이 중복되고, 실제로는 UI가 API 흐름을 결정하므로 HTTP API로 구현.

### 5.3 JWT Stateless 인증

| 항목      | JWT (선택)                | Session          |
|---------|-------------------------|------------------|
| API 적합성 | Stateless HTTP API와 일관됨 | 서버 상태 유지 필요      |
| 확장성     | 서버 간 상태 공유 불필요          | 세션 저장소(Redis) 필요 |
| 성능      | DB 조회 없이 토큰 검증          | 매 요청 세션 조회       |
| 보안      | 토큰 탈취 시 대응 어려움          | 서버에서 즉시 무효화 가능   |

> HTTP API 설계에 맞춰 Stateless 인증 방식 선택. 토큰 탈취 대응은 짧은 만료 시간으로 완화.

### 5.4 Soft Delete

| 항목     | Soft Delete (선택)         | Hard Delete |
|--------|--------------------------|-------------|
| 데이터 복구 | 가능                       | 불가능         |
| 감사 추적  | 삭제 이력 보존                 | -           |
| 쿼리 복잡도 | WHERE isActive = true 추가 | 단순          |
| 저장 공간  | 계속 증가                    | 정리됨         |

> 주문/상품 데이터는 법적 보관 의무가 있을 수 있고, 실수로 삭제 시 복구가 가능해야 하므로 Soft Delete 선택.

### 5.5 단일 UPDATE

| 항목         | 단일 UPDATE (선택)    | @Version 낙관적 락 | SELECT FOR UPDATE 비관적 락 |
|------------|-------------------|----------------|-------------------------|
| 구현 복잡도     | 낮음                | 중간 (재시도 로직 필요) | 중간 (트랜잭션 관리 필요)         |
| 동시성 처리     | WHERE 조건으로 원자적 처리 | 충돌 시 예외 발생     | 락 획득까지 대기               |
| 충돌 빈도 높을 때 | 효율적               | 재시도 오버헤드 증가    | 대기 시간 증가                |
| 확장성        | 단일 DB에서 충분        | 분산 환경에서 유리     | 락 경합으로 병목 가능            |
| 데드락 위험     | 없음                | 없음             | 있음                      |

> 현재 규모에서는 단일 UPDATE 쿼리로 충분. 향후 트래픽 증가 시 @Version 도입 고려.

## 6. 기술적 도전과 해결

### 6.1 OSIV(Open Session In View) 비활성화와 N+1 문제 해결

#### 문제

- Controller에서 LAZY 로딩된 연관 엔티티 접근 시 `LazyInitializationException` 발생
- 트랜잭션 범위가 불명확하여 예상치 못한 쿼리 발생
- 목록 조회 시 N+1 쿼리 문제 (Order 100건 조회 → Product 쿼리 100회)

#### 해결

```yaml
# application.yml
spring.jpa.open-in-view: false
```

- Service 레이어에서 필요한 데이터를 DTO로 변환 완료
- 인증 시 Fetch Join으로 User + Role 단일 쿼리 조회

```java
// UserRepository.java - 인증 시 Role을 함께 로딩
@Query("SELECT u FROM User u JOIN FETCH u.role WHERE u.username = :username")
Optional<User> findByUsernameWithRole(@Param("username") String username);
```

- 목록 조회 시 `@BatchSize`로 IN절 배치 로딩

```java
// Product.java, Role.java - 엔티티 클래스 레벨에 적용
@Entity
@BatchSize(size = 100)
public class Product extends SoftDeleteEntity {
}
```

| 시나리오             | 개선 전      | 개선 후             |
|------------------|-----------|------------------|
| 인증 (User + Role) | 2쿼리       | 1쿼리 (Fetch Join) |
| Order 100건 조회    | 1 + 100쿼리 | 1 + 1쿼리 (IN절 배치) |
| User 50건 조회      | 1 + 50쿼리  | 1 + 1쿼리 (IN절 배치) |

### 6.2 권한 검증 누락 방지

#### 문제

- 새로운 Controller 메서드 추가 시 `@PreAuthorize` 누락 가능성
- 코드 리뷰에서 발견하지 못하면 보안 취약점으로 이어짐

#### 해결

- SecurityConfig에서 기본적으로 `.anyRequest().authenticated()` 설정
- URL 패턴과 메서드 레벨 보안을 이중으로 적용
- 테스트 코드에서 권한별 접근 검증 필수화

### 6.3 테스트 데이터 격리

#### 문제

- 통합 테스트 간 데이터 간섭으로 간헐적 실패
- 테스트 순서에 따라 결과가 달라지는 문제

#### 해결

**application-test.yml - 테스트 전용 설정**

```yaml
spring:
  config.activate.on-profile: test
  datasource:
    url: jdbc:h2:mem:testdb;MODE=PostgreSQL  # 인메모리 DB
  flyway:
    enabled: false  # 마이그레이션 비활성화
  jpa:
    hibernate.ddl-auto: create-drop  # 테스트마다 스키마 재생성
```

**@BeforeEach로 테스트별 데이터 초기화**

```java

@SpringBootTest
@ActiveProfiles("test")
@Transactional  // 각 테스트 후 자동 롤백
class OrderIntegrationTest {

    @BeforeEach
    void setUp() {
        // 기존 데이터 정리
        orderRepository.deleteAll();
        productRepository.deleteAll();
        userRepository.deleteAll();
        roleRepository.deleteAll();

        // 테스트별 독립적인 데이터 seed
        adminRole = roleRepository.save(createRole("ROLE_ADMIN"));
        product = productRepository.save(createProduct("고구마", 100));
    }
}
```

| 설정                            | 역할                       |
|-------------------------------|--------------------------|
| `@ActiveProfiles("test")`     | H2 인메모리 DB, Flyway 비활성화  |
| `@Transactional`              | 테스트 종료 시 자동 롤백           |
| `@BeforeEach` + `deleteAll()` | 명시적 데이터 초기화로 테스트 간 격리 보장 |

## 7. 향후 계획

| 항목    | 현재      | 목표                             |
|-------|---------|--------------------------------|
| 캐싱    | 없음      | Redis로 상품 목록 캐싱                |
| 모니터링  | 로그만     | OpenTelemetry + OpenObserve 연동 |
| CI/CD | 수동 배포   | GitHub Actions + Docker        |
| 검색    | LIKE 쿼리 | PostgreSQL pg_trgm 전문 검색       |

이지훈 | kontang5@icloud.com | [github.com/kontang5](https://github.com/kontang5)
