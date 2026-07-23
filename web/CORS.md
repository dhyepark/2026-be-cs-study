# CORS (Cross-Origin Resource Sharing)

## 학습 목표

- Origin과 Same-Origin Policy가 무엇인지 설명할 수 있다.
- CORS가 필요한 이유와 브라우저·서버의 역할을 구분할 수 있다.
- 단순 요청과 Preflight 요청의 차이를 HTTP 메시지로 설명할 수 있다.
- 주요 CORS 헤더와 자격 증명 요청의 제약을 설명할 수 있다.
- Spring MVC와 Spring Security에서 CORS를 설정할 수 있다.
- CORS 오류를 진단하고 면접의 꼬리 질문에 답할 수 있다.

---

## 먼저 한 문장으로 정리

CORS는 서버가 HTTP 응답 헤더를 통해 **어떤 출처의 브라우저 코드에 응답을 공유할지 허용**하고, 브라우저가 그 허용 여부를 검사하는 HTTP 기반 메커니즘이다.

```text
Same-Origin Policy = 다른 출처의 응답을 함부로 읽지 못하게 하는 브라우저 보안 정책
CORS              = 서버가 허용한 교차 출처에는 응답을 공유할 수 있게 하는 메커니즘
```

CORS는 서버에 요청을 보낼 수 있는 모든 방법을 차단하는 방화벽이 아니다. 서버는 허용 범위를 응답 헤더로 표현하고, 이를 실제로 집행하는 주체는 주로 **브라우저**다.

---

## Origin이란?

Origin, 즉 출처는 URL의 다음 세 요소로 결정된다.

```text
Origin = Scheme(프로토콜) + Host(호스트) + Port(포트)
```

기준 URL을 `https://service.example:443/members`라고 해보자.

| URL | 동일 출처 여부 | 이유 |
| --- | --- | --- |
| `https://service.example/orders` | 동일 | 경로는 Origin에 포함되지 않고 HTTPS 기본 포트는 443이다. |
| `http://service.example/members` | 다름 | Scheme이 다르다. |
| `https://api.service.example/members` | 다름 | Host가 다르다. |
| `https://service.example:8443/members` | 다름 | Port가 다르다. |

경로, 쿼리 문자열, 프래그먼트는 Origin 비교에 영향을 주지 않는다.

```text
https://service.example/members?page=1
└─ scheme ─┘ └──── host ────┘ └ path/query ┘

기본 포트를 생략했다면 https는 443, http는 80으로 비교한다.
```

### Site와 Origin은 다르다

쿠키의 `SameSite`에서 말하는 Site와 CORS의 Origin은 같은 개념이 아니다.

예를 들어 `https://app.example.com`과 `https://api.example.com`은 일반적으로 같은 Site로 볼 수 있지만 Host가 다르므로 서로 다른 Origin이다. 따라서 쿠키가 전송되는지와 CORS 검사를 통과하는지는 각각 따져야 한다.

---

## Same-Origin Policy

Same-Origin Policy(SOP)는 한 출처에서 로드된 문서나 스크립트가 다른 출처의 리소스와 상호작용하는 방식을 제한하는 브라우저 보안 정책이다.

이 정책이 없다면 악성 사이트가 사용자의 브라우저에 저장된 인증 정보를 이용해 다른 서비스의 민감한 응답을 읽을 수 있다.

```text
1. 사용자가 bank.example에 로그인해 인증 쿠키를 가지고 있다.
2. 사용자가 evil.example에 접속한다.
3. evil.example의 JavaScript가 bank.example의 계좌 API를 요청한다.
4. SOP가 없다면 악성 코드가 계좌 정보를 읽어 외부로 전송할 수 있다.
```

SOP는 교차 출처 상호작용을 전부 같은 방식으로 금지하지 않는다.

- 교차 출처 쓰기: 링크 이동이나 HTML 폼 제출처럼 대체로 가능한 경우가 있다.
- 교차 출처 삽입: `<img>`, `<script>`, `<link>` 등은 조건에 따라 가능하다.
- 교차 출처 읽기: 민감한 정보 유출을 막기 위해 대체로 제한된다.

이때 신뢰할 수 있는 다른 출처에는 응답을 읽도록 허용할 방법이 필요하다. 그 방법이 CORS다.

---

## CORS가 필요한 상황

프런트엔드와 API 서버의 Origin이 다르면 브라우저의 `fetch`나 `XMLHttpRequest` 요청은 CORS 검사의 대상이 된다.

```text
프런트엔드: https://frontend.example
API 서버:   https://api.example
             └ Host가 다르므로 Cross-Origin
```

전체 흐름은 다음과 같다.

```text
JavaScript
    │ 교차 출처 요청
    ▼
브라우저 ───── HTTP 요청 + Origin ─────▶ API 서버
    ▲                                      │
    └── CORS 응답 헤더 검사 ◀── HTTP 응답 ┘
          │
          ├─ 허용: 응답을 JavaScript에 공개
          └─ 거부: JavaScript에서 응답 접근 차단
```

역할을 구분하면 다음과 같다.

| 주체 | 역할 |
| --- | --- |
| 프런트엔드 코드 | 요청 URL, 메서드, 헤더, 자격 증명 포함 여부 등을 정한다. |
| 브라우저 | `Origin`과 Preflight 관련 헤더를 붙이고, 서버 응답을 검사해 JavaScript에 공개할지 결정한다. |
| 서버 | 허용할 Origin, 메서드, 헤더, 자격 증명 여부를 CORS 응답 헤더로 표현한다. |

### Postman이나 curl에서는 왜 요청이 성공할까?

CORS와 SOP는 브라우저가 웹 페이지의 JavaScript를 제한하는 메커니즘이다. Postman, `curl`, 백엔드 서버 간 통신은 브라우저의 SOP 집행 대상이 아니므로 CORS 응답 헤더가 없어도 응답을 읽을 수 있다.

```text
브라우저 JavaScript → SOP/CORS 검사 적용
Postman, curl       → 브라우저 정책 미적용
서버 → 서버         → 브라우저 정책 미적용
```

따라서 "Postman에서는 되는데 브라우저에서는 안 된다"면 네트워크 연결이나 API 로직보다 CORS 설정을 먼저 의심할 수 있다.

---

## CORS 요청의 두 흐름

CORS 요청은 설명상 다음 두 종류로 나눈다.

| 구분 | 실제 요청 전 OPTIONS 요청 | 핵심 동작 |
| --- | --- | --- |
| 단순 요청(Simple Request) | 없음 | 실제 요청을 바로 보낸 뒤 응답 공유 여부를 검사한다. |
| Preflight 요청 | 있음 | OPTIONS로 허용 여부를 확인한 뒤 실제 요청을 보낸다. |

"단순 요청"은 현재 Fetch 표준이 직접 사용하는 용어는 아니다. Fetch 표준의 **CORS-safelisted method와 CORS-safelisted request-header 조건을 만족해 Preflight가 필요 없는 요청**을 흔히 이렇게 부른다.

### 중요한 차이

```text
단순 요청 실패
실제 요청 전송 ──▶ 서버가 처리할 수 있음 ──▶ 브라우저가 응답 읽기를 차단

Preflight 실패
OPTIONS 전송 ──▶ 허용되지 않음 ──X── 실제 요청은 전송하지 않음
```

따라서 CORS를 "모든 교차 출처 요청이 서버에 도착하지 못하게 막는 정책"이라고 설명하면 정확하지 않다.

---

## 단순 요청(Simple Request)

단순 요청은 다음 핵심 조건을 모두 만족해 Preflight를 생략하는 요청이다.

### 허용되는 메서드

- `GET`
- `HEAD`
- `POST`

### JavaScript에서 설정할 수 있는 주요 허용 헤더

- `Accept`
- `Accept-Language`
- `Content-Language`
- `Content-Type` — 아래의 제한된 미디어 타입만 가능
- `Range` — 단일 범위 값 등 추가 조건을 만족해야 함

### 허용되는 Content-Type

- `application/x-www-form-urlencoded`
- `multipart/form-data`
- `text/plain`

헤더 값에도 길이와 안전하지 않은 바이트 등 세부 제한이 있다. 또한 `XMLHttpRequest.upload` 이벤트 리스너나 `ReadableStream` 사용 여부도 조건에 영향을 줄 수 있다. 면접에서는 먼저 메서드, 직접 설정한 헤더, `Content-Type` 세 기준을 정확히 설명하면 된다.

`application/json`이나 `Authorization`은 safelist 조건에 포함되지 않는다. 따라서 다음 요청은 `POST`여도 일반적으로 Preflight가 발생한다.

```javascript
fetch("https://api.example/members", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Authorization": "Bearer token"
  },
  body: JSON.stringify({ name: "June" })
});
```

### 단순 GET 요청 예시

브라우저는 요청에 `Origin`을 넣는다. 이 헤더는 브라우저가 관리하며 프런트엔드 JavaScript가 임의로 설정할 수 없다.

```http
GET /members/1 HTTP/1.1
Host: api.example
Origin: https://frontend.example
```

서버가 이 Origin에 응답을 공유하려면 다음과 같이 응답한다.

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://frontend.example
Vary: Origin
Content-Type: application/json

{"id":1,"name":"June"}
```

브라우저는 `Access-Control-Allow-Origin`이 요청의 Origin과 맞는지 확인한다. 일치하면 JavaScript에 응답을 공개하고, 없거나 일치하지 않으면 응답 접근을 차단한다.

`Vary: Origin`은 서버 응답이 요청의 `Origin`에 따라 달라질 수 있음을 캐시에게 알린다. 여러 Origin을 허용 목록에서 동적으로 골라 응답한다면 잘못된 Origin용 응답이 재사용되지 않도록 함께 설정해야 한다.

---

## Preflight 요청

브라우저는 safelist 범위를 벗어나는 교차 출처 요청을 바로 보내지 않는다. 먼저 `OPTIONS` 요청으로 실제 요청의 메서드와 헤더를 서버가 허용하는지 확인한다. 이를 Preflight Request, 즉 사전 요청이라고 한다.

다음과 같은 `PUT` 요청을 보내려는 상황을 생각해보자.

```javascript
fetch("https://api.example/members/1", {
  method: "PUT",
  headers: {
    "Content-Type": "application/json",
    "Authorization": "Bearer token"
  },
  body: JSON.stringify({ name: "June" })
});
```

### 1. 브라우저가 Preflight 요청을 보낸다

```http
OPTIONS /members/1 HTTP/1.1
Host: api.example
Origin: https://frontend.example
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: content-type, authorization
```

- `Origin`: 실제 요청을 시작한 출처
- `Access-Control-Request-Method`: 실제 요청에서 사용할 메서드
- `Access-Control-Request-Headers`: 실제 요청에서 사용할 비-safelisted 헤더

### 2. 서버가 허용 범위를 응답한다

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://frontend.example
Access-Control-Allow-Methods: GET, PUT
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 600
Vary: Origin
```

브라우저는 요청하려던 Origin, 메서드, 헤더가 모두 허용되는지 확인한다.

### 3. 성공하면 실제 요청을 보낸다

```http
PUT /members/1 HTTP/1.1
Host: api.example
Origin: https://frontend.example
Content-Type: application/json
Authorization: Bearer token

{"name":"June"}
```

실제 응답에도 `Access-Control-Allow-Origin` 등 필요한 CORS 헤더가 있어야 한다. Preflight 응답에만 CORS 헤더를 넣고 실제 응답에서 빠뜨리면 최종 응답을 JavaScript가 읽을 수 없다.

### Preflight 결과 캐시

`Access-Control-Max-Age: 600`은 같은 조건의 Preflight 결과를 최대 600초 동안 재사용할 수 있다는 뜻이다. 이 캐시는 일반 HTTP 응답 캐시와 별개인 브라우저의 CORS Preflight 캐시다. 브라우저별 최대 캐시 시간이 설정값보다 짧을 수 있다.

---

## 주요 CORS 헤더

### 요청 헤더

요청 헤더는 일반적으로 브라우저가 자동으로 설정한다.

| 헤더 | 사용 시점 | 의미 |
| --- | --- | --- |
| `Origin` | CORS 요청 및 출처 정보가 필요한 요청 | 요청을 시작한 Origin |
| `Access-Control-Request-Method` | Preflight | 이후 실제 요청에서 사용할 메서드 |
| `Access-Control-Request-Headers` | Preflight | 이후 실제 요청에서 사용할 비-safelisted 헤더 목록 |

### 응답 헤더

| 헤더 | 의미 |
| --- | --- |
| `Access-Control-Allow-Origin` | 응답을 공유할 Origin 한 개 또는 자격 증명 없는 요청에서의 `*` |
| `Access-Control-Allow-Methods` | Preflight 이후 허용할 실제 요청 메서드 |
| `Access-Control-Allow-Headers` | Preflight 이후 실제 요청에서 허용할 헤더 |
| `Access-Control-Allow-Credentials` | 자격 증명이 포함된 요청의 응답을 공개할지 여부 |
| `Access-Control-Expose-Headers` | JavaScript가 읽을 수 있도록 추가로 공개할 응답 헤더 |
| `Access-Control-Max-Age` | Preflight 결과를 캐시할 시간(초) |

### Access-Control-Expose-Headers가 필요한 이유

CORS 응답에 성공해도 JavaScript가 모든 응답 헤더를 읽을 수 있는 것은 아니다. 기본적으로 공개되는 safelisted response header 외에 `X-Request-Id` 같은 헤더를 읽게 하려면 서버가 명시해야 한다.

```http
Access-Control-Expose-Headers: X-Request-Id
```

```javascript
const response = await fetch("https://api.example/members/1");
console.log(response.headers.get("X-Request-Id"));
```

---

## 자격 증명을 포함한 요청

CORS에서 자격 증명은 주로 쿠키, HTTP 인증 정보, TLS 클라이언트 인증서 등을 의미한다.

교차 출처 `fetch`에서 쿠키를 포함하려면 클라이언트가 명시적으로 요청해야 한다.

```javascript
fetch("https://api.example/members/me", {
  credentials: "include"
});
```

서버의 실제 응답에는 다음 두 헤더가 필요하다.

```http
Access-Control-Allow-Origin: https://frontend.example
Access-Control-Allow-Credentials: true
```

### 왜 *를 사용할 수 없을까?

자격 증명이 있는 요청에서 다음 설정은 허용되지 않는다.

```http
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```

쿠키처럼 사용자별 민감한 정보가 포함될 수 있는데 모든 Origin에 응답을 공개하면 안 되기 때문이다. 자격 증명 요청에서는 신뢰하는 Origin을 구체적으로 응답해야 한다. `Access-Control-Allow-Methods`, `Access-Control-Allow-Headers`, `Access-Control-Expose-Headers`의 `*`도 자격 증명 요청에서는 일반적인 와일드카드로 동작하지 않으므로 필요한 값을 명시하는 편이 안전하다.

### Preflight 요청 자체에는 자격 증명이 없다

표준상 Preflight의 credentials mode는 `same-origin`이므로 교차 출처 Preflight에는 쿠키 같은 자격 증명이 포함되지 않는다. 다만 이후 실제 요청에 자격 증명을 허용하려면 Preflight 응답에도 `Access-Control-Allow-Credentials: true`를 포함해야 한다.

### CORS를 통과해도 쿠키가 전송되지 않을 수 있다

CORS 허용은 쿠키 정책을 무시하지 않는다. 쿠키에는 별도로 다음 조건이 적용된다.

- `SameSite` 속성
- HTTPS에서의 `Secure` 속성
- Domain과 Path 범위
- 브라우저의 서드파티 쿠키 차단 정책

따라서 자격 증명 요청은 다음 조건을 모두 점검해야 한다.

```text
클라이언트 credentials 설정
        +
서버의 명시적 Allow-Origin
        +
Allow-Credentials: true
        +
쿠키 속성과 브라우저 쿠키 정책
```

---

## Spring MVC에서 CORS 설정하기

### 전역 설정

여러 컨트롤러에 같은 정책을 적용한다면 `WebMvcConfigurer`로 URL 패턴별 정책을 설정할 수 있다.

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

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

| 설정 | 의미 |
| --- | --- |
| `addMapping` | CORS 정책을 적용할 서버 URL 패턴 |
| `allowedOrigins` | 응답을 공유할 신뢰 Origin 목록 |
| `allowedMethods` | 허용할 HTTP 메서드 |
| `allowedHeaders` | 실제 요청에서 허용할 헤더 |
| `allowCredentials` | 자격 증명 요청 허용 여부 |
| `maxAge` | Preflight 결과 캐시 시간(초) |

운영 환경에서는 `allowedOrigins`를 설정 파일로 분리할 수 있지만, 요청의 `Origin`을 검증 없이 그대로 응답해서는 안 된다.

### 컨트롤러 단위 설정

특정 컨트롤러나 메서드만 허용하려면 `@CrossOrigin`을 사용할 수 있다.

```java
@CrossOrigin(origins = "https://frontend.example")
@RestController
@RequestMapping("/api/members")
public class MemberController {
    // ...
}
```

정책이 여러 곳에 흩어지면 허용 범위를 파악하기 어려우므로 공통 정책은 전역 설정으로 관리하고, 예외적인 경우에만 컨트롤러 단위 설정을 사용하는 편이 이해하기 쉽다.

---

## Spring Security와 함께 사용하기

Spring Security를 사용하면 CORS 처리가 인증 필터보다 먼저 이뤄져야 한다. Preflight 요청에는 보통 `JSESSIONID` 같은 쿠키가 없기 때문에 Security가 먼저 검사하면 미인증 요청으로 판단해 거절할 수 있다.

Spring MVC에 CORS 설정이 있고 별도의 `CorsConfigurationSource`가 없다면 Spring Security가 MVC 설정을 사용하도록 연결할 수 있다.

```java
import static org.springframework.security.config.Customizer.withDefaults;

@Bean
SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
            .cors(withDefaults())
            .authorizeHttpRequests(authorize -> authorize
                    .anyRequest().authenticated()
            );
    return http.build();
}
```

필터 흐름을 단순화하면 다음과 같다.

```text
Preflight OPTIONS
      │
      ▼
CORS 처리 ── 거부
      │ 허용
      ▼
Spring Security 인증·인가
      │
      ▼
Controller
```

애플리케이션에 여러 `CorsConfigurationSource`가 있거나 Security Filter Chain마다 정책이 다르면 어떤 설정을 사용할지 명시해야 한다. 또한 실제 API의 성공 응답뿐 아니라 인증 실패, 인가 실패, 예외 응답에도 필요한 CORS 헤더가 붙는지 확인해야 한다.

---

## 자주 발생하는 오류

### No Access-Control-Allow-Origin header

응답에 `Access-Control-Allow-Origin`이 없거나 요청 Origin과 일치하지 않는 경우다.

확인할 내용:

- 정확한 프런트엔드 Origin이 허용 목록에 있는가?
- `http`와 `https`, 포트 번호를 구분했는가?
- 정상 응답뿐 아니라 오류 응답에도 헤더가 있는가?

### Response to preflight request doesn't pass access control check

Preflight 응답이 올바르지 않은 경우다.

- 서버나 프록시가 `OPTIONS`를 허용하는가?
- 응답 상태가 성공 상태인가?
- 요청하려는 메서드가 `Access-Control-Allow-Methods`에 있는가?
- 요청하려는 헤더가 `Access-Control-Allow-Headers`에 있는가?

### Credential is not supported with wildcard origin

`credentials: "include"`를 사용하면서 서버가 `Access-Control-Allow-Origin: *`로 응답하는 경우다. 신뢰할 Origin을 명시하고 `Access-Control-Allow-Credentials: true`를 함께 사용해야 한다.

### 서버 로그에는 요청이 있는데 브라우저에서는 실패한다

단순 요청이거나 실제 요청 응답의 CORS 헤더가 잘못된 경우, 서버는 이미 요청을 처리했지만 브라우저가 응답을 JavaScript에 공개하지 않을 수 있다. 서버가 요청을 받았다는 사실과 브라우저 코드가 응답을 읽을 수 있다는 사실은 별개다.

### 프록시 뒤에서만 실패한다

애플리케이션 설정이 맞더라도 프록시나 API Gateway가 다음 문제를 만들 수 있다.

- `OPTIONS` 요청 차단
- CORS 응답 헤더 제거 또는 중복 추가
- 리다이렉트 응답 생성
- 오류 응답에 CORS 헤더 누락

한 계층에서 정책을 일관되게 책임지고, 응답에 `Access-Control-Allow-Origin`을 여러 번 중복해서 넣지 않도록 한다.

---

## 디버깅 순서

브라우저 개발자 도구의 Network 탭에서 다음 순서로 확인한다.

1. 요청 URL의 Scheme, Host, Port를 현재 페이지와 비교한다.
2. `OPTIONS` 요청이 있는지와 그 상태 코드를 확인한다.
3. 요청의 `Origin`, `Access-Control-Request-Method`, `Access-Control-Request-Headers`를 확인한다.
4. Preflight와 실제 응답 양쪽의 `Access-Control-Allow-*` 헤더를 확인한다.
5. Spring Security, 프록시, 예외 처리 응답에서도 CORS 헤더가 유지되는지 확인한다.

`curl`로 응답 헤더를 재현할 수도 있다.

```bash
curl -i -X OPTIONS 'https://api.example/members/1' \
  -H 'Origin: https://frontend.example' \
  -H 'Access-Control-Request-Method: PUT' \
  -H 'Access-Control-Request-Headers: content-type, authorization'
```

다만 `curl`은 CORS 정책을 집행하지 않는다. 이 명령은 서버가 어떤 헤더를 응답하는지 확인하기 위한 것이다.

---

## CORS에 대한 흔한 오해

### CORS는 서버 보안 기능이므로 허용되지 않은 요청은 서버에 도착하지 않는다?

아니다. CORS 정책을 집행하는 핵심 주체는 브라우저다. 단순 요청은 서버에 실제로 도착해 처리될 수 있고, 브라우저가 응답 읽기만 막을 수 있다. 브라우저가 아닌 클라이언트는 CORS와 무관하게 요청할 수 있다.

### CORS를 허용하면 인증 없이 API를 호출할 수 있다?

아니다. CORS와 인증·인가는 서로 다른 문제다.

```text
CORS      = 어느 Origin의 브라우저 코드에 응답을 공개할 것인가?
인증      = 요청한 사용자는 누구인가?
인가      = 그 사용자가 이 작업을 수행할 권한이 있는가?
```

### Preflight는 인증 요청인가?

아니다. Preflight는 이후의 교차 출처 요청에서 사용할 Origin, 메서드, 헤더를 서버가 허용하는지 확인한다. 사용자 신원을 확인하는 인증 절차가 아니다.

### 개발 환경에서만 발생하는 프런트엔드 문제인가?

아니다. 개발 서버와 API 서버의 포트가 달라 자주 보일 뿐이다. 운영에서도 서브도메인, 별도 API 도메인, CDN 등의 Origin이 다르면 동일하게 고려해야 한다.

---

## CORS와 CSRF의 차이

CORS와 CSRF는 관련이 있지만 해결하려는 문제가 다르다.

| 구분 | CORS | CSRF 방어 |
| --- | --- | --- |
| 주요 목적 | 교차 출처 응답을 브라우저 코드에 공개할지 제어 | 사용자가 의도하지 않은 상태 변경 요청 방지 |
| 주된 집행 주체 | 브라우저 | 서버의 CSRF 검증과 쿠키 정책 |
| 보호 대상 | 주로 응답 읽기 | 상태 변경 요청 |
| 대표 수단 | `Access-Control-Allow-*` | CSRF Token, SameSite Cookie, Origin 검증 등 |

단순한 교차 출처 폼 제출은 Preflight 없이 서버에 도착할 수 있다. 그러므로 CORS가 있다고 해서 CSRF 방어가 자동으로 되는 것은 아니다. 상태 변경 API는 인증·인가, CSRF 방어, 입력 검증 등을 별도로 설계해야 한다.

---

## 운영 시 주의점

- 필요한 Origin, 메서드, 헤더만 최소 범위로 허용한다.
- 요청의 `Origin`을 허용 목록 검증 없이 그대로 `Access-Control-Allow-Origin`으로 반환하지 않는다.
- 자격 증명을 허용할 때는 구체적인 Origin을 사용하고 신뢰 수준을 높게 본다.
- 동적 Origin 응답에는 캐시 오염을 막기 위해 `Vary: Origin`을 고려한다.
- 정상 응답뿐 아니라 401, 403, 500 같은 오류 응답도 점검한다.
- CORS를 인증, 인가, CSRF 방어, 네트워크 접근 제어의 대체재로 사용하지 않는다.

---

## 면접 답변

### 30초 답변

> CORS는 Cross-Origin Resource Sharing의 약자로, 서버가 어떤 Origin의 브라우저 코드에 응답을 공유할지 HTTP 헤더로 허용하는 메커니즘입니다. 브라우저는 Same-Origin Policy 때문에 기본적으로 다른 Origin의 응답 읽기를 제한하는데, 서버의 `Access-Control-Allow-Origin` 같은 헤더를 검사해 예외를 허용합니다. 즉 서버가 허용 범위를 표현하고 브라우저가 이를 집행합니다.

### 1분 답변

> CORS는 브라우저의 Same-Origin Policy를 필요한 범위에서 완화하기 위한 HTTP 기반 메커니즘입니다. Origin은 Scheme, Host, Port로 구분합니다. 교차 출처 요청이 safelist 조건을 만족하면 실제 요청을 바로 보내고 응답 헤더를 검사하지만, `PUT`, `application/json`, `Authorization`처럼 조건을 벗어나면 브라우저가 먼저 `OPTIONS` Preflight로 Origin, 메서드, 헤더의 허용 여부를 확인합니다. 쿠키 같은 자격 증명을 포함한다면 서버는 와일드카드 대신 구체적인 Origin과 `Access-Control-Allow-Credentials: true`를 응답해야 합니다. CORS는 브라우저가 집행하므로 Postman이나 서버 간 통신에는 적용되지 않으며, 인증·인가나 CSRF 방어를 대신하지도 않습니다.

---

## 예상 면접 질문과 꼬리 질문

### 1. CORS란 무엇인가요?

- 꼬리 질문: Same-Origin Policy와 어떤 관계인가요?
- 꼬리 질문: CORS 정책을 실제로 집행하는 주체는 누구인가요?

### 2. Origin은 무엇으로 결정되나요?

- 꼬리 질문: 도메인이 같고 포트가 다르면 같은 Origin인가요?
- 꼬리 질문: 경로나 쿼리 문자열이 다르면 어떻게 되나요?

### 3. 단순 요청과 Preflight 요청의 차이는 무엇인가요?

- 꼬리 질문: 단순 요청의 메서드와 `Content-Type` 조건을 말해보세요.
- 꼬리 질문: `POST application/json`은 왜 Preflight가 발생하나요?

### 4. Preflight 요청은 어떻게 동작하나요?

- 꼬리 질문: 왜 `OPTIONS`를 사용하나요?
- 꼬리 질문: `Access-Control-Request-Method`와 `Access-Control-Allow-Methods`의 차이는 무엇인가요?
- 꼬리 질문: Preflight 결과를 캐시할 수 있나요?

### 5. Postman에서는 성공하는데 브라우저에서는 왜 CORS 오류가 날까요?

- 꼬리 질문: 서버 간 통신에도 CORS 설정이 필요한가요?

### 6. `Access-Control-Allow-Origin: *`를 사용하면 안 되는 경우는 언제인가요?

- 꼬리 질문: 쿠키를 포함한 요청에는 어떤 응답 헤더가 필요한가요?
- 꼬리 질문: CORS 설정이 맞는데 쿠키가 전송되지 않는다면 무엇을 확인해야 하나요?

### 7. CORS와 CSRF는 어떤 차이가 있나요?

- 꼬리 질문: 단순 요청이 서버까지 도착할 수 있다는 사실이 왜 중요한가요?

### 8. 서버 로그에는 요청이 처리됐는데 브라우저가 CORS 오류를 표시할 수 있나요?

- 꼬리 질문: Preflight 실패와 실제 응답의 CORS 검사 실패는 어떻게 구분하나요?

### 9. Spring Security를 사용할 때 CORS 설정에서 주의할 점은 무엇인가요?

- 꼬리 질문: Preflight가 인증 필터에서 401이나 403을 받는 이유는 무엇인가요?

### 10. 모든 Origin을 허용하면 CORS 문제를 해결한 것 아닌가요?

- 꼬리 질문: 요청 Origin을 그대로 응답하는 구현은 어떤 위험이 있나요?

---

## 핵심 체크리스트

- Origin은 Scheme, Host, Port의 조합이다.
- SOP는 브라우저의 중요한 보안 정책이고, CORS는 허용된 교차 출처 응답 공유 메커니즘이다.
- 서버가 CORS 헤더를 응답하고 브라우저가 정책을 집행한다.
- 단순 요청은 실제 요청이 먼저 전송될 수 있다.
- Preflight는 `OPTIONS`로 실제 요청의 메서드와 헤더 허용 여부를 확인한다.
- 자격 증명 요청에서는 구체적인 Origin과 `Access-Control-Allow-Credentials: true`가 필요하다.
- Postman, `curl`, 서버 간 통신에는 브라우저의 CORS 제한이 적용되지 않는다.
- CORS는 인증·인가와 CSRF 방어를 대체하지 않는다.
- Spring Security보다 CORS 처리가 먼저 이뤄져야 한다.

---

## 참고 자료

- [Fetch Standard - CORS protocol](https://fetch.spec.whatwg.org/#http-cors-protocol)
- [MDN - Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)
- [MDN - Same-origin policy](https://developer.mozilla.org/en-US/docs/Web/Security/Defenses/Same-origin_policy)
- [Spring Framework - CORS](https://docs.spring.io/spring-framework/reference/web/webmvc-cors.html)
- [Spring Security - CORS](https://docs.spring.io/spring-security/reference/servlet/integrations/cors.html)
