---
title: "동시성 제어 (Concurrency Methods)"
date: "2025-03-24"
tags: ["동시성제어"]
draft: false
---

## 동시성제어(concurrency control) 정의

### 멀티스레드 관점에서의 동시성 제어

여러스레드가 동시에 공유자원인 프로세스의 데이터, 파일, 힙, 코드 를 접근하는 것을 제어하는 것을 의미합니다.

멀티스레드 환경의 스레드는 메인메모리 로부터 값을 복사하여 CPU 캐시에 저장하여 작업을 합니다.

CPU가 2개 이상이라면 멀티스레드 환경에서 각 스레드는 서로다른 CPU에서 동작하고 있으며 이는 각 스레드가 같은 변수에 대해 읽기/쓰기 동작을 수행할 시 각 CPU 캐시에 저장된 값이 다르기 때문에 변수값 불일치 문제가 발생합니다.

이러한 불일치 문제를 해결하기 위한 방법인 동기화 기법이 필요합니다

**스레드(thread) 와 멀티스레드(multi-threads)**

![multi-threads](../images/multi-threads.png)

- 스레드(thread)는 **프로세스를 구성하는 실행의 흐름 단위** 입니다.
- 하나의 프로세스는 여러개의 스레드를 가질 수 있습니다.
- 스레드를 이용하면 하나의 프로세스에서 여러부분을 동시에 실행할 수 있습니다.

- 스레드는 프로세스 내에서 각기 다른 스레드 id, 프로그램 카운터 값을 비롯한 레지스터 값, 스택으로 구성됩니다.

![multi-threads-consistants](../images/multi-thread-consistants.png)

- 프로세스의 스레드들은 실행에 필요한 최소한의 정보(프로그램 카운터를 포함한 레지스터, 스택)만을 유지한 채 **프로세스 자원을 공유하며 실행**됩니다.

### 데이터베이스 관점에서의 동시성제어 정의

![concurrency-db](../images/concurrency-db.png)

다중 사용자 환경을 지원하는 데이터베이스시스템에서 여러 트랜잭션들이 성공적으로 동시에 실행될 수 있도록 지원하는 기술을 의미합니다. 트랜잭션간의 간섭으로 문제가 발생하지 않도록 트랜잭션의 실행순서를 제어하는 기법입니다. 동시성제어는 실생활에 사용되고 있는 소프트웨어 서비스에서도 많이 활용되고 있습니다.

- 금융서비스에서 금액 송금/저금
- 쇼핑몰서비스에서 주문처리/재고감소/결제요청
- 동시 접속자수가 많은 실시간 앱에서 채팅/댓글/좋아요 수 증가
- 포인트나, 캐시백을 적립/사용
- 이벤트 응모와 선착순쿠폰 발급

---

## 동시성제어를 하지 않으면 어떤 문제가 발생할까?

#### 멀티스레드 관점에서의 문제 - Race Condition

여러 스레드가 동시 다발적으로 임계구역의 코드를 행하여 데이터의 일관성이 깨지는 문제를 의미합니다. 즉 공유자원에 해당되는 데이터가 덮어쓰게 됩니다.

예를들어 우리가 읽고 업데이트할 때 동기화가 없다면 여러 스레드의 인스턴스에서 잘못된 결과가 발생할 수 있습니다.

![race-condition-2](../images/race-condition-2.png)

![race-condition-1](../images/race-condition-1.png)

예를 들어, 스레드A와 스레드B가 count 값을 공유하고 있을 경우입니다.

1단계. 읽기: 메모리에서 현재 count값을 읽습니다.
2단계. 계산: 1단계에서 읽은값이 1씩 증가합니다.
3단계. 쓰기: 메모리에 저장된 count값을 업데이트합니다.

스레드 A에서 읽기 -> 증가 -> 쓰기 작업 순으로 진행후에 스레드B가 작업을 해야되는데
스레드 A의 작업 중 중간에 스레드 B의 개입으로 count 값이 2 로 증가하는게 아닌 1만 증가하게 됩니다.

#### 데이터베이스 관점에서의 문제들

| 문제점                            | 설명                                                                                                                    |
| --------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| **갱신내용 손실(lost update)**    | 트랜잭션들이 동일한 데이터를 동시에 갱신하는 경우에 발생합니다.                                                         |
|                                   | 이전 트랜잭션이 데이터를 갱신한 후에 트랜잭션을 종료하기전에 나중 트랜잭션이 갱신값을 덮어쓰는 경우에 발생합니다.       |
| **현황파악 오류(dirty read)**     | 트랜잭션의 중간 수행결과를 다른 트랜잭션이 참조함으로써 오류가 발생합니다.                                              |
| **모순성(inconsistency)**         | 두 트랜잭션이 동시에 실행할 때 데이터베이스가 일관성이 없는 상태로 남는 문제가 발생합니다.                              |
| **연쇄복귀 (cascading rollback)** | 복수의 트랜잭션이 데이터 공유시 특정 트랜잭션이 처리를 취소할때 다른 트랜잭션이 처리한 부분에 대한 취소가 불가능합니다. |

---

## 동시성제어 제어기법

### 데이터베이스 관점에서 동시성제어에는 어떤 제어기법이 있을까?

#### 잠금(locking)

![locking](../images/locking.png)

잠금(locking)은 하나의 트랜잭션이 실행하는 동안 특정 데이터 항목에 대해서 다른 트랜잭션이 동시에 접근하지 못하도록한 상호배제(mutual exclusive) 입니다. 하나의 트랜잭션이 데이터 항목에 대해서 잠금(lock)을 설정하면 잠금을 설정한 트랜잭션이 잠금해제(unlock)할 때까지 데이터를 독점적으로 사용할 수 있습니다.

- 잠금(lock)의 종류
  | 종류 | 주요 개념 |
  |---- | ---- |
  | **공유락**(shared lock: S-lock)| 공유잠금한 트랜잭션은 데이터 항목에 대해서 읽기(read)만 가능합니다. |
  | | 다른 트랜잭션도 읽기(read)만을 실행할 수 있는 형태 입니다.|
  | **배타락**(exclusive lock: X-lock)|전체잠금한 트랜잭션은 데이터 항목에 대해서 읽기(read)와 쓰기(write) 모두 가능합니다.|
  || 다른 트랜잭션은 읽기(read)와 쓰기(write)를 모두 할 수 없습니다.|

- 비관적락(pessimistic lock)과 낙관적락(optimistic lock)

비관적락과 낙관적락 모두 동시성제어를 위한 대표적인 방법이며 여러개의 트랜잭션이 같은 데이터를 접근할 때 데이터의 일관성을 어떻게 보장할 것인가에 대한 접근방식이기도 합니다.

| 락의 종류                      | 정의                                                                                                                                                                                |
| ------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **비관적락**(pessimistic lock) | 다른트랜잭션이 읽기/쓰기 접근할 수 없도록 미리 Lock을 걸어서 다른트랜잭션이 데이터에 접근하지 못하도록 차단을 합니다.                                                               |
| **낙관적락**(optimistic lock)  | 다른트랜잭션이 읽기/쓰기 접근을 할 수 있지만 저장시점에서 버전정보를 비교해서 변경여부를 판단합니다. 만일 버젼이 달라서 충돌이 발생하면 예외를 발생시켜 저장하지 못하도록 막습니다. |

![locking_unit](../images/locking-unit.png)

잠금의 단위는 **잠금의 대상이 되는 데이터 객체의 크기** 를 의미합니다.
작게는 레코드의 필드값, 하나의 레코드, 물리적 입출력 단위가 되는 디스크블록이 될 수 있으며, 크게는 테이블이나 데이터베이스까지 하나의 잠금의 단위가 될 수 있습니다.

잠금의 단위가 클수록 동시성(병행성) 수준은 낮아지고, 동시성 제어기법은 간단해집니다.
반면에 잠금의 단위가 작을수록 동시성(병행성) 수준은 높아지고, 동시성 제어기법 관리는 복잡해집니다.

상호배제인 잠금은 교착상태(deadlock)가 발생할 수 있습니다.

#### 타임스탬프 순서 기법

트랜잭션을 식별하기 위해 DBMS 가 부여하는 유일한 식별자인 타임스탬프를 지정하여 트랜잭션간의 순서를 미리선택하는 기법입니다. 데이터베이스 시스템에 들어오는 트랜잭션 순서대로 타임스탬프를 지정하여 동시성 제어의 기준으로 사용됩니다.

---

## Java 언어로 가능한 동시성제어 방식은 어떤게 있을까?

| 범주                        | 방법                                   | 설명                                                                                                             |
| --------------------------- | -------------------------------------- | ---------------------------------------------------------------------------------------------------------------- | --- |
| JVM 레벨                    | `synchronized`, `ReentrantLock`        | 한 JVM 내에서만 유효한 락이며, 멀티 인스턴스 환경에서는 무의미하다.                                              |     |
|                             | JVM 내 큐기반 처리 `ConcurrentHashMap` | 동시성 제어를 위한 직렬화 전략으로 여러요청이 한번에 와도 각 요청을 큐에 넣고 하나씩 처리한다.                   |
| DB 레벨                     | 비관적락(Pessimistic Lock)             | 조회시 락을 걸고 다른 트랜잭션을 차단하여 데이터충돌을 미연에 방지한다. 데드락이 발생하여 성능저하가 올 수 있다. |
|                             |                                        | `@Lock(LockModeType.PESSIMISTIC_WRITE)`                                                                          |
|                             | 낙관적락(Optimistic Lock)              | 충돌가능성을 감안하고 수정하며, 충돌이 발생하면 롤백한다                                                         |
| 분산환경                    | Redisson, Redis Lock, Zookeeper 등     | 분산락 구현 가능. 멀티인스턴스 환경에서 락공유가 가능하다.                                                       |
| 비동기식 직렬화 큐기반 처리 | Kafka, RabbitMQ 등으로 직렬화 처리     | 비동기식 직렬화 방식                                                                                             |

- 직렬화(serialization): 객체의 상태를 바이트스트림으로 변환하여 파일이나 네트워크를 통해 전송할 수 있게하는 과정이며, 분산시스템에서도 객체의 상태를 전송하고 수신자측에서는 이를 다시 객체로 복원하여(역직렬화)하여 사용할 수 있습니다.

### 보고서를 작성해보면서 느낀점

동시성 제어와 그와 관련된 운영체제 개념들을 정리를 이해하고나서 동시성제어에 대한 자료조사 아티클들을 읽어보니 조금씩 정리가 되는거 같았습니다. 왜 더 기본기를 중요하다고 하는게 몸으로 깨달았습니다. 앞으로도 이 Lock을 이용한 동시성제어를 깊게 다룰텐데, 여기서 더 확장된 데이터베이스에서의 락, Redis의 분산락, Kafka와 RabbitMQ와 같은 비동기식 큐를 이용한 직렬화에 대한 아티클들을 읽어보고싶습니다.

## 참고자료

- (도서) 혼자공부하는 컴퓨터구조+운영체제
- [동시성 제어 DB 운영](https://i-bada.blogspot.com/2012/04/db_24.html)
- [동시성 제어 개요](http://www.jidum.com/jidums/view.do?jidumId=282)
- [동시성 제어 기법 잠금 locking 기법](https://medium.com/pocs/%EB%8F%99%EC%8B%9C%EC%84%B1-%EC%A0%9C%EC%96%B4-%EA%B8%B0%EB%B2%95-%EC%9E%A0%EA%B8%88-locking-%EA%B8%B0%EB%B2%95-319bd0e6a68a)
