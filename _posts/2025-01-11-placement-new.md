---
layout: post
title: 99%가 잘못 쓰는 placement new
tags: [C++]
author: copyrat90
last_modified_at: 2025-01-11T12:00:00+09:00
---

제목 어그로를 좀 끌어봤는데, 이만큼 직관적이면서 어그로 없는 제목이 안 떠오르는 걸 어쩌겠는가.\
그리고 인터넷에서 보이는 거의 대부분 글에서 placement new가 잘못 쓰이고 있기도 하고.

# `operator new` vs. new expression

우선 혼동의 여지가 있는 용어부터 정리하자.

1. [`operator new`](https://en.cppreference.com/w/cpp/memory/new/operator_new) : 요청한 바이트 수만큼의 공간을 동적 할당해주는 연산자
1. [new expression](https://en.cppreference.com/w/cpp/language/new) : 위 `operator new` 연산자를 활용해 공간을 동적 할당 + 그 공간에다가 생성자를 호출하는 표현식


1. [`operator delete`](https://en.cppreference.com/w/cpp/memory/new/operator_delete) : `operator new`로 할당한 공간을 해제해주는 연산자
1. [delete expression](https://en.cppreference.com/w/cpp/language/delete) : new expression으로 생성된 객체의 소멸자를 호출 + 할당된 공간을 `operator delete`로 해제해주는 표현식

즉, `operator new`는 순수하게 메모리 할당만 하고, new expression은 메모리 할당 + 객체 생성을 한다.

간단하게 아래와 같은 예제 코드로 확인해볼 수 있다.

```cpp
#include <cstddef>
#include <iostream>
#include <new>

void* operator new(std::size_t size, const char* msg) {
    std::cout << msg << std::endl;
    return ::operator new(size);
}

void operator delete(void* ptr, const char* msg) {
    std::cout << msg << std::endl;
    ::operator delete(ptr);
}

struct MyData {
    MyData() { std::cout << "MyData()" << std::endl; }
    ~MyData() { std::cout << "~MyData()" << std::endl; }
};

int main() {
    // new expression
    MyData* exp = new ("Allocate space for `exp`!") MyData;
    // delete expression
    delete exp;  // delete expression은 placement-args 기능이 없다.

    std::cout << "Deleted space for `exp`..." << std::endl;

    // `MyData`의 alignment는 `__STDCPP_DEFAULT_NEW_ALIGNMENT__`를
    // 넘지 않기 때문에, 아래 static_cast<MyData*>를 사용 가능.
    // (보통 64-bit 환경에서 16-bytes 경계값으로 기본 align됨)
    static_assert(alignof(MyData) <= __STDCPP_DEFAULT_NEW_ALIGNMENT__);

    // new operator
    MyData* oper = static_cast<MyData*>(
        ::operator new(sizeof(MyData), "Allocate space for `oper`!"));
    // delete operator
    ::operator delete(oper, "Delete space for `oper`!");
}
```

실행 결과는 다음과 같다.

```
Allocate space for `exp`!
MyData()
~MyData()
Deleted space for `exp`...
Allocate space for `oper`!
Delete space for `oper`!
```

# placement new

흔히들 [placement new](https://en.cppreference.com/w/cpp/language/new#Placement_new)라고 부르는 기능을 엄밀하게 설명하면,\
[new expression에 *placement-args*를 넣어](https://en.cppreference.com/w/cpp/language/new), new expression이 다른 오버로딩된 `operator new`를 호출할 수 있도록 하는 기능이다.

위 예제 코드에서 다뤘던 `const char* msg`를 전달하는 new expression도 넓은 의미에서는 placement new라고 할 수 있다.

그런데 placement new를 사용할 때, operator new 오버로드 중 주로 [non-allocating placement deallocation function (9번, 10번)](https://en.cppreference.com/w/cpp/memory/new/operator_new)을 호출하기 때문에,\
이걸 호출하는 버전을 placement new 라고 퉁쳐서 부르기도 한다. 헷갈리게시리...\
앞으로 다루는 내용은 이 동적 할당하지 않는 버전의 placement new 이다.

방금 말한 non-allocating placement deallocation function은 이렇게 생겼다.
```cpp
void* operator new (std::size_t size, void* ptr);
```

이녀석은 이름답게, **동적 할당을 하지 않고, `ptr` 변수를 있는 그대로 반환한다.**\
그 결과, 이걸 호출하는 placement new 또한 동적 할당을 하지 않고, *placement-arg* 로서 넣은 **`void*` 위치에다가 객체를 생성하게 된다.**

## 빠지기 쉬운 함정들

### 1. 소멸자 호출을 빼먹기

이건 너무 기초적인 실수라 언급만 하고 넘어가도록 하겠다.

placement new로 생성한 객체는 반드시 소멸자를 직접 호출해야지, `operator delete`나 delete expression으로 처리하려고 들면 안된다.\
둘 다 결국 메모리를 해제하려 드는데, 동적 할당을 하지 않은 영역을 해제하려 들면 바로 heap allocator가 오류를 낼 것이다.

### 2. 생성할 객체 Type과 alignment가 맞지 않는 주소를 전달

이것도 비교적 잘 알려진 실수이지만, 흔한 실수이기 때문에 한번 다뤄본다.

```cpp
// alignas(MyData)를 빼먹었다.
char data_storage[sizeof(MyData)];
::new (static_cast<void*>(data_storage)) MyData;
```

보통 전달할 메모리 공간이 `MyData`를 넣을 수 있는 크기인지는 고려해 buffer를 만들지만,\
그 공간의 alignment를 `MyData`에 맞추는 걸 빼먹는 경우가 많다.\
이걸 빼먹으면 unaligned access가 발생할 위험이 있다.

문제는 unaligned access로 인한 버그가 환경에 따라 발생하지 않는 경우가 많아, 실수한 줄도 모른다는 점이다.\
대부분 환경에서 메모리 할당시 기본 alignment가 8-bytes가 넘기도 하고,\
AMD64를 포함해 여러 아키텍처에서 unaligned access를 해도 crash를 내지 않고\
2번의 memory access로 퉁치고 넘어가버리기도 해서 그렇다.

그렇지만 2번의 memory access가 된다는 건 atomic access 보장이 깨진다는 소리이므로, 멀티스레드 환경에서 torn read/write가 발생하는 경우는 관찰할 수 있다.

예를 들면 AMD64 아키텍처에서는 cache line 단위로의 atomic access 만 보장하는데,\
아래와 같이 `std::atomic` 변수라도 cache line 경계에 걸치도록 만들 경우 torn write가 발생한다.

```cpp
#include <algorithm>
#include <atomic>
#include <cstddef>
#include <cstdint>
#include <format>
#include <iomanip>
#include <iostream>
#include <new>
#include <thread>
#include <vector>

static_assert(std::atomic_uint32_t::is_always_lock_free);

static constexpr std::uint32_t WRITE_VAL = 0x1111'1111u;

void worker(std::atomic_uint32_t* atomic_num, std::uint32_t store_val) {
    for (;;) {
        // 0x1111... / 0x2222... / 0x3333... 등 대입
        atomic_num->store(store_val);
        // torn read 발생 확인
        const auto read_num = atomic_num->load();
        if (read_num % WRITE_VAL != 0) {
            std::cout << std::format("TORN READ! value was: 0x{:x}\n",
                                     read_num);
            std::terminate();
        }
    }
}

int main() {
    constexpr std::size_t CACHE_LINE_SIZE =
        std::hardware_destructive_interference_size;
    char atomic_storage[sizeof(std::atomic_uint32_t) + CACHE_LINE_SIZE];

    // 일부러 cache line 경계 위치에 걸리도록 unaligned 위치를 잡아보자.
    char* obj_addr = atomic_storage;
    for (std::size_t i = 0; i < CACHE_LINE_SIZE; ++i) {
        if ((std::uintptr_t(obj_addr + 1)) % CACHE_LINE_SIZE == 0)
            break;
        else
            ++obj_addr;
    }

    // atomic 변수를 cache line 경계에 생성
    std::atomic_uint32_t* atomic_num =
        ::new (static_cast<void*>(obj_addr)) std::atomic_uint32_t;

    std::cout << "atomic variable constructed at: 0x" << std::hex
              << reinterpret_cast<std::uintptr_t>(obj_addr) << std::endl;
    std::cout << "cache line size: 0x" << CACHE_LINE_SIZE << std::endl;

    // 여러 스레드에서 worker() 실행
    // 0x1111... / 0x2222... / 0x3333... 등 대입하며 torn write 발생 확인
    const unsigned cores =
        std::thread::hardware_concurrency()
            ? std::clamp(std::thread::hardware_concurrency(), 2u, 15u)
            : 2u;
    std::vector<std::thread> threads;
    threads.reserve(cores);
    for (unsigned i = 0; i < cores; ++i)
        threads.emplace_back(worker, atomic_num, WRITE_VAL * i);
    for (auto& th : threads) th.join();

    atomic_num->~atomic<std::uint32_t>();
}
```

멀티코어 AMD64 아키텍처에서 실행해보면 금방 오류를 볼 수 있다.
```cpp
atomic variable constructed at: 0x7f401130013f
cache line size: 0x40
TORN READ! value was: 0x33333311
terminate called without an active exception
Aborted
```

위와 같이 분명 `std::atomic` 변수임에도 unaligned address에 객체를 생성했기 때문에,\
`0x33333333`을 쓰는 스레드와 `0x11111111`을 쓰는 스레드 간 경합으로 `0x33333311`이 쓰여버리는 참사가 일어났다.


### 3. `void*`가 아닌 포인터를 전달

아래와 같이 `void*`가 아닌 포인터를 전달하는 경우는?

```cpp
alignas(MyData) char data_storage[sizeof(MyData)];
// `void*` 대신 `char*`가 전달되었다.
::new (data_storage) MyData;
```

이것도 일반적인 상황에서는 문제가 발생하지 않는다.\
왜냐면 `operator new` 오버로딩을 안하는 경우가 많기 때문.

하지만 제일 앞에서 본 코드처럼 `const char*` 오버로딩이 되어있다면 어떨까?\
다시 가져와보면:
```cpp
void* operator new(std::size_t size, const char* msg) {
    std::cout << "Message: " << msg << std::endl;
    return ::operator new(size);
}
```

그냥 `const char* msg`를 추가로 받아서, 메시지를 출력해주는 기능이다.\
그런데 가만, 위에서 전달한 `data_storage`가 `char*`이지 않은가?\
그렇다. `data_storage`에 객체를 생성하려던 의도와 달리, **내가 정의한 동적 할당을 하는 오버로드가 호출되어버린다.**

```cpp
#include <cstddef>
#include <iostream>
#include <new>

// char* 받아서 `msg`를 출력하는 오버로드
void* operator new(std::size_t size, const char* msg) {
    std::cout << "Message: " << msg << std::endl;
    return ::operator new(size);
}

void operator delete(void* ptr, const char* msg) {
    std::cout << "Message: " << msg << std::endl;
    ::operator delete(ptr);
}

struct MyData {
    char str[64];
    MyData() { std::cout << "MyData()" << std::endl; }
    ~MyData() { std::cout << "~MyData()" << std::endl; }
};

int main() {
    alignas(MyData) char data_storage[sizeof(MyData)];

    // `data_storage`에 객체를 생성하는 게 아니라,
    // 그걸 `msg`로 해석해 출력하고, 동적 할당한 건 leak 되어 버린다.
    ::new (data_storage) MyData;

    // 생성된 적도 없는 `data_storage` 위치에서 소멸을 한다.
    // 소멸자 정의에 따라 온갖 undefined behavior를 야기할 수 있다.
    reinterpret_cast<MyData*>(data_storage)->~MyData();
}
```

결과는 다음과 같다.

```cpp
Message: 
MyData()
~MyData()

=================================================================
==48112==ERROR: LeakSanitizer: detected memory leaks

Direct leak of 64 byte(s) in 1 object(s) allocated from:
    #0 0x7ff04f022548 in operator new(unsigned long) ../../../../src/libsanitizer/asan/asan_new_delete.cpp:95
    #1 0x560b2d7e9305 in operator new(unsigned long, char const*) (/home/copyrat90/a.out+0x1305) (BuildId: b38949a2ad9f4160d9d1c4e06ac28e53493bfd68)
    #2 0x560b2d7e940e in main (/home/copyrat90/a.out+0x140e) (BuildId: b38949a2ad9f4160d9d1c4e06ac28e53493bfd68)
    #3 0x7ff04e9a71c9 in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:58
    #4 0x7ff04e9a728a in __libc_start_main_impl ../csu/libc-start.c:360
    #5 0x560b2d7e91e4 in _start (/home/copyrat90/a.out+0x11e4) (BuildId: b38949a2ad9f4160d9d1c4e06ac28e53493bfd68)

SUMMARY: AddressSanitizer: 64 byte(s) leaked in 1 allocation(s).
```

엉뚱하게 `Message:` 출력이 들어가 있으며, AddressSanitizer가 memory leak을 감지한 것을 볼 수 있다.

이걸 고치려면 명시적으로 `void*`로 캐스팅해서 전달만 하면 된다.

```cpp
::new (static_cast<void*>(data_storage)) MyData;
```

고치고 난 실행 결과는 다음과 같다.
```
MyData()
~MyData()
```

#### 나는 `operator new` 오버로딩 안 할 건데?

나는 `operator new` 오버로딩 안 할건데 괜찮지 않나요? 라고 생각할 수도 있는데,\
차후에 링크하게 된 라이브러리가 자기 마음대로 `operator new`를 오버로딩하면, 여태까지 잘 동작하던 코드가 그로 인해 버그를 뿜는 상황에 빠질 수도 있다.

사실 근데 라이브러리가 global `operator new`를 오버로딩 하는 경우가... 있을지는 모르겠다.\
그나마 [CRT 디버깅](https://learn.microsoft.com/en-us/cpp/c-runtime-library/find-memory-leaks-using-the-crt-library?view=msvc-170)처럼 디버깅용 라이브러리가 `new`를 제멋대로 재정의하는 경우?


### 4. global scope resolution operator를 빼먹기

마지막으로 할 수 있는 실수는 `new` 앞에 `::`를 빼먹는 경우이다.

```cpp
alignas(MyData) char data_storage[sizeof(MyData)];
// `new` 앞에 `::`를 빼먹었다.
new (static_cast<void*>(data_storage)) MyData;
```

이건 또 왜 문제라는걸까?

`::`를 빼먹으면, global scope가 아닌 곳에서도 `operator new`를 찾으려고 시도한다.\
만일 **생성하려는 타입의 클래스 내부에 `operator new`가 오버로딩되어 있는 경우**, overload resolution을 거기서 멈춘다.\
문제는 전달된 인자 목록이 불일치해도 거기서 멈춰버린다는 것이며, 이는 컴파일 오류를 야기한다.

```cpp
#include <cstddef>
#include <iostream>
#include <new>

struct MyData {
    MyData() { std::cout << "MyData()" << std::endl; }
    ~MyData() { std::cout << "~MyData()" << std::endl; }
    void* operator new(std::size_t size) {
        std::cout << "MyData()::operator new" << std::endl;
        return ::operator new(size);
    }
};

int main() {
    alignas(MyData) char data_storage[sizeof(MyData)];

    // `MyData::operator new`에는 `void*`를 인자로 받는 오버로드가 없다고
    // 불평한다.
    new (static_cast<void*>(data_storage)) MyData;

    reinterpret_cast<MyData*>(data_storage)->~MyData();
}
```

컴파일하면
```cpp
test.cpp: In function ‘int main()’:
test.cpp:19:44: error: no matching function for call to ‘MyData::operator new(sizetype, void*)’
   19 |     new (static_cast<void*>(data_storage)) MyData;
      |                                            ^~~~~~
test.cpp:8:11: note: candidate: ‘static void* MyData::operator new(std::size_t)’
    8 |     void* operator new(std::size_t size) {
      |           ^~~~~~~~
test.cpp:8:11: note:   candidate expects 1 argument, 2 provided
```

이걸 막으려면, `new` 앞에 `::`를 붙여 global scope에서만 찾도록 제약을 걸면 된다.

#### 나는 `T::operator new` 오버로딩 안 할 건데?

나는 클래스에다가 `operator new` 오버로딩 안 할건데 괜찮지 않나요? 라고 생각할 수도 있는데,\
차후에 링크하게 된 라이브러리가 자기 클래스 일부에 `operator new`를 오버로딩하는 경우에, 템플릿으로 placement new 하는 코드에서 그 클래스를 템플릿 매개변수로 전달할 경우 컴파일 오류가 날 수 있다.

[이 상황이 발생한 GitHub Issue 예시.](https://github.com/GValiente/butano/issues/81)\
Code Examples -> Example 1 을 보면, 예제로 만든 외부 클래스 `Test1`이 `Test1::operator new`를 오버로딩했다.\
그리고 저 프로젝트 코드의 `bn::pool<T, MaxSize>::create()` 내부 placement new가 global scope resolution operator를 빼먹어서, `bn::pool<Test1, 3>::create()` 호출 시에 컴파일 오류가 난 걸 볼 수 있다.\
(참고로 지금은 문제가 수정되었다.)

## 결론

한마디로 요약하면 이렇게 쓰면 된다.

```cpp
// 생성할 객체의 alignment와 size를 고려한 buffer 만들기
alignas(MyData) char data_storage[sizeof(MyData)];
// `::`와 `void*` 캐스팅 잊지 말기
MyData* ptr = ::new (static_cast<void*>(data_storage)) MyData;
// 소멸자 호출 잊지 말기
ptr->~MyData();
```

# 번외: `std::destroy_at` & `std::construct_at`

C++17에서 [`std::destroy_at`](https://en.cppreference.com/w/cpp/memory/destroy_at)이, C++20에서 [`std::construct_at`](https://en.cppreference.com/w/cpp/memory/construct_at)이 추가되었다.

각각 명시적 소멸자 호출, placement new를 대체한다고 보면 된다.

```cpp
alignas(MyData) char data_storage[sizeof(MyData)];
MyData* ptr = std::construct_at(reinterpret_cast<MyData*>(data_storage));
std::destroy_at(ptr);
```

[이 Stackoverflow 답변글을 보면](https://stackoverflow.com/a/52971541/12875525), `std::destroy_at`은 가끔 명시적 소멸자 이름 찾기가 애매한 경우에 쓰기 좋다고 하고,\
`std::construct_at`은 그냥 짝맞추기 위해서 나왔다는 것 같다.

당장 위 atomic 관련 예제에서도 `std::atomic_uint32_t` 가 alias 라서 소멸할 때는 `~atomic<std::uint32_t>()`로 소멸시켰는데, `std::destroy_at()`을 썼으면 이런 문제를 생각할 필요도 없었겠다.

# 참고 자료

* [Stackoverflow - Using placement new in generic programming](https://stackoverflow.com/a/57539415/12875525)
* [Stackoverflow - Why isn't there a std::construct_at in C++17?](https://stackoverflow.com/a/52971541/12875525)

*마지막 수정 : {{ page.last_modified_at }}*
