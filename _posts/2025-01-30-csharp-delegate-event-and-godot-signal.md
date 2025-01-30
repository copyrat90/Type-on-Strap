---
layout: post
title: C#의 `delegate`와 `event`, 그리고 Godot의 Signal
tags: [C#, Godot]
author: copyrat90
last_modified_at: 2025-01-30T12:01:00+09:00
---

C#에서 Observer pattern을 구현하는 방법과, Godot에서 그걸 활용해 Signal handling하는 방법.

# `delegate`

C#에서 메서드를 저장하는 타입을 선언하는 방법.\
C++의 [`std::function`](https://en.cppreference.com/w/cpp/utility/functional/function)과 비슷하지만, 차이점도 있다.
```cs
// int를 반환하고 (int, int)를 매개변수로 받는 delegate 타입 선언
delegate int Calculation(int, int)
...
// 위 Calculate 타입의 변수 선언
Calculate myCalc = null;
```

아래는 더 자세한 사용 예시.
```cs
class Program
{
    // int를 반환하고 (int, int)를 매개변수로 받는 delegate 타입 선언
    delegate int Calculation(int a, int b);

    // 예시용 정적 메서드
    static int Add(int x, int y)
    {
        return x + y;
    }

    class TrapCard
    {
        int _data;
        public TrapCard(int data)
        {
            _data = data;
        }

        // 예시용 멤버 함수
        public int Calc(int a, int b)
        {
            return _data;
        }
    }

    static void Main()
    {
        Calculation myCalc;

        // 정적 메서드 저장한 경우
        myCalc = Add;
        Console.WriteLine($"7 + 5 = {myCalc(7, 5)}");

        // 익명 함수 저장한 경우
        // (여기서의 delegate 키워드는 익명 함수 선언용으로, 위 delegate 타입 선언과는 다름)
        myCalc = delegate (int x, int y)
        {
            return x - y;
        };
        Console.WriteLine($"7 - 5 = {myCalc(7, 5)}");

        // 람다식 저장한 경우
        // (람다식은 익명 함수를 좀 더 쉽게 만드는 문법)
        myCalc = (int i, int j) =>
        {
            return i * j;
        };
        // 참고로 명령 1개 반환하는 람다식은 이렇게 축약해서 쓸 수도 있음
        myCalc = (i, j) => i * j;

        Console.WriteLine($"7 * 5 = {myCalc(7, 5)}");

        // 특정 객체의 메서드 호출을 저장
        // (이 경우, 호출 전에 해당 객체가 사라지면 큰일남)
        TrapCard trap = new TrapCard(457);
        myCalc = trap.Calc;

        Console.WriteLine($"7 trap 5 = {myCalc(7, 5)}");
    }
}
```
실행 결과:
```
7 + 5 = 12
7 - 5 = 2
7 * 5 = 35
7 trap 5 = 457
```

그리고 1개의 delegate 변수에 `+=` 연산자로 여러 개의 메서드를 연결하는 것도 가능하다.\
그러면 호출 시에 추가된 순서대로 전부 호출된다.\
C++ 이었으면 `intrusive_list<std::function>` 식으로 썼어야 했을 터.
```cs
class Program
{
    delegate void Printer();

    static void Main()
    {
        Printer printer = () => Console.WriteLine("Woof!");

        static void meow() { Console.WriteLine("Meow!"); }
        // += 로 지역 함수를 추가 연결했음
        printer += meow;

        // 호출 시 위 2개 함수가 모두 호출
        printer();

        // -= 로 지역 함수를 등록 해제
        printer -= meow;

        // 이제 "Woof!" 출력만 호출됨
        printer();
    }
}
```
결과:
```
Woof!
Meow!
Woof!
```

# `event`
위 delegate 를 사용하는 일반적인 패턴은\
`Subject` 클래스에서 public 으로 delegate 변수를 하나 노출하고,\
`Observer` 클래스에서 그 delegate 변수 호출 시 호출될 handler 메서드를 등록하는 식이다.

관찰당하는 `Subject`는 `Observer`라는 구체 클래스를 모르는 상태에서도 `Observer`가 등록한 handler 메서드를 호출할 수 있다.\
이걸 [*observer pattern*](https://gameprogrammingpatterns.com/observer.html)이라고 부른다.

```cs
class Program
{
    class Subject
    {
        // Action 타입의 delegate 변수 선언.
        // (참고로, Action<Param1, Param2, ...> 은 미리 정의된 반환값 없는 형식의 delegate)
        // (미리 정의된 반환값 있는 Func<Param1, Param2, ..., Result> 형식 delegate 도 존재)
        public Action DiedInAMinute;

        public void Die(int seconds)
        {
            if (seconds <= 60)
            {
                // ?. 연산자를 쓰면 null 아닐 때만 호출되게 할 수 있음
                DiedInAMinute?.Invoke();
            }
        }
    }

    class Observer
    {
        public Observer(Subject subject)
        {
            subject.DiedInAMinute += OnDiedInAMinute;
        }

        void OnDiedInAMinute()
        {
            Console.WriteLine("Achievement unlocked: Superfast death");
        }
    }

    static void Main()
    {
        Subject subject = new Subject();
        Observer observer = new Observer(subject);

        // subject 가 조건에 따라 내부 `DiedInAMinute` 변수에 연결된 메서드(들)을 호출함
        subject.Die(5);
    }
}
```

그러나 위처럼 delegate 변수를 public 으로 공개해 버리면,\
`Subject` 가 아닌 클래스에서 delegate 변수를 호출시켜버릴 위험성이 존재한다.
```cs
// 비정상적 방식으로 강제 호출 (public 필드라서 가능)
subject.DiedInAMinute();
```

이럴 때 delegate 변수 앞에 `event` 키워드를 앞에 붙이면 public 이어도 `Subject` 에서만 호출 가능하도록 강제할 수 있다.
```cs
public event Action DiedInAMinute;
```

# Godot 의 Signal

고도 엔진에서는 C#의 delegate 와 event 를 활용하여\
[엔진 내부의 Signal 을 받아 handling 할 수 있도록 하였다.](https://docs.godotengine.org/en/stable/tutorials/scripting/c_sharp/c_sharp_signals.html)

또한 직접 커스텀 Signal을 만들 수도 있는데, 이땐 좀 특이하게\
`[Signal]` Attribute를 붙인 delegate 타입 하나만 정의해야 하고, 변수명 뒤를 `EventHandler`로 끝내야 한다.
```cs
[Signal] public delegate void DiedInAMinuteEventHandler();
```

그러면 Godot의 빌드스크립트(?)가 알아서 그 위치에 뒤 `EventHandler`를 뗀 event delegate 변수를 생성해 줘,\
외부에서는 그 변수에 handler를 등록할 수 있다.
```cs
// Godot 빌드스크립트(?)가 자동 생성한 이벤트 변수, 직접 작성할 필요 없음
private global::DiedInAMinuteEventHandler backing_DiedInAMinute;
public event global::DiedInAMinuteEventHandler @DiedInAMinute {
        add => backing_DiedInAMinute += value;
        remove => backing_DiedInAMinute -= value;
}
```
(코드 생성이 정확히 어떤 원리인지는 모르겠다.\
Reflection을 쓸 것으로 추측되긴 하는데, 어떻게 빌드조차 안 했는데 바로 생성되지?)

## Signal emission

이렇게 정의한 Signal을 발생시키려면\
[C#식 Invoke는 못 쓰고, `EmitSignal(SignalName.DiedInAMinute);` 식으로 호출해야 한다.](https://docs.godotengine.org/en/stable/tutorials/scripting/c_sharp/c_sharp_signals.html#signal-emission)

## Signal await

메서드 실행 도중에 [특정 Signal 이 오기를 대기했다가 이어서 실행하려면 `await ToSignal(..)`을 쓸 수 있다.](https://docs.godotengine.org/en/stable/tutorials/scripting/c_sharp/c_sharp_signals.html#signals-as-c-events)

```cs
using Godot;

public partial class Main : Node
{
    Button _button;
    Label _label;

    int _counter = 0;

    public override void _Ready()
    {
        _button = GetNode<Button>("Button");
        _label = GetNode<Label>("Label");
        // Button에 들어있는 Pressed Signal에 `OnButtonPressed` handler 등록
        _button.Pressed += OnButtonPressed;
    }

    // Signal handler는 Task 반환이 불가능해서, 어쩔 수 없이 void 반환
    async void OnButtonPressed()
    {
        _label.Text = $"Pressed button for...";
        // 0.5초짜리 SceneTreeTimer 만들고, Timeout Signal을 대기
        await ToSignal(GetTree().CreateTimer(0.5f), SceneTreeTimer.SignalName.Timeout);
        _label.Text = $"Pressed button for... {++_counter} times!";
    }
}
```

그런데 가만, 위 예제에서 `await`을 사용하는데, 왜 별도의 동기화가 필요 없을까?

보통 C#에서 await 문이 [`TaskAwaiter`](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.taskawaiter?view=net-8.0)를 받는 것과 달리,\
Godot의 Signal을 받을 때는 [`Godot.SignalAwaiter`](https://haxegodot.github.io/godot/godot/SignalAwaiter.html)를 받는다.

보통 C# await에서 `TaskAwaiter`를 받을 때는 장치 I/O를 대기하는 경우가 많은데,\
그 경우 장치 I/O가 끝나면 (Windows 기준) [C# 런타임이 갖고 있는 IOCP 내 worker thread pool에서 I/O Completion Packet을 처리한다.](https://blog.stephencleary.com/2013/11/there-is-no-thread.html)\
그 말은, `await` 이후에 재개하는 녀석은 IOCP의 worker thread라는 것으로, **I/O 요청을 건 thread와 재개하는 thread가 다를 수 있다는 말이다.**\
(이걸 확인해보고 싶다면, `await Task.Delay(..)` 앞 뒤로 `CurrentThread.ManagedThreadId`를 출력해보라. 값이 다를 것이다.)\
따라서, **상황에 따라 thread간 동기화가 필요할 수 있다.**

반면, 고도 엔진에서는 기본적으로 게임 루프를 돌리는 main thread가 Signal emission도 담당하는 것으로 보인다.\
이 말은, main thread에서 Signal을 발생시켰다면 `await ToSignal(..)`로 **대기가 끝난 이후에도 같은 thread라서 동기화가 필요없다는 말이다.**\
그래서 인터넷에 도는 예제들 대부분이 `await`을 해 놓고도 별도의 동기화를 사용하지 않는 것으로 생각된다.

# Lapsed listener problem

만일 [Signal 을 받는 listener가 `+=` 을 해 놓은 상태에서 사라져버리면, handler가 해제되지 않는 leak이 발생한다.](https://docs.godotengine.org/en/stable/tutorials/scripting/c_sharp/c_sharp_signals.html#no-automatic-disconnection-a-custom-signal)\
따라서, **listener가 먼저 사라지는 경우, 반드시 사라지기 전에 `-=` 으로 handler를 제거해줘야 한다.**\
아니면 `+=` 대신에 `Target.Connect(MyClass.SignalName.MySignal, Callable.From(OnMySignal));` 식으로 등록하면 자동 해제된다고 한다.

이게 얼마나 흔한 실수인지, 아예 [*Lapsed listener problem*](https://en.wikipedia.org/wiki/Lapsed_listener_problem)라고 위키피디아 항목까지 있다.


*마지막 수정 : {{ page.last_modified_at }}*
