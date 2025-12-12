# File System

- provides the mechanism for
  - on-line storage of and access to both data and programs
  - of the operating system and all the users of the computer system
- consists of two distinct parts
  - a collection of files, each storing related data
  - a directory structure, which organizes all the files in the system
- access methods
  - sequential access
  - direct access/relative access
- 파일은 운영체제에 의하여 정의되고 구현되는 추상적인 자료형이다
  - 파일은 논리 레코드의 연속으로서, 바이트, 행 또는 좀 더 복합적인 자료 항목들이다
  - 운영체제는 다양한 레코드형을 사용자에게 제공하거나 아니면 사용자가 프로그램상으로 정의하도록 해준다
- 운영체제의 가장 중요한 문제는 논리적인 파일을 하드 디스크나 NVM 장치 같은 실제 저장장치에 어떻게 매핑시키는지 이다
  - 보통 물리 레코드는 논리 레코드보다 훨씬 작기 때문에 논리 레코드를 물리 레코드에 연관시켜야한다
  - 이 작업은 운영체제에 의해 제공되거나 사용자의 응용 프로그램에서 할 수 있다
- 파일 할당 방식은 크게보면 sequential access(순차 접근), linked access(연속 접근), index access(인덱스 접근)이 있다
  - sequential acceess(Contiguous Allocation): 파일을 디스크의 하나의 연속된 공간에 저장하며, fragment의 발생이 쉽고 파일 크기 변경이 어렵다
  - linked access(Linked Allocation): 파일의 각 블록이 디스크의 임의의 위치에 저장되며, 각 블록은 다음 블록의 주소를 포인터로 가진다. 단편화가 발생하지 않지만 random access가 느리고 포인터 저장을 위한 오버헤드가 발생한다. FAT(File Allocation Table) 방식이 이 방법이다
    - Windows는 FAT와 달리 인덱스 할당 방식의 개념을 매우 발전시켜 사용한다
      - MFT(Master file Table, 볼륨 내의 모든 파일과 디렉터리에 대한 정보를 저장하는 테이블)을 이용해 $DATA속성을 이용해 클러스터 리스트를 저장한다 
  - index access(Indexed Allocation): 파일당 인덱스 블록(Index Block)을 하나씩 할당하고, 이 인덱스 블록에 파일 구성 블록들의 주소 목록을 모아둔다. 파일당 인덱스 블록을 위한 공간이 필요하며, 대용량 파일의 경우 인덱스 블록의 크기 제한 문제가 발생할 수 있다(unix/linux의 i-node가 해당 방식을 사용한다)
- 인덱스 할당이나 NTFS나 비슷한 방식인데 방법이 조금 다를뿐이다

| 특징 | 인덱스 할당 (개념) | NTFS (MFT 기반) |
| :--- | :--- | :--- |
| **관리 단위** | 파일당 1개의 인덱스 블록 | 볼륨(디스크) 전체의 MFT |
| **중앙 집중화** | 분산 (파일마다 인덱스) | 중앙 집중화 (모든 파일 정보를 MFT에 통합) |
| **메타데이터 위치** | 데이터 블록 외부에 별도 존재 (i-node 등) | 데이터 위치와 메타데이터가 한 엔트리에 통합 |
| **작은 파일 처리** | 인덱스 블록만 이용 | 데이터 자체를 MFT 엔트리에 직접 저장 (상주 데이터) |
| **효율성** | 랜덤 접근에 효율적 | 작은 파일의 접근에서 최상의 효율 제공 |