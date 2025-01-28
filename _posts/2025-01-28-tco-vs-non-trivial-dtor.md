---
layout: post
title: Tail call optimization vs. Non-trivial destructor
tags: [C++]
author: copyrat90
last_modified_at: 2025-01-28T19:44:00+09:00
---

Non-trivial destructor를 가지는 지역 변수가 있는 경우, 일반적으로 Tail-call optimization이 막힌다.

Tail-call optimization이 뭔지 잘 모르는 독자가 있을 수 있으므로, 우선 간략하게 설명하겠다.


# Tail-call optimization (TCO)

Tail-call optimization (TCO) 를 한 문장으로 요약하면 아래와 같다.

> 최후에 수행할 명령이 함수일 때, 해당 함수의 새로운 stack frame 쌓는 대신, 현재 stack frame을 재사용하는 최적화

함수 호출 시 추가적인 stack frame 을 쌓아야 하는 이유는,\
caller의 context를 저장해놨다가, 호출이 끝나면 복원하기 위함이다.\
그래야 callee가 반환된 후 caller의 남은 작업을 수행할 수 있을 테니.

그런데, callee가 반환될 시점에 caller에 남은 작업이 없는 경우라면?\
굳이 caller로 돌아갈 이유가 없어지니, caller의 context를 저장할 필요가 없이\
그냥 현재 stack frame에 callee에 전달된 매개 변수 등의 정보를 덮어 쓰고, 적절한 위치로 jump하면 된다.

> 참고로 Tail-call optimization을 흔히 *꼬리 재귀 최적화*라고 번역하는데, 이건 반만 맞는 말이다.\
> 프로그래밍 언어에 따라 꼭 재귀함수가 아니라도 Tail-call optimization이 가능한 경우도 있다.\
> 단지 Tail-call optimization이 드라마틱한 효과를 내는 상황이 재귀함수 호출이고, 꼬리 재귀만 최적화하는 언어도 있어서 혼용될 뿐.
> 
> 정확하게는 **꼬리 호출 최적화**라고 번역하는 편이 나았을 것이다.


## 피보나치 재귀함수 예시

```c
unsigned fibo(unsigned n) {
    if (n <= 1)
        return 1;
    return fibo(n-1) + fibo(n-2);
}
```

위 피보나치 재귀함수를 x86 clang 에서 `-O2` 옵션으로 컴파일하면 아래 assembly가 생성된다.

```c
fibo(unsigned int):
        push    ebx
        push    edi
        push    esi
        sub     esp, 16
        call    .L0$pb
.L0$pb:
        pop     ebx
.Ltmp0:
        add     ebx, offset _GLOBAL_OFFSET_TABLE_+(.Ltmp0-.L0$pb)
        mov     edi, dword ptr [esp + 32]
        mov     esi, 1
        cmp     edi, 2
        jb      .LBB0_4
        xor     esi, esi
.LBB0_2:
        lea     eax, [edi - 1]
        mov     dword ptr [esp], eax
        call    fibo(unsigned int)
        add     edi, -2
        add     esi, eax
        cmp     edi, 1
        ja      .LBB0_2
        inc     esi
.LBB0_4:
        mov     eax, esi
        add     esp, 16
        pop     esi
        pop     edi
        pop     ebx
        ret
```

C 소스코드와 달리, assembly에서 재귀 호출은 1번만 일어났다.\
그 이유는 아까 말한대로, 마지막으로 수행하는 `fibo(n-2)`의 경우 현재 stack frame을 재사용하기 때문이다.

assembly를 분석해 보자.

```c
.Ltmp0:
        add     ebx, offset _GLOBAL_OFFSET_TABLE_+(.Ltmp0-.L0$pb)
        mov     edi, dword ptr [esp + 32]    // edi = n;
        mov     esi, 1                       // esi = 1;
        cmp     edi, 2                       // if (edi <= 1)
        jb      .LBB0_4                      //     goto .LBB0_4;
        xor     esi, esi                     // esi = 0;
.LBB0_2:
        lea     eax, [edi - 1]               // eax = n-1;
        mov     dword ptr [esp], eax         // 위 매개변수를 스택에 넣기
        call    fibo(unsigned int)           // fibo(n-1) 재귀호출
/* 여기부터 fibo(n-2)를 현재 stack frame에서 수행하는 절차 */
        add     edi, -2                      // edi = n-2;
        add     esi, eax                     // esi += fibo(n-1) 반환값
        cmp     edi, 1                       // if (edi > 1)
        ja      .LBB0_2                      //     goto .LBB0_2;    // fibo(n-2) 효과를 얻게 됨
        inc     esi                          // esi += 1;            // fibo(edi)가 1 반환하는 경우
.LBB0_4:
        mov     eax, esi                     // 반환값 esi 사용
        /* 에필로그 생략 */
```

`fibo(n-1)`을 수행할 때는, 그 이후에 해야 할 작업(`fibo(n-2)` 수행 후 더하기)이 남아있어, 그냥 재귀 호출이 되었다.

반면, `fibo(n-2)`를 호출할 때는, 해야할 작업이 기존 결과에 누산 하나 뿐인데,\
이 누산 context만 `esi`에 유지하고 `goto`로 위로 올라감으로써, 마치 `fibo(n-2)`를 실행한 효과를 냈다.

> 사실 누산도 추가 작업에 해당하므로 최적화가 안 될 수도 있다. (clang이 똑똑해서 최적화해준 듯.)\
> 그걸 피하려면 누산용 변수를 매개변수로 따로 전달하면 된다.

만약 이게 피보나치가 아니라 팩토리얼처럼 재귀 호출이 1번만 있는 경우였다면,\
아예 재귀 호출 자체가 사라지고 조건부 `goto` 하나만 남았을 것이다.\
그런 경우, 분명 재귀호출을 짰는데 사실상 반복문을 짠 것과 마찬가지가 된다.


# 하지만 non-trivial destructor가 출동한다면 어떨까?

문제는 C++로 오면, 함수 반환 과정에서 지역 객체를 소멸시켜야 한다는 점이다.

```cpp
void func(int n) {
    std::string hey = "Destroy me at the end of the call, please.";
    if (n <= 0)
        return;
    func(n-1);
}
```

`func(n-1)`이 꼬리 재귀처럼 보이나, 사실 마지막으로 하는 작업은 `hey` 객체의 소멸자 호출이다.\
그래서 Tail-call optimization이 일어날 수가 없다.

아니, `hey`가 마지막 `func(n-1)` 호출 시에 사용되지 않고 있으니, 컴파일러가 그걸 판단해서 `hey` 소멸자를 먼저 호출해주면 안 되나?\
...라는 위험한 생각을 할 수도 있는데, 안 된다.

만일 `hey`가 `std::string`이 아니라 사용자 정의 class instance였고, 소멸자에서 전역변수를 수정한다면...?\
`func(n-1)` 호출이 끝나기 전까지, 전역변수의 값이 미리 바뀌어 있게 되고, 그건 잘못된 로직 순서가 된다.\
(사실 깊게 생각해보면 `std::string`인 위 경우도 마찬가지다. 전역 힙 메모리 관리자의 상태를 수정할 테니까...)

반면, 소멸자가 trivial해서 side-effect가 없다는 게 보장이 될 경우, 소멸자 호출이 필요 없으므로, Tail-call optimization이 가능하다.

알다시피 컴파일은 Translation Unit 단위로 일어나므로, 소멸자의 구현을 `func()` 컴파일하는 쪽에서는 알 수 없는 경우가 대부분이다.\
따라서 대부분의 경우, 소멸자가 non-trivial하게 정의된 지역 객체가 있다면 tail call optimization은 안 된다고 보면 된다.

Link-Time Optimization을 활성화한다면 달라질 수도 있을까 싶긴 한데... 실험하기는 귀찮으므로 패스.


## 지역 객체 먼저 소멸시키기

위 함수를 다음과 같이 수정하면 어떻게 될까?

```cpp
void func(int n) {
    // 명시적인 scope 추가
    {
        std::string hey = "Destroy me at the end of the call, please.";
        if (n <= 0)
            return;
    } // 명시적인 scope 종료, 이 지점에서 `hey`가 소멸된다.

    func(n-1); // 이 호출 이후에 할 작업이 진짜로 없다.
}
```

이 경우에는 지역 객체 소멸자를 명시적으로 `func(n-1)` 이전에 호출했으므로,\
`func(n-1)`이 마지막 호출이 되어 tail call이 맞는게 된다.\
따라서 Tail-call optimization이 가능하다.


# 참고 자료

* [Wikipedia - Tail call](https://en.wikipedia.org/wiki/Tail_call)
* [Stack Overflow - Tail recursion in C++](https://stackoverflow.com/a/2693719)


*마지막 수정 : {{ page.last_modified_at }}*
