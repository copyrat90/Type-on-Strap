---
layout: post
title: C++20 concepts 써보기
tags: [C++]
author: copyrat90
last_modified_at: 2024-07-25T12:00:00+09:00
---

C++에서 템플릿 관련 컴파일 에러는 읽기 힘들기로 악명이 높다.\
(1줄만 고치면 되는 문제인데 오류 메시지가 1000줄이 되는 바람에 터미널에서 잘려서 오류 메시지를 읽지 못하는 경험도 해봤다... -_-)

C++20에 도입된 concepts를 이용하면, 템플릿 제약조건을 만족하지 못하는 함수/클래스에 대해서 에러 메시지를 좀 더 읽기 좋게 출력하도록 만들 수 있다.\
또한, 코드를 사용하는 입장에서도 어떤 제약조건을 만족해야 하는지 명시적으로 문서화되는 효과가 있다.

그러니, 한번 써보자.

# 콘셉트 *(concepts)*
> 클래스 템플릿이나 함수 템플릿의 매개변수를 제한하는 데 사용되는 요구사항을 정의

## 문법

### 콘셉트 정의 *(concept definition)*

> 템플릿 매개변수로 전달된 타입에 대한 제약조건에 이름을 붙여 정의하는 것

```cpp
template <parameter-list>
concept concept-name = constraints-expression;
```

* `constraints-expression`
  * `bool`로 평가될, 컴파일 타임 상수로 평가될 표현식을 여기에 적으면 된다.

#### 예시

```cpp
// `T`가 열거형인지 판별하는 `Enum` 콘셉트를 정의
template <typename T>
concept Enum = std::is_enum<T>::value;
```

### 콘셉트 표현식 *(concept expression)*

> 앞에서 정의한 콘셉트를 실제로 사용한 표현식

```cpp
concept-name<argument-list>
```

* 콘셉트 표현식은 `true`나 `false`로 평가된다.

#### 예시

```cpp
enum class MyFruit { APPLE, BANANA, MELON };
class MyClass {};

// 앞 예시에서 정의한 `Enum` 콘셉트를 사용
static_assert(Enum<MyFruit>);
static_assert(!Enum<MyClass>);
```

## 제약 표현식 *(constraints expression)*

앞서 말한 제약 표현식 자리에는 반드시 `bool`로 평가될, 컴파일 타임 상수 표현식을 적어야 한다.\
그런데, 요구사항을 `bool`을 반환하는 표현식으로 어떻게 적을 수 있을까?

예를 들어, 어떤 클래스에 멤버 함수 `unique_id()`가 존재하고, 그 멤버가 `std::size_t`를 반환하는 것을 검증하는 콘셉트 `UIDSizeT`를 정의하고 싶다면?

물론 C++11 식으로 `<type_traits>` 헤더에 존재하는 템플릿 메타프로그래밍 API를 이용해 해결할 수도 있을 것이다.

```cpp
// `T`가 `unique_id()` 멤버함수를 가지며, 그 반환형은 `std::size_t`인지 검증하는 콘셉트
template <typename T>
concept UIDSizeT = std::is_same<decltype(std::declval<T>().unique_id()), std::size_t>::value;
```

근데... 필자만 그런지 모르겠지만, 정말 정말 보기 싫게 생겼다.

다행히 이것보다 깔끔하게 요구사항을 표현하는 *요구 표현식 (requires expression)*도 콘셉트와 함께 추가됐다.


### 요구 표현식 *(requires expression)*

> 요구사항을 만족하는지 여부를 `bool`로 반환하는 표현식

```cpp
requires (parameter-list) { requirements; }
```

* `parameter-list` : 매개변수, 생략 가능하다.
* `requirements` : 요구사항, 여러 개 있다면 세미콜론으로 구분해 적어야 한다.

`requirements`에 적을 수 있는 요구사항은 [단순*(simple)*, 타입*(type)*, 복합*(compound)*, 중첩*(nested)*의 4가지로 나뉜다.](https://en.cppreference.com/w/cpp/language/requires#Requirements)

#### 1. 단순 요구사항 *(simple requirements)*

> `requires`로 시작하지 않는 단순한 표현식

* 표현식이 유효한지만 컴파일러가 검증한다.

##### 예시

```cpp
template <typename T>
concept Addable = requires (T a, T, b) {
    a + b; // 단순 표현식의 예시; a와 b 사이의 덧셈 연산이 유효한지 체크됨
};
```

#### 2. 타입 요구사항 *(type requirements)*

> `typename` 으로 해당 타입이 존재할 수 있는지 검증

##### 예시

```cpp
template <typename T>
concept HasValueType = requires {
    typename T::value_type; // 타입 요구사항: `T` 안에 `value_type`이라는 타입이 있어야 함
};
```

#### 3. 복합 요구사항 *(compound requirements)*

> 표현식이 noexcept 인지 & 표현식 타입이 type-constraint를 만족하는지 검증

```cpp
{ expression } noexcept(optional) -> type-constraint(optional);
```

* `expression` : 평가될 표현식 1개만, 여기엔 세미콜론이 없다는 것에 주의하자.
* `type-constraints` : 타입 제약 조건, 여기에 `expression`의 타입이 첫번째 타입 매개변수로 자동으로 전달된다.

##### 예시

```cpp
// `T`가 `unique_id()` 멤버함수를 가지며, 그 반환형은 `std::size_t`인지 검증하는 콘셉트
template <typename T>
concept UIDSizeT = requires(T obj) {
    { obj.unique_id() } -> std::same_as<std::size_t>; // 복합 요구사항
});
```

#### 4. 중첩 요구사항 *(nested requirements)*

> 요구사항 안에 요구사항을 넣고 싶을 때 사용

```cpp
requires constraint-expression;
```

##### 예시

```cpp
template <typename T>
concept C = requires (T t) {
    requires sizeof(t) == 4; // 중첩 요구사항
    ++t; --t; t++; t--; // 단순 요구사항들
};
```

## 조합

이미 존재하는 콘셉트 표현식을 `&&` 나 `||` 로 합쳐서 쓸 수도 있다.

### 예시

```cpp
// Incrementable과 Decrementable 콘셉트를 합쳐 새로운 콘셉트 작성
template <typename T>
concept IncrementableAndDecrementable = Incrementable<T> && Decrementable<T>;
```

## 표준 콘셉트

[`<concepts>` 헤더에 미리 정의된 표준 콘셉트가 많으니, 적절히 활용하자.](https://en.cppreference.com/w/cpp/concepts)

## `auto`에 제약 조건 지정하기

`auto` 쓰는 변수, 함수 리턴 타입, 축약 함수 템플릿(C++20), 제네릭 람다 표현식(C++14)에 쓸 수 있다고 한다.

```cpp
Incrementable auto val1 = 1; // `int` 추론, Incrementable 하므로 OK
Incrementable auto val2 = "abc"s; // `std::string` 추론, Incrementable 하지 않으므로 컴파일 오류
```

## 타입 제약 조건과 함수/클래스 템플릿

이제 콘셉트를 만들었으니, 이걸 템플릿 함수/클래스에 적용해봐야 할 것이다.

전혀 어려울 것이 없다. 다만 방법이 2가지로 나뉜다.

### `typename` 대체

첫번째로, `typename` 대신에 concept를 쓰는 방법이 있다.

이 때, `T`가 해당 concept의 첫번째 타입 매개변수로 자동으로 전달된다.

#### 예시

```cpp
template <std::convertible_to<bool> T>
void handle(const T& obj);
```

### 요구 구문 *(requires clause)*

둘째로, 요구 구문을 쓰는 방법이 있다.

이 때, 아래처럼 앞에 쓸 수도, 뒤에 쓸 수도 있다.

```cpp
template <typename T> requires 상수_표현식
void func();
```
```cpp
template <typename T>
void func() requires 상수_표현식;
```

요구 구문은 `상수_표현식`이라면 뭐든 가능하므로, 꼭 concept를 쓰지 않아도 된다.\
예를 들어, `<type_traits>`에 있는 TMP API를 활용할 수도 있을 것이다.

#### 예시

```cpp
template <typename T>
    requires std::convertible_to<T, bool>
void handle(const T& obj);
```

## 타입 제약 조건과 클래스 멤버 함수

클래스의 멤버 함수에 추가적인 제약 조건을 명시하는 것도 가능하다.

### 예시

```cpp
template <typename T>
class GameBoard
{
public:
    void move(int xSrc, int ySrc, int xDest, int yDest) requires std::movable<T>;
}
```

위와 같이 명시하면, 아무 `T`에 대해 `GameBoard<T>`를 인스턴스화 할 수는 있겠으나,\
`GameBoard<T>::move()` 멤버 함수는 `T`가 `std::movable<T>`를 만족할 때만 호출할 수 있을 것이다.

## 타입 제약 조건과 템플릿 특수화

특정 타입 제약 조건을 만족하는 타입들만 클래스 템플릿 특수화/함수 템플릿 오버로딩도 할 수 있다.

### 예시

```cpp
// 일반적인 `T`에 대해서는 이 버전이 호출됨
template <typename T>
std::size_t find(const T& value);

// `T`가 부동 소수점인 경우 이 버전이 호출됨
template <std::floating_point T>
std::size_t find(const T& value);
```




-----

## 참고자료
* [전문가를 위한 C++ (개정5판) (C++20): 콘셉트](https://hanbit.co.kr/store/books/look.php?p_code=B5989104254)
* cppreference.com
  * [Constraints and concepts](https://en.cppreference.com/w/cpp/language/constraints)
  * [Requires expression](https://en.cppreference.com/w/cpp/language/requires)

*마지막 수정 : {{ page.last_modified_at }}*
