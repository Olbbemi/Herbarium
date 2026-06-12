---
kind: code
id: cpp-concepts
title: C++20 Concepts
summary: 템플릿 타입 파라미터에 컴파일 타임 제약을 이름 붙은 술어로 다는 기능. 만족 못 하면 명확한 에러를 내고 오버로드 해석에 참여한다. SFINAE 대비 가독성과 에러 메시지가 크게 낫다.
category: language-feature
language: cpp
since: C++20
tags: [cpp, cpp20, concepts, templates, constraints, generic-programming]
relations:
  contrasts_with: [sfinae]
  related: [template-metaprogramming, type-traits]
---

## 한 줄 정의

템플릿 타입 파라미터에 컴파일 타임 제약(requirement)을 이름 붙은 술어로 달아, 만족하지 못하면 명확한 에러를 내고 오버로드 해석에도 참여시키는 기능이다.

## 문법 형태

- concept 정의: `template<typename T> concept Name = <bool 컴파일타임 표현식>;`
- requires 표현식: `requires(T a) { a.foo(); { a + a } -> std::same_as<T>; }`
- 적용 방법:
  - `template<Name T>` -- 타입 파라미터 자리에 concept
  - `template<typename T> requires Name<T>` -- requires 절
  - `void f(Name auto x)` -- 축약 함수 템플릿

## 최소 예제

```cpp
#include <concepts>

template<typename T>
concept Addable = requires(T a, T b) {
    { a + b } -> std::same_as<T>;
};

template<Addable T>
T sum(T a, T b) { return a + b; }
```

## 이전 방식과의 차이

```cpp
// before (C++17, SFINAE)
template<typename T, typename = std::enable_if_t<std::is_integral_v<T>>>
T inc(T x) { return x + 1; }

// after (C++20, concept)
template<std::integral T>
T inc(T x) { return x + 1; }
```

- 제약을 못 맞추면 SFINAE 는 난해한 에러, concept 는 "constraint not satisfied" 류의 명확한 에러를 낸다.
- concept 는 이름이 붙어 재사용·문서화가 된다.

## 주의점

- **문법·파싱**
  - `requires` 절(제약 선언)과 `requires` 표현식(요구사항 집합)은 별개다.
  - `requires requires` 가 합법이라 헷갈린다.
- **동작·안전성**
  - `requires` 표현식은 유효성(컴파일 가능 여부)만 검사하고 실행하지 않는다.
  - `{ a + b } -> std::same_as<T>` 는 런타임 값과 무관하다.
- **호환·지원**
  - subsumption(제약 부분순서)은 atomic constraint 단위로 비교한다.
  - 의미가 같아도 표현이 다르면 더 특수하다고 인정 안 될 수 있다.

## 버전·지원

- 표준 concepts 는 `<concepts>` 헤더에 있다.
- GCC 10+ 에서 대체로 동작한다. Clang·MSVC 의 완성도는 버전별 차이가 있다.

## 참고 자료

- cppreference, "Constraints and concepts" -- https://en.cppreference.com/w/cpp/language/constraints
- cppreference, `<concepts>` -- https://en.cppreference.com/w/cpp/concepts
- P0734R0, "Wording Paper, C++ extensions for Concepts"
