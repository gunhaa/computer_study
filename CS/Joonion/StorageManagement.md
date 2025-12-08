# Storage Management

- Mass-Storage
  - non-volatile, secondary storage system of a computer
  - usually, HDD(Hard Disk Drive) or NVM(Non-Volatile Memory)
    - HDD는 물리적이고, NVM은 전기적이다
    - HDD는 물리적으로 움직이기에 I/O가 오래 걸리고, NVM은 전기식이기에 I/O가 상대적으로 빠르다
  - sometimes, magnetic tapes, optical disks, or cloud storage
    - using the structure of RAID systems
- HDD Scheduling 목표
  - to minimize the access time(or seek time)
  - to maximize data transfer bandwidth
  - seek time을 줄여 disk bandwidth(throughput, 처리량)를 높이는 것이 목표이다
    - use FIFO, SCAN, Circular-SCAN hdd scheduling algorithm 
      - scan의 경우 위치를 찾으며 요청을 처리, c-scan의 경우 헤드를 한쪽에 다다를때 까지 처리후, 반대쪽으로 스캔한다

## Storage Device Management

- 운영체제는 저장장치와 관련하여 몇 가지 책임을 져야한다(드라이브의 초기화, 드라이브로부터의 부팅 및 손상된 블록의 복구 등)
- Boot block
  - For a computer to start running, when it is powered up,
    - it must have an initial program to run
  - A bootstrap loader is stored in NVM flash memory,
    - and mapped to a known memory location
  - 부트스트랩 로더는 시스템 마더보드의 NVM 플래시 메모리 펌웨어에 저장되며 알려진 메모리 위치에 매핑된다
    - CPU 레지스터에서 장치 컨트롤러 및 메인 메모리의 내용에 이르기까지 시스템의 모든 측면을 초기화한다
  - Windows 시스템은 부트코드를 하드 디스크의 첫 번째 논리 블록 또는 NVM 장치의 첫 번째 페이지에 배치한다(MBR, 이곳은 Master Boot Record라고 불린다)
    - 시스템이 부팅 파티션을 식별하면 해당 파티션의 첫 번째 섹터/페이지(부트 섹터라고 불린다)를 읽고 이 섹터/페이지는 시스템을 커널로 안내한다
    - MBR은 디스크의 전체 구조와 부팅 가능한 파티션의 위치를 알려주는 역할을 하는 반면, 부트 섹터는 각 개별 파티션의 시작점에 위치하여 운영 체제 커널을 로드하는 코드를 담고 있다
