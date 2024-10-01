# stock-management

how to solve concurrency issues in stock management

<br />

### 해결방법 1 - synchronized

- Java에서 지원하는 기능
- Spring의 `@Transactional`이 설정되어 있다면, 메서드가 랩핑되어 실행되므로 실제 트랜잭션 커밋 전에 다른 메서드가 실행하여 재고 수량이 맞지 않게 된다.\
-> Spring의 `@Transactional`을 제거해 이 문제를 해결
- 하지만 `synchronized`는 하나의 프로세스 안에서만 락이 보장된다. 서버가 여러 대라면 데이터 접근이 여러 곳에서 가능해진다. 따라서 서버1이 재고를 조회하고 갱신하기 전에, 서버2가 재고를 조회한다면 둘
  다 접근이 가능해지고 결국 race condition이 발생하게 된다.

-> 실제 운영 서비스는 여러 대 서버를 사용하기 때문에 `synchronized`는 거의 사용되지 않는다.

<br />
<br />

### 해결방법 2 - DB

<b>1. Pessimistic Lock</b>
- 실제로 데이터에 Lock 을 걸어서 정합성을 맞추는 방법
- exclusive lock을 걸고, 다른 트랜잭션에서는 lock이 해제되기 전에 데이터를 가져갈 수 없다.
- 데드락 주의
- 쿼리 메서드에 `@Lock(LockModeType.PESSIMISTIC_WRITE)` 어노테이션을 이용해 쉽게 Pessimistic Lock을 걸 수 있다.
- 실제 쿼리는 `for update`가 추가되어 실행

장점
1. 충돌이 빈번하게 일어난다면 Optimistic Lock보다 성능이 좋을 수 있다.
2. Lock을 통해 업데이트 제어 → 데이터 정합성 보장

단점
1. 별도의 Lock을 잡기 때문에 성능 감소가 있을 수 있다.

<br />

<b>2. Optimistic Lock</b>
- 실제로 Lock 을 이용하지 않고 버전을 이용함으로써 정합성을 맞추는 방법
- version 필드를 추가해, 데이터를 읽을 때와 반영할 때 version을 비교한다. 반영 시 내가 읽은 버전이 아닌 다른 버전이라면 application에서 다시 읽은 후에 작업을 수행
  - Entity에 version 컬럼 추가 및 `@Version` 어노테이션 설정. 이때 `javax.persistence.Version` 사용
- 쿼리 메서드에 `@Lock(LockModeType.Optimistic)` 어노테이션을 이용해 쉽게 Optimistic Lock을 걸 수 있다.
- `for update` 없이 version을 통해 정합성 확인
- 실행에 오류가 있다면 Xms 이후에 다시 조회하고, 수정하는 작업을 진행
  - 이 역할을 위한 Facade 클래스 생성

장점
1. 별도의 Lock을 걸지 않아 Pessimistic Lock보다 성능 상 이점이 있다.

단점
1. 개발자가 실패로직을 직접 작성해줘야 한다.

<br />

→ 만약 충돌이 빈번하게 일어날 것으로 예상된다면 Pessimistic Lock을 추천, 그렇지 않다면 Optimistic Lock을 추천.

<br />

<b>3. Named Lock</b>
- 이름을 가진 메타데이터 Lock
- 실제 데이터에 락을 거는 것이 아닌, 별도의 공간에 락을 걸어서 사용한다.
- Pessimistic Lock은 로우나 테이블 단위로 걸지만, Named Lock은 메타데이터에 락을 건다.
- 트랜잭션이 종료될 때 락이 자동으로 해제되지 않아, 별도의 명령어로 해제를 수행해주거나 선점시간이 끝나야 해제된다.

장점
1. NameLock은 주로 분산락을 구현할 때 사용한다. Pessimistic Lock은 timeout을 구현하기 힘들지만 Named Lock은 timeout을 구현하기 쉽다. 

단점
1. 트랜잭션 종료시에 Lock 해제 및 세션 관리를 잘해줘야 하고 실제 구현할 때는 구현방법이 복잡해질 수 있다.

<br />
<br />

### 해결방법 3 - Redis
Redis에서 동시성 문제를 해결할 때 사용하는 라이브러리 Lettuce, Redisson을 이용해 해결

<br />

<b>1. Lettuce</b>
- setnx 명령어를 활용하여 분산락 구현
  - setnx(set if not exist) : key-value를 설정할 때, 값이 없을 때만 set하는 명령어
- spin lock 방식
  - 락을 획득하려는 스레드가 락을 사용할 수 있는지 반복적으로 확인하면서 락 획득을 시도하는 방식
  - 따라서 개발자가 직접 retry로직을 작성해줘야 한다.

장점
1. 구현이 간단하다 
2. spring-data-redis를 이용하면 lettuce를 기본으로 사용하기 때문에 별도의 라이브러리를 사용하지 않아도 된다. 
3. Lettuce를 활용한 방법은 Named Lock과 거의 비슷하다. Named Lock과는 달리, 세션 관리에 신경을 안 써도 된다는 장점이 있다.

단점
1. spin lock 방식이기 때문에 동시에 많은 스레드가 lock 획득 대기 상태라면 redis에 부하가 갈 수 있다.

<br />
