---
title: "정적 팩토리 메소드"
date: "2025-04-13"
draft: true
---

## 서론

단위테스트의 목데이터를 만들때 생성자를 호출하게되어 연관관계를 갖는 도메인을 setter로 설정시켜서 목데이터를 생성하는데 점점 무거워지는 불편함을 겪었습니다.

```java
@Test
void 신규_임시예약상태의_예약도메인_생성에_성공된다() {
    // given
    long userId = 1L;
    long concertSeatId = 1L;
    User user = new User(userId, "은강");

    Concert concert = new Concert(1L, "항해와 함께하는 TDD 단위테스트 콘서트", "항해플러스"); // 👈 도메인 엔티티 생성자를 직접호출!
    ConcertDate concertDate = new ConcertDate(1L, LocalDate.now(), true, "뚝섬 한강공원"); // 👈 연관관계 도메인엔티티 생성자를 직접호출!
    ConcertSeat concertSeat = new ConcertSeat(concertSeatId, 5, 15000, true); // 👈 연관관계 도메인엔티티 생성자를 직접호출!
    concertSeat.setConcert(concert); // 👈 문제의 setter!
    concertSeat.setConcertDate(concertDate); // 👈 문제의 setter!

    // when & then
    Reservation result = assertDoesNotThrow(() -> reservationService.initTemporaryReservedStatus(user, concertSeat));
    assertEquals(null, result.getReservedAt());
    assertEquals(PENDING_PAYMENT, result.getStatus());
    assertEquals(user, result.getUser());
    assertEquals(concert, result.getConcert());
    assertEquals(concertDate, result.getConcertDate());
    assertEquals(concertSeat, result.getConcertSeat());
}
```

목데이터를 나타낼때마다 뚱뚱해지는 코드길이와, 목데이터를 셋팅하는데 계속 new 연산자로 생성자를 호출해야되는걸까? 라는 의구심이 들었습니다.
엔티티와 연결된 도메인이 1개면 생성자호출,setter 로 코드라인 2줄을 차지하는데, 만일 연결된 도메인이 10개라면 코드라인수가 20개를 차지해서 테스트코드를 작성하는데 큰 불편함을 겪었습니다. 만일 위의 문제의 코드처럼 진행된다면 연결 도메인개수가 늘어날수록 작성해야되는 테스트코드도 방대해진다는 결론을 내렸습니다.

이에 대한 내용으로 코치님께 리뷰포인트로 질문을 했고 그에 대한 답변을 받았습니다.
도메인 모델이 자신의 정보를 관리하지 않고 외부에서 변경이 가능할 경우 책임의 경계가 바깥까지 확장될 수 있기 때문에 "정적 팩토리 메서드" 로 접근을 하는게 좋다고 하셨습니다.

하지만 저는 '정적 팩터리 메서드' 가 무엇인지 들어봤지만 그 용어의 정의를 모른 상태였습니다.
직접 검색을하며 학습을 할 수 밖에 없었습니다.

---

## 정적 팩토리 메소드

정적 팩토리 메서드(Static Factory Method)는 객체의 생성을 담당하는 클래스 메서드 입니다.
`new` 연산자를 직접 사용하지 않고도 클래스 내에서 선언되어 있는 메서드를 통해서 내부의 `new` 연산자를 이용하여 객체를 생성해서 반환하는 것을 의미합니다.

저는 이펙티브 자바 책을 읽지 않았지만, 참고블로그의 작성자는 이펙티브 자바 도서를 근거하여
new 연산자를 사용할 때보다 정적 팩토리 메서드를 사용할 때 얻게되는 장점이 있습니다.

- 1. 이름을 가질 수 있다.
- 2. 호출 될때마다 인스턴스를 새로 생성하지 않아도 된다.
- 3. 반환 타입의 하위 타입 객체를 반환할 수 있는 능력이 있다.
- 4. 입력 매개변수에 따라 매번 다른 클래스의 객체를 반환할 수 있다.
- 5. 정적 팩토리 메서드를 작성하는 시점에는 반환할 객체의 클래스가 존재하지 않아도 된다.

### (1) 이름을 가질 수 있다

객체지향 프로그래밍에서는 객체는 자신에게 주어진 행동, 역할, 책임을 갖고 있습니다.
객체 자신의 역할을 수행하기 위해서 객체 생성시 역할 수행이라는 생성 목적에 따라 생성자를 구별해서 사용합니다.
이때 new 연산자를 이용하여 객체를 생성하게 되면 프로그래머는 해당 생성자의 내부 구조를 알고 있어야 목적에 맞게 객체를 생성할 수 있습니다.

하지만 정적팩토리 메소드를 사용하게되면 메소드 네이밍에 따라 반환될 객체들의 특성을 묘사할 수 있습니다.
메소드명에 따라 코드의 가독성이 상승하는 장점을 갖습니다.

### (2) 호출할 때 마다 새로운 객체를 생성할 필요가 없다

생성자를 통해서 객체를 만드는 경우 초기화 값이 동일하더라도 인스턴스 주소가 전부 다른 현상이 있습니다.
즉 호출할 때마다 매번 서로다른 인스턴스 주소를 가진 객체가 생성됩니다.

정적 팩토리 메소드를 사용하게되면 여러번 호출하더라도 동일한 인스턴스 주소로 나타낼 수 있습니다.
즉 정적팩토리 메소드와 캐싱구조를 함께 사용하면 매번 객체를 새롭게 만들 필요가 없어집니다.

### (3) 반환 타입의 하위 타입 객체를 반환할 수 있는 능력이 있다

생성자를 사용하게되면 생성되는 객체의 클래스가 하나로 고정됩니다. 하지만 정적 팩토리 메소드를 사용하게되면 반환할 객체의 클래스를 자유롭게 선택할 수 있는 유연성을 가지게됩니다. 정적 팩토리 메소드를 사용하게되면 객체를 생성할때 분기처리를 통해서 하위 타입의 객체를 반환할 수 있습니다.

### (4) 입력 매개변수에 따라 매번 다른 클래스의 객체를 반환 할 수 있다

인스턴스를 서로 다르게 반환하는 기능을 추가하고 싶다면, 매개변수에 따라 다른 클래스의 인스턴스를 반환해주면 됩니다.

### (5) 정적 팩토리 메서드를 작성하는 시점에는 반환할 객체의 클래스가 존재하지 않아도 된다.

서비스 제공자 프레임워크와 JDBC 를 만드는 근간이기도하고, 구현체가 전혀 존재하지 않아도 인터페이스만으로 정적 팩토리 메소드를 작성할 수 있습니다.

### 정적 팩토리 메소드의 단점

- 1. 상속에는 public 혹은 protected 생성자가 필요하므로 정적 팩토리 메소드만 제공할 경우 상속이 불가능하다.
- 2. 다른 개발자들이 정적팩토리 메소드를 찾기가 어렵다

### 정적 팩토리 메소드 네이밍

단점2를 보완하기 위해서 널리 알려진 규약을 통해서 정적 팩토리 메소드를 명명하는 것이 좋습니다. 이펙티브 자바에서 소개하는 정적 팩토리 메소드 네이밍 컨벤션은 아래와 같습니다.

1. **from**

매개변수 하나를 받아서 해당 타입의 인스턴스를 반환하는 형변환 메소드

```java
Date d = Date.from(instant);
```

2. **of**

여러 매개변수를 받아서 적합한 타입의 인스턴스를 반환하는 집계 메소드

```java
Set<Rank> pockerCards = EnumSet.of(JACK, QUEEN, KING);
```

3. **valueOf**

`from` 과 `of` 의 더 자세한 버젼

```java
BigInteger prime = BigInteger.valueOf(Integer.MAX_VALUE);
```

4. **instance** 혹은 **getInstance**

```java
StackWalker luke = StackWalker.getInstance(options);
```

5. **create** 혹은 **newInstance**

instance 혹은 getInstance 와 비슷하지만, 매번 새로운 인스턴스를 생성하여 반환함을 보장합니다.

```java
Object newArray = Array.create(classObject, arrayLen);
```

6. **getType**

getInstance 와 같으나, 현재 클래스가 아닌 다른 클래스의 인스턴스를 생성할 때 사용합니다.
Type은 팩토리 메소드가 반활할 객체 타입을 적습니다.

```java
FileStore fs = Files.getFileStore(path);
```

7. **newType**

createInstance 와 같으나, 현재 클래스가 아닌 다른 클래스의 인스턴스를 생성할 때 사용합니다.
Type은 팩토리 메소드가 반활할 객체 타입을 적습니다.

```java
BufferedReader br = Files.newBufferedReader(path);
```

8. **type**

getType 과 newType 의 간결한 버젼입니다.

```java
List<Complaint> litany = Collections.list(legacyLitany);
```

- 참고
  - [cjh8746.log - 정적 팩토리 메소드](https://velog.io/@cjh8746/%EC%A0%95%EC%A0%81-%ED%8C%A9%ED%86%A0%EB%A6%AC-%EB%A9%94%EC%84%9C%EB%93%9CStatic-Factory-Method)
  - [Abel - 정적 팩토리 메소드](https://tjdtls690.github.io/studycontents/java/2023-01-30-static_factory_method_01/)
  - [Hudi - 정적 팩토리 메소드](https://hudi.blog/effective-java-static-factory-method/)
  - [인파 - 정적 팩토리 메소드](https://inpa.tistory.com/entry/GOF-%F0%9F%92%A0-%EC%A0%95%EC%A0%81-%ED%8C%A9%ED%86%A0%EB%A6%AC-%EB%A9%94%EC%84%9C%EB%93%9C-%EC%83%9D%EC%84%B1%EC%9E%90-%EB%8C%80%EC%8B%A0-%EC%82%AC%EC%9A%A9%ED%95%98%EC%9E%90)

---

## 정적 팩토리 메소드를 사용하여 코드를 개선해보자

```

```

---

## 결론

객체는 각자 책임과 역할을 갖고 있습니다. 목적에 맞게 객체의 책임과 역할을 활용하려면 객체의 생성이 필요합니다.
하지만 생성자를 호출하게되면 서론에서 소개한 문제처럼 객체가 필요할 때마다 `new` 연산자로 생성자를 직접 호출하게되면 생성자 호출로인해서 코드의 가독성이 떨어지게되는걸 직접 경험해봤습니다.

각 도메인이 갖는 책임과 역할을 객체지향스럽게 해야된다면 이펙티브자바 도서를 한번쯤은 읽어보고, 읽은 내용을 바탕으로 예제코드를 작성하면서 나만의 언어로 정의하는 연습을 해봐야될 거 같습니다.
