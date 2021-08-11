---
layout: post
title: Meme 으로 알아보는 C++20 Constraints and Concepts
tags: [Meme, C++, C++20, Template Metaprogramming, SFINAE, Constraints and Concepts]
author: copyrat90
last_modified_at: 2021-08-07T17:53:00+09:00
---


[웃기는 Reddit 발 C++ Meme](https://www.reddit.com/r/ProgrammerHumor/comments/kvz1y2/then_everything_changed_when_the_nation_attacked/) 을 봤다.

![](/assets/img/posts/2021-08-07-cpp-verbose-meme-but-sfinae-invalid/meme.png)

C 에 ++ 붙이면 얼마나 장황해지는지를 압축해서 보여준다.

잠깐 피식거리다 생각해보니 **사실 이 Meme 은 좀 문제가 많다.**

### 1. 제시된 C 코드와 C++ 코드의 기능이 다르다.

Meme 에서의 C 코드는 `int` 자료형에만 작동하는 반면, C++ 코드는 `모든 숫자 자료형 T` 에 작동하는 코드다.\
뿐만 아니라, C++ 코드는 [`constexpr` *컴파일 타임 상수*](https://en.cppreference.com/w/cpp/language/constexpr)를 반환하는 함수고, [`[[nodiscard]]` *반환값을 사용하지 않으면 경고*](https://en.cppreference.com/w/cpp/language/attributes/nodiscard)가 나는 함수다.\
그리고 [`noexcept` *예외가 안난다고 컴파일러에게 힌트*](https://en.cppreference.com/w/cpp/language/noexcept)도 줬다.

사실 제시된 C 코드와 같은 기능을 구현하려면 C++ 코드에 C 코드를 그대로 사용하면 된다.\
이런 점을 보면 약간 억지 Meme 이라는 생각이 든다.

### 2. 제시된 C++ 코드에 오류가 있다.

이건 사실 [Reddit 덧글](https://www.reddit.com/r/ProgrammerHumor/comments/ll0ijo/then_everything_changed_when_the_nation_attacked/gnqokqz?utm_source=share&utm_medium=web2x&context=3)을 보기 전까지 알아차리지 못한건데, **제시된 C++ 코드에 오류가 있다.**\
이걸 이해하려면 템플릿 메타프로그래밍(TMP)과 `std::enable_if` 의 동작 원리를 알아야 한다.

[이 블로그](https://modoocode.com/295)에 자세하게 설명이 돼 있다.\
여기서는 간단하게 왜 오류인지만 짚어보도록 하자.

참고 : <https://en.cppreference.com/w/cpp/types/enable_if>
```cpp
template< bool B, class T = void >
struct enable_if;
```
`std::enable_if` 타입은 템플릿 인자 `B`에 `true`로 추론되는 식이 들어가면 내부에 `type`이라는 이름의 멤버 *type alias* 가 생기고, `false` 값이 들어가면 내부에 `type` 가 정의되지 않는 타입이다.\
위 참고 링크에 있는 예상 구현을 보면 위 설명 그대로인 것을 알 수 있다.
```cpp
template<bool B, class T = void>
struct enable_if {};
 
template<class T>
struct enable_if<true, T> { typedef T type; };
```
이제 `B` 에다가 우리가 원하는 제약조건을 걸고, `std::enable_if<B, T>::type` 에 접근해보자.\
그러면 [SFINAE](https://en.cppreference.com/w/cpp/language/sfinae) 에 의해 `B` 조건이 `true`인 템플릿만 성공적으로 오버로딩되고, `false`면 `std::enable_if<B, T>::type` 이 정의되지 않았으므로 오버로딩에 실패할 것이다.

문제는 Meme 코드에서는 `std::enable_if<B, T>::type` 에 접근을 안하고, `std::enable_if<B, T>` 에만 접근하고 있다는 점이다.
```cpp
// Meme code. WRONG!
template<typename T, typename = std::enable_if<std::is_arithmetic<T>::value, T>>
[[nodiscard]] static constexpr auto Add(const T& a, const T& b) noexcept -> T
{
    return a + b;
}
```
`std::enable_if<B, T>` 자체는 항상 정의되므로, 위 코드는 어떤 타입의 `T`가 들어가도 오버로딩이 되어버린다.\
사실상 제약조건을 안 건 것이나 마찬가지인 것이다.\
제대로 하려면 첫 줄을 아래 중 하나와 같이 썼어야 했다.
```cpp
// 1-1. 끝에 ::type 을 붙여 type 멤버 alias 에 접근
//      이 경우 std::enable_if<B, T>::type 가 dependent name 이라,
//      컴파일러는 이걸 기본적으로 non-type 이라고 가정해버리기 때문에, 앞에 typename 을 따로 붙여야 한다.
//      http://www.cplusplus.com/forum/beginner/103508/#msg557627
//      C++ 정말 너무 복잡하다...
template<typename T, typename = typename std::enable_if<std::is_arithmetic<T>::value, T>::type>
...
// 1-2. 사실 std::enable<B, T> 의 T 부분은 void 로 디폴트 타입이 정해져 있고,
//      타입이 중요한 부분이 아니므로 빼도 된다.
template<typename T, typename = typename std::enable_if<std::is_arithmetic<T>::value>::type>
...
// 1-3. C++14 부터 `::type` 에 대한 alias 인 `_t`, `::value` 에 대한 alias 인 `_v` 를 제공한다.
//      이 경우에는 typename 을 앞에 따로 안 붙여도 된다. alias 에 이미 들어있기 때문에.
template<typename T, typename = std::enable_if_t<std::is_arithmetic_v<T>>>
...
// 2-1. 템플릿 인자에 타입이 아니라 값을 받는 방식으로도 구현이 가능하다.
//      이 경우엔 std::enable<B, T> 의 인자 T 가 void 면 값을 못 넣으니, 다른 자료형으로 바꿔야한다.
template<typename T, std::enable_if_t<std::is_arithmetic_v<T>, int> = 0>
...
// 2-2. 자료형 바꾸기 귀찮으면 * 찍어서 void * = nullptr 형식으로도 쓸 수 있다.
template<typename T, std::enable_if_t<std::is_arithmetic_v<T>>* = nullptr>
...
```

물론 Meme 이 지적한대로 *제대로 된 방법을 써도 코드가 지저분하다는 건 여전히 유효*하다.\
... **C++20 에 [Constraints and Concepts](https://en.cppreference.com/w/cpp/language/constraints) 가 도입되기 전까지는** 말이지.

* * *

## [Constraints and Concepts (C++20)](https://en.cppreference.com/w/cpp/language/constraints)

기존 C++ 템플릿에 제약조건을 명시하려면 위처럼 기괴한 TMP 코드를 써야 했는데,\
표준 위원회 분들도 이게 너무 변태같다고 생각하셨는지 조금 쉽게 [제약조건(Constraints)](https://en.cppreference.com/w/cpp/language/constraints)을 명시하는 방법이 등장했다.

C# 에서 Generic 에 where 문으로 제약조건을 거는 것과 비슷하게, C++20 에서는 `requires` 를 쓰면 된다!

```cpp
// 3-1. C++20 Constraints 활용
template <typename T>
requires std::is_arithmetic_v<T>
...
```

와우, 정말 간단해졌다!

추가로, `Concepts` 까지 이용하면 이런 제약조건을 정의해놓고 재사용하는 것도 가능하다!\
제약조건이 여러 식을 `&&` 와 `||` 로 섞어야 할 정도로 복잡하다면 쓸만할 것이다.

```cpp
// 3-2. C++20 Constraints and Concepts 활용
template <typename T>
concept Arithmetic = std::is_arithmetic_v<T>;

template <Arithmetic T>
[[nodiscard]] static constexpr auto Add(const T &a, const T &b) noexcept -> T
{
    return a + b;
}
```

이 기능들을 도입하면, 치환에 완전히 실패했을 때 나오는 컴파일 오류 메시지도 한결 친절해진다.

원래는 아래와 같이 뭐가 문제인지 알기 어려웠던 오류가...

```
PS C:\Users\Home\Desktop> arm-none-eabi-g++ test.cpp -std=c++20
test.cpp: In function 'int main()':
test.cpp:14:21: error: no matching function for call to 'Add(std::string, std::string)'
   14 |     std::cout << Add(std::string("foo"), std::string("bar")) << '\n';
      |                  ~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
test.cpp:6:37: note: candidate: 'template<class T, std::enable_if_t<is_arithmetic_v<T> >* <anonymous> > constexpr T Add(const T&, const T&)'
    6 | [[nodiscard]] static constexpr auto Add(const T &a, const T &b) noexcept -> T
      |                                     ^~~
test.cpp:6:37: note:   template argument deduction/substitution failed:
In file included from d:\...\arm-none-eabi\include\c++\11.1.0\bits\move.h:57,
                 from d:\...\arm-none-eabi\include\c++\11.1.0\bits\nested_exception.h:40,
                 from d:\...\arm-none-eabi\include\c++\11.1.0\exception:148,
                 from d:\...\arm-none-eabi\include\c++\11.1.0\ios:39,
                 from d:\...\arm-none-eabi\include\c++\11.1.0\ostream:38,
                 from d:\...\arm-none-eabi\include\c++\11.1.0\iostream:39,
                 from test.cpp:1:
d:\...\arm-none-eabi\include\c++\11.1.0\type_traits: In substitution of 'template<bool _Cond, class _Tp> using enable_if_t = typename std::enable_if::type [with bool _Cond = false; _Tp = void]':
test.cpp:5:69:   required from here
d:\...\arm-none-eabi\include\c++\11.1.0\type_traits:2514:11: error: no type named 'type' in 'struct std::enable_if<false, void>'
 2514 |     using enable_if_t = typename enable_if<_Cond, _Tp>::type;
      |           ^~~~~~~~~~~
```

아래와 같이 친절하게 어디가 문제인지 알려주는 오류 메시지로 바뀐다!

```
PS C:\Users\Home\Desktop> arm-none-eabi-g++ test.cpp -std=c++20
test.cpp: In function 'int main()':
test.cpp:15:21: error: no matching function for call to 'Add(std::string, std::string)'
   15 |     std::cout << Add(std::string("foo"), std::string("bar")) << '\n';
      |                  ~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
test.cpp:7:37: note: candidate: 'template<class T>  requires  is_arithmetic_v<T> constexpr T Add(const T&, const T&)'
    7 | [[nodiscard]] static constexpr auto Add(const T &a, const T &b) noexcept -> T
      |                                     ^~~
test.cpp:7:37: note:   template argument deduction/substitution failed:
test.cpp:7:37: note: constraints not satisfied
test.cpp: In substitution of 'template<class T>  requires  is_arithmetic_v<T> constexpr T Add(const T&, const T&) [with T = std::__cxx11::basic_string<char>]':
test.cpp:15:21:   required from here
test.cpp:7:37:   required by the constraints of 'template<class T>  requires  is_arithmetic_v<T> constexpr T Add(const T&, const T&)'
test.cpp:6:15: note: the expression 'is_arithmetic_v<T> [with T = std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >]' evaluated to 'false'
    6 | requires std::is_arithmetic_v<T>
      |          ~~~~~^~~~~~~~~~~~~~~~~~
```

이걸 보고 있자니 왜 C++20 에 와서야 이 기능이 도입됐는지 참 안타깝다.\
진작에 도입됐어야 편하게 쓸텐데...\
C++20 은 너무 최신버전이라 실사용되려면 한참 걸릴테니, `std::enable_if` 도 어떻게 쓰이는 지 잘 알아둬야겠다.

* * *

사실 Meme 코드에 몇 가지 지적사항이 더 있다.

> 1. 왜 숫자 타입을 쓸데없이 참조로 전달하느냐? 어차피 몇 바이트 되지도 않는데?
> 2. 함수 프로토타입에 왜 굳이 쓸데없는 후행 리턴 타입을 쓰느냐?

1번은 일리 있는 지적이고, 2번은 스타일 문제로 치부할 수 있겠으나, 어차피 둘 다 사소한 문제다.

여튼 모든 걸 고치고 C++20 기능까지 사용하면 아래와 같이 C 코드 못지 않은 짧은 코드가 나온다.
```cpp
template <typename T>
requires std::is_arithmetic_v<T>
[[nodiscard]] static constexpr T Add(T a, T b) noexcept
{
    return a + b;
}
```

뭐, 이 정도면 C++ 가 그렇게 장황하지도 않은 것 같다.

_마지막 수정 : {{ page.last_modified_at }}_
