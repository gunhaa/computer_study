# Acls와 Acl

- acls: Consumer에게 그룹을 할당할 때 사용하는 entity이다
  - Kong Admin에 consumer 하단의 Authorization에서 확인 가능하며, 특정 consumer가 소속된 그룹을 의미한다 
- acl: Route나 Service에 적용하여 접근 통제 규칙을 정의하는 plugin이다  
  - plugin의 형태로 존재하며, 어떤 id를 가지는지에 따라(route, service, global) 어떤 엔티티의 규칙인지를 알 수 있다
  - acl plugin의 allow: ["{acls}"]를 통해 접근 가능을 관리할 수 있다
  - enabled/disabled 형태로 acl을 관리하는 플러그인이다

