---
layout: post
title: Observer 패턴 with C++
tags: [Design Pattern]
author: copyrat90
last_modified_at: 2022-05-05T12:00:00+09:00
---

[(원문) Game Programming Patterns - Observer](http://gameprogrammingpatterns.com/observer.html)\
[(번역) 디자인 패턴 다시 보기 : 관찰자 패턴](https://www.hanbit.co.kr/channel/series/series_view.html?cms_code=CMS4660566726&hcs_idx=3)

게임 개발하다가 필요해서 위 링크된 글을 참고해 Observer 패턴을 구현해봤다.\
진행하며 사고한 과정을 정리해 글로 써본다.

# 프로젝트 설명

시리즈 첫글이니까 프로젝트 설명을 간략히 써야겠다.

요새 재미로 [Creepy Castle이라는 인디게임](https://store.steampowered.com/app/450440/Creepy_Castle/)을 [GBA](https://en.wikipedia.org/wiki/Game_Boy_Advance)로 포팅 중이다.\
GBA는 임베디드 시스템이니까 [I/O Register에 Address 넣는 방식으로 코딩하는 것...](https://en.wikipedia.org/wiki/Memory-mapped_I/O) **은 아니고.**\
그런 동작을 High-level API로 잘 추상화해놓은 [Butano engine](https://github.com/GValiente/butano)을 사용하는 중이다.

언어는 C++20을 사용하고, [Heap 메모리 사용은 최대한 피해야 하는 환경이다.](https://gvaliente.github.io/butano/faq.html#faq_containers)\
SDL/SFML급의 추상화된 API를 제공하지만, 하드웨어 제약사항을 항상 생각하며 코딩해야한다.\
(예를 들어, 할당해 쓸 수 있는 스택+힙 메모리가 고작 **288 KiB 뿐**이다...)

# Observer 패턴 도입 배경

## 커플링(coupling)

아래와 같이 주인공이 적을 때리는 상황을 생각해보자.\
(참고로 이건 원본 게임 스샷이다. 포팅본은 UI도 채 안 만들었다.)

![](/assets/img/posts/2022-05-05-observer-pattern-old/creepy-castle-attack.png)

적을 때리면, **우측 상단 UI**에 보이는 **몹 이름**과 **몹의 HP**가 업데이트된다.\
때리는 순간, **UI를 그리는 객체**가 **마지막으로 때린 몹 객체를 찾아** 이름과 HP를 읽어 UI를 다시 그려야 한다.

단순히 생각하면, 배틀 시스템 관련 코드에서 UI를 업데이트하는 함수를 호출할 수도 있다.\
하지만 그러면, **배틀 시스템**과 **UI**라는 서로 큰 관계가 없는 두 객체가 커플링(coupling)된다.\
이런 식으로 이것저것 커플링하다보면, 게임 시스템 전체가 서로서로를 호출하는 엉망진창 스파게티 구조가 될 것이다.

## 커플링 제거

이럴 때 Observer 디자인 패턴을 도입하면, 커플링을 제거할 수 있다.\
우선 배틀 시스템은 UI는 신경쓰지 않고, 몹 객체의 HP를 떨어뜨리는 함수를 호출한다.\
해당 몹 객체는 자신의 상태가 변화하였으므로, 멤버 함수 `notify()`를 호출해 자신의 상태가 변했음을 알린다.\
UI 객체는 모든 몹 객체를 관찰하고 있다가, `notify()`가 호출되어 HP 떨어진 몹을 알게 되면, 이에 맞춰 UI를 다시 그린다.

아니 그러면 모든 몹 객체랑 UI가 커플링된거 아닌가? 싶을 것이다.\
몹 객체는 UI 클래스의 구조 전체를 아는 것이 아니라, 자신의 `notify()`가 호출될 때 UI에서 호출해야 하는 **이벤트 핸들러 멤버 함수 딱 1개만을 아는 것**이다.\
나중에 코드를 보면 알겠지만, Enemy class는 UI class를 include 하지 않고, IObserver 인터페이스만 include한다.


# Observer 패턴 구현

우선, 각 몹 객체는 `IObservable`을 상속받는다. ([GPP 책](http://gameprogrammingpatterns.com/observer.html)의 Subject이다. 이름은 [C#에서 따와 바꿨다.](https://docs.microsoft.com/en-us/dotnet/api/system.iobservable-1?view=net-6.0))\
그리고 UI는 `IObserver`를 상속받는다. (얘는 그대로 Observer이다.)

`IObserver`는 관찰하고 싶은 `IObservable`에 자기 자신을 등록한다. 이를 구독한다고 한다.\
`IObservable`은 위처럼 자신을 구독중인 `IObserver`들을 연결 리스트로 저장해둔다.\
그리고 자신의 상태가 변하면(즉, 이벤트가 발생하면), 멤버 함수 `notify()`를 호출한다.\
`notify()`는 연결 리스트를 순회하며 모든 관찰자의 `onNotify()` 함수를 호출한다(Callback).

## 0차 시도

맨 처음으로는 사실 인터페이스가 아니라 멤버 함수를 `std::function<void(EventArgs)>`을 저장하는 연결 리스트에 람다로 묶어 저장하도록 짰는데...\
이상하게 람다식을 컨테이너에 저장해뒀다 호출하는 코드가 GBA를 하얗게 질리게 만들었다.\
PC 환경에서는 작동하는 코드인데...\
어쨌든 안되는걸 어쩌랴, 그냥 [GPP 책](http://gameprogrammingpatterns.com/observer.html)의 방법을 따라 인터페이스로 만들기로 했다.

## 1차 시도

[1차 시도 GitHub 커밋](https://github.com/copyrat90/creepy_castle_gba/tree/f8a737a16c40225705c0355ede7b2c5143a5b9c7)

일단 위에 써놓은 그대로 구현했는데, [`notify(EventArg e)`로 인자를 고정시켰다.](https://github.com/copyrat90/creepy_castle_gba/blob/f8a737a16c40225705c0355ede7b2c5143a5b9c7/include/event/IObservable.h#L31)\
그리고 [연결 리스트를 `bn::list<IObserver*, MAX_OBSERVERS>` 로 선언](https://github.com/copyrat90/creepy_castle_gba/blob/f8a737a16c40225705c0355ede7b2c5143a5b9c7/include/event/IObservable.h#L38)했다.\
(연결 리스트인데 최대 개수가 제한되는 이유는, [Butano 제공 컨테이너는 환경상 Heap을 안쓰기 때문이다.](https://gvaliente.github.io/butano/faq.html#faq_containers))

한가지 GPP 글과 다르게 구현된 점은, [`IObserver`가 `IObservable`의 포인터와 연결 리스트 반복자까지 저장하고 있다는 점이다.](https://github.com/copyrat90/creepy_castle_gba/blob/f8a737a16c40225705c0355ede7b2c5143a5b9c7/include/event/IObserver.h#L26)\
이래야만 특정 대상을 구독 해제하고 싶을 때, 반복자를 써서 $O(1)$에 해제할 수 있기 때문.

[이 코드는 이런 식으로 사용된다.](https://github.com/copyrat90/creepy_castle_gba/blob/f8a737a16c40225705c0355ede7b2c5143a5b9c7/src/tests.cpp#L98)

### 문제점 1. `notify(EventArg e)`로 인자가 고정됨

`EventArg`는 단순 enum이다.\
마치 SIGNAL처럼, 그저 어떤 이벤트가 발생했는지만 알 수 있을 뿐, 상세 정보는 알 수가 없다.\
(예를 들어 데미지량, 몹 이름 등을 알아낼 수가 없다.)\
게다가 게임의 모든 SIGNAL이 전부 `EventArg`이므로, [enum 하나가 무진장 비대해진다.](https://github.com/copyrat90/creepy_castle_gba/blob/f8a737a16c40225705c0355ede7b2c5143a5b9c7/include/event/EventArg.h#L6)

그렇다면 `EventArg`를 class로 만들고 파생 클래스를 통해 다양한 정보를 전달받는 방법은 어떨까?\
이러면 어떤 이벤트가 발생했는지 알려면 `dynamic_cast`를 해야하고, 파생 클래스가 우후죽순 난립할 수 있는게 문제다.

이 문제점을 해결하려면, `EventArg` 타입을 고정하는 대신 템플릿 인자`<EArg>`로 전달해\
`IObserver<EArg>, IObservable<EArg>` 처럼 하면 된다. [C#이 이렇게 하고 있다.](https://docs.microsoft.com/en-us/dotnet/api/system.iobserver-1?view=net-6.0)\
이러면 상황에 따라 `EArg`를 enum으로 할 수도, class로 할 수도 있다.

### 문제점 2. `bn::list<T, MaxSize>`의 크기 상한 고정

몇번이고 말했듯이 [Butano 컨테이너는 Heap을 안 써서, 크기가 고정](https://gvaliente.github.io/butano/faq.html#faq_containers)이다.\
이로 인해 2가지 문제점이 생긴다.

1. [관찰자가 저장 중인, 대상의 포인터와 연결 리스트 반복자 목록](https://github.com/copyrat90/creepy_castle_gba/blob/f8a737a16c40225705c0355ede7b2c5143a5b9c7/include/event/IObserver.h#L26)의 크기가 고정이라,
모든 관찰자가 고정 크기의 대상보다 많이 관찰을 못한다.
2. [대상이 저장 중인, 관찰자들의 연결 리스트](https://github.com/copyrat90/creepy_castle_gba/blob/f8a737a16c40225705c0355ede7b2c5143a5b9c7/include/event/IObservable.h#L24)의 크기가 고정이라, 모든 대상이 고정 크기 이상의 관찰자에게 관찰당할 수 없다.

크기를 작게 잡으면, 게임 내 수많은 몹들을 담아야하는 UI 클래스 관찰자가 공간 부족으로 터질 것이고,\
크기를 크게 잡으면, 끽해야 1~2개 저장하는 관찰자도 메모리를 심하게 잡아먹을 것이다.

대상도 관찰자의 리스트를 관리하고, 관찰자도 대상의 리스트를 관리하므로, intrusive list는 사용 불가능하다.\
이 문제를 해결하려면, 리스트 노드 Object Pool을 만들어 노드를 재사용해야하는데... 너무 복잡해진다.\
일단 이 문제의 해결은 당장은 보류해야겠다. 메모리 부족해지면 그때 가서 하지 뭐...

## 2차 시도

[2차 시도 GitHub 커밋](https://github.com/copyrat90/creepy_castle_gba/tree/db4deacbf53c7adc49bd39687c035a747e74cc93)

1차 시도의 문제점 1을 해결.\
[`IObservable<EArg>, IObserver<EArg>`처럼 템플릿으로 고쳤다.](https://github.com/copyrat90/creepy_castle_gba/blob/db4deacbf53c7adc49bd39687c035a747e74cc93/include/event/IObserve.h#L20)

템플릿이라서 구현부를 헤더로 합쳐야 하는데, 두 개의 템플릿이 서로의 멤버 함수를 참조해야 하기 때문에,\
하나만 쓰고 싶어도 무조건 둘 다 include 해야해서, 그냥 2개 헤더를 하나로 합쳤다.

이제 `EArg`로 enum뿐만 아니라 class도 받을 수 있어, 다양한 형태의 이벤트 정보를 전달하는 것이 가능해졌다.

게다가 서로 다른 `EArg`를 갖는 여러 대상을 관찰하는 것도 가능하다.\
`IObserver<A>, IObserver<B>`를 [다중 상속받아 관찰](https://github.com/copyrat90/creepy_castle_gba/blob/db4deacbf53c7adc49bd39687c035a747e74cc93/src/tests.cpp#L36)하면 된다.\
다만 이때는 [`observe()` 함수가 자동적으로 오버로딩되지는 않기 때문](https://stackoverflow.com/questions/51690394/overloading-member-function-among-multiple-base-classes)에, [이런 식으로 using 선언을 해줘야한다.](https://github.com/copyrat90/creepy_castle_gba/blob/db4deacbf53c7adc49bd39687c035a747e74cc93/src/tests.cpp#L40)

비슷하게, 하나의 대상이 여러 `EArg`의 이벤트를 발생시키도록 다중 상속할 수도 있을 것이다.\
다만, `A`, `B`타입의 이벤트를 발생시키는 대상을 `A`, `B`타입의 이벤트를 관찰하는 관찰자로 관찰하는 것은\
`observe()`가 모호한 호출이 되므로 불가능하다.

### 문제점 1. 대상을 삭제하는데 $O(N)$ 소요

관찰자가, 대상의 관찰자 리스트의 반복자를 갖고 있으므로, 관찰자의 삭제는 $O(1)$에 가능하다.\
하지만, 이를 위해 관찰자는 대상의 반복자와 포인터를 별도 리스트에 관리해야하고, 대상이 삭제될 때는 이 2개를 모두 삭제해야 한다.\
그러려면 인덱싱이 필수여서, `bn::vector`를 사용하게 되고, 배열의 원소를 찾아 지우는 건 $O(N)$이 소요된다.


# 실제 게임 시스템에 적용

(나중에 시스템 구현에 적용되면 수정할 예정)

*마지막 수정 : {{ page.last_modified_at }}*
