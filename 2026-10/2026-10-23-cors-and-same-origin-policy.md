# 오늘의 개발 지식: CORS와 동일 출처 정책

예상 읽기 시간: 약 10~15분

프론트엔드에서 API를 호출했는데 브라우저 콘솔에 `CORS policy` 오류가 나타납니다. 그런데 같은 주소를 서버에서 호출하거나 API 테스트 도구로 요청하면 정상 응답이 돌아옵니다. 왜 브라우저에서만 문제가 생길까요?

답은 브라우저의 **동일 출처 정책(Same-Origin Policy)**과, 서로 다른 출처 간 접근을 허용하는 **CORS(Cross-Origin Resource Sharing)**에 있습니다. 오늘은 이 둘의 관계와 인증, 보안, 장애 진단까지 살펴보겠습니다.

## 1. 출처란 무엇인가?

출처(origin)는 URL의 **프로토콜, 호스트, 포트**로 구성됩니다. 세 가지가 모두 같아야 동일 출처입니다.

| URL | `https://example.com`과 비교 |
|---|---|
| `https://example.com/posts` | 동일 출처: 경로는 출처의 일부가 아님 |
| `http://example.com` | 다른 출처: 프로토콜이 다름 |
| `https://api.example.com` | 다른 출처: 호스트가 다름 |
| `https://example.com:8443` | 다른 출처: 포트가 다름 |

예를 들어 프론트엔드가 `https://app.example.com`에서 실행되고 API가 `https://api.example.com`에 있다면, 같은 회사의 서비스여도 두 출처는 다릅니다. [MDN: Origin](https://developer.mozilla.org/en-US/docs/Glossary/Origin)

동일 출처 정책은 한 출처의 스크립트가 다른 출처의 데이터를 마음대로 읽지 못하게 제한합니다. 사용자가 은행과 악성 사이트를 동시에 열어 두더라도, 악성 사이트의 JavaScript가 은행 응답을 자유롭게 읽지 못하도록 하는 데 중요한 역할을 합니다.

## 2. CORS는 이 제한에 대한 합의다

CORS는 서버가 응답 헤더를 통해 **어떤 출처의 브라우저 코드가 응답을 읽어도 되는지** 알리는 방식입니다.

예를 들어 브라우저가 다음 요청을 보냅니다.

```http
GET /api/profile HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
```

서버가 해당 출처를 허용한다면 응답에 다음 헤더를 포함할 수 있습니다.

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.example.com
Content-Type: application/json
```

브라우저는 요청 출처와 응답의 허용 출처를 비교해 JavaScript가 응답을 읽게 할지 결정합니다.

여기서 중요한 구분이 있습니다.

> CORS는 서버의 인증·인가를 대신하지 않습니다. 브라우저가 교차 출처 응답을 스크립트에 공개할지 판단하는 규칙입니다.

API 테스트 도구나 서버 간 요청은 일반적으로 브라우저의 CORS 검사를 적용하지 않습니다. 따라서 `curl`에서는 성공하고 브라우저에서는 실패하는 상황이 가능합니다. [MDN: CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)

## 3. 요청은 갔는데 응답을 읽지 못할 수도 있다

흔한 오해는 “CORS 오류가 나면 서버에 요청이 전혀 도착하지 않았다”는 것입니다. 항상 그렇지는 않습니다.

일부 교차 출처 요청은 별도 사전 검사 없이 실제 요청이 먼저 전송됩니다. 서버는 요청을 처리했지만 응답에 필요한 CORS 헤더가 없다면, 브라우저는 JavaScript가 그 응답을 읽지 못하게 할 수 있습니다.

따라서 다음 상황이 생길 수 있습니다.

1. 브라우저가 서버에 요청을 보냅니다.
2. 서버가 데이터를 변경합니다.
3. 응답에 적절한 CORS 헤더가 없습니다.
4. 화면의 JavaScript는 CORS 오류로 실패한 것처럼 보입니다.

**화면에서 오류가 났다고 서버의 상태 변경이 없었다고 단정하면 안 됩니다.** 주문이나 결제처럼 재시도에 주의해야 하는 요청이라면 서버 기록과 실제 처리 상태부터 확인해야 합니다.

## 4. 프리플라이트: 실제 요청 전에 하는 확인

일부 요청은 실제 요청 전에 브라우저가 `OPTIONS` 요청을 먼저 보냅니다. 이것이 **프리플라이트(preflight)**입니다.

예를 들어 JSON과 `Authorization` 헤더를 사용하는 요청을 보낸다고 가정합니다.

```js
fetch("https://api.example.com/orders", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Authorization": "Bearer example-token",
  },
  body: JSON.stringify({ productId: 123 }),
});
```

브라우저는 실제 `POST` 전에 대략 다음과 같이 확인할 수 있습니다.

```http
OPTIONS /orders HTTP/1.1
Origin: https://app.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: authorization, content-type
```

서버는 허용하는 출처, 메서드, 헤더를 응답합니다.

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: POST
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 600
```

검사가 성공해야 브라우저가 실제 요청을 이어서 보냅니다. 프리플라이트는 모든 요청에 발생하는 것은 아닙니다. 요청 메서드와 헤더, 콘텐츠 타입 등에 따라 필요 여부가 달라집니다. `application/json`이나 `Authorization`을 사용하는 위 예제는 프리플라이트가 필요한 경우입니다. [MDN: Preflight request](https://developer.mozilla.org/en-US/docs/Glossary/Preflight_request)

`Access-Control-Max-Age`는 프리플라이트 결과를 일정 시간 재사용하도록 하는 설정입니다. 다만 브라우저마다 자체 제한이 있을 수 있으므로 설정한 시간 그대로 반드시 유지된다고 가정하면 안 됩니다.

## 5. 쿠키를 포함하는 요청은 더 주의해야 한다

브라우저에서 교차 출처 요청에 쿠키 같은 자격 증명을 포함하려면 클라이언트와 서버 양쪽 설정이 맞아야 합니다.

클라이언트 예시:

```js
fetch("https://api.example.com/profile", {
  credentials: "include",
});
```

서버 응답 예시:

```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
Vary: Origin
```

이때 `Access-Control-Allow-Origin: *`와 자격 증명을 함께 사용하는 방식은 허용되지 않습니다. 서버는 허용할 출처를 명시해야 합니다. 또한 `credentials: "include"`가 있다고 해서 쿠키가 반드시 전송되는 것은 아닙니다. 쿠키의 `SameSite`, `Secure`, 도메인 설정과 브라우저의 쿠키 정책에도 영향을 받습니다. [MDN: CORS와 자격 증명](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)

`Vary: Origin`은 서버가 요청의 출처에 따라 다른 `Access-Control-Allow-Origin` 값을 반환할 때 캐시가 응답을 구분하도록 돕습니다. 이를 빠뜨리면 공유 캐시가 한 출처에 대한 응답을 다른 출처 요청에 재사용하는 문제가 생길 수 있습니다.

특히 `Origin` 값을 아무 검증 없이 그대로 응답에 반사하면서 자격 증명까지 허용하는 설정은 피해야 합니다. 신뢰할 수 있는 정확한 출처 목록을 기준으로 비교해야 합니다. [MDN: 안전한 CORS 설정](https://developer.mozilla.org/en-US/docs/Web/Security/Practical_implementation_guides/CORS)

## 6. CORS와 CSRF는 같은 문제가 아니다

CORS와 CSRF는 관련이 있지만 서로 다른 개념입니다.

- **CORS**: 브라우저가 교차 출처 JavaScript에 응답을 공개해도 되는지 판단하는 규칙입니다. 일부 요청에는 사전 검사도 수행합니다.
- **CSRF**: 사용자가 로그인한 상태를 악용해 공격자가 의도하지 않은 상태 변경 요청을 보내게 하는 공격입니다.

CORS를 엄격하게 설정해도 일부 요청은 서버에 도착할 수 있으므로 CORS만으로 모든 CSRF를 막을 수는 없습니다.

쿠키 기반 인증에서 상태를 변경하는 요청에는 서비스 특성에 맞춰 CSRF 토큰, `Origin` 검사, 적절한 `SameSite` 쿠키 등 방어 수단을 고려해야 합니다. 무엇보다 서버는 요청마다 인증과 인가를 검증해야 합니다. [MDN: CSRF](https://developer.mozilla.org/en-US/docs/Web/Security/Attacks/CSRF)

또한 **출처(origin)**와 **사이트(site)**는 같은 개념이 아닙니다. 두 서브도메인은 출처가 달라도 같은 사이트로 취급될 수 있습니다. 따라서 CORS의 출처 비교와 쿠키의 `SameSite` 동작을 같은 기준이라고 생각하면 안 됩니다.

## 7. 프론트엔드와 백엔드에서 확인할 것

프론트엔드에서는 먼저 브라우저 개발자 도구의 Network 탭을 확인합니다.

1. 실제 요청 전에 `OPTIONS`가 발생했는지 확인합니다.
2. `OPTIONS`가 실패했다면 상태 코드와 응답 헤더를 확인합니다.
3. 실제 요청이 전송됐다면 그 응답에도 `Access-Control-Allow-Origin`이 있는지 봅니다.
4. 로그인 쿠키가 필요하다면 `credentials` 설정과 쿠키 정책을 확인합니다.
5. 리다이렉트나 오류 응답에서만 헤더가 누락되지 않았는지 확인합니다.

백엔드에서는 다음을 확인합니다.

1. 정확한 프론트엔드 출처를 허용하고 있는가?
2. 프리플라이트에 필요한 메서드와 헤더를 허용하는가?
3. 인증 미들웨어가 `OPTIONS` 요청을 예상치 못하게 차단하지 않는가?
4. 성공 응답뿐 아니라 401, 403, 500 등에서도 필요한 CORS 헤더가 적용되는가?
5. 프록시나 CDN이 헤더를 제거하거나 잘못 캐시하지 않는가?
6. 교차 출처 스크립트가 `ETag` 같은 응답 헤더를 읽어야 한다면 `Access-Control-Expose-Headers`에 명시했는가?

특히 어제 다룬 `ETag`를 브라우저에서 읽고 `If-Match`로 다시 보낸다면, 서버는 교차 출처 요청에서 **ETag 노출**과 **If-Match 요청 헤더 허용**을 모두 고려해야 합니다. 이 조합에서는 프리플라이트도 확인해야 합니다.

## 8. 자주 하는 실수

**“프론트엔드에서 `Access-Control-Allow-Origin`을 요청 헤더에 넣으면 된다.”**  
아닙니다. 이 헤더는 원칙적으로 서버가 응답에서 보내는 허용 정보입니다. 브라우저 코드가 임의의 허용 응답을 만들어 낼 수는 없습니다.

**“CORS 오류니까 서버 요청은 실패했다.”**  
항상 그렇지 않습니다. 프리플라이트가 실패하면 실제 요청이 전송되지 않지만, 실제 요청이 전송된 뒤 응답 접근이 차단된 경우도 있습니다. Network 탭에서 둘을 구분하세요.

**“모든 출처를 허용하면 인증 문제도 해결된다.”**  
CORS 허용과 인증·인가는 별개입니다. 오히려 민감한 응답을 광범위하게 공개할 위험이 생길 수 있습니다.

**“개발 환경 프록시가 되니 운영 환경도 문제없다.”**  
개발 서버가 요청을 프록시하면 브라우저가 같은 출처로 요청하는 것처럼 보일 수 있습니다. 운영 환경의 도메인 구성과 프록시 설정도 별도로 확인해야 합니다.

## 마무리

CORS 문제를 해결할 때 핵심은 단순히 헤더 하나를 추가하는 것이 아닙니다. **브라우저가 어느 단계에서 무엇을 차단했는지**, 그리고 서버가 어떤 출처와 자격 증명을 허용해야 하는지 이해하는 것입니다.

기억할 내용은 세 가지입니다. 출처는 프로토콜·호스트·포트로 구분합니다. CORS는 인증·인가를 대체하지 않습니다. 그리고 오류가 보이더라도 실제 요청이 서버에 도착했는지는 별도로 확인해야 합니다.
