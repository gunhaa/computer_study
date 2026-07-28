# key-auth(plugin) & credentials

- key-auth plugin
  - route나 service에 적용되어 `이 API에 접근하려면 API Key 검증을 거쳐야 한다`는 인증 정책
  - 클라이언트가 보낸 요청에서 API Key를 어디서 추출할지(Header, Query Parameter, Body 등), 인증 실패 시 어떤 상태 코드를 줄지 설정한다
- Consumer credentials
  - Consumer에게 발급되어 저장된 key
  - 요청에 포함된 API Key 값이 DB에 등록된 어떤 Consumer의 credentials와 일치하는지 찾아 사용자 식별(key-auth뿐만 아니라 jwt, basic-auth, oauth2 가능)

## 동작

1. 클라이언트 요청: 클라이언트가 `GET /orders -H "apikey: secret-api-key-1234"`로 API를 호출한다

2. key-auth 플러그인 동작: Route(혹은 service)에 설정된 key-auth가 요청 헤더에서 secret-api-key-1234라는 키 값을 추출한다

3. credentials 매칭: Kong은 등록된 Consumer들의 credentials postgre(혹은 메모리캐시)에서 조회하여, 해당 키 값(secret-api-key-1234)을 소유한 Consumer(loopy)를 찾는다

4. 인증 완료: 식별된 Consumer 명의로 승인되어 upstream으로 전달된다(식별된 Consumer 정보를 Upstream 요청 header에 자동으로 주입, e.g. X-Consumer-ID/X-Consumer-Username)