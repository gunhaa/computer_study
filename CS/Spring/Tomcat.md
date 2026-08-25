# Tomcat의 요청 및 연결 처리 흐름 (NIO Connector: Default)

### 1. TCP 3-Way Handshake 및 OS Queue 진입

1. 외부 Client로부터 서버의 해당 포트로 TCP 연결 요청(SYN)이 도착한다
2. OS 커널의 TCP 스택이 3-Way Handshake를 시작하며, 연결 진행 중인 상태의 소켓은 OS의 SYN queue에 들어간다
3. Client로부터 마지막 ACK 패킷이 도착하여 3-Way Handshake가 완결되면, 소켓은 accept queue (backlog queue)로 이동한다
  - accept queue의 최대 크기는 Tomcat의 `accept-count` 설정값과 OS의 `kern.ipc.somaxconn(Mac)/net.core.somaxconn(Linux)` 설정값 중 더 작은 값으로 지정된다
  - accept queue가 가득 차면 추가 요청은 drop되거나(default 동작) RST 패킷(sysctl net.ipv4.tcp_abort_on_overflow = 1 설정 시, 바로 실패)이 반환되며, Client 측에서는 TCP의 SYN 재전송(Exponential Backoff) 혹은 Connection Refused 에러(RST패킷 설정의 경우)가 발생한다

### 2. Tomcat의 Accept 및 NIO 기반 비동기/논블로킹 처리

4. Tomcat의 acceptor 스레드가 OS accept queue에서 준비된 소켓 연결을 `accept()`하여 가져온다
5. acceptor 스레드는 이 커넥션을 Java NIO selector를 관리하는 poller 스레드에게 전달한다
  - acceptor가 연결을 받아온 직후 스레드풀의 worker 스레드를 바로 할당하지 않는다
  - 대신 poller 스레드가 selector를 읽기 이벤트(OP_READ) 발생 여부를 논블로킹으로 감시한다
    - poller 스레드는 selector를 감시하며, 읽기 이벤트 발생해 읽을 준비가 완료되면 worker thread에 할당한다
    - poller 스레드는 kernel 기반 I/O multiplexing(`epoll_wait`) 대기 방식을 사용한다
  - worker thread는 요청을 처리(parsing)한다

### 3. 처리 및 응답
6. Worker Thread는 HTTP 요청을 수신하여 Servlet Container(Spring MVC 등)로 전달 후 로직을 수행한다
7. 처리가 끝나면 응답을 Client로 전송하고, Keep-Alive 여부에 따라 커넥션을 유지(poller thread 재등록)하거나 종료한다