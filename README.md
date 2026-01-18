# common-security

커머스 플랫폼 마이크로서비스를 위한 JWT 기반 인증 및 보안 설정 라이브러리

## 개요

`common-security`는 커머스 플랫폼의 JWT 검증 및 Spring Security 설정을 제공합니다.  
이 모듈은 **토큰 검증**에만 집중하며, 토큰 발급은 `auth-service`에서 처리합니다.  
각 서비스는 `common-security`를 import하여 자체적으로 jwt 검증 로직을 수행합니다.  
패키지는 Github Packages로 배포되며, semantic versioning을 따릅니다.

## 주요 기능

- **JWT 토큰 검증**: JWT 토큰을 검증하고 인증 정보를 추출
- **Spring Security 자동 설정**: 서블릿 기반 애플리케이션의 보안을 자동으로 구성
- **Public 엔드포인트 등록**: 인증 없이 접근 가능한 엔드포인트를 정의하는 인터페이스 기반 메커니즘
- **@AuthId 어노테이션**: 컨트롤러 메서드 파라미터에 인증된 사용자 ID를 직접 주입
- **Stateless 세션 관리**

### 필수 설정

이 모듈을 사용하는 서비스는 JWT secret을 설정해야 합니다 (auth-service와 동일해야 함)

```yaml
jwt:
  secret: ${JWT_SECRET_KEY}
  header: Authorization # 기본값
  prefix: "Bearer " # 기본값
```

## 제공 기능

### 1. 기본 인증

`jwt.secret`이 설정되면 자동 설정이 활성화됩니다. 추가 설정이 필요 없습니다.

```kotlin
@RestController
@RequestMapping("/api/orders")
class OrderController(
    private val orderService: OrderService
) {
    @GetMapping("/{id}")
    fun getOrder(@PathVariable id: String): Order {
        // 유효한 JWT 토큰이 필요
        return orderService.getOrder(id)
    }
}
```

### 2. 인증된 사용자 ID 추출

`@AuthId` 어노테이션을 사용하여 인증된 사용자의 ID를 주입받을 수 있습니다:

```kotlin
@RestController
@RequestMapping("/api/orders")
class OrderController(
    private val orderService: OrderService
) {
    @GetMapping("/my-orders")
    fun getMyOrders(@AuthId userId: Long?): List<Order> {
        userId ?: throw UnauthorizedException("인증이 필요합니다")
        return orderService.getOrdersByUserId(userId)
    }
}
```

### 3. Public 엔드포인트 등록

`PublicEndpointProvider`를 구현하여 인증을 우회하는 엔드포인트를 정의합니다:

```kotlin
@Component
class OrderServicePublicEndpointProvider : PublicEndpointProvider {
    override fun publicEndpoints(): Array<String> = arrayOf(
        "/api/orders/public/**",
        "/actuator/health/**"
    )
}
```

**기본 Public 경로** (항상 허용됨):

- `/actuator/**`
- `/health`
- `/swagger-ui/**`
- `/v3/api-docs/**`

## 아키텍처

### 컴포넌트 구조

```
common-security/
├── jwt/
│   ├── JwtProperties                    # JWT 설정 프로퍼티
│   ├── JwtTokenProvider                 # 토큰 검증 및 파싱
│   ├── JwtAuthenticationFilter          # 토큰 추출을 위한 서블릿 필터
│   ├── JwtAuthenticationEntryPoint      # 인증 실패 에러 핸들러
│   └── JwtUserPrincipal                 # 사용자 인증 데이터
├── config/
│   ├── SecurityConfig                   # Spring Security 설정
│   ├── PublicEndpointProvider           # Public 엔드포인트 인터페이스
│   ├── JwtSecurityAutoConfiguration     # JWT 빈 자동 설정
│   └── WebMvcAutoConfiguration          # @AuthId 리졸버 자동 등록
└── resolver/
    ├── @AuthId                           # 사용자 ID 주입 어노테이션
    └── AuthIdArgumentResolver            # SecurityContext에서 userId 추출
```

### 인증 흐름

```
1. 클라이언트 요청
   ↓
2. JwtAuthenticationFilter
   - Authorization 헤더에서 Bearer 토큰 추출
   - JwtTokenProvider로 토큰 검증
   ↓
3. SecurityContextHolder
   - JwtUserPrincipal(userId, roles) 저장
   ↓
4. 컨트롤러 메서드
   - @AuthId 어노테이션으로 userId 추출
   - 비즈니스 로직 실행
   ↓
5. 응답
```

### 토큰 검증 상태

`JwtTokenProvider`는 세 가지 검증 결과 중 하나를 반환합니다:

| 결과      | 설명                                  | HTTP 상태        |
| --------- | ------------------------------------- | ---------------- |
| `Valid`   | 토큰이 유효하고 만료되지 않음         | 200 OK           |
| `Expired` | 토큰 서명은 유효하지만 만료됨         | 401 Unauthorized |
| `Invalid` | 토큰 서명이 유효하지 않거나 형식 오류 | 401 Unauthorized |

## 자동 설정

다음 조건에서 자동 설정이 활성화됩니다:

1. **JwtSecurityAutoConfiguration**: `jwt.secret` 프로퍼티가 설정됨
2. **WebMvcAutoConfiguration**: 애플리케이션이 SERVLET 타입

등록 경로:

```
src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

### 제공되는 의존성 (`api` 스코프)

이 모듈을 사용하는 서비스는 자동으로 다음을 포함합니다.

- `spring-boot-starter-security`
- `spring-boot-starter-web`
- `jjwt-api` (0.12.5)

별도로 선언할 필요가 없습니다.

### 자동 설정 제외

커스텀 보안 설정이 필요한 경우, 자동 설정에서 제외할 수 있습니다.

```kotlin
@SpringBootApplication(exclude = [JwtSecurityAutoConfiguration::class])
class MyApplication
```

그런 다음 자체 `SecurityConfig`를 제공하세요.

## 기술 스택

- **Kotlin**: 1.9.25
- **Java**: 21
- **Spring Boot**: 3.5.8
- **Spring Security**: (Spring Boot 스타터를 통해 제공)
- **JJWT**: 0.12.5

## 관련 모듈

- **common-core**: 공통 응답 모델, 예외 처리, 이벤트 스키마
- **common-observability**: 로깅 및 MDC 관리
- **auth-service**: JWT 토큰 발급 (검증을 위해 이 모듈 사용)
