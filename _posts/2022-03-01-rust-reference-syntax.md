---
layout: post
title: C++과 다른 Rust 참조자
tags: [Rust]
author: copyrat90
last_modified_at: 2022-03-01T12:00:00+09:00
---

Rust 의 참조자는 C++ 의 참조자랑 비슷하기보다는 차라리 C/C++ 의 포인터와 비슷해보인다.\
(필자는 Rust는 처음이고, 실험하면서 정리하는 중이므로, 틀린 정보가 있을 수 있다.)

## 1. 다른 객체 가리키도록 변경 가능

C++ 에서는 한번 참조자가 특정 변수를 가리키면, 가리키는 변수를 절대로 변경할 수 없다.\
반면, **Rust 에서는 참조자 자체를 `mut`으로 선언하면, 뭘 가리킬지를 변경할 수 있다.**

이 때문에 C/C++ 의 `const int* const pNum` 비슷하게,\
*1. 가리키는 변수*와 *2. 참조자 변수 자체*의 immutability 를 따로 명시해야 한다.

```rust
let mut num1 = 1;
let mut num2 = 2;

// 1. ref1 변경 OK, (*ref1) 변경 X.
// const int* ref1 = &num1;
let mut ref1: &i32 = &num1;
ref1 = &num2; // num2 가리키도록 변경.
*ref1 = -1; // [E0594] `ref1` is a `&` reference, so the data it refers to cannot be written

// 2. ref1 변경 X, (*ref1) 변경 OK.
// int* const ref1 = &num1;
let ref1: &mut i32 = &mut num1;
*ref1 = 7; // 역참조한 객체의 값 변경
ref1 = &mut num2; // [E0384] cannot assign twice to immutable variable `ref1`

// 3. ref1 변경 OK, (*ref1) 변경 OK.
// int* ref1 = &num;
let mut ref1: &mut i32 = &mut num1;
ref1 = &mut num2;
*ref1 = 7;

// 4. ref1 변경 X, (*ref1) 변경 X.
// const int* const ref1 = &num1;
let ref1: &i32 = &num1;
*ref1 = -1; // [E0594] `ref1` is a `&` reference, so the data it refers to cannot be written
ref1 = &num2; // [E0384] cannot assign twice to immutable variable `ref1`
```

## 2. 선언과 동시에 초기화 불필요

C++ 과는 달리, **Rust 의 참조자는 선언과 동시에 초기화하지 않아도 된다.**

Rust의 모든 immutable 변수들은, 선언과 동시에 초기화하지 않아도 된다.\
나중에 딱 1번만 초기화한다면 아무 문제 없이 쓸 수 있다.

immutable 참조자도 일종의 변수이므로, 위의 규칙이 적용된다.

```rust
fn main() {
    let ref1: &i32;
    let num1: i32;

    // 초기화되지 않은 변수를 참조(사용)하려고 하면 오류
    // ref1 = &num1; // [E0381] borrow (or use) of possibly-uninitialized `num1`
    
    num1 = 3;
    ref1 = &num1;
}
```

당연하지만, 1번도 초기화하지 않은 변수를 사용하면 컴파일 오류가 발생한다.


## 3. 참조의 참조 생성 가능

C++ 과는 달리, **Rust 에서는 참조자를 가리키는 참조자를 만들 수 있다.**

**`&&T` 는 T 의 참조의 참조를 의미한다.**\
(T의 r-value reference 가 아니다. Rust 에는 r-value reference 개념이 따로 존재하지 않는 듯하다.)

```rust
struct MyNum {
    num: i32,
}

impl MyNum {
    fn get_num(&self) -> i32 {
        self.num
    }
}

fn main() {
    let num1: MyNum = MyNum { num: 42 };
    let r1: &MyNum = &num1;
    let r2: &&MyNum = &r1; // Rust 에 r-value reference 따윈 없다. 이건 그저 참조의 참조일 뿐.
    let r3: &&&MyNum = &r2;
    let r9: &&&&&&&&&MyNum = &&&&&&r3;

    // 함수 호출 시에는 자동으로 dereference가 일어난다. 심지어 중첩 참조자라도.
    // 참고할만한 링크: https://stackoverflow.com/q/28519997
    r9.get_num(); // (*********r9).get_num() 과 동일
}
```

*마지막 수정 : {{ page.last_modified_at }}*
