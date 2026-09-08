# 🔒 Module 9: Authentication, Authorization & Security Architecture

> **Core Philosophy:** *Security is not a feature added at the end; it is an architectural foundation. A modern distributed system enforces the Zero-Trust model: authenticate every hop, authorize every action, protect data at rest and in transit, and assume the internal network is compromised.*

---

## 📌 Table of Contents
1. [The Problem: Security Holes in Distributed Systems](#1-the-problem-security-holes-in-distributed-systems)
2. [Authentication vs Authorization](#2-authentication-vs-authorization)
3. [JSON Web Tokens (JWT) Deep Dive: Cryptographic Signing & Validation](#3-json-web-tokens-jwt-deep-dive-cryptographic-signing--validation)
4. [Access Tokens vs Refresh Tokens: The Secure Lifecycle](#4-access-tokens-vs-refresh-tokens-the-secure-lifecycle)
5. [OAuth 2.0 & OpenID Connect (OIDC) Protocols](#5-oauth-20--openid-connect-oidc-protocols)
6. [Role-Based (RBAC) vs Attribute-Based (ABAC) Access Control](#6-role-based-rbac-vs-attribute-based-abac-access-control)
7. [Distributed Rate Limiting: Algorithms & Atomic Redis Lua Scripts](#7-distributed-rate-limiting-algorithms--atomic-redis-lua-scripts)
8. [Zero Trust Architecture: Mutual TLS (mTLS) & Secrets Management](#8-zero-trust-architecture-mutual-tls-mtls--secrets-management)
9. [Java / Spring Boot Security Filter & Rate Limiter Implementation](#9-java--spring-boot-security-filter--rate-limiter-implementation)
10. [Side-by-Side Security Architecture Trade-off Table](#10-side-by-side-security-architecture-trade-off-table)
11. [Interview Rapid Q&A Checklist](#11-interview-rapid-qa-checklist)

---

## 1. The Problem: Security Holes in Distributed Systems

- **The Perimeter Trap (Castle-and-Moat):** Traditional architectures assume that once traffic passes the outer firewall, internal services can trust each other without encryption or auth. If an attacker breaches one container, they gain access to the entire company database!
- **Stateless Verification Dilemma:** How can 50 microservices authenticate 100,000 requests/second without hammering the central Auth Database with 100,000 DB queries/sec?
- **Solution:** Cryptographically signed **JSON Web Tokens (JWT)** verified locally in CPU memory without network calls.

---

## 2. Authentication vs Authorization

- **Authentication (AuthN):** *"Who are you?"* (Verifying identity via password, biometric, or MFA).
- **Authorization (AuthZ):** *"What are you allowed to do?"* (Verifying permissions, e.g. "Can User 42 delete Order 999?").

---

## 3. JSON Web Tokens (JWT) Deep Dive

```
A JWT is 3 base64url-encoded strings separated by periods:
[ Header ] . [ Payload ] . [ Cryptographic Signature ]

1. Header:
   {"alg": "RS256", "typ": "JWT"}

2. Payload (Claims):
   {"sub": "10492", "name": "Asha", "role": "EDITOR", "exp": 1715000000}

3. Signature:
   RSASHA256(base64Url(Header) + "." + base64Url(Payload), privateKey)
```
- **How Verification Works:**
  - The Auth Server signs the JWT using its **Private Key**.
  - All internal microservices download the Auth Server's **Public Key** (JWKS) once at startup.
  - Any microservice can verify the signature using the public key in **< 0.1ms of pure CPU time**, with zero database queries!

---

## 4. Access Tokens vs Refresh Tokens: The Secure Lifecycle

```
[ Client App ] ── 1. POST /login (Credentials) ────────────────▶ [ Auth Server ]
[ Client App ] ◀─ 2. Access Token (15 min) + Refresh Token (7d) ── [ Auth Server ]

[ Client App ] ── 3. GET /orders (Bearer AccessToken) ─────────▶ [ Order Service ]
[ Client App ] ◀─ 4. HTTP 401 Unauthorized (Token Expired) ──── [ Order Service ]

[ Client App ] ── 5. POST /refresh (RefreshToken in HttpOnly) ─▶ [ Auth Server ]
[ Client App ] ◀─ 6. New Access Token (15 min) ──────────────── [ Auth Server ]
```
- **Why short-lived access tokens?** If an access token is intercepted over public Wi-Fi, it becomes useless in minutes.
- **Why refresh tokens in HttpOnly cookies?** Prevents malicious JavaScript (XSS attacks) from reading the token.

---

## 5. Distributed Rate Limiting: Algorithms & Atomic Lua Scripts

### Token Bucket Algorithm (Industry Standard - Stripe/AWS):
- A bucket holds up to $B$ tokens and refills at rate $R$ tokens/second.
- Each incoming request takes 1 token. If the bucket is empty, return **HTTP 429 Too Many Requests**.
- Allows bursts up to bucket capacity $B$, but limits average throughput to rate $R$.

### Atomic Redis Lua Script (Prevents Race Conditions):
```lua
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local current = tonumber(redis.call(get, key) or "0")

if current + 1 > limit then
    return 0 -- Rate limit exceeded!
else
    redis.call("INCRBY", key, 1)
    if current == 0 then
        redis.call("EXPIRE", key, ARGV[2])
    end
    return 1 -- Allowed
end
```

---

## 6. Java / Spring Boot Security Filter & Rate Limiter

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenValidator tokenValidator;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {

        String authHeader = request.getHeader("Authorization");

        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            String token = authHeader.substring(7);
            try {
                Claims claims = tokenValidator.validateAndExtractClaims(token);

                String userId = claims.getSubject();
                String role = claims.get("role", String.class);

                UsernamePasswordAuthenticationToken authentication =
                    new UsernamePasswordAuthenticationToken(
                        userId,
                        null,
                        List.of(new SimpleGrantedAuthority("ROLE_" + role))
                    );

                SecurityContextHolder.getContext().setAuthentication(authentication);

            } catch (JwtException ex) {
                log.warn("Invalid JWT token: {}", ex.getMessage());
                response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                response.getWriter().write("{"error": "Unauthorized token"}");
                return;
            }
        }

        filterChain.doFilter(request, response);
    }
}
```

---

## 7. Side-by-Side Security Architecture Trade-off Table

| Token Strategy | Latency | Revocation Speed | Storage Overhead |
| :--- | :--- | :--- | :--- |
| **Stateful Sessions (Redis)**| Network call to Redis (1-2ms) | Immediate (Delete key) | High memory in Redis cluster |
| **Stateless JWT** | In-memory CPU verify (<0.1ms) | Delayed (until TTL expires) | Zero backend memory |
| **Hybrid JWT with Blocklist**| In-memory CPU + Redis check on revoke | Immediate | Minimal (only store revoked tokens) |

---

## 8. Interview Rapid Q&A Checklist
- *How do you revoke a stateless JWT immediately if a phone is stolen?* (Store the revoked `token_id` (JTI) in a Redis **Blocklist** with a TTL equal to the remaining expiration time of that specific token).
- *What is mTLS (Mutual TLS)?* (Both the client and the server present X.509 cryptographic certificates to verify each other's identity before establishing a connection; standard for microservice-to-microservice mesh networks).
