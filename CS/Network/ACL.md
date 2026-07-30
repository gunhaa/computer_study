# ACL Access Control List

- 방화벽, 웹사이트 등에서 정보 이용 주체가 정보(객체)에 대해 어떤 권한을 가지는지 정해 놓은 목록
  - 예시를 들자면 'A 파일에 대해 A 사용자는 읽기와 쓰기 권한을, B 사용자는 읽기 권한을 가진다'와 같이 사용자 이름과 권한을 표시해 두는 것이 있다

## Kong ACL

- Kong gateway의 경우 plugin으로 ACL을 사용할 수 있다
  - key-auth plugin을 사용해 key auth를 이용한 controll을 할 수 있다(ACL과 연관이 없는 별도의 플러그인)
  - ACL plugin을 사용해 Allow List와 Deny List를 작성할수있다