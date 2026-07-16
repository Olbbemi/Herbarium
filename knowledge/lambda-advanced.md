---
kind: code
id: lambda-advanced
title: 제네릭 람다와 캡처 규칙
summary: |
  매개변수를 auto 로 받아 인자 타입마다 별개로 인스턴스화되는 람다다.
  캡처 절로 주변 스코프의 변수와 this 를 끌어온다.
  std::variant 를 std::visit 로 처리할 때 각 대안을 하나의 auto 람다로 일반화하는 데 자주 쓴다.
category: language-feature
language: cpp
since: C++14
tags: [cpp, cpp14, lambda, generic-lambda, capture, std-visit, std-variant]
relations: [contrasts_with:overload-pattern, related:optional-variant-any, related:if-constexpr, related:init-capture]
---

## 한 줄 정의

매개변수를 `auto` 로 받아 호출 시점의 인자 타입마다 별개로 인스턴스화되는 람다(제네릭 람다)다.  
캡처 절로 주변 스코프의 변수와 `this` 를 끌어온다.  
`std::variant` 를 `std::visit` 로 처리할 때 각 대안 타입을 하나의 `auto` 람다로 일반화 처리하는 데 자주 쓴다.

## 문법 형태

```
[캡처](auto 매개변수) -> 반환타입 { 본문 }
```

- 캡처 절 갈래:
  - `[]` -- 아무것도 캡처 안 함
  - `[x]` / `[&x]` -- 변수 x 를 값/참조로 캡처
  - `[=]` / `[&]` -- 쓰인 변수 전부를 값/참조로 암시 캡처
  - `[this]` -- 현재 객체의 포인터를 캡처(멤버를 `this->` 로 접근)
  - `[*this]` -- 현재 객체를 값으로 복사 캡처 (C++17~)
- `auto` 매개변수가 제네릭 람다의 핵심이다.
  - 컴파일러가 람다를 "템플릿 `operator()` 를 가진 익명 함수 객체"로 만든다.
  - 인자 타입마다 별도 실체가 생긴다(컴파일 타임, 런타임 다형성 아님).
- 반환 타입(`-> T`)은 생략하면 추론된다.
  - 단 `std::visit` 처럼 모든 분기의 반환 타입이 하나로 합쳐져야 하는 경우 명시하면 안전하다.

## 최소 예제

캡처 없는 제네릭 람다로 variant 의 각 대안을 일반화 처리(반환 타입 명시):

```cpp
// value 는 std::variant<...>
auto tag = std::visit([](const auto& alt) -> std::string_view {
    using T = std::decay_t<decltype(alt)>;
    if constexpr (std::is_same_v<T, Ping>) return "ping";
    else if constexpr (std::is_same_v<T, Say>) return "say";
    else return "other";
}, value);
```

`[this]` 로 현재 객체를 캡처해 각 대안을 멤버 오버로드로 위임:

```cpp
// handle 은 대안 타입별 오버로드 집합
return std::visit([this](const auto& alt) { return handle(alt); }, value);
```

`const auto& alt` 한 줄이 `Ping`, `Say`, `Quit` 각각에 대해 `handle(const Ping&)`, `handle(const Say&)`, `handle(const Quit&)` 를 오버로드 해소로 골라 부른다.  
대안이 늘어도 람다는 그대로다.

## 이전 방식과의 차이

- C++11 람다는 매개변수 타입을 반드시 명시해야 했다.
  - variant 대안마다 다른 타입을 받으려면 대안 수만큼 람다(또는 함수 객체 오버로드)를 따로 써야 했다.
- C++14 의 `auto` 매개변수(제네릭 람다)로 대안 타입을 하나의 람다가 일반 처리한다.
- 대안마다 **다른** 동작이 필요하면 제네릭 람다 대신 "overload 패턴"을 쓴다.
  - 여러 람다를 상속으로 묶어 오버로드 집합을 만드는 관용구다.
  - 위 예제는 `if constexpr` 분기 또는 멤버 오버로드 위임으로 같은 효과를 낸다.

```cpp
// 참고: overload 패턴 (대안마다 다른 람다)
template <class... Fs> struct overload : Fs... { using Fs::operator()...; };
template <class... Fs> overload(Fs...) -> overload<Fs...>;  // C++17 CTAD

std::visit(overload{
    [](const Ping& p) { /* ... */ },
    [](const Say&  s) { /* ... */ },
    [](const Quit&)   { /* ... */ },
}, value);
```

## 주의점

- **수명·소유권**
  - `[this]` 는 객체 포인터를 캡처한다 -- 람다가 원래 객체보다 오래 살면(비동기 콜백, 저장된 `std::function` 등) dangling `this` 가 된다.
  - 객체를 복사해 안전하게 들고 가려면 `[*this]`(C++17~)를 쓴다.
  - 방문 결과로 `std::string_view`·참조·포인터를 반환할 때, 분기 안의 지역 임시를 가리키면 반환 즉시 dangling 이다.
  - `string_view` 반환은 대안이 들고 있는 문자열이나 리터럴처럼 방문 후에도 살아있는 대상만 가리켜야 한다.
- **동작·안전성**
  - `std::visit` 는 모든 대안의 방문 결과가 하나의 공통 반환 타입이어야 한다 -- 분기마다 다르면 컴파일 에러다.
  - 첫 예제처럼 `-> std::string_view` 로 못박으면 각 분기가 그 타입으로 변환된다.
  - variant 가 valueless_by_exception 상태면 `std::visit` 는 `std::bad_variant_access` 를 던진다.
- **문법·파싱**
  - 제네릭 람다는 인자 타입마다 `operator()` 가 인스턴스화된다 -- 한 분기라도 그 타입에 컴파일 안 되면(특정 대안엔 없는 멤버 접근 등) 에러다.
  - `if constexpr` 로 타입별 분기를 컴파일 타임에 잘라내야 한다. 일반 `if` 는 모든 가지가 컴파일돼야 하므로 부족하다.

## 흔한 오해 / 통념 정정

> **오해:** "`[=]` 로 캡처하면 멤버 변수가 값으로 복사된다."
> - `[=]` 는 멤버를 직접 복사하지 않고 `this` 포인터를 캡처한다.
> - 멤버 접근은 실제로는 `this->멤버` 라 원본 객체를 참조한다.
> - 멤버 자체를 복사하려면 `[*this]`(C++17) 또는 `[member = 멤버]` 초기화 캡처를 쓴다.
> - `[=]` 를 통한 `this` 암시 캡처는 C++20 에서 deprecated 다 -- 필요하면 `[=, this]` 로 명시한다.

> **오해:** "제네릭 람다는 런타임 다형성(가상 함수 비슷한 것)이다."
> - `auto` 매개변수는 **템플릿**으로 풀린다.
> - 타입 결정과 오버로드 해소는 전부 컴파일 타임이며 가상 디스패치가 아니다.
> - `std::visit` 이 하는 variant 대안 선택만이 런타임 요소다.

## 버전·지원

- 제네릭 람다(`auto` 매개변수): C++14~
- 초기화 캡처(`[x = expr]`): C++14~
- `[*this]` 복사 캡처: C++17~
- `std::variant` / `std::visit`: C++17~
- 템플릿 매개변수 람다(`[]<class T>(T x){}`): C++20~
- `[=]` 를 통한 `this` 암시 캡처 deprecated: C++20 (P0806R2)
- 위 최소 예제·전체 예제는 `-std=c++17` 이상으로 컴파일한다.

## 참고 자료

- cppreference, Lambda expressions -- <https://en.cppreference.com/w/cpp/language/lambda>
- cppreference, std::visit -- <https://en.cppreference.com/w/cpp/utility/variant/visit>
- cppreference, std::variant -- <https://en.cppreference.com/w/cpp/utility/variant>

## 전체 예제

빌드: `g++ -std=c++17 lambda_advanced.cpp -o lambda_advanced` (또는 `clang++ -std=c++17`).

```cpp
#include <variant>
#include <string>
#include <string_view>
#include <type_traits>
#include <iostream>

// 예제용 대안 타입들
struct Ping { int seq; };
struct Say  { std::string text; };
struct Quit {};

using Message = std::variant<Ping, Say, Quit>;

// (1) 캡처 없는 제네릭 람다 + 반환 타입 명시 + if constexpr 분기
std::string_view tag_of(const Message& msg) {
    return std::visit([](const auto& alt) -> std::string_view {
        using T = std::decay_t<decltype(alt)>;
        if constexpr (std::is_same_v<T, Ping>) return "ping";
        else if constexpr (std::is_same_v<T, Say>) return "say";
        else return "quit";
    }, msg);
}

// (2) [this] 캡처 제네릭 람다 -> 멤버 오버로드로 위임
class Router {
public:
    std::string route(const Message& msg) {
        return std::visit([this](const auto& alt) { return handle(alt); }, msg);
    }

private:
    std::string handle(const Ping& p) { return prefix() + "ping " + std::to_string(p.seq); }
    std::string handle(const Say& s)  { return prefix() + "say " + s.text; }
    std::string handle(const Quit&)   { return prefix() + "quit"; }

    std::string prefix() const { return "[router] "; }
};

int main() {
    Router r;
    Message m1 = Ping{7};
    Message m2 = Say{"hi"};
    Message m3 = Quit{};

    std::cout << tag_of(m1) << " | " << r.route(m1) << "\n"; // ping | [router] ping 7
    std::cout << tag_of(m2) << " | " << r.route(m2) << "\n"; // say | [router] say hi
    std::cout << tag_of(m3) << " | " << r.route(m3) << "\n"; // quit | [router] quit
}
```
