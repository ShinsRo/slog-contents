# OAuth2 — 글 계획

## 시리즈 구성

### 1편: 신뢰를 위임한다는 것

역사와 구조, 동작 흐름을 다룬다. 독자적으로도 읽힐 수 있어야 하지만,
글 말미에 "이 흐름에서 각 엔티티가 실제로 무엇을 어떻게 다뤄야 하는가"를 2편에서 본격적으로 다룬다는 명분을 만들어야 한다.

**다룰 내용:**
- 역사 — OAuth 1.0 → 2.0 전환이 아니라, 인증/인가 표준의 변천사 전체
  - Kerberos / SAML → OAuth2 + OIDC → FIDO2 / Passkeys 흐름
  - 각 시대마다 어떤 문제를 풀려 했는가
  - OAuth2가 그 흐름 어디쯤에 있는가
- 핵심 엔티티 4개
- 증표 개념 (흐름에서 발생하는 데이터의 종류)
- Grant Types 동작 흐름 (Authorization Code + PKCE 중심, 나머지는 간략히)
- OIDC — "소셜 로그인은 사실 OIDC"

### 2편 — 각 엔티티의 관리 전략 (본론)

**핵심 주제:**
> OAuth2 흐름에서 발생하는 데이터를 각 엔티티가 어떻게 관리해야 하는가

- Client의 토큰 저장 전략
- Authorization Server의 발급/추적/폐기
- Resource Server의 검증과 캐싱
- 솔루션 (Keycloak, Zitadel 등) — 직접 구현 vs 위임

---

## 핵심 개념 정리

### 핵심 엔티티

RFC 6749는 "Role"이라는 표현을 쓰지만, Resource Owner가 사람일 수도 시스템일 수도 있다는 점에서 "엔티티"가 더 정확한 표현이다.

- **Resource Owner** — 자원에 대한 접근 권한을 부여할 수 있는 엔티티
  - 사람: 소셜 로그인에서 사용자가 앱에게 권한 위임
  - 시스템: M2M / MSA에서 서비스 자체가 주체 (Client Credentials)
- **Client** — 자원에 접근하려는 애플리케이션
  - Confidential Client: 서버 사이드, 시크릿 보관 가능
  - Public Client: SPA / 모바일, 시크릿 보관 불가 → PKCE 필요
- **Authorization Server** — 동의 받고 토큰 발급 (Google, Kakao 로그인 서버)
- **Resource Server** — 보호된 자원 보유, Access Token 검증 후 응답

### 증표 (흐름에서 발생하는 데이터)

OAuth2에서 발생하는 데이터는 "신뢰의 증표"다. Authorization Server가 신뢰의 원천이고, 발급한 것들이 엔티티 사이를 흘러다닌다.

수명이 짧을수록 노출 피해가 작고, 수명이 길수록 저장 전략이 중요해진다.

| 데이터 | 수명 | 노출 시 피해 범위 |
|---|---|---|
| client_secret | 영구 | 해당 앱의 모든 사용자 |
| AS 서명 키 (private) | 수개월 | 모든 토큰 위조 가능 |
| refresh_token | 수일~수주 | 장기 접근 탈취 |
| access_token | ~1시간 | 해당 토큰 scope 범위 |
| authorization_code | 10분, 1회용 | 즉시 토큰 교환 가능 |

### Grant Types

| Grant Type | 용도 | 비고 |
|---|---|---|
| Authorization Code | 서버 사이드 앱, 소셜 로그인 | 현재 권장 방식 |
| Authorization Code + PKCE | SPA, 모바일 | Public Client 필수 |
| Client Credentials | 서버 간 통신 (사용자 없음) | M2M |
| Implicit | (구) SPA | Deprecated |
| Resource Owner Password | (구) 직접 자격증명 | Deprecated |

### OIDC (OpenID Connect)

OAuth2가 "이 앱이 내 자원에 접근해도 되는가"를 다룬다면,
OIDC는 "이 사람이 누구인가"를 OAuth2 위에 얹은 것.

- **ID Token** (JWT) — 사용자 신원 정보 포함 (sub, email, name 등)
- **UserInfo 엔드포인트** — 추가 사용자 정보 조회
- **`openid` scope** — OIDC 흐름을 트리거하는 스코프
- "소셜 로그인 = OAuth2" 라고 알고 있지만 실제로는 OIDC
- OAuth2만으로는 "누구인지"를 알 수 없음

---

## 2편 세부 내용 (미결)

### 엔티티별 관리 전략

**Client**
- access_token: 메모리 저장 권장 (XSS 방어), httpOnly 쿠키 차선
- refresh_token: 안전한 저장소 필요, 서버 사이드 보관 권장
- client_secret: 환경변수 / Secret Manager, 절대 코드에 포함 금지
- PKCE `code_verifier`: 요청 단위 임시 저장 후 즉시 폐기
- `state` 파라미터: CSRF 방지용, 세션에 저장 후 검증

**Authorization Server**
- authorization_code: 단기(10분), 1회용, 재사용 시 관련 토큰 전체 폐기
- access_token 형식: JWT(자체 검증) vs Opaque(서버 조회 필요) 선택
- refresh_token Rotation: 사용 시 새 토큰 발급 + 이전 토큰 무효화
- 토큰 폐기 엔드포인트 (RFC 7009)
- JWKS 엔드포인트: RS가 서명 검증에 쓰는 공개키 제공

**Resource Server**
- JWT: JWKS로 서명 검증, 만료 / scope 확인
- Opaque: Authorization Server에 Introspection 요청 (RFC 7662)
- 검증 결과 캐싱 전략 (매 요청마다 JWKS 조회는 비효율)

### 솔루션

#### 셀프 호스팅

| 솔루션 | 언어 | 특징 |
|---|---|---|
| Keycloak | Java | Red Hat. 가장 많이 쓰이는 오픈소스 IAM |
| Zitadel | Go | 클라우드 네이티브, 최근 주목 |
| Ory Hydra | Go | 경량 OAuth2/OIDC 서버, 헤드리스 |
| Authentik | Python | 관리 UI 세련됨, 설치 쉬움 |
| Dex | Go | OIDC 브릿지, 외부 IdP 연결 특화 |
| Spring Authorization Server | Java | Spring 공식 OAuth2 AS |

#### SaaS

| 솔루션 | 특징 |
|---|---|
| Auth0 (Okta) | 가장 많이 쓰이는 SaaS IAM |
| AWS Cognito | AWS 생태계 통합 |
| Clerk | 개발자 경험 중심, 최근 인기 |
| SuperTokens | 오픈소스 + SaaS 하이브리드 |

---

## 메모 / 미결 사항

- 비교 대상: OAuth 1.0 vs 2.0, OAuth2 vs SAML, JWT와의 관계 → 1편에 넣을지 2편에 넣을지
- Refresh Token 탈취 시나리오와 Rotation 전략 깊이
- PKCE code_verifier / code_challenge 생성 과정까지 다룰지
- 구현 예시 언어: Spring Kotlin 위주
