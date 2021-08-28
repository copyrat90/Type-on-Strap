---
layout: post
title: 전역 상수를 헤더에 넣어보자!
tags: [C++, C++17, ODR, inline, constexpr]
author: copyrat90
last_modified_at: 2021-08-29T00:02:00+09:00
---

헤더에 전역 상수를 넣고 여기저기서 참조하는 접근법 3가지!

> 1. `inline constexpr T` (C++17)
> 2. `extern const T`(선언) + 따로 정의
> 3. `const T` and `constexpr T`

비슷한듯 미묘하게 다르다. [특히, non-static 인 constexpr 변수는 inline 이 아니다!](https://stackoverflow.com/a/57407675/12875525)\
참고로 순서는 `g++ -std=c++17 main.cpp ham*.cpp` 기준으로, 출력 파일의 용량이 작은 순서다.

| 전역 상수 | 출력물 용량 |
|:---|:---|
| `inline constexpr T` (C++17) | 16,664 bytes |
| `extern const T`(선언) + 따로 정의 | 16,696  bytes |
| `const T` and `constexpr T` | 16,744 bytes |

`const T` 와 `constexpr T` 는 출력 결과가 동일하다. (해시값이 같음)\
아래는 실험한 소스 코드.

### 1. `inline constexpr T` (C++17)
```cpp
// main.cpp
int main() { return 0; }

// ham.hpp
#pragma once
inline constexpr int answer = 42;

// ham1.cpp
#include "ham.hpp"
int ham1() { return answer; }

// ham2.cpp
#include "ham.hpp"
int ham2() { return answer; }

// ham3.cpp
#include "ham.hpp"
int ham3() { return answer; }
```

출력물 용량이 가장 작은 방법.\
objdump 심볼 테이블에 answer 심볼이 아예 존재하지 않는다!

> **장점**\
> 중복되는 심볼이 없는게 아니라, 아예 심볼이 없다.\
> 그 결과 용량이 제일 작다.

> **단점**\
> 전역 상수의 값을 바꾸면 포함하는 TU 를 전부 재컴파일 해야한다.\
> (여기서는 `ham1.cpp`, `ham2.cpp`, `ham3.cpp`)

### 2. `extern const T`(선언) + 따로 정의
```cpp
// main.cpp
int main() { return 0; }

// ham.hpp
#pragma once
extern const int answer;  // 선언

// ham1.cpp
#include "ham.hpp"
const int answer = 42;    // 정의
int ham1() { return answer; }

// ham2.cpp
#include "ham.hpp"
int ham2() { return answer; }

// ham3.cpp
#include "ham.hpp"
int ham3() { return answer; }
```

이 방법은 `answer` 의 값을 바꿔도, `ham1.cpp` 파일만 재컴파일하면 된다는 장점이 있다.

objdump 심볼 테이블 상에는 `answer` 가 `.rodata` 영역에 **g**lobal **O**bject 로 존재한다.
```
$ objdump -t a.out
...
0000000000002004 g     O .rodata	0000000000000004              answer
...
```

> **장점**
> 1. 상수의 값을 바꿔도, TU 하나만 재컴파일하면 된다.
> 2. 전역 심볼이 1개만 존재해서, 용량이 작다.

> **단점**\
> 상수의 선언과 정의를, 헤더와 구현 파일로 나눠 작성해야한다. 귀찮다.

### 3. `const T` and `constexpr T`

1번에서 `inline` 을 지운 방법이다.\
거기에다 `constexpr` 을 `const` 로 바꿔도 결과는 동일.

```cpp
// main.cpp
int main() { return 0; }

// ham.hpp
#pragma once
constexpr int answer = 42;
// const int answer = 42;

// ham1.cpp
#include "ham.hpp"
int ham1() { return answer; }

// ham2.cpp
#include "ham.hpp"
int ham2() { return answer; }

// ham3.cpp
#include "ham.hpp"
int ham3() { return answer; }
```

헤더에 이걸 쓰는 건 정말로 나쁜 방법이다.\
헤더를 포함하는 `ham*.cpp` 파일마다 `answer` 가 **l**ocal **O**bject 로 중복돼서 존재하게 된다.

```
$ objdump -t a.out
...
0000000000000000 l    df *ABS*	0000000000000000              ham1.cpp
0000000000002004 l     O .rodata	0000000000000004              _ZL6answer
0000000000000000 l    df *ABS*	0000000000000000              ham2.cpp
0000000000002008 l     O .rodata	0000000000000004              _ZL6answer
0000000000000000 l    df *ABS*	0000000000000000              ham3.cpp
000000000000200c l     O .rodata	0000000000000004              _ZL6answer
...
```

웬만하면 헤더에는 쓰지 말자.\
구현 파일에는 써도 되는데, [static 을 붙이거나, 익명 namespace 에다가 넣으면 암시적으로 inline 취급되어, 1번이랑 같아진다.](https://stackoverflow.com/a/57407675/12875525)

> **단점**
> 1. 상수 헤더를 포함하는 모든 TU 마다 심볼이 중복 존재하게 된다.\
> 그 결과, 용량이 쓸데없이 늘어난다.
> 2. 전역 상수의 값을 바꾸면 포함하는 TU 를 전부 재컴파일 해야한다.\
> (여기서는 `ham1.cpp`, `ham2.cpp`, `ham3.cpp`)


* * *

참고할만한 링크
1. [objdump Symbol table](https://stackoverflow.com/a/16471895/12875525)
2. [메모리 구조](https://ffun.tistory.com/entry/%EB%A9%94%EB%AA%A8%EB%A6%AC-%EA%B5%AC%EC%A1%B0)

*마지막 수정 : {{ page.last_modified_at }}*
