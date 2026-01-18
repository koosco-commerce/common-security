# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Module Purpose

The `common-security` module provides JWT validation and security configuration for all microservices in the commerce platform. Token issuance is handled by `auth-service`; this module is validation-only.

Services that depend on this module automatically receive:
- JWT token validation via servlet filter
- Spring Security configuration with stateless sessions
- Public endpoint registration mechanism
- `@AuthId` annotation for extracting authenticated user ID

## Key Components

### JWT Package (`com.koosco.commonsecurity.jwt`)

| Class | Purpose |
|-------|---------|
| `JwtProperties` | Configuration properties bound to `jwt.*` prefix. Requires `jwt.secret` to match auth-service. |
| `JwtTokenProvider` | Validates tokens and extracts `Authentication` objects. Returns `TokenValidationResult` (Valid/Expired/Invalid). |
| `JwtAuthenticationFilter` | Servlet filter that extracts Bearer token from Authorization header and sets `SecurityContextHolder`. |
| `JwtAuthenticationEntryPoint` | Returns JSON error response for unauthorized requests. |
| `JwtUserPrincipal` | Data class holding `userId` and `roles` extracted from token claims. |

### Security Configuration (`com.koosco.commonsecurity.config`)

| Class | Purpose |
|-------|---------|
| `SecurityConfig` | Main Spring Security configuration. Disables CSRF, sets stateless sessions, wires JWT filter. |
| `PublicEndpointProvider` | Interface for services to register endpoints that bypass authentication. |
| `JwtSecurityAutoConfiguration` | Auto-configures JWT beans when `jwt.secret` property is present. |
| `WebMvcAutoConfiguration` | Auto-registers `AuthIdArgumentResolver` for `@AuthId` annotation support. |

### Resolver (`com.koosco.commonsecurity.resolver`)

| Class | Purpose |
|-------|---------|
| `@AuthId` | Annotation for controller method parameters to inject authenticated user ID. |
| `AuthIdArgumentResolver` | Extracts `userId` from `SecurityContextHolder` and converts to `Long`. Returns `null` if unauthenticated. |

## PublicEndpointProvider Pattern

Services register public (unauthenticated) endpoints by implementing `PublicEndpointProvider`:

```kotlin
@Component
class MyServicePublicEndpointProvider : PublicEndpointProvider {
    override fun publicEndpoints(): Array<String> = arrayOf(
        "/api/myservice/public/**",
        "/actuator/health/**",
    )
}
```

The `SecurityConfig` collects all `PublicEndpointProvider` beans and permits those paths.

**Default public paths** (always permitted):
- `/actuator/**`
- `/health`
- `/swagger-ui/**`
- `/v3/api-docs/**`

## @AuthId Annotation Usage

Use `@AuthId` in controller methods to get the authenticated user's ID:

```kotlin
@GetMapping("/profile")
fun getProfile(@AuthId userId: Long?): ResponseEntity<UserProfile> {
    // userId is null if request is unauthenticated
    userId ?: throw UnauthorizedException()
    return ResponseEntity.ok(userService.getProfile(userId))
}
```

The resolver extracts `userId` from `JwtUserPrincipal` in `SecurityContextHolder` and converts it to `Long`.

## Auto-Configuration Behavior

Auto-configuration activates when:
1. `jwt.secret` property is set (triggers `JwtSecurityAutoConfiguration`)
2. Web application type is SERVLET (triggers `WebMvcAutoConfiguration`)

Registered in:
```
src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

## Required Configuration

Services using this module must set the JWT secret (must match auth-service):

```yaml
jwt:
  secret: ${JWT_SECRET_KEY}
  header: Authorization      # default
  prefix: "Bearer "          # default
```

## Dependencies

This module provides (via `api` scope):
- `spring-boot-starter-security`
- `spring-boot-starter-web`
- `jjwt-api` (0.12.5)

Services do not need to declare these dependencies separately.

## GitHub Packages

Published to GitHub Packages. Consuming services need authentication:

```properties
# ~/.gradle/gradle.properties
gpr.user=your-github-username
gpr.token=your-github-token
```

Or via environment variables: `GH_ACTOR`, `GH_TOKEN`
