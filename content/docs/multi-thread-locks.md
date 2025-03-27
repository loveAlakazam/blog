---
title: "synchronized 와 ReentrantLock 은 어떤 차이점이 있을까?"
date: "2025-03-26"
draft: false
---

## 동시성제어를 테스트하는 것은 단위테스트에 적합할까? 통합테스트에 적합할까?

JVM 내에서 동시성테스트를 하려고한다면, synchornized, ReentrantLock, JVM 내 큐(queue)를 사용하는 방법이 존재합니다. 그러나 동시성제어를 테스트할때 통합테스트에서 실행해야되는건지 아니면 유닛테스트에서 테스트를 해야될지 애매할겁니다.

단위테스트는 가장 작은 테스트인만큼 1개 메서드/함수 단위로 독립적인 기능을 빠르게 검증하기 위한 테스트입니다.
동시요청이 발생할 때 큐를 이용해서 순서를 제공해주거나 잠금(locking)연산을 수행하여 다른요청이 접근하지 못하도록 막거나, 큐를 이용해서 순서를 보장해줘야하는 역할까지 검증을 해야되기 때문에 단위테스트만으로는 어려울거같습니다.

즉, 동시성은 여러개의 스레드가 동시에 접근하거나 실행될 때 발생하는 문제를 검증해야하므로, 단일 스레드 환경에서 실행되는 단위테스트만으로는 동시적인 상황을 재현하기가 어렵습니다. **따라서 동시성을 테스트하려면 통합테스트로 검증** 해야됩니다.

---

## 미션1: 동일한 사용자가 동시에 포인트 충전을 요청한 경우

> [테스트 대상]
> 동일한 사용자가 동시에 포인트 1000원을 100번 충전 요청한 경우
>
> 정상적으로 처리되야하며 총 10만원을 보유해야한다

### (공통) 통합테스트 코드

```java
@SpringBootTest
@AutoConfigureMockMvc
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class ChargeConcurrencyTest {
 @Autowired
 private MockMvc mockMvc;

 @Autowired
 private PointService pointService;

 @Autowired
 private UserPointRepository userPointRepository;

 @Autowired
 private PointHistoryRepository pointHistoryRepository;

 @Autowired
 private ObjectMapper objectMapper;

 @Autowired
 private UserPointLockManager userPointLockManager;


 @BeforeEach
 void setUp() {
  // UserPoint 의 초기 포인트값을 0 으로한다.
  userPointRepository.save(1L, 0L);
 }

@Test
 void 동시에_100번_충전요청을_요청했을때_정상적으로_처리되며_충전요청후_보유잔액이_100000원이면_성공이다() throws Exception {
  // given
  long id = 1L;
  int threadCount = 100;
  long chargeAmount = 1000L;

  ExecutorService executor = Executors.newFixedThreadPool(10); // 스레드풀 10개
  CountDownLatch latch = new CountDownLatch(threadCount); // 요청가능한 스레드개수
  ChargeRequestBody requestBody = new ChargeRequestBody(chargeAmount);
  String json = objectMapper.writeValueAsString(requestBody);

  // when
  for(int i = 0 ; i < threadCount ; i++) {
   executor.submit(()-> {
    try {
      // 충전 API 호출
     mockMvc.perform(patch("/point/"+id+"/charge")
      .contentType(MediaType.APPLICATION_JSON)
      .content(json));
    } catch (Exception e) {
     e.printStackTrace();
     throw new RuntimeException(e);
    } finally {
     latch.countDown(); // 요청가능 스레드 개수 감소
    }
   });
  }

  latch.await(); // 다 끝날 때까지 대기
  Thread.sleep(1000); // 1초 정도 대기후 최종 포인트 확인

  // then
  long expectedPoint = chargeAmount * threadCount;
  long finalPoint = userPointRepository.findById(id).point();
  assertEquals(expectedPoint, finalPoint, "충전후 포인트값이 예상값("+expectedPoint+")과 실제값("+finalPoint+")이 서로다릅니다.");
 }
}
```

### synchronized

- `synchronized`는 하나의 스레드만 임계영역(critical section)에 접근하도록 보장하는 키워드로, 공유자원에 대한 동시접근을 차단하여 Race Condition을 방지합니다.

- Java에서 제공하는 synchronized 키워드는 모니터락(MonitorLock)을 이용하여 동기화를 지원합니다.

- Monitor은 OS가 아닌 프로그래밍 수준에서 제공하는 동기화 메커니즘입니다. Java 모든 객체(인스턴스)는 내부적으로 고유한 모니터를 가지며, 이를 획득하고 해제하며 동기화 작업을 수행합니다. 즉, 모니터와 락은 서로 상호보완적인 관계로 대체하거나 비교할 수 있는 개념이 아닙니다.

- 스레드 상태

| 스레드 상태     | 설명                                                                           |
| :-------------- | ------------------------------------------------------------------------------ |
| NEW             | 스레드가 새로 생성되었지만 아직 시작하지 않은 상태                             |
| RUNNABLE        | 동기화된 락이 풀리기를 기다리는 상태. synchronized 사용시 발생                 |
| BLOCKED         | 동기화 락이 풀리기를 기다리는 상태                                             |
| WAITING         | 다른 스레드의 특정 작업이 완료되기를 무한정 기다리는 상태. wait(), join() 호출 |
| TIMEOUT_WAITING | 특정시간동안 대기하는 상태. sleep(), wait(timeout), join(timeout) 호출         |
| TERMINATED      | 스레드의 실행이 완료된 상태                                                    |

> 특징

- 락을 획득하지 못한 스레드는 RUNNABLE 상태에서 BLOCKED 상태로 전환됩니다. 락을 획득할 때까지 대기하며, 이동안 CPU 실행 스케줄링에서 제외됩니다.

- 여러 스레드가 대기중일 경우, 락 획득 순서는 보장되지 않습니다.

- synchronized 블록안에서 변수의 메모리 가시성 문제가 자동으로 해결되므로 별도의 volatile 선언이 필요하지 않습니다.
  - volatile 키워드: 해당변수는 모두 읽기와 쓰기 작업이 CPU 캐시가 아닌 _메인메모리에서_ 이뤄지는 것을 의미한다.

> 단점

- BLOCKED 상태의 스레드는 락이 풀릴때까지 무한 대기를 하며 synchronized는 인터럽트, 타임아웃을 지원하지 않습니다.

- 다만 wait()을 통해 Blocked 상태인 스레드를 WAITING 으로 변경하고, 해당 스레드를 인터럽트 시킬 수 있습니다.

- 공정성문제: 락이 돌아왔을 때 BLOCKED 상태의 여러 스레드 중에 어떤 스레드가 락을 획득할지 알 수 없습니다.

> 충전 서비스 내부로직에 synchronized 예약어를 추가하여 임계구역 블록 지정하기

```java
// PointServiceImpl

@RequiredArgsConstructor
public class PointServiceImpl implements PointService {

 private final UserPointRepository userPointRepository;
 private final PointHistoryRepository pointHistoryRepository;
 private final UserPointLockManager userPointLockManager;
 private static final Logger log = LoggerFactory.getLogger(PointServiceImpl.class);

  ...

 @Override
 public synchronized ChargeResponse charge(ChargeRequest request) {

  long id = request.id();
  long amount = request.amount();

   // 로그기록
   log.info("::: 🔒 Lock acquired for userId: {}, thread: {}", id, Thread.currentThread().getName());UserPoint myPoint = this.userPointRepository.findById(id);

   // 포인트 충전
   long pointAfterCharge = myPoint.charge(amount);

   // 포인트내역에 '충전' 기록
   this.pointHistoryRepository.insert(id, amount, TransactionType.CHARGE);

   // 보유포인트 정보 수정
   UserPoint result = this.userPointRepository.save(id, pointAfterCharge);
   log.info("::: thread {} 작업완료:: 유저 id {}의 충전후 보유 포인트: {}",Thread.currentThread().getName(),  id, result.point() );
   return ChargeResponse.from(result);
  }
}
```

> 실행결과: 성공

### ReentrantLock

`ReentrantLock`은 동일한 쓰레드가 여러번 락을 획득할 수 있는 재진입이 가능한 락으로 `synchronized` 보다 더 정밀한 락제어와 락 확인 상태, 타임아웃, 인터럽트 처리등이 가능한 클래스입니다. 또한 `ReentrantLock`은 실시간 제어, deadlock 회피, 락 상태 진단 등 고급제어가 필요할 때 적합합니다.

> 유저포인트 ID(id) 마다 락을 관리 - ConcurrentHashMap 으로 분리된 락들을 관리

```java
// UserPointLockManager

@Component
public class UserPointLockManager {
    // 사용자별 분리된 락을 관리하는 맵
    // 1. 사용자 ID(id) 마다 하나의 고유한 락(Object)를 저장하는 맵
    // 2. synchronized에 넘길 락을 하나만 쓰면 전역락(exclusive lock)이 되므로 사용자별로 분리된 락객체를 관리
 private final ConcurrentHashMap<Long, ReentrantLock> locks = new ConcurrentHashMap<>();

 public ReentrantLock getLock(long id) {
    // locks.computeIfAbsent(id, key -> new Object());
    // 1. 사용자 ID(id)에 해당하는 락객체가 이미 있으면 그 객체를 반환하고, 없다면 새로운 락을 넣는다.
    // 2. 사용자 ID(id)별 하나의 고유한 락을 필요할 때만 만들고 중복으로 만들지않도록 보장한다.
  return locks.computeIfAbsent(id, key -> new ReentrantLock());
 }
}
```

> 충전 서비스 내부로직에 try블록을 임계구역 블록 지정하고
> 임계구역 입구에는 `lock.lock()`은 잠금상태인지(이미 요청작업을 수행하고있는 중인지) 아닌지를 확인하고, 잠겨있다면 끝날때까지 기다려야합니다. 반대로 열려있는 상태라면 임계영역에 진입하여 잠궈야합니다.
> 작업결과에 상관없이 요청작업 수행이 완료되면 `lock.unlock()`으로 잠금을 해제합니다.

```java
// PointServiceImpl

@Service
@RequiredArgsConstructor
public class PointServiceImpl implements PointService {

 private final UserPointRepository userPointRepository;
 private final PointHistoryRepository pointHistoryRepository;
 private static final Logger log = LoggerFactory.getLogger(PointServiceImpl.class);
 private final UserPointLockManager userPointLockManager;

 @Override
 public ChargeResponse charge(ChargeRequest request) {
  long id = request.id();
  long amount = request.amount();


  // 락 획득하여 다른요청이 들어오지 못하도록 임계구역을 잠금
  ReentrantLock lock = userPointLockManager.getLock(id);
  lock.lock();
  try {
      // try 블록안은 임계구역 이므로, 하나의 요청이 작업을 수행
      log.info("::: 🔒 Lock acquired for userId: {}, thread: {}", id, Thread.currentThread().getName());

      // 공유자원 정의
      UserPoint userPoint = this.userPointRepository.findById(id);
      long myPoint = userPoint.point();

      this.pointHistoryRepository.insert(id, amount, TransactionType.CHARGE); // 포인트내역에 '충전' 기록

      UserPoint result = this.userPointRepository.save(id, myPoint + amount); // 포인트 충전

      return ChargeResponse.from(result);
  } finally {
   lock.unlock(); // 락을 반환하여 임계구역을 잠금해제
  }
 }
}
```

> 실행결과: 성공

---

## synchronized 와 ReentrantLock 은 어떤 차이점이 있을까?

- synchronized는 메서드블록으로 락을 획득하면 블록내에서 다른 스레드가 접근하는 것을 막습니다. 하지만 락의 해제하는 코드가 존재하지 않습니다.

반면 ReentrantLock은 synchronized 보다 더 많은 인터페이스들을 제공해줍니다. 대표적으로 잠금해제(unlock)을 제공해주어 잠금을 해제할 시기를 명시적으로 나타낼 수 있습니다.

- 잠금관리의 유연성: 명시적으로 잠금획득(lock())/ 잠금해제(unlock()) 을 제공해줍니다.
- 잠금을 시도하는 기능: `tryLock()` 스레드가 무기한 대기하지 않고 잠금을 획득할 수 있도록 합니다.
- 인터럽트 가능한 잠금획득: `lockInterruptibly()` 스레드가 중단될 수 있는 상황에서 잠금을 획득할 수 있도록 합니다.
- 공정성 정책 옵션: ReentrantLock 의 공정성 정책을 지정할 수 있습니다. 공정한 잠금은 오래 기다리는 스레드의 우선순위를 정하여 스레드의 기아를 피하는데 사용됩니다. synchronized 메커니즘은 공정성을 보장하지 않습니다.
- 시간 잠금 대기: `tryLock(long timeout,TimeUnit unit)` 스레드가 지정된 기간동안 잠금을 획득하려고 시도할 수 있습니다. 스레드가 잠금을 기다리는 시간을 제한해야하는 시나리오에서 사용됩니다.

> 미션내용: 동일한 사용자가 아닌 여러사용자이 동시에 충전을 했을 때, 둘의 차이는 어떻게 될까?

두개의 락은 모두 다 비관적락이며, 스레드가 이미 작업을 수행중이라면, 다른 스레드들은 앞의 스레드의 작업이 끝날때까지 기다려야합니다. 해당 미션내용을 진행했을 때, 두개의 락이 진행하는데 걸리는 시간과 실행결과를 측정해보고자합니다.

---

## 미션2: 서로다른 사용자가 동시에 포인트 충전을 요청한 경우

> [테스트 대상]
> 서로다른 사용자 10명이 동시에 포인트 1000원을 10번 충전했을 경우
>
> 정상적으로 처리되야하며 각 유저는 총 만원을 보유해야한다.

### (공통) 통합테스트 코드

```java
 @Test
 void 서로_다른유저_10명이_동시에_1000원_충전을_요청했을때_정상적으로_합산에_성공한다() throws Exception {
  // given
  int numberOfUsers = 10; // 인원수
  int requestPerUser = 10; // 한사람당 요청개수

  // 포인트 초기화
  for(long id = 1L; id < numberOfUsers ; id++) {
   userPointRepository.save(id, 0L);
  }

  ExecutorService executor = Executors.newFixedThreadPool(5); // 스레드풀: 5개. (요청 5개 동시에 실행가능)
  CountDownLatch latch = new CountDownLatch(numberOfUsers * requestPerUser); // 10 * 10
  long chargeAmount = 1000L;

  // when
  for(long id = 1L; id <= numberOfUsers; id++) {
   ChargeRequest request = new ChargeRequest(id, chargeAmount);

   // 유저 1명당 10번을 요청한다.
   for(int i = 0 ; i < requestPerUser; i++) {
    long uid = id;
    executor.submit(() -> {
     try {
      if(uid == 1L) sleep(1000);
      pointService.charge(request);
     } catch(Exception e) {
      e.printStackTrace();
      throw new RuntimeException(e);
     } finally {
      latch.countDown();
     }
    });
   }
  }

  latch.await();
  sleep(1000);

  // then
  log.info("::: 테스트 종료후 유저별 보유포인트 조회 :::");
  for(long id = 1L; id <= numberOfUsers; id++){
   long actualPoint = userPointRepository.findById(id).point();
   log.info("유저 ID {} 의 보유포인트: {}", id, actualPoint);
  }

  for(long id =1L; id <= numberOfUsers; id++) {
   long actualPoint = userPointRepository.findById(id).point();
   assertEquals(requestPerUser * chargeAmount, actualPoint,"UserPoint id "+id+ "인 회원의 포인트는 예상금액과 다릅니다: "+ actualPoint + "원" );
  }
 }
```

- 특정스레드를 슬립을 시키지 않은채 동시성제어 테스트를 실행시킨 결과

> - synchronized : 성공 (44s)
>
> - ReentrantLock : 성공 (31s)

ReentrantLock 이 상대적으로 실행속도가 빠른편입니다. 공정성을 제공하고 해시맵으로 락을 구분하기 때문입니다. 그에반면 락의 구분이 없습니다. synchronized는 `synchronized` 키워드로 부여된 charge함수블록이 임계구역임을 나타냅니다.

만일 특정 유저에게만 1초의 Thread.sleep 을 했을 경우 synchronized와 ReentrantLock 간의 실행시간차이가 점점 더 커집니다.

```java
for(long id = 1L; id <= numberOfUsers; id++) {
    ChargeRequest request = new ChargeRequest(id, chargeAmount);

    // 유저 1명당 10번을 요청한다.
    for(int i = 0 ; i < requestPerUser; i++) {
        long uid = id;
        executor.submit(() -> {
            try {
                if(uid == 1L) sleep(1000); // id=1 번인 유저는 1초 대기
                pointService.charge(request);
            } catch(Exception e) {
                e.printStackTrace();
                throw new RuntimeException(e);
            } finally {
                latch.countDown();
            }
        });
    }
}
```

> - synchronized : 성공 (47s)
>
> - ReentrantLock : 성공 (33s)

---

## 결론

- 현재 synchronized, ReentrantLock 등 JVM내에서만 활용되는 락이며 하나의 서버에서는 가능합니다.
- synchronized 은 임계구역을 블록으로 지정하여 나타내지만 단순하게 동시성제어를 처리하는데 좋습니다.
- synchronized 는 간단하게 한개의 스레드에만 접근하도록 설정해줍니다. 공유데이터에 하나의 스레드만이 접근이 가능하다는 조건이 하나의 프로세스에만 보장됩니다. 이러한 특징때문에 scale-out 하게되면 서버가 여러대일때 동시성을 보장하지 않습니다.
- ReentrantLock 은 임계구역을 진입하기전에는 lock를 임계구역을 빠져나오면 unlock 으로 명시하여 synchronized 보다는 정교하게 나타냅니다.
- 동시성제어와 관련된 테스팅은 작성할 때는 단위테스트보다 통합테스트가 적합합니다.
- 선입선출 자료구조인 큐(queue)를 이용해서 동시성제어 를 해결할 수 있습니다. 하지만 Lock다르게 여러 요청이 한번에 들어와도 각 요청을 큐에넣고 순차적으로 처리하는방식입니다. 순서가 있기 때문에 동시성을 제거시켜서 해결하는 방법입니다.

- 참고
  - [Java Synchronized vs ReentrantLock: In-Depth Comparison for Optimal Locking Choice](https://anytech.medium.com/java-synchronized-vs-reentrantlock-which-lock-should-i-use-really-0cd8bd7cd3e5)
  - [Java의 동기화 기법 - synchronized & ReentrantLock 에 대해](https://dkswhdgur246.tistory.com/76)
  - [스프링에서의 동시성 처리](https://velog.io/@pak4184/%EC%8A%A4%ED%94%84%EB%A7%81%EC%97%90%EC%84%9C%EC%9D%98-%EB%8F%99%EC%8B%9C%EC%84%B1-%EC%B2%98%EB%A6%AC-5azfj19q)
