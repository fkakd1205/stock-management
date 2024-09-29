# stock-management

how to solve concurrency issues in stock management

<br />

### 해결방법 1 - synchronized

* Java에서 지원하는 기능
* Spring의 `@Transactional`이 설정되어 있다면, 메서드가 랩핑되어 실행되므로 실제 트랜잭션 커밋 전에 다른 메서드가 실행하여 재고 수량이 맞지 않게 된다.\
-> Spring의 `@Transactional`을 제거해 이 문제를 해결
* 하지만 `synchronized`는 하나의 프로세스 안에서만 락이 보장된다. 서버가 여러 대라면 데이터 접근이 여러 곳에서 가능해진다. 따라서 서버1이 재고를 조회하고 갱신하기 전에, 서버2가 재고를 조회한다면 둘
  다 접근이 가능해지고 결국 race condition이 발생하게 된다.

-> 실제 운영 서비스는 여러 대 서버를 사용하기 때문에 `synchronized`는 거의 사용되지 않는다.