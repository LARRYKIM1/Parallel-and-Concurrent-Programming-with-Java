# CH10 - Designing Parallel Programs

- Parallel 디자인 단계
  - Partitioning
  - Communication
  - Agglomeration
  - Mapping

![image](https://user-images.githubusercontent.com/49010295/144940783-614431b7-80ab-4488-9aab-4b261be1a55e.png)

## Partitioning

![image](https://user-images.githubusercontent.com/49010295/144940329-ea8408b1-347b-4a5e-8ec3-82073510c222.png)

- 단계 설명:  문제를 보다 작은 여러 개의 문제들로 분해해야 하는 것
- decomposition (병렬 프로그래밍을 위한 첫 단계)
  - domain decomposition
    - 전체 데이터를 가능한 동일한 크기로 나누어 각 프로세스에 할당하고 프로세스는 할당 받은 데이터에 대해 동일한 일련의 계산을 수행하는 것
    - block decomposition
    - cyclic decomposition
  - functional decomposition
    - 각 프로세스가 서로 다른 계산을 할당 받아 수행하는 것. 
    - 각 프로세스는 동일한 데이터 또는 서로 다른 데이터를 가지고 **서로 다른 함수의 계산을 수행**하게 된다.
    - ![image](https://user-images.githubusercontent.com/49010295/144938793-65264824-60ff-4277-ac89-ae5904cf838e.png)

## Communication

![image](https://user-images.githubusercontent.com/49010295/144940365-2a410fff-d4e2-40fa-8185-fd6e1b9732da.png)

- 단계 설명: Coordinate task execution and share information
- 비유: 컵케익 12개가 있고 색깔별로 3개씩 총 4종류를 만든다고 할때 서로 고른 색상을 공유해야 한다.
- Point-to-Point communication
  - sender-receiver (producer-consumer)
  - 하지만, 10개 작업을 10개로 쪼개서 consumer에게 처리 시킨후 다시 gather 한다고 할때는 비효율적. 
  - 중앙화보단, 트리처럼 나눠지게 
- Synchronous blocking vs Asynchronous nonblocking
- 고려해야될 요소: overhead, latency bandwidth

## Agglomeration

![image](https://user-images.githubusercontent.com/49010295/144940389-cca7347c-197e-447d-b23c-e7a0f22fedc4.png)

- 통신하는 프로세스들을 그룹화하는 방식으로 효율적이게 작동하게 도와준다. 
- granularity 공식 = computation / communication
- Fine-Grained Parallelism
  - 장: workload가 골고루 분배된다
  - 단: 통신과 동기화에 overhead
- Coarse-Grained Parallelism
  - 장: 통신이 적어, 계산에 집중 가능
  - 단: 비효율적 로드발란싱

## Mapping

![image](https://user-images.githubusercontent.com/49010295/144940262-11feb559-5367-424c-bc41-a7f40225b6a2.png)

- 단계 설명: specify where each task will execute
- 적용 불가한 환경: single-core 프로세서, 자동화 task 스케줄링
- 목적: 실행 시간을 줄이자. 
- 전략
  - 여러 프로세서에서 concurrently 실행되게
  - 한 프로세서에서 빈번하게 통신하게 

## QUIZ

- Why does the mapping design stage not apply to applications written for common desktop operating systems?
  - The operating system automatically handles scheduling threads to execute on each processor core.
- Which stage of the parallel design process focuses on specifying where each task will execute?
  - mapping
- What is an advantage of using coarse-grained parallelism with a small number of large tasks?
  - high computation-to-communication ratio
- What is an advantage of using fine-grained parallelism with a large number of small tasks?
  - good load-balancing
- Granularity can be described as the ratio of `_____` over `_____`.
  - computation; communication
- Which stage of the parallel design process focuses on combining tasks and replicating data or computation as needed to increase program efficiency?
  - agglomeration
- Which scenario describes the best use case for a point-to-point communication strategy?
  - A small number of tasks need to communicate with each other.
- Which stage of the parallel design process focuses on coordinating task execution and how they share information?
  - communication
- Why does the partitioning design stage occur before the communication stage?
  - You need to know how the problem will be divided in order to assess the communication needs between individual tasks.
- Which stage of the parallel design process focuses on breaking the problem down into discrete pieces of work?
  - partitioning



## 참고문헌

- https://www.kma.go.kr/aboutkma/intro/supercom/super/super_program.jsp?printable=true&
- 



