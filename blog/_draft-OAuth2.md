# OAuth2 — 글 계획

## 다룰 내용

- 역사 — OAuth 1.0 → 2.0 전환이 아니라, 인증/인가 표준의 변천사 전체
  - Kerberos / SAML → OAuth2 + OIDC → FIDO2 / Passkeys 흐름
  - 각 시대마다 어떤 문제를 풀려 했는가
  - OAuth2가 그 흐름 어디쯤에 있는가
- 핵심 엔티티 4개
- 증표 개념 (흐름에서 발생하는 데이터의 종류)
- Grant Types 동작 흐름 (Authorization Code + PKCE 중심, 나머지는 간략히)
- OIDC — "소셜 로그인은 사실 OIDC"

---

## 역사

OAuth2 이전

프로토콜	연도	특징
HTTP Basic / Digest Auth	1990s	username:password 전달. 가장 단순
Kerberos	1988	티켓 기반. Windows AD / 기업 내부망
SAML 1.0 / 2.0	2002/2005	XML 기반 SSO. 기업 환경에서 여전히 많이 쓰임
OpenID 1.0 / 2.0	2005/2007	분산 신원 인증. OIDC의 전신. 지금은 거의 사용 안 함
OAuth 1.0 / 1.0a	2007/2009	서명 기반. 복잡해서 OAuth2로 대체
OAuth2 이후 / 확장

프로토콜 / 표준	연도	특징
OIDC	2014	OAuth2 위에 인증 레이어 추가
PKCE	2015	Public Client 보안 확장
FIDO2 / WebAuthn	2018	비밀번호 없는 인증. 생체인식, 하드웨어 키
Passkeys	2022~	FIDO2 기반. Apple / Google / MS 공동 추진
DPoP	2023	토큰을 키 쌍에 바인딩. 탈취 방지 강화
GNAP	진행 중	"OAuth3"로 불림. OAuth2 한계 해결 시도
OAuth 2.1	초안	OAuth2 모범 사례 통합 (Implicit, ROPC 공식 제거)
목적이 다른 인접 표준들

SCIM — 사용자 계정 프로비저닝 (인증이 아니라 계정 동기화)
SPIFFE / SPIRE — 쿠버네티스 등 워크로드 신원 (서비스 간 mTLS)
Verifiable Credentials / DID — W3C 탈중앙화 신원. 아직 초기 단계
흐름으로 보면 SAML → OAuth2 + OIDC → Passkeys / FIDO2 방향, 서버 간 신원은 mTLS / SPIFFE 쪽으로 가는 추세

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

## 메모 / 미결 사항

- 비교 대상: OAuth 1.0 vs 2.0, OAuth2 vs SAML, JWT와의 관계 → 어디에 넣을지
- PKCE code_verifier / code_challenge 생성 과정까지 다룰지
