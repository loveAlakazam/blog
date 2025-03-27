---
title: "객체의 책임(도메인 주도)"
date: "2025-03-26"
draft: false
---

## 목적

- UserPointService 의 충전/사용 메서드를 리팩터링한다
- 절차지향적인 코드를 객체지향적으로 변경해보자
- 냄새나는 코드?

## 포인트 충전 로직 개선시키기

### 수정 이전 (절차지향적)

- 포인트 충전 내부로직 플로우

```java
// PointServiceImpl.java

@Service
@RequiredArgsConstructor
public class PointServiceImpl implements PointService {
    @Override
    public ChargeResponse charge(ChargeRequest request)
    {
        long id = request.id();
        long amount = request.amount();

        UserPoint userPoint = this.userPointRepository.findById(id);
        long myPoint = userPoint.point();

        // 포인트내역에 '충전' 기록
        this.pointHistoryRepository.insert(id, amount, TransactionType.CHARGE);

        // 포인트 충전 -> 포인트 업데이트
        UserPoint result = this.userPointRepository.save(id, myPoint + amount);
        return ChargeResponse.from(result);
    }
}
```

### 수정 이후 (객체지향적)

- 1. 포인트 에게 '충전' 역할을 부여하기

```java
public record UserPoint(
        long id,
        long point,
        long updateMillis
) {

    public static UserPoint empty(long id) {
        return new UserPoint(id, 0, System.currentTimeMillis());
    }

    /**
     * UserPoint 정책
     *
     * - MIN_CHARGE_POINT_AMOUNT: 최소 포인트 충전금액
     * - MAX_CHARGE_POINT_AMOUNT: 최대 포인트 충전금액
     *
     * - MIN_USE_POINT_AMOUNT: 최소 포인트 사용금액
     * - MAX_USE_POINT_AMOUNT: 최대 포인트 사용금액
     */
    public static int MIN_CHARGE_POINT_AMOUNT = 1000;
    public static int MAX_CHARGE_POINT_AMOUNT = 50000;

    public static int MIN_USE_POINT_AMOUNT = 100;
    public static int MAX_USE_POINT_AMOUNT = 50000;


    // 포인트 충전에 대한 정책
    public UserPoint charge(long amount) {
        long point = this.point + amount; // 포인트 충전
        return new UserPoint(id, point, updateMillis); // 충전된 값으로 신규 인스턴스를 리턴
    }
}
```

UserPoint가 클래스라면 set으로 데이터를 변경하여 수정할 수 있습니다.
하지만 record 클래스는 데이터를 읽을 수 있지만, 데이터변경을 금지하기 때문에 set 이 허용하지 않기때문에 포인트충전된 값으로 포인트 객체를 생성하여 리턴하였습니다.

- 2. 포인트 충전 로직 개선하기

```java
// PointServiceImpl.java

@Service
@RequiredArgsConstructor
public class PointServiceImpl implements PointService {
 @Override
 public ChargeResponse charge(ChargeRequest request) {
  long id = request.id();
  long amount = request.amount();

  // 보유 포인트 조회
  UserPoint myPoint = this.userPointRepository.findById(id);

  // 포인트 계산 및 충전 (객체의 책임 부여)
  myPoint.charge(amount);

  // 포인트내역에 '충전' 기록
  this.pointHistoryRepository.insert(id, amount, TransactionType.CHARGE);

  // 보유포인트 정보 수정
  UserPoint result = this.userPointRepository.save(id, myPoint.point());

  // 응답DTO로 감싸서 리턴
  return ChargeResponse.from(result);
 }
}
```

---

## 포인트 사용 로직 개선시키기

### 수정 이전 (절차지향적)

- 포인트 사용 내부로직 플로우

  > 1. `보유 포인트 조회(db)`
  > 2. `보유포인트와 사용포인트 비교`
  > 3. `포인트 내역 기록(db)`
  > 4. 보유포인트- 사용포인트 계산
  > 5. `보유포인트 업데이트(db)`
  >
  > - (db) 는 의존성인 래포지토리의 메서드를 이용

- 문제점 분석

- `this.userPointRepository.save(id, myPoint - amount)` : '사용' 메서드안에 포인트사용 이 포함되어있습니다.

- UserPoint 와 관련된 역할이 객체인 UserPoint가 책임을 지지않고 PointService의 use()함수가 책임을 지고 있습니다.

```java
// PointServiceImpl.java

@Service
@RequiredArgsConstructor
public class PointServiceImpl implements PointService {
 @Override
 public UseResponse use(UseRequest request) {
  long id = request.id();
  long amount = request.amount();

  UserPoint userPoint = this.userPointRepository.findById(id);
  long myPoint = userPoint.point();

  // 잔고가 사용금액보다 부족할 경우
  if( amount > myPoint ) {
   throw new CustomBusinessException(ErrorCode.OVER_USE_AMOUNT_VALUE_THAN_BALANCE_POLICY);
  }

  // 포인트내역에 '사용' 기록
  this.pointHistoryRepository.insert(id, amount, TransactionType.USE);

  // 포인트 사용
  UserPoint result = this.userPointRepository.save(id, myPoint - amount);
  return UseResponse.from(result);
 }
}
```

### 수정 이후 (객체지향적)

- 어떻게 개선할건가?

  > 1. `보유 포인트 조회(db)`
  > 2. ((객체의 책임)) `보유포인트와 사용포인트 비교` -> 보유포인트 - 사용포인트 계산 후 신규포인트를 리턴
  > 3. `포인트 내역 기록(db)`
  > 4. `보유 포인트 업데이트(db)`
  >
  > - (db) 는 의존성인 래포지토리의 메서드를 이용

- 1. UserPoint

```java
public record UserPoint(
        long id,
        long point,
        long updateMillis
) {

    ...(생략)...

    // 포인트 사용에 대한 정책
    public UserPoint use(long amount) {
        // 잔고가 사용금액보다 부족할 경우 CustomBusinessException 예외처리
        if(amount > this.point) {
            throw new CustomBusinessException(ErrorCode.OVER_USE_AMOUNT_VALUE_THAN_BALANCE_POLICY);
        }
        long point = this.point - amount; // 포인트 사용
        return new UserPoint(id, point, updateMillis);
    }
}

```

- 2. UserPointServiceImpl

```java
// PointServiceImpl.java

@Service
@RequiredArgsConstructor
public class PointServiceImpl implements PointService {
    @Override
    public UseResponse use(UseRequest request) {
        long id = request.id();
        long amount = request.amount();

        UserPoint myPoint = this.userPointRepository.findById(id);

        // 포인트 사용
        myPoint = myPoint.use(amount);

        // 포인트내역에 '사용' 기록
        this.pointHistoryRepository.insert(id, amount, TransactionType.USE);

        // 보유 포인트 수정
        UserPoint result = this.userPointRepository.save(id, myPoint.point());

                // 응답 DTO로 감싸서 리턴
        return UseResponse.from(result);
    }
}
```

- 3. 테스트코드 수정

```java
@Test
 void 보유포인트_10000원에_5000원_충전하면_잔액은_15000원이된다() {
  // given
  long id = 1L;
  long initialPoint = 10000L; // 초기 보유 잔액
  long amount = 5000L; // 충전금액

  UserPoint myPoint = new UserPoint(id, initialPoint, 100L);
  long pointAfterCharged = myPoint.charge(amount); // 객체에게 책임을 부여

  when(userPointRepository.findById(id)).thenReturn(myPoint); // 충전전 보유잔액
  when(userPointRepository.save(id, pointAfterCharged))
   .thenReturn(new UserPoint(id, pointAfterCharged, myPoint.updateMillis())); // 충전후 보유잔액


  // when
  ChargeResponse response = pointService.charge(new ChargeRequest(id, amount)); // 포인트 5000원 충전 수행

  // then
  assertEquals(myPoint.charge(amount), response.point()); // 충전후 예상값과 실제값 비교
  verify(pointHistoryRepository, times(1)).insert(id, amount, TransactionType.CHARGE); // 포인트내역 insert 호출검증
  verify(userPointRepository, times(1)).save(id, pointAfterCharged );  // save(포인트정보 수정) 호출검증
 }
```

## 결론 및 내 생각

객체지향적이라는 것은 마치 객체가 살아움직이는 것처럼 능동적으로 자기역할에 대해서 책임을 지는 것을 의미한다고 생각합니다.

그러므로 객체에 대한 역할을 객체에게 부여시키는 연습을 해봤고, 이를 통해 코드라인이 줄어들었음을 확인할 수 있게되었습니다.

객체에게 역할을 부여하고 그 역할에 대한 책임을 지게끔하는게 객체지향이면서 도메인주도개발(DDD)이 아닌가 싶습니다.

코드가 변경되면 테스트코드도 같이 변경되어야합니다.
그래서 코드도 리팩터링할때도 코드의 변화가 있다면 테스트코드도 알맞게 변경해야됩니다.

코드를 예쁘게 짤려고하다보면 계속 고민을 하게되는거 같습니다.
처음에는 잘 안보이다가도 다른팀원들과 의논을 나누게되면 코드를 변경하게되는데...

요즘든 생각은 DTO클래스내부에서 직접 유효성검사를 하기보다는 비즈니스 정책과 관련되어있는 (예를들어 UserPoint 클래스의 `id값은 양수여야한다`, `amount값은 최소금액은 XX원 이상이어야한다` 등에 대한) 유효성검사 검증은 UserPoint객체에서 책임함수를 따로 만들어야할까? 라는 생각이 듭니다.

```java
public record UserPoint(
        long id,
        long point,
        long updateMillis
) {

    ...(생략)...

    // 유효성검사에 대한 정책
    // 1. id에 대한 정책
    public static void validateId(long id) {
        if(id <= 0) throw new CustomInvalidRequestException("id는 양수여야합니다.");
    }

    // 2. 충전 금액의 파라미터 인자 amount에 대한 정책
    public static void validateChargeAmount(long amount) {
        if(amouont <= 0 ) throw new CustomInvalidRequestException("amount는 양수여야합니다.");
        if(amount <= MIN_CHARGE_POINT_AMOUNT) throw new CustomInvalidRequestException("amount는 최소 1000원 이상이어야 합니다.");
        if(amount > MAX_CHARGE_POINT_AMOUNT) throw new CustomInvalidRequestException("amount 최댓값은 50000원 입니다.");

    }

    ...
}
```

첫번째로는 비즈니스 정책만큼 가장중요한게 엔티티 객체입니다.
중요한 엔티티에게 관련 비즈니스 정책을 부여해주는 거죠.

두번째로는 이러한 책임의 부여는 변경할 가능성이 높고, 심지어 정책이 사라질 수 있는 변경에 대비할 수 있습니다.

정책과 관련된 책임을 엔티티에게 주게되면, 즉 객체지향적으로 코드를 작성하게되면 절차지향적으로 했을 때 코드로 했을 때보다 불필요한 코드를 줄일 수 있다는걸 경험했습니다.
