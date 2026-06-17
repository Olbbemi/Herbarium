---
id: tdd
title: TDD (Test-Driven Development)
summary: 테스트를 먼저 쓰고 그 실패(RED)를 확인한 뒤 통과시키는 최소 코드를 쓰는 RED-GREEN-REFACTOR 사이클. TDD를 정의하는 것은 테스트의 존재가 아니라 작성 순서와 실패 확인이다.
category: software-engineering
tags: [testing, tdd, red-green-refactor, test-first, software-design]
relations: [contrasts_with:characterization-test, contrasts_with:e2e-testing, same_family:bdd, related:regression-test, related:unit-test, related:test-double]
---

## 핵심

TDD의 본질은 "테스트로 코드를 검증한다"가 아니라 **순서**다. 이 순서가 RED-GREEN-REFACTOR 사이클이다:

1. 테스트를 **먼저** 쓴다.
2. 그 테스트가 **실패하는 것을 눈으로 확인**한다.
3. 통과시키는 최소 코드를 쓴다.

실패(RED)를 먼저 봐야 "이 테스트가 실제로 그 동작을 잡고 있다"는 증거가 생긴다.

## "테스트 기반으로 코드를 만든다"가 빠뜨리는 것

"테스트 기반으로 코드를 만든다" 또는 "코드를 만들면서 테스트도 같이 만들어 green으로 만든다"는 서술은 방향은 맞지만 TDD의 정의로는 불완전하다. 여기엔 **순서**와 **실패 확인**이라는 핵심 두 가지가 빠져 있다.

- 코드를 먼저 쓰고 나중에 테스트를 붙여도 "테스트 기반"이고 결국 green이 된다. 하지만 그건 TDD가 아니다.
- TDD를 TDD로 만드는 건 "테스트를 **먼저** 쓴다(test-first)" + "그 테스트가 **실패하는 걸 본다**(see it fail)"이다.

## RED-GREEN-REFACTOR 사이클

1. **RED**
   - 아직 존재하지 않거나 미완성인 동작에 대한 테스트를 **먼저** 작성하고 실행한다.
   - 테스트는 **실패해야 한다**. 이 실패를 눈으로 확인하는 것이 핵심이다.
2. **GREEN**
   - 그 테스트를 통과시키는 **최소한의** 코드를 작성한다.
   - 멋지게 만들 필요 없이, 통과만 시킨다.
3. **REFACTOR**
   - 테스트가 green인 상태를 유지하면서 코드(및 테스트)의 구조를 정리한다.
   - 테스트가 안전망 역할을 한다.

사이클은 보통 작게(한 동작 단위) 빠르게 돈다.

## 왜 "실패를 먼저 봐야" 하는가

RED 단계를 건너뛰면 테스트 자체가 잘못됐을 위험을 못 잡는다. 흔한 사고:

- 단언(assert)을 빠뜨렸거나 항상 통과하도록 잘못 쓴 테스트
- 대상 동작과 무관한 것을 검증하는 테스트
- 코드가 이미 통과시켜서 "추가됐는지조차 모르는" 테스트

코드를 쓰기 전에 테스트를 돌려 **실패하는 것을 확인**하면, 나중에 green이 됐을 때 "이 테스트가 실제로 방금 만든 동작 때문에 통과한다"는 인과가 보장된다.<br>
즉 RED는 **테스트가 진짜 동작을 잡고 있다는 증거**를 만드는 단계다. 이것이 "코드 먼저, 테스트 나중" 방식과 TDD를 가르는 결정적 차이다.

## TDD가 아닌 것 -- 인접 모드와의 경계

구현 단계에서 테스트로 동작을 보장하는 방식은 TDD 하나가 아니다. 보통 다음 세 모드가 공존하며, 그중 진짜 TDD는 첫 번째뿐이다.

| 모드 | 순서 | 첫 실행 결과 | 핵심 |
|------|------|------------|------|
| **진짜 TDD** | 테스트 -> 코드 | RED(실패) | 실패를 먼저 보고 통과시키는 코드를 쓴다 |
| **회귀(characterization) 먼저** | 코드 -> 테스트 | 바로 GREEN | 이미 존재하는 동작을 테스트로 "박제"한다 |
| **E2E만** | (단위 테스트 없음) | -- | 단위 테스트 없이 실제 바이너리/엔드포인트로만 검증 |

- **회귀(characterization) 테스트**: 코드가 이미 있어 테스트가 처음부터 green이다. RED 단계가 원천적으로 없으므로 TDD가 아니다. 목적은 "현재 동작을 고정해 이후 변경 시 깨지면 알려주기"다.
- **E2E만**: 진입점/배선 지점(예: 진입점에서 의존성을 엮는 코드)처럼 단위 테스트로 잡기 번거로운 부분을 실제 바이너리 실행으로만 검증하는 방식. 단위 수준 RED-GREEN 사이클을 돌지 않는다.

## 용어 정리: 회귀 == regression

"회귀 테스트"와 "regression test"는 같은 말이다(회귀 == regression). 회귀 테스트는 "한때 동작하던 것이 변경 후에도 여전히 동작하는지 다시 확인"하는 모든 테스트를 가리키는 넓은 범주다.<br>
그 안에서 **characterization test**는 "(원래 의도와 무관하게) 현재 코드가 보이는 동작을 그대로 포착해 고정하는" 갈래다. 즉 "회귀 먼저"라고 할 때의 그 테스트는 회귀 테스트의 한 종류인 characterization에 해당한다.

## 흔한 오해 / 반례

> **오해:** "테스트가 있으면 TDD다"
> - 테스트의 존재가 아니라 **작성 순서(test-first)와 실패 확인(RED)**이 TDD를 정의한다.
>
> > **반례:** 코드 작성 후 붙인 테스트는 (유용하더라도) TDD가 아니다.

> **오해:** "green이면 됐다"
> - green만으로는 테스트가 무엇을 잡는지 보장되지 않는다.
>
> > **반례:** 한 번도 RED였던 적 없는 테스트는 항상 통과하는 빈 테스트일 수도 있다.

> **오해:** "TDD는 모든 코드에 적용해야 한다"
> - 단위 테스트 비용이 높은 부분(진입점 배선 등)은 E2E로 가는 게 합리적이다.
> - 기존 레거시는 characterization으로 가는 게 합리적이다.
> - 모드 선택은 대상의 성격에 따른다.

> **오해:** "REFACTOR는 선택"
> - REFACTOR는 사이클의 정식 단계다.
> - green 안전망 위에서 구조를 정리하는 단계를 생략하면 누적 부채로 사이클 속도가 떨어진다.

## 참고 자료

- Kent Beck, *Test-Driven Development: By Example* (2002) -- RED-GREEN-REFACTOR 원전
- Martin Fowler, "TestDrivenDevelopment" -- https://martinfowler.com/bliki/TestDrivenDevelopment.html
- Michael Feathers, *Working Effectively with Legacy Code* (2004) -- characterization test 개념의 출처
- "Characterization test" -- https://en.wikipedia.org/wiki/Characterization_test
