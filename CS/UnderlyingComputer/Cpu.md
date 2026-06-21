# Chapter 4. 트랜지스터에서 CPU로, 이보다 더 중요한 것은 없다

## Section 4.2. CPU는 유휴상태일 때 무엇을 할까?

- CPU가 유지되기 위해서는 어떤 프로세스든 실행이 되고 있어야 한다
  - Window에서는 가장 낮은 우선순위로 `System Idle Process`가 존재한다
    - 더 이상 스케줄링 할 프로세스가 없을 때 `halt`명령이 실행된다
    - 이는 process를 일시중지 시키는 `suspend`와는 다르다
    - `System Idle Process`는 `halt`명령어를 실행하는 순환이며, `halt`명령어를 실행해 저전력 상태로 진입한다
  ```C
  // Linux kernel의 구현 예시
  while (1)
  {
    // 재스케쥴링이 필요 없다면(할 일이 없다면)
    while (!need_resched()) 
    {
        // halt명령을 실행시킨다
        cpuidle_idle_call();
    }
  }
  ```  
  - 위 코드가 컴퓨터 시스템이 유휴 상태일때 CPU가 하는 일이며 핵심은 `halt`명령을 실행시키는 것이다
  - 가장 많이 실행되는 명령어는 대부분 `halt`이다
- 그렇다면 무한 순환 중인 `halt`상태의 탈출은 어떻게 해야할까?
  - 운영체제는 일정 시간마다 timer interrupt를 생성하고, CPU는 interrupt신호를 감지해 interrupt처리 프로그램을 실행한다
  - 즉, `System Idle Process`가 실행 중일때 timer interrupt를 통해 일시 중지시키고 interrupt 처리 함수는 시스템에 준비 완료된 프로세스가 있는지 확인하고 없다면 `System Idle Process`를 게속 실행한다
  - 즉, OS의 유휴 상태는 CPU가 무한 루프를 돌며 전력을 낭비하는 Spin 대기 방식을 사용하지 않고, halt 명령어로 CPU를 잠재운 뒤 주기적인 timer Interrupt 신호를 통해 깨어나는 방식으로 저전력을 유지하는 것이다
    - CPU가 깨어날 때를 제외하고는 완전히 멈춰(halt) 있으며, 특정 주기마다 외부 타이머 하드웨어가 던져주는 인터럽트 신호를 수동적으로 받아서 깨어나 할 일을 확인하는 구조다