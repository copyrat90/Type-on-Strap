---
layout: post
title: 공간을 2배씩 늘리는 Semi-Lockfree Object Pool
tags: [Lockfree, Data Structure]
author: copyrat90
last_modified_at: 2024-12-11T12:00:00+09:00
---

공개된 Lockfree Object Pool 구현체는 많이 있다.\
그런데 다들 예약된 Object가 다 떨어지면, Object를 1개만 추가로 동적 할당해 기존 linked list에 붙이는 식으로 구현한다. (예: [boost freelist](https://github.com/brycelelbach/boost.lockfree/blob/5eed06058c25ca71b8a9732dbf7a0f54434dfebb/boost/lockfree/detail/freelist.hpp#L72))

`std::vector` 마냥, **공간이 부족할 시, 예약된 Object를 기존 capacity의 2배가 되도록 한방에 할당해서, 동적 할당의 횟수를 줄여보면 어떨까?**

(사실 이렇게 구현하려면, 할당이 일어날 때만큼은 lock이 걸려야 해서, semi-lockfree가 되긴 한다...)

# 개념

## 기존 Freelist

우선 기존 Freelist 방식의 구현을 생각해보자.\
각 Node를 next pointer로 연결하고, 제일 앞을 `node_head` pointer로 가리켜 거기에 Node를 추가하고 삭제하는 식이다.

![](/assets/img/posts/2024-12-11-semi-lockfree-object-pool/01.svg)

```cpp
struct Node {
    T data;
    Node* next;
};
...
std::atomic<Node*> _node_head;
```

하지만 위 방법 그대로 한다면, 추가 Node가 필요할 때마다 1개씩 동적 할당할 수밖에 없다.

## Block 단위 Freelist

잦은 동적 할당을 피하려면 어떻게 해야 할까?\
커다란 Block을 할당하고, 이를 여러 개의 Node로 쪼개서 사용하는 방법을 생각할 수 있다.

![](/assets/img/posts/2024-12-11-semi-lockfree-object-pool/02.svg)


```cpp
struct Block {
    Block* next;
    std::byte* alloc_addr; // unaligned allocation address to deallocate later

    std::size_t count; // number of `Node`s in this block
    // Node nodes[count];
};
...
std::mutex _block_mutex;
Block* _block_head;
std::atomic<Node*> _node_head;
```

이런 구조면, Block 전체의 소유권은 Object Pool이 관리하되, 그 공간의 일부를 자른 Node의 pointer를 Object를 할당받는 측에게 넘기는 식이 된다.\
따라서 Block을 추가 할당한다고 기존 Node의 pointer가 달라지면 안 되므로,`std::vector`와 달리 내부 Node & Object를 move / copy 할 수는 없다.\
따라서, Block 추가 할당 시에도 기존 Block을 유지하기 위해, Block 자체도 linked list 형식으로 관리한다.

또한, 기존 capacity의 2배씩 증가시킬 수 있으려면, Block의 크기는 가변적이어야 한다.\
따라서, `struct Block` 안에 `Node`들을 고정 크기 배열로 넣는 게 불가능하다.\
그래서, 우선 (이번에 필요한 전체 Block 크기 + alignment를 위한 패딩) 만큼의 공간을 byte 단위로 동적 할당 후, `Block`(Block 헤더) 부분과 내부 `Node`들의 메모리 위치를 직접 계산해 써야 한다.\
`Block`과 `Node`의 메모리 위치를 수동으로 잡을 때, 그 둘의 alignment를 맞춰야 하는데, 이를 위해 [`std::align()`](https://en.cppreference.com/w/cpp/memory/align)을 이용하면 편하다.

그리고, Block을 할당할 때만큼은, 다른 thread들도 할당이 끝나기를 기다려야 하므로, 어쩔 수 없이 `_block_mutex`로 lock을 걸 수 밖에 없다.\
이것마저 lockfree 하게 처리하면, Node가 부족해졌을때 모든 thread가 자기만의 거대한 Block을 동적 할당해 버려서 메모리가 낭비될 위험이 있기 때문이다.\
이걸 피하기 위해, lock을 걸고 나서, 혹시 다른 thread가 이미 할당을 마쳐 새로운 `_node_head`를 쓸 수 있게 되었는지 double-checking이 들어가게 된다.

```cpp
void add_new_block() {
    std::lock_guard<std::mutex> block_guard(_block_mutex);
    // double check if new block allocation is still required
    if (!_node_head.load()) {
        ... /* really allocate a new block & connect it */
    }
}
```

(그래도 추가 Block 할당이 필요없을 때는 lock이 걸리지 않으므로, semi-lockfree라고 불러도 괜찮을 듯 싶다.)

[`add_new_block()` 전체 코드](https://github.com/copyrat90/NetBuff/blob/1205d17a283f672f02898074687564a8ce06c5e8/include/NetBuff/LockfreeObjectPool.hpp#L268-L343)

## 객체 재사용 vs 메모리만 재사용

Object Pool을 구현함에 있어, 2가지 사용례를 생각할 수 있다.

1. Object를 한번 생성하면, Object Pool에서 할당/반납시에도 계속 소멸시키지 않고 재사용하다가, Object Pool이 삭제될 때 그제서야 소멸시키기
1. 매번 Object Pool에서 할당 시 Object 생성 -> 반납 시 소멸시키기

Object 생성/소멸 자체가 무거운 작업이라면, 1번을 원할 수도 있다.\
다만 이 경우, 할당받은 Object가 재사용된 객체일 수 있으므로, Object가 적절한 초기화용 멤버 함수를 제공해야 할 것이다.

반대로 매번 즉시 소멸을 원한다면 2번이 적합할 수도 있다.\
이렇게 되면, Object 반납 시 `Node` 내부에 객체를 유지할 필요가 없으므로, `next` pointer와 Object를 union으로 감싸 메모리를 조금 아낄 수도 있다.
```cpp
struct Node {
    union {
        Node* next;
        alignas(T) std::byte data[sizeof(T)]; // space to store object
    };
};
```

그래서 뭘로 구현할까? 둘 다 그럴듯한데...\
나는 그냥 템플릿 매개변수 `bool CallDestructorOnDestroy`를 받아서, `false`면 1번, `true`면 2번으로 동작하도록 구현했다.

```cpp
template <typename T, bool CallDestructorOnDestroy>
class LockfreeObjectPoolTraits {
protected:
    struct Node {
        Node* next;
#if NB_OBJ_POOL_CHECK
        LockfreeObjectPoolTraits* pool; // run-time check if enabled
#endif
        alignas(T) std::byte data[sizeof(T)];
        bool constructed; // whether the `obj` is alive or not
    };
};

template <typename T>
class LockfreeObjectPoolTraits<T, true>
{
protected:
    struct Node {
        union {
            Node* next;
            alignas(T) std::byte data[sizeof(T)];
        };
#if NB_OBJ_POOL_CHECK
        LockfreeObjectPoolTraits* pool; // run-time check if enabled
#endif
    };
};

template <typename T, bool CallDestructorOnDestroy, ...>
class LockfreeObjectPool final : private LockfreeObjectPoolTraits<T, CallDestructorOnDestroy>, ... {
    ...
};
```

Object를 할당 및 해제하는 함수에서는, `CallDestructorOnDestroy` 값에 따라 객체를 매번 생성할 수도, 아닐 수도 있도록 `if constexpr`로 다르게 처리했다.

```cpp
template <typename... Args>
[[nodiscard]] auto construct(Args&&... args) -> T& {
    ...
    if constexpr (CallDestructorOnDestroy) {
        // always construct a fresh new object
        ::new (static_cast<void*>(cur->data)) T(std::forward<Args>(args)...);
    } else {
        // only construct if it was never constructed before
        if (!cur->constructed) {
            ::new (static_cast<void*>(cur->data)) T(std::forward<Args>(args)...);
            cur->constructed = true;
        }
    }
    return reinterpret_cast<T&>(cur->data);
}

void destroy(T& obj) {
    ...
    if constexpr (CallDestructorOnDestroy)
        obj.~T();
    ...
}
```

## Lockfree하게 `node_head` 교체

이 부분은 여타 Lockfree Stack 구현체와 대동소이하다.

### Naive한 원리

#### Push

Push 상황은 2가지가 있는데, Object를 할당받은 측에서 반납하는 경우와, `add_new_block()` 끝에서 추가된 Node들을 전부 연결하는 상황이 그것이다.

두 상황 모두 동일하게 처리할 수 있다.\
기존 `node_head`가 가리키는 Node를 새로운 Node의 next로 설정하고, `node_head`를 새로운 Node로 교체하면 된다.\
이걸 atomic하게 처리하기 위해 일단 새로운 Node를 설정하고, CAS를 통해 `node_head` 교체를 시도해본 후, 실패하면 새로운 Node의 next를 변화된 `node_head`로 재설정 후 재시도한다.

Lockfree Stack의 경우, 아래와 같이 구현해 볼 수 있다.
```cpp
void NaiveLockfreeStack::push(const T& data) {
    Node* old_top = _top.load();
    Node* new_node = new Node{data, old_top};
    while (!_top.compare_exchange_weak(new_node->next, new_node))
        ;
}
```

Object Pool도 위와 동일한 원리로 구현할 수 있다.\
Object Pool 코드는 밑에서 ABA 문제까지 해결하고 난 이후에 살펴보도록 하자.

#### Pop

Pop 상황은, Object 할당 요청이 들어와서 Node를 하나 꺼내야 하는 경우다.

기존 `node_head`가 가리키는 Node를 가져온 후, 그것의 next를 새로운 `node_head`가 되도록 교체하면 된다.\
이걸 atomic하게 처리하기 위해 일단 기존 head를 가져오고, CAS를 통해 `node_head`를 기존 head의 next로 교체 시도해본 후, 실패하면 기존 head를 변화된 `node_head`로 재설정 후 재시도한다.

Lockfree Stack의 경우, 아래와 같이 구현해 볼 수 있다.
```cpp
auto NaiveLockfreeStack::pop() -> std::optional<T> {
    std::optional<T> result;
    Node* old_top = _top.load();
    while (old_top && !_top.compare_exchange_weak(old_top, old_top->next))
        ;
    if (old_top) {
        result = old_top->data;
        delete old_top; // possible dangling pointer on other thread(s)
    }
    return result;
}
```

사실, 위 Stack 코드에서 `delete old_top;`을 하면, 다른 thread에서 같은 pointer를 갖고 `old_top->next`로 dangling pointer 참조하는 것을 막을 수가 없다.\
하지만 지금 구현하는 건 Object Pool이고, 내 구현은 Object Pool이 사라지기 전까지는 어떤 메모리도 해제하지 않으므로 이 문제가 나지 않는다.

### ABA 문제

#### 문제 상황

그렇지만 위 Naive한 구현은 여전히 문제점을 갖고 있다. 바로 ABA 문제다.

`NaiveLockfreeStack::pop()` 코드를 다시 보자.\
만일, 어떤 thread가 `Node* old_top = _top.load();`까가만 수행한 상태에서 정지한다고 해보자.\
그 상태에서 다른 thread들이 `pop()`->`pop()`->`push()`를 정상 수행하는데, 첫번째 `pop()`한 Node와 같은 주소의 객체를 마지막에 다시 `push()`한다면?

![](/assets/img/posts/2024-12-11-semi-lockfree-object-pool/03.svg)


위 그림을 예시로 보면, 정지된 thread는 처음에 `old_top` 주소를 `0x01`(A)로 읽은 상태였는데, 중간에 다른 thread들이 `0x02`, `0x03`으로 top을 바꾸는 것(B)을 눈치채지 못하고 `0x01`로 돌아온 최종 상태(A)에서 다시 재개될 수 있다.\
그렇게 되면, CAS는 성공하지만, `old_top->next`는 이제 존재하지 않는 `0x02`이므로, `_top`이 `0x02`로 세팅되어 사라진 Node를 가리키게 되어버린다.

#### 해결책

기존 head를 얻는 것과 CAS 수행 사이에, 연결 상태가 바뀌었다는 것을 인지할 방법이 필요하다.\
그런데 head의 주소만 가지고는 위와 같은 상황에서 그 판단이 불가능하므로, 별도의 정보가 필요하다는 결론이 나온다.

간단하게는 별도의 tag를 두고, `pop()`을 할 때마다 tag 변수를 증가시켜, (tag, ptr) 중 어느 하나라도 다르다면 CAS를 실패시키는 방법이 있다.

문제는 2개의 정보를 한번에 CAS에 넣어 검사할 수 있느냐인데...\
안타깝게도 [모든 64-bit 아키텍처가 Double-Width CAS 명령어를 지원하진 않아서](https://timur.audio/dwcas-in-c), [정렬된 16 byte `T`에 대해 `std::atomic<T>::is_always_lock_free` 체크를 통과하지 못하는 경우가 대부분이다.](https://godbolt.org/z/679YfGPMK)

그럼 이 방법을 안 되는건가? 싶지만 꼼수가 있다.\
대부분 64-bit 아키텍처에서, Page Table 수준에서 user-space 가상 주소의 범위를 제한하는데, user-space는 다들 상위 비트를 0으로 채워놓은 주소를 사용한다.\
이를 이용, pointer에서 미사용되는 상위 bit에 tag를 같이 욱여넣어, 64-bit CAS로 한방에 처리하는 꼼수를 쓸 수 있다.

그럼 아키텍쳐별로 미사용 상위 bit 너비를 알아야할텐데, 안타깝게도 프로그램 내부에서 이걸 얻어올 방법이 없다.\
Windows에서는 [`SYSTEM_INFO` 구조체](https://learn.microsoft.com/en-us/windows/win32/api/sysinfoapi/ns-sysinfoapi-system_info)를 통해 알아낼 수 있는데, 대응되는 Linux의 API는 찾지 못했다.\
Linux Kernel 쪽 코드를 보면 `VA_BITS` 매크로가 있긴 한데, 공개된 User-level API가 안 보인다.

어쩔 수 없이 유저가 값을 지정하도록 해야 하는데, 비교적 안전한 기본값을 잡아야겠다.\
일단 몇몇 아키텍처를 찾아본 결과:

* AMD64: [최대 56-bit user-space VA](https://www.kernel.org/doc/html/latest/arch/x86/x86_64/mm.html#complete-virtual-memory-map-with-5-level-page-tables)
* AArch64: [최대 52-bit user-space VA](https://docs.kernel.org/arch/arm64/memory.html)
* RISC-V SV57: [최대 56-bit user-space VA](https://docs.kernel.org/next/arch/riscv/vm-layout.html#risc-v-linux-kernel-sv57)

웬만하면 56-bit 초과로는 잘 안 쓰는 것 같으니, 56-bit를 기본값으로 잡았다.\
다시 말해, tag로 쓸 수 있는 값은 8-bit가 된다.

그런데 그러면 256번의 `pop()`만 일어나도, tag가 한바퀴를 돌아서 중복된 (tag, ptr)가 나와버릴 위험성이 있지 않을까?\
좀 더 생각해보면, `Node`도 `alignof(Node)`로 정렬을 맞추느라 하위 비트 일부도 0으로 채워져 있으니, 이 부분까지 사용할 수 있다.\
안에 `next` pointer가 들어있으니, 최소 3비트는 추가로 쓸 수 있다.

```cpp
#define NB_VA_BITS 56

class TaggedNodePtr {
private:
    std::uintptr_t _tagged_addr;

public:
    static constexpr std::size_t UPPER_TAG_BITS = 64 - NB_VA_BITS;
    static constexpr std::uintptr_t UPPER_TAG_MASK = ((std::uintptr_t(1) << UPPER_TAG_BITS) - 1) << NB_VA_BITS;

    static constexpr std::size_t LOWER_TAG_BITS = std::countr_zero(alignof(Node));
    static constexpr std::uintptr_t LOWER_TAG_MASK = alignof(Node) - 1;

    static constexpr std::uintptr_t TAG_MASK = UPPER_TAG_MASK | LOWER_TAG_MASK;

public:
    auto get_node_ptr() const noexcept -> Node* {
        return reinterpret_cast<Node*>(_tagged_addr & ~TAG_MASK);
    }
    
    auto get_tag() const noexcept -> std::uintptr_t {
        return ((_tagged_addr & UPPER_TAG_MASK) >> (NB_VA_BITS - LOWER_TAG_BITS))
                 | (_tagged_addr & LOWER_TAG_MASK);
    }

    void set_tag(std::uintptr_t tag_value) noexcept {
        const std::uintptr_t upper = (tag_value & (UPPER_TAG_MASK >> (NB_VA_BITS - LOWER_TAG_BITS)))
                                        << (NB_VA_BITS - LOWER_TAG_BITS);
        const std::uintptr_t lower = tag_value & LOWER_TAG_MASK;

        _tagged_addr = (_tagged_addr & ~TAG_MASK) | (upper | lower);
    }

    void increase_tag() noexcept {
        set_tag(get_tag() + 1);
    }
};
```

[실제 구현 코드](https://github.com/copyrat90/NetBuff/blob/main/include/NetBuff/TaggedPtr.hpp)는 템플릿으로 분리(`TaggedPtr<Node>`)하고, `static_assert` 체크와 `operator->` 오버로딩 등을 포함하고 있어서 좀 길다.

이제 `_node_head`의 타입을 `std::atomic<Node*>` 대신 `std::atomic<TaggedPtr<Node>>`로 교체하면, (tag, ptr)을 64-bit CAS 한 번에 수행할 수 있다.

### 실제 구현

```cpp
template <typename... Args>
[[nodiscard]] auto construct(Args&&... args) -> T& {
    TaggedPtr<Node> cur = _node_head.load();
    for (;;) {
        // if there's no unused node available, allocate a new block and try getting one again
        while (!cur) {
            add_new_block();
            cur = _node_head.load();
        }

        // if got the candidate `cur` node, prepare `next` node
        TaggedPtr<Node> next(cur->next);
        next.set_tag(cur.get_tag() + 1); // prevent ABA problem w/ increasing tag

        // try exchanging `_node_head` to `next`, and break if succeeds
        if (_node_head.compare_exchange_weak(cur, next))
            break;
    }
    ...
    if constexpr (CallDestructorOnDestroy) {
        ::new (static_cast<void*>(cur->data)) T(std::forward<Args>(args)...);
    }
    else {
        if (!cur->constructed) {
            ::new (static_cast<void*>(cur->data)) T(std::forward<Args>(args)...);
            cur->constructed = true;
        }
    }
    return reinterpret_cast<T&>(cur->data);
}

void destroy(T& obj) {
    Node& node = *reinterpret_cast<Node*>(reinterpret_cast<std::byte*>(&obj) - offsetof(Node, data));
    ...
    if constexpr (CallDestructorOnDestroy)
        obj.~T();

    TaggedPtr<Node> old_head = _node_head.load();
    TaggedPtr<Node> new_head(&node);
    for (;;) {
        // prepare `new_head`
        node.next = old_head.get_node_ptr();
        new_head.set_tag(old_head.get_tag());

        // try exchanging `_node_head` to `new_head`, and break if succeeds
        if (_node_head.compare_exchange_weak(old_head, new_head))
            break;
    }
    ...
}
```

# 전체 코드

[전체 코드](https://github.com/copyrat90/NetBuff/blob/lockfree-obj-pool/include/NetBuff/LockfreeObjectPool.hpp)

# 테스트

## 테스트 코드로 검증

이제 이걸 어떻게 테스트 할 수 있을까?

단순하게 시스템 논리 프로세서 개수만큼 스레드를 생성해 `construct()`와 `destroy()`를 마구 호출하는 걸로 시작할 수 있겠다.

그리고, 할당받은 Object에 현재 thread의 id를 써 놓은 다음, 잠시 뒤에 다시 확인해서 같은 thread id가 유지되고 있는지를 체크해 볼 수도 있다.\
만약 thread id가 수정됐다면, 다른 thread에도 중복으로 같은 Node를 할당해 준 버그일 것이다.

혹은, 미리 Object Pool에 일정량의 Object를 할당해놓고, 그 개수만큼을 모든 스레드가 나눠 할당받고, Object Pool의 capacity가 그것을 초과하는지를 검사할 수도 있겠다.

내 Object Pool 구현체의 경우, [Object Pool이 소멸될 때 반납하지 않은 Object가 남아 있다면 오류 로그를 남기는 코드](https://github.com/copyrat90/NetBuff/blob/1205d17a283f672f02898074687564a8ce06c5e8/include/NetBuff/LockfreeObjectPool.hpp#L132-L139)가 들어가 있어서, 그걸로 leak된 Node를 검사한다.

[테스트 코드](https://github.com/copyrat90/NetBuff/blob/lockfree-obj-pool/tests/lop_validate_automatic.cpp)

위 테스트 코드를 CTest를 이용해 AMD64와 AArch64 아키텍처에서 며칠간 돌려봤는데, 아직까지는 정상이다.

## [Sanitizer](https://github.com/google/sanitizers)

위 테스트 코드에 AddressSanitizer도 사용해봤는데, 일단은 문제가 없는 것으로 나온다.

ThreadSanitizer의 경우에는 data race가 난다고 뜨나, 모두 `cur->next` Node가 이미 사용자에게 Object로 할당된 경우로, 이 경우 어차피 CAS가 실패해 재시도하므로 상관없다.

*마지막 수정 : {{ page.last_modified_at }}*
