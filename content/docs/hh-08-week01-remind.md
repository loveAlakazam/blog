---
title: "항해99 1주차 회고 (WIL)"
date: "2025-03-29"
draft: false
tags: ["항해99 플러스백엔드 8기", "항해99"]
---

## 멘토링하면서 깨달은 점

- 통합테스트/유닛테스트 의 범위는 정해져있지 않다.
- 통합테스트에 대한 최소 기준은 내가 호출할 친구가 통합테스트로 검증되어있는지 확인을 합니다.
  - 단위함수가 아닌 2개이상의 함수나 외부의존성이 있는지
- e2e 관점으로도 통합테스트를 할 수 있다.
  - 비개발자 를 대상으로 내부동작보다는 호출했을때 실질적은 결과를 보여주는 테스트도 괜찮은 편.
  -
- 가까운 대상을 테스트하는게 좋다
- DTO입장에서는 서비스 메서드함수 한개가 통합테스트가 될 수도 있다.
- 단위테스트는 테스트 대상외의것들을 (서비스의 의존성을 비롯한 테스트 대상에는 해당되지않지만 서비스에 연결되어있는 대상들을) Mocking 을 해야한다.
- 객체에게 책임을 부여하면 불필요한 코드를 제외할 수 있다.

---

## 문제 - 이번주차를 지나며 겪었던 문제가 무엇이었나요?

- 빠르게 2일내로 풀어봤지만 멘토링을 받은 이후에 제 코드를 개선이 필요하다는 걸 깨달았습니다.
- 코드개선하면서 깃브랜치를 reset, force push 해야되는 번거로움이 있었습니다 ㅎㅎ;
- 특히 객체에게 책임을 부여하는 부분과 그에대한 테스트코드 수정에서 시간이 많이 소요됐습니다.

## 시도 - 문제를 해결하기 위해 어떤 시도를 하셨나요?

- 각 테스트를 수행후에 데이터를 삭제하게 됐는데요. Reflection 을 이용한 삭제는 안정성과 유지보수측면에서 좋지 않다는 생각이 들었습니다. 심지어 코드도 길구요.

## 해결 - 문제를 어떻게 해결하셨나요?

제가 퍼스트 펭귄이되어, 먼저 팀원들에게 코드리뷰를 부탁했습니다.
코드리뷰 이후에는 팀원들로부터 개인별적으로 의견을 주셨습니다.

## 알게된 것 - 문제를 해결하기 위해 시도하며 새롭게 알게된 것은 무엇인가요?

- '집중중' / '외출중' 등 나의 상태를 상대방이 파악할 수 있도록 공유해놓자.
- 직접 객체생성으로 의존성주입해서 BeforeEach 로 셋팅하면 이전 테스트 데이터에 영향받지 않을 수 있다.
- DirtyContext를 이용하면 스프링컨텍스트를 재호출시켜 이전테스트데이터에 영향을 받지 않을 수 있다.
- 과제제출과 관련된 커밋은 제출직전에 해놓자
- 팀원들끼리 모여서 적극적으로 리뷰를 요청하고 생각을 나누다보니 제가 모르는걸 배우게되었습니다.

## 지난 목표 회고 - 지난주에 설정해두었던 목표는 달성하셨나요? 잘된 것은 무엇이고 안된 것은 무엇인가요?

- 과제의 항목은 아니지만 한발짝 더 나아가서 ReentrantLock 과 syncrhonized 차이에 대한 실험을 테스트를 해봤습니다. 이러한 실험정신이 잘한거 같습니다.
- 잘 안된 것은 너무 잘해보겠다는 욕심으로 밤을 새웠는데 다음날에 컨디션이 안좋더라고요... 역시나 시간관리가 어려웠습니다.

## 다음 목표 설정 - 반복적인 성장을 위한 실천 가능한 단기적인 목표를 설정해보세요

- 초심을 잃지말기 : Comfort Zone에서 벗어나서 나만의 정답을 만들어 나가기
- 너무 밤새워서 번아웃 오지않게 체력관리 잘하기.
- 잠 푹 잘자기
- 잘 듣기고 나만의 언어로 정리후에 가독성있게 글로 정리해보기.
