# CORS Study Guide Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 백엔드 면접 준비용 CORS 학습 문서를 작성하고 저장소의 웹 주제 목차에서 접근할 수 있게 한다.

**Architecture:** 하나의 `web/CORS.md`가 개념, HTTP 흐름, Spring 설정, 디버깅, 면접 질문을 순서대로 설명한다. `README.md`는 기존 웹 주제 목록의 CORS 항목만 문서 링크로 바꾸며, 다른 파일이나 학습 주제는 변경하지 않는다.

**Tech Stack:** Markdown, HTTP/CORS, Fetch Standard, Java, Spring MVC, Spring Security

## Global Constraints

- 발표 시간은 10~20분을 기준으로 한다.
- 설명 언어는 한국어다.
- 프레임워크에 독립적인 HTTP 메시지와 Java/Spring 설정을 함께 사용한다.
- 예시는 `https://frontend.example`에서 `https://api.example`로 요청하는 상황으로 통일한다.
- Fetch 표준, MDN, Spring 공식 문서를 우선 근거로 사용한다.
- Private Network Access, WebSocket, WebTransport, Canvas tainting, 프록시 제품별 설정은 다루지 않는다.
- 사용자가 이미 수정한 `database/트랜잭션_격리_수준.md`, `history/question_history.md`, `network/HTTP_&_HTTPS.md`는 변경하거나 커밋하지 않는다.

---

### Task 1: CORS 학습 문서 작성

**Files:**
- Create: `web/CORS.md`

**Interfaces:**
- Consumes: `docs/superpowers/specs/2026-07-23-cors-study-design.md`의 범위와 설명 원칙
- Produces: README에서 연결할 완성된 CORS 스터디 문서

- [ ] **Step 1: 공식 자료에서 핵심 사실을 검증한다**

다음 자료를 확인하고 각 주장과 예시를 대조한다.

- Fetch Standard CORS protocol: `https://fetch.spec.whatwg.org/#http-cors-protocol`
- MDN CORS guide: `https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS`
- MDN Same-origin policy: `https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy`
- Spring Framework CORS reference: `https://docs.spring.io/spring-framework/reference/web/webmvc-cors.html`
- Spring Security CORS reference: `https://docs.spring.io/spring-security/reference/servlet/integrations/cors.html`

검증할 항목은 Origin의 구성, CORS-safelisted request 조건, Preflight의 `OPTIONS` 요청, 주요 CORS 헤더, 자격 증명 요청과 와일드카드 제약, Preflight 캐시, Spring MVC·Security 연결 방식이다.

- [ ] **Step 2: 문서 골격과 핵심 설명을 작성한다**

`web/CORS.md`에 다음 순서로 내용을 작성한다.

1. `# CORS (Cross-Origin Resource Sharing)`
2. 학습 목표
3. 먼저 한 문장으로 정리
4. Origin과 Same-Origin Policy
5. CORS가 필요한 이유와 적용 주체
6. CORS 요청의 전체 흐름
7. 단순 요청과 Preflight 요청
8. 주요 CORS 헤더
9. 자격 증명을 포함한 요청
10. Spring MVC와 Spring Security 설정
11. 흔한 오류와 디버깅 순서
12. CORS 보안 오해와 CSRF와의 차이
13. 30초·1분 면접 답변
14. 예상 질문과 꼬리 질문
15. 핵심 체크리스트와 공식 참고 자료

Origin 비교 표에는 스킴, 호스트, 포트 세 요소를 포함한다. `https://service.example:443`을 기준으로 스킴·호스트·포트가 달라지는 예시와 경로만 달라지는 동일 출처 예시를 넣는다.

- [ ] **Step 3: 단순 요청 HTTP 예시를 작성한다**

다음 요청과 응답을 사용하고, 브라우저가 응답의 `Access-Control-Allow-Origin`을 검사한 뒤 JavaScript에 응답 공개 여부를 결정한다고 설명한다.

```http
GET /members/1 HTTP/1.1
Host: api.example
Origin: https://frontend.example
```

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://frontend.example
Vary: Origin
Content-Type: application/json

{"id":1,"name":"June"}
```

CORS-safelisted request의 메서드는 `GET`, `HEAD`, `POST`로 정리하고, safelisted request header 및 허용되는 `Content-Type` 세 가지(`application/x-www-form-urlencoded`, `multipart/form-data`, `text/plain`)를 설명한다. `application/json` 또는 `Authorization` 헤더가 있으면 일반적으로 Preflight 대상이라는 예시를 넣는다.

- [ ] **Step 4: Preflight HTTP 예시를 작성한다**

다음 요청·응답·실제 요청의 세 단계를 순서대로 설명한다.

```http
OPTIONS /members/1 HTTP/1.1
Host: api.example
Origin: https://frontend.example
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: content-type, authorization
```

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://frontend.example
Access-Control-Allow-Methods: GET, PUT
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 600
Vary: Origin
```

```http
PUT /members/1 HTTP/1.1
Host: api.example
Origin: https://frontend.example
Content-Type: application/json
Authorization: Bearer token

{"name":"June"}
```

Preflight 성공 후에만 브라우저가 실제 요청을 보내며, `Access-Control-Max-Age`는 Preflight 결과를 캐시하는 시간임을 적는다.

- [ ] **Step 5: 주요 헤더와 Credentialed Request를 정리한다**

요청 헤더 `Origin`, `Access-Control-Request-Method`, `Access-Control-Request-Headers`와 응답 헤더 `Access-Control-Allow-Origin`, `Access-Control-Allow-Methods`, `Access-Control-Allow-Headers`, `Access-Control-Allow-Credentials`, `Access-Control-Expose-Headers`, `Access-Control-Max-Age`를 표로 정리한다.

`fetch` 예시는 다음과 같이 작성한다.

```javascript
fetch("https://api.example/members/me", {
  credentials: "include"
});
```

응답에는 구체적인 Origin과 `Access-Control-Allow-Credentials: true`가 필요하며, `Access-Control-Allow-Origin: *`를 사용할 수 없다고 설명한다. CORS 허용과 별개로 쿠키의 `SameSite`, `Secure`, 서드파티 쿠키 정책이 적용된다는 점도 명시한다.

- [ ] **Step 6: Spring 설정 예시를 작성한다**

Spring MVC 전역 설정은 다음 코드와 각 옵션 설명을 포함한다.

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("https://frontend.example")
                .allowedMethods("GET", "POST", "PUT", "DELETE")
                .allowedHeaders("Content-Type", "Authorization")
                .allowCredentials(true)
                .maxAge(600);
    }
}
```

컨트롤러 단위 설정으로 `@CrossOrigin(origins = "https://frontend.example")`을 소개한다. Spring Security 사용 시 MVC 설정을 재사용하는 다음 구성을 포함한다.

```java
@Bean
SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
            .cors(Customizer.withDefaults())
            .authorizeHttpRequests(authorize -> authorize
                    .anyRequest().authenticated()
            );
    return http.build();
}
```

Preflight 요청에는 보통 인증 쿠키가 없으므로 CORS 처리가 Spring Security 인증보다 먼저 수행되어야 한다는 이유를 설명한다.

- [ ] **Step 7: 디버깅·보안·면접 섹션을 작성한다**

디버깅 순서는 다음 다섯 단계로 고정한다.

1. 요청 URL의 스킴·호스트·포트를 비교한다.
2. 브라우저 Network 탭에서 `OPTIONS` 요청 유무와 상태를 확인한다.
3. 요청의 `Origin`, `Access-Control-Request-*` 헤더를 확인한다.
4. Preflight와 실제 응답 양쪽의 `Access-Control-Allow-*` 헤더를 확인한다.
5. Spring Security, 프록시, 예외 응답에서도 CORS 헤더가 유지되는지 확인한다.

보안 섹션에는 다음 사실을 명시한다.

- CORS는 브라우저가 응답을 JavaScript에 공개할지를 통제하며 서버 인증·인가를 대체하지 않는다.
- 단순 요청은 서버까지 전송될 수 있으므로 CORS만으로 상태 변경 요청이나 CSRF를 막을 수 없다.
- 요청의 Origin을 검증 없이 그대로 허용하면 신뢰 경계가 사라진다.
- CORS 오류를 없애기 위해 모든 Origin·메서드·헤더를 무조건 허용하면 안 된다.

30초 답변은 정의, SOP와의 관계, 서버 헤더와 브라우저 검사의 세 문장으로 구성한다. 1분 답변은 여기에 단순 요청·Preflight·자격 증명 제약·비브라우저 클라이언트 차이를 추가한다. 예상 질문에는 최소한 CORS 적용 주체, Postman에서 성공하는 이유, Preflight 발생 조건, `*`와 쿠키, CORS와 CSRF, 서버가 요청을 받았는데 브라우저에서 실패하는 이유를 포함한다.

- [ ] **Step 8: 문서 자체 검증을 수행한다**

Run:

```powershell
rg -n "TBD|TODO|FIXME|XXX" web/CORS.md
git diff --check -- web/CORS.md
rg -n "^## |Access-Control-|Same-Origin|Preflight|Spring Security|CSRF|30초|1분" web/CORS.md
```

Expected: 첫 명령은 결과 없음, `git diff --check`는 오류 없음, 마지막 명령은 설계에 포함된 각 핵심 섹션과 용어를 출력한다.

- [ ] **Step 9: CORS 문서만 커밋한다**

```powershell
git add -- web/CORS.md
git diff --cached --check
git commit -m "docs: add CORS interview study guide"
```

Expected: 새 커밋에는 `web/CORS.md`만 포함된다.

---

### Task 2: README 웹 목차 연결

**Files:**
- Modify: `README.md`

**Interfaces:**
- Consumes: Task 1에서 생성한 `web/CORS.md`
- Produces: 저장소 루트에서 CORS 문서로 이동할 수 있는 상대 링크

- [ ] **Step 1: CORS 항목을 링크로 변경한다**

`README.md`의 웹 주제 목록에서 다음 줄을 찾는다.

```markdown
- CORS
```

다음과 같이 변경한다.

```markdown
- [CORS](web/CORS.md)
```

- [ ] **Step 2: 링크와 변경 범위를 검증한다**

Run:

```powershell
Test-Path 'web/CORS.md'
rg -n -F '[CORS](web/CORS.md)' README.md
git diff --check -- README.md
git diff -- README.md
```

Expected: `Test-Path`는 `True`, `rg`는 CORS 링크 한 줄을 출력, `git diff --check`는 오류 없음, README diff에는 CORS 항목 한 줄의 변경만 나타난다.

- [ ] **Step 3: README 변경만 커밋한다**

```powershell
git add -- README.md
git diff --cached --check
git commit -m "docs: link CORS guide from README"
```

Expected: 새 커밋에는 `README.md`만 포함된다.

---

### Task 3: 최종 범위와 문서 완성도 검증

**Files:**
- Verify: `web/CORS.md`
- Verify: `README.md`

**Interfaces:**
- Consumes: Task 1의 본문과 Task 2의 목차 링크
- Produces: 발표 및 면접 준비에 바로 사용할 수 있는 검증된 산출물

- [ ] **Step 1: 최종 파일과 Git 상태를 확인한다**

Run:

```powershell
git log -3 --oneline
git status --short
git show --stat --oneline HEAD~1
git show --stat --oneline HEAD
```

Expected: 최근 두 커밋이 CORS 본문과 README 링크를 각각 포함한다. 기존 사용자 변경 세 파일은 작업 트리에 그대로 남아 있고 CORS 커밋에는 포함되지 않는다.

- [ ] **Step 2: Markdown 구조와 코드 블록 쌍을 확인한다**

Run:

```powershell
$corsText = Get-Content -Raw -Encoding UTF8 'web/CORS.md'
([regex]::Matches($corsText, '```')).Count % 2
(Select-String -Path 'web/CORS.md' -Pattern '^# ').Count
(Select-String -Path 'web/CORS.md' -Pattern '^## ').Count
```

Expected: 코드 펜스 나머지는 `0`, H1 개수는 `1`, H2 개수는 문서 구성에 필요한 복수의 섹션을 나타내는 양수다.

- [ ] **Step 3: 공식 참고 링크와 핵심 면접 항목을 최종 확인한다**

Run:

```powershell
rg -n "fetch.spec.whatwg.org|developer.mozilla.org|docs.spring.io" web/CORS.md
rg -n "Postman|curl|Access-Control-Allow-Credentials|Access-Control-Max-Age|SameSite|CSRF" web/CORS.md
```

Expected: Fetch, MDN, Spring 공식 자료가 모두 있고, 비브라우저 클라이언트·자격 증명·Preflight 캐시·쿠키·CSRF 관련 설명이 각각 존재한다.
