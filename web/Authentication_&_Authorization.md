# Authentication & Authorization

Authentication(인증)은 사용자가 누구인지 확인하는 과정이고, Authorization(인가)은 인증된 사용자가 어떤 자원에 접근하거나 어떤 작업을 수행할 수 있는지 확인하는 과정이다.

두 개념은 항상 함께 사용되지만 목적이 다르다.

- Authentication : "당신이 누구인가?"
- Authorization : "당신이 이것을 해도 되는가?"

예를 들어 로그인은 Authentication이고, 관리자 페이지 접근 권한을 확인하는 과정은 Authorization이다.

---

## Authentication

Authentication은 사용자의 신원을 확인하는 과정이다.

대표적인 인증 방식은 다음과 같다.

- 아이디 / 비밀번호
- OTP
- 휴대폰 인증
- 이메일 인증
- 생체 인증
- OAuth 로그인

사용자가 로그인하면 서버는 입력한 인증 정보를 검증하고, 성공하면 해당 사용자를 인증된 사용자로 판단한다.

### 인증 과정

아이디와 비밀번호 로그인은 다음과 같이 동작한다.

```text
Client

↓

아이디 / 비밀번호 입력

↓

Server

↓

DB 조회

↓

비밀번호 비교

↓

Authentication 성공
```

이후 서버는 로그인 상태를 유지하기 위해 Session이나 Token을 발급한다.

### 비밀번호는 왜 Hash해서 저장하는가?

비밀번호를 그대로 저장하면 DB가 유출되는 순간 모든 비밀번호가 함께 노출된다.

따라서 비밀번호는 Hash 함수를 이용해 저장한다.

```text
1234

↓

BCrypt

↓

$2a$10$...
```

로그인 시에는 입력한 비밀번호를 다시 Hash하여 저장된 값과 비교한다.

---

## Authorization

Authorization은 인증된 사용자가 어떤 작업을 수행할 수 있는지 확인하는 과정이다.

Authentication이 끝난 이후 수행된다.

```text
게시글 조회

USER 가능
ADMIN 가능
```

```text
회원 삭제

USER 불가
ADMIN 가능
```

처럼 권한에 따라 접근을 제한한다.

### Role 기반 인가

가장 많이 사용하는 방식이다.

사용자에게 역할(Role)을 부여한다.

```text
USER

ADMIN

MANAGER
```

API마다 필요한 Role을 지정한다.

```text
/admin/**

↓

ADMIN만 접근 가능
```

### 리소스 소유권 기반 인가

Role만으로는 충분하지 않은 경우도 있다.

예를 들어 USER 권한이라도 자신의 게시글만 수정할 수 있어야 한다.

```text
현재 로그인 사용자 ID

=

게시글 작성자 ID
```

인 경우에만 수정을 허용한다.

---

## Authentication과 Authorization의 차이

Authentication은 사용자의 신원을 확인하는 과정이다.

Authorization은 인증된 사용자의 권한을 확인하는 과정이다.

둘의 차이를 정리하면 다음과 같다.

| Authentication | Authorization |
|----------------|---------------|
| 누구인가?          | 무엇을 할 수 있는가?  |
| 로그인            | 접근 제어         |
| 먼저 수행          | 인증 이후 수행      |

---

## 401과 403

Authentication과 Authorization의 차이는 HTTP 상태 코드에서도 나타난다.

### 401 Unauthorized

인증이 실패했거나 인증 정보가 없는 경우이다.

예를 들어

- 로그인하지 않은 사용자
- 만료된 JWT
- 잘못된 Access Token

등이 해당된다.

### 403 Forbidden

인증은 성공했지만 권한이 없는 경우이다.

예를 들어

```text
USER가

/admin/users

접근
```

처럼 권한이 부족한 경우 반환된다.

---

# 로그인 상태는 어떻게 유지하는가?

HTTP는 Stateless 프로토콜이다.

즉, 이전 요청을 기억하지 않는다.

따라서 로그인에 성공하더라도 다음 요청에서는 다시 사용자가 누구인지 알 수 없다.

그래서 인증 정보를 매 요청마다 함께 전달해야 한다.

대표적인 방식이 Session과 JWT이다.

---

## Session 기반 인증

Session 방식에서는 서버가 로그인 상태를 관리한다.

로그인에 성공하면 서버는 Session을 생성하고 Session ID를 클라이언트에게 전달한다.

이후 클라이언트는 Cookie를 통해 Session ID를 함께 보낸다.

```text
로그인

↓

Session 생성

↓

Session ID 발급

↓

Cookie 저장

↓

요청마다 Cookie 전송

↓

Session 조회

↓

사용자 확인
```

### 장점

- 서버에서 로그인 상태를 관리한다.
- 로그아웃 시 즉시 Session을 제거할 수 있다.
- 토큰 구조를 직접 관리할 필요가 없다.

### 단점

- 서버 메모리를 사용한다.
- 서버가 여러 대라면 Session 공유가 필요하다.

---

## JWT 기반 인증

JWT(JSON Web Token)는 서버가 토큰을 발급하는 방식이다.

로그인 성공 후 JWT를 발급하고, 클라이언트는 이후 모든 요청에서 JWT를 함께 전송한다.

```text
로그인

↓

JWT 발급

↓

Client 저장

↓

Authorization Header 전송

↓

JWT 검증

↓

사용자 확인
```

### JWT 구조

JWT는 다음 세 부분으로 구성된다.

```text
Header

.

Payload

.

Signature
```

Header에는 알고리즘 정보가 저장된다.

Payload에는 사용자 정보와 만료 시간이 저장된다.

Signature는 토큰 위조 여부를 검증하는 역할을 한다.

### 장점

- 서버가 로그인 상태를 저장하지 않는다.
- 서버를 여러 대 운영하기 쉽다.
- 확장성이 좋다.

### 단점

- 발급된 토큰을 즉시 회수하기 어렵다.
- 토큰이 탈취되면 만료 전까지 사용할 수 있다.

---

## Session과 JWT 비교

| Session       | JWT          |
|---------------|--------------|
| 서버가 로그인 상태 관리 | 클라이언트가 토큰 보관 |
| 서버 메모리 사용     | 서버 메모리 사용 X  |
| 로그아웃 즉시 가능    | 토큰 회수 어려움    |
| Session 공유 필요 | 서버 확장에 유리    |

Session과 JWT는 어느 한쪽이 항상 더 좋은 것이 아니다.

서비스 규모, 운영 방식, 보안 요구사항에 따라 적절한 방식을 선택해야 한다.

---

# 핵심 정리

Authentication은 사용자의 신원을 확인하는 과정이다.

Authorization은 인증된 사용자의 권한을 확인하는 과정이다.

HTTP는 Stateless이므로 로그인 상태를 유지하기 위한 방법이 필요하다.

대표적인 방식은 Session과 JWT이며, 각각 장단점이 존재한다.

Authentication과 Authorization은 대부분의 웹 서비스에서 함께 사용되며, 안전한 사용자 인증과 접근 제어를 위한 핵심 개념이다.