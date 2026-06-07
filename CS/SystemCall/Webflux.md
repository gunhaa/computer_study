# epoll과 webflux

- epoll의 I/O 모델은 reactor를 채택했다
- webflux는 이렇게 만들어진 epoll system call을 최대한 잘 이용하기 위해 reactor 패턴을 핵심 패턴으로 채택했다
  - os(linux)에서 더 효율적인 system call이 추가되어도(io_uring) 안정성이 최우선이라 epoll을 사용한다
    - netty-incubator-transport-io_uring를 통해 사용 할 수 있다
  - Windows 환경 내 WebFlux의 IOCP 활용 불가 원인은 Java NIO Selector의 select system call로 구현되어 있기 때문이다
    - Linux의 epoll에 대응하는 고성능 멀티플렉싱 시스템 콜의 부재
      - linux의 경우 nio가 epoll로 구현되어 있다
    - select system call은 fd가 많아질 경우 한계가 있다
    - select system call
      - fd를 전부 감시해주는 epoll의 이전 세대의 system call로써, 모든 fd를 직접 폴링으로 감시해서 많은 오버헤드가 있다
      - 이전 세대의 os는 fd가 많을 수 없다고 생각해 설계되어, 완료 통지가 가기는 하지만 epoll과 다르게 정확한 위치를 통지하는 것이 아닌 완료 사실만 통지하기에, fd를 전부 탐색(O(N))해서 많은 오버헤드가 있다
- 동작
  - webflux epoll의 reactor 추상화를 그대로 사용한 모델이다
  - webflux(netty) boss thread가 epoll을 client의 요청을 accept해서 socket connection(fd)을 만든다
  - socket connection(fd)의 완료 통지를 worker thread pool의 특정 thread와 매핑하여 특정 worker가 event를 통지받을 수 있도록 한다
  - socket connection(fd)는 재사용되며(http protocol의 keep-alive 설정 사용시), connection의 판별은 os network stack의 tcp connection을 통해 진행된다

## Netty 예제 자바코드

```java
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.util.Iterator;
import java.util.concurrent.ConcurrentLinkedQueue;
import java.util.concurrent.atomic.AtomicInteger;

public class MiniNettyServer {
    private static final int PORT = 8080;
    private static final int WORKER_COUNT = 2; // Worker 스레드 개수

    public static void main(String[] args) throws IOException {
        // 1. Worker 풀 생성 (각 Worker는 자신만의 epoll/Selector 루프를 가짐)
        WorkerThread[] workers = new WorkerThread[WORKER_COUNT];
        for (int i = 0; i < WORKER_COUNT; i++) {
            workers[i] = new WorkerThread("Netty-Worker-" + i);
            workers[i].start();
        }

        // 2. Boss 스레드 시작
        BossThread boss = new BossThread(PORT, workers);
        boss.setName("Netty-Boss-Thread");
        boss.start();
        
        System.out.println("🚀 Mini-Netty 서버가 " + PORT + " 포트에서 시작되었습니다.");
    }

    // --- [ BOSS THREAD: 연결 접수 전담 ] ---
    static class BossThread extends Thread {
        private final ServerSocketChannel serverChannel;
        private final Selector bossSelector;
        private final WorkerThread[] workers;
        private final AtomicInteger roundRobin = new AtomicInteger(0);

        public BossThread(int port, WorkerThread[] workers) throws IOException {
            this.workers = workers;
            this.bossSelector = Selector.open(); // 리눅스에선 epoll_create 호출됨
            this.serverChannel = ServerSocketChannel.open();
            this.serverChannel.configureBlocking(false); // 논블로킹 설정
            this.serverChannel.bind(new InetSocketAddress(port));
            // Boss는 오직 ACCEPT(접속 요청) 이벤트만 감시한다
            this.serverChannel.register(bossSelector, SelectionKey.OP_ACCEPT);
        }

        @Override
        public void run() {
            try {
                while (!Thread.currentThread().isInterrupted()) {
                    bossSelector.select(); // epoll_wait: 새로운 클라이언트가 올 때까지 블로킹
                    Iterator<SelectionKey> keys = bossSelector.selectedKeys().iterator();

                    while (keys.hasNext()) {
                        SelectionKey key = keys.next();
                        keys.remove();

                        if (key.isAcceptable()) {
                            // 클라이언트 접속! Socket Connection(FD) 생성
                            SocketChannel clientChannel = serverChannel.accept();
                            clientChannel.configureBlocking(false);
                            
                            // [핵심] 라운드로빈으로 Worker Thread 하나를 점찍어서 FD를 토스함
                            int workerIndex = roundRobin.getAndIncrement() % workers.length;
                            WorkerThread chosenWorker = workers[workerIndex];
                            
                            System.out.println(" New Connection! " + clientChannel.getRemoteAddress() 
                                    + " -> 할당된 워커: " + chosenWorker.getName());
                            
                            // 워커에게 소켓 등록을 요청
                            chosenWorker.registerNewChannel(clientChannel);
                        }
                    }
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    // --- [ WORKER THREAD: 할당받은 소켓의 Read/Write 이벤트만 전담 ] ---
    static class WorkerThread extends Thread {
        private final Selector workerSelector;
        // Boss 스레드가 안전하게 소켓을 넘겨주기 위한 큐
        private final ConcurrentLinkedQueue<SocketChannel> registerQueue = new ConcurrentLinkedQueue<>();

        public WorkerThread(String name) throws IOException {
            super(name);
            this.workerSelector = Selector.open(); // 각 워커마다 개별 epoll 루프를 가짐
        }

        public void registerNewChannel(SocketChannel channel) {
            registerQueue.add(channel);
            workerSelector.wakeup(); // select()로 대기 중인 워커 스레드를 깨움
        }

        @Override
        public void run() {
            try {
                ByteBuffer buffer = ByteBuffer.allocate(1024);

                while (!Thread.currentThread().isInterrupted()) {
                    // epoll_wait: 자기가 담당하는 소켓들에 데이터가 들어올 때까지 대기
                    workerSelector.select(); 

                    // Boss가 새로 넣어준 소켓이 있다면 내 epoll 관심 목록에 등록(OP_READ)
                    processRegistrationQueue();

                    Iterator<SelectionKey> keys = workerSelector.selectedKeys().iterator();
                    while (keys.hasNext()) {
                        SelectionKey key = keys.next();
                        keys.remove();

                        if (key.isReadable()) {
                            SocketChannel clientChannel = (SocketChannel) key.channel();
                            buffer.clear();
                            int read = clientChannel.read(buffer);

                            if (read == -1) { // 클라이언트가 연결을 끊음
                                System.out.println(" Connection Closed: " + clientChannel.getRemoteAddress());
                                clientChannel.close();
                                continue;
                            }

                            // Keep-Alive 실감하기: 동일한 FD로 계속 요청이 들어옴
                            buffer.flip();
                            byte[] data = new byte[buffer.limit()];
                            buffer.get(data);
                            System.out.println("[" + getName() + "] 데이터 수신: " + new String(data).trim());
                            
                            // 에코(Echo) 응답 보낸 후 소켓을 닫지 않고 유지(Keep-Alive)
                            clientChannel.write(ByteBuffer.wrap("HTTP/1.1 200 OK\r\nContent-Length: 5\r\n\r\nHELLO".getBytes()));
                        }
                    }
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        private void processRegistrationQueue() throws ClosedChannelException {
            while (!registerQueue.isEmpty()) {
                SocketChannel channel = registerQueue.poll();
                // 이제 이 소켓(FD)은 이 워커 스레드가 끊어질 때까지 전담 마크함
                channel.register(workerSelector, SelectionKey.OP_READ);
            }
        }
    }
}
```

- gemini로 생성한 netty 예제코드
- 결국 eventloop의 실체는 fd(socket connection)를 감시하는 `while (!Thread.currentThread().isInterrupted())`루프이다
  - `selector.select(); // epoll_wait: 새로운 클라이언트가 올 때까지 블로킹` 을 통해 blocking으로 cpu의 spin을 막고 이벤트를 감시하는 것이 핵심이다
  - Boss EventLoop: `bossSelector.select()/key.isAcceptable()`로 새로운 클라이언트 연결(Accept)을 대기
    - boss eventloop(main)은 socket의 connection까지만 관리하며, connection이 맺어진 socket의 참조를 worker eventloop로 넘겨 감시시킨다
  - Worker EventLoop: `workerSelector.select()/key.isReadabke()`로 소켓의 데이터 입출력(Read/Write)을 대기

### 바로 worker.register()를 하지 못하고, wakeup()을 통해 깨워야 하는 이유

- worker는 select()를 통해 자신의 자원(keys: 등록된 socket, selectedKeys: event가 발생한 socket)에 락을 건다
- boss.register()를 바로 할 경우 worker의 자원을 획득 하려고 lock을 시도하기 때문에 lock 경합으로 인한 blocking이 발생한다
  - 이 경우 boss는 자바 모니터 락을 획득하기 위해 BLOCKED상태로 멈추고, worker는 select() system call을 호출하여 kernel의 I/O 통지를 기다리며 sleep 상태가 되어있다
- 그래서 자원 획득의 경합을 막기위해 `ConcurrentLinkedQueue<SocketChannel>`를 통해 accept된 socket의 참조를 넣은 후 wakeup()을 통해 깨워 worker가 직접 추가시키도록 해야한다