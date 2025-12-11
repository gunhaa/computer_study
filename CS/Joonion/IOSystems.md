# I/O Systems, 입출력 시스템

- 컴퓨터의 두 가지 주요 작업은 계산(Computing)과 입출력 작업(I/O)이다
- Two main jobs of a computer: I/O and computing
  - In many cases, the main job is I/O
    - for instance, web browsing, file editing, youtube, game, and so force
  - The role of Operation System in I/O is
    - to manage and control I/O operations and I/O devices
- 시스템 버스

  ![images1](images/iosystems1.png)
 
  - 버스란?
    - 버스는 공유 전송 매체이다. (Shared Transmission Medium)
    - 한 번에 한 장치만 성공적으로 전송이 가능하다
    - 하나 이상의 장치들이 공동으로 여러 선을 사용한다면, 이러한 선을 버스라고 부른다
    - 버스의 정의는 회선의 집합으로써 이를 통해 어떻게 해야 메시지들을 주고받을 수 있는지를 정한 프로토콜까지를 포함한다
  - 시스템 버스
    - 시스템 버스는 컴퓨터의 주요 구성요소들을 연결하는 버스이다

## Memory - Mapped I/O 메모리 맵드 입출력

- 입출력 전송을 하기 위해 처리기는 어떻게 명령어와 데이터를 컨트롤러에 전달하는가?
- 모든 컨트롤러(하드웨어)는 (제어용 or 데이터용)레지스터를 가지고 있다
  - processor는 이 컨트롤러의 레지스터에 비트 패턴을 쓰거나 읽음으로써 입출력을 수행한다
- 입출력 장치 컨트롤러(하드웨어)는 보통 네개의 레지스터로 구성되어있다
  - 입력 레지스터(data-in): 호스트가 입력을 얻기 위해 읽기를 수행
  - 출력 레지스터(data-out): 호스트가 데이터를 출력하기 위해 쓰기를 수행
  - 상태 레지스터(status): 호스트가 읽는 용도이며, 이 비트들은 현재의 명령의 완료/읽어도 되는지/오류 등의 상태를 알린다
  - 제어 레지스터(control): 호스트가 입출력 명령을 내리거나 장치의 모드를 변경하기 위해 쓰기를 수행하는 대상이다
  - 여기서 호스트는 OS와 하드웨어적으로 실행하는 CPU를 의미한다

1. Memory Mapped I/O
- 물리적인 I/O 레지스터에 접근하기 위해, 시스템 설계자가 메인 메모리(RAM) 주소 공간의 일부를 할당하여 이 주소들을 컨트롤러 레지스터에 연결하는 것이다
2. Polling(busy-waiting)[PIO]
- reading the status register repeatedly until the busy bit becomes clear
- CPU가 I/O 장치의 작업 완료 여부를 확인하기 위해 지속적으로 상태를 감시하는 방식이다
- 비효율적이기에 많이 사용되지 않는다
3. interrupt[PIO]
- CPU has a wire called the interrupt-request line
- If CPU detects an interrupt, it jumps to an ISR(interrupt service routine) to handle an interrupt
- The addresses of ISRs is specified in the interrupt vector table
  - ISR 내부에서는 CPU가 직접 I/O 컨트롤러의 데이터 레지스터에서 데이터를 한 바이트(혹은 워드)씩 읽어와 메인 메모리로 옮기는 작업을 수행한다(폴링도 이와같이 한바이트씩 메인 메모리로 옮긴다)
- CPU 하드웨어는 interrupt-request line이라고 불리는 선을 하나 갖으며, CPU는 매 명령어를 끝내고 다음 명령어를 수행하기 전에 늘 이 선을 검사한다
- interrupt를 이용하는 기술이 바로 async처리이다(대표적으로 nodejs의 async-await)
  - 정확하게는 interrupt는 I/O 완료를 알리는 하드웨어/커널의 수단이고, async/await는 이를 사용자 레벨에서 효율적으로 사용하기 위한 소프트웨어 문법이라고 구분할 수 있다
4. DMA(Direct Memory Access)
- used to avoid programmed I/O[PIO] (one byte at a time)
- useful for handling large data transfer
- DMA를 사용하지 않는다면 인터럽트 방식 I/O는 데이터가 준비될 때마다 CPU가 개입하여 한 바이트씩 옮기는 방식이다 
  - DMA만이 CPU 개입 없이 블록 단위로 대량의 데이터를 한 번에 전송할 수 있다
  - DMA의 가장 큰 특징은 데이터 전송 과정에서 CPU가 반복적으로 관여할 필요가 없다는 것이다
  - 이는 Programmed I/O (PIO) 방식이 '한 바이트씩(one byte at a time)' CPU의 LOAD/STORE 명령을 통해 데이터를 옮기는 비효율성을 해소한다
- I/O에 CPU가 개입하는 것(PIO)은 예전 방식이다
  - DMA가 최신 방식이다
  - 현대 IOCP/epoll의 async system call은 DMA를 이용해서 I/O를 효율적으로 처리한다
    - IOCP(Input Output Completion Port)는 DMA를 통해 I/O가 완료된 후 그 결과를 스레드에게 효율적으로 전달하여 처리하게 할지 관리하는 OS의 port 역할을 한다
    - IOCP의 Port는 DMA 장치로부터 직접 통지를 받는 물리적인 포트를 의미하는 것이 아니라, OS 내부에 존재하는 논리적인 구조를 의미한다
    - DMA 등을 통해 커널 수준에서 비동기 I/O 작업이 실제로 완료되면, 커널은 이 완료된 작업의 정보(완료 패킷)를 이 큐(Port)에 넣어준다
    - 애플리케이션의 워커 스레드들은 GetQueuedCompletionStatus() 함수를 호출하여 이 큐(Port)에 도착한 완료 패킷을 가져간다
    - 큐(Port)에 들어있는 것은 작업이 완료되었다는 `통지 정보`이며, async I/O 요청 시 애플리케이션은 OS에게 Buffer의 메모리 주소를 넘겨준다(Buffer와 hardware가 DMA를 수행하는 것이 DMA의 핵심이다)
    - 이 Buffer는 보통 애플리케이션이 미리 확보해 둔 메인 메모리(RAM)의 특정 위치이며, 완료시 신호를 받고 Buffer의 데이터를 애플리케이션이 수거해 사용한다
    - 즉, `Buffer와 hardware가 DMA를 수행하는 것이 DMA의 핵심이며, async system call의 핵심이다`