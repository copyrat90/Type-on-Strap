---
layout: post
title: Fix your timestep! 정리
tags: [Gamedev]
author: copyrat90
last_modified_at: 2022-11-16T12:00:00+09:00
---

게임에서 Frame rate 에 관한 문제가 크게 2가지가 있다.
1. 게임을 일정한 속도로 진행시켜야 한다는 문제.
1. 게임의 Frame rate 제한을 두어 CPU를 불태우지 않도록 하는 문제.

이 중 1번 문제에 대해서는 아래의 유명한 글에서 다룬다.\
<https://gafferongames.com/post/fix_your_timestep/>

이 글에서는 위 링크된 글을 토대로 1번 문제를 그림과 함께 살펴보도록 한다.\
2번 문제에 대해서는 간단히 살펴보고, 추후에 글을 업데이트 하든 해야겠다.

* 여기서 timestep 이란, delta time 을 의미한다.
* 예시 코드 스니펫은 C++20, [SFML](https://www.sfml-dev.org)을 사용한다.


# 필요성
```cpp
// 게임 루프
while (game.isRunning()) {
    game.processEvents();
    game.update();
    game.render();
}
```

위 게임 루프의 경우, 컴퓨터의 성능에 따라 게임 루프가 다른 속도로 실행될 것이다.\
그 결과, 성능이 빠른 컴퓨터에서는 게임이 빠르게 진행되고, 

게임이 환경에 따라 서로 다른 속도로 진행되는 문제가 있으니, 뭔가 조치가 필요하다.

# 1. Variable delta time
## 코드
```cpp
sf::Clock clock;

while (game.isRunning()) {
    // sf::Clock::restart() 는 해당 타이머의 경과 시간을 반환하고, 재시작
    sf::Time frameTime = clock.restart();
    game.processEvents();
    game.update(frameTime); // 경과 시간에 맞는 상태 업데이트 진행
    game.render();
}
```

## 설명
한 번의 게임 루프를 도는 동안, 경과 시간을 `frameTime` 으로 계산해서 `Game::update(frameTime)` 으로 전달한다.\
전달받은 경과 시간을 토대로 적은 시간이 흘렀으면 조금 이동하고, 많은 시간이 흘렀으면 많이 이동하는 식으로 게임을 업데이트 하면 될 것이다.

## 문제점
게임 루프가 엄청 느리게 실행되는 바람에, `frameTime` 이 5초나 걸렸다고 해보자.\
`이동거리 = 속도 * frameTime` 으로 이동 처리를 한다면, 위치가 너무 많이 달라져서\
중간에 벽이 있어도 무시하고 그대로 뚫고 넘어갈 수 있는 문제가 발생할 수 있다.

(그림)

저런 극단적인 경우가 아니더라도, 경과 시간에 따라 미묘하게 게임 물리가 다르게 느껴질 상황이 많을 것이다.

# 2. Semi-fixed timestep
## 아이디어
한 번의 게임 루프 진행시에,\
게임의 논리적 상태를 일정 단위(`SEC_PER_FRAME`)로 나눠서 업데이트한다.

예를 들어 이번 게임 루프에 `frameTime == 2.7/60` 만큼 진행시켜야 하면, 아래처럼 3번 나눠서 업데이트를 수행한다.
1. `deltaTime == 1/60 (SEC_PER_FRAME)` 주고 업데이트
1. `deltaTime == 1/60` 주고 업데이트
1. `deltaTime == 0.7/60` 주고 업데이트

그 뒤에 화면을 1번 그린다.

(N-1)번까지는 `SEC_PER_FRAME` 으로 timestep 이 고정되지만,\
마지막 1번은 그것보다 적은 timestep 으로 업데이트가 수행되므로 **반쪽짜리 고정 타임스텝** *Semi-fixed timestep* 이라 부르는 듯하다.

## 코드
```cpp
const int      FPS           = 60;
const sf::Time SEC_PER_FRAME = sf::seconds(1.f / FPS);

sf::Clock clock;

while (game.isRunning()) {
    sf::Time frameTime = clock.restart();
    // 죽음의 소용돌이 임시방편 해결책, 아래 주석을 해제해 frameTime 상한을 둔다.
    // frameTime = std::min(frameTime, sf::seconds(0.25f));
    while (frameTime > sf::seconds(0.f)) {
        sf::Time deltaTime = std::min(frameTime, SEC_PER_FRAME);
        frameTime -= deltaTime;
        game.processEvents();
        game.update(deltaTime);
    }
    game.render();
}
```

## 장단점
이제 각 `update()` 에 전달되는 `deltaTime` 의 상한이 생겼다.\
게임 상태를 1번 업데이트 할 때는 아무리 오래 걸려도 `SEC_PER_FRAME` 를 초과하지 않는다는 말.

대신에 1번의 게임 루프 실행 시에 `update()` 가 여러 번 호출될 수 있다.\
만일 update 가 render 보다 무거운 작업이라면, 성능이 안 좋아질 수 있다.\
심할 경우 아래에서 설명할 죽음의 소용돌이 *spiral of death* 문제가 발생할 수 있다.

## 문제점과 해결

### 죽음의 소용돌이
1번의 게임 루프를 처리한다고 하자.\
여러 번의 update 를 통해 게임의 물리 상태를  `X` 만큼 진행시켰다.\
그런데 그 모든 물리 업데이트를 처리하는 데 `X` 보다 더 긴 시간 `Y` 가 걸린다면?

다음 번 루프에는 `Y` 시간의 물리 업데이트를 처리해야 할 것이다.\
마찬가지로 그걸 처리하는 데도 `Y` 보다 더 긴 시간이 걸릴 것이고...\
이런 식으로 해야 할 물리 처리 `frameTime` 이 매 루프마다 점점 불어나면\
결국 무한히 지연될텐데, 이를 **죽음의 소용돌이** *spiral of death* 라고 부른다.

### 해법
이걸 해결하는 궁극적인 방법은 `X` 만큼의 게임 물리 상태 업데이트하는데\
`X`보다 훨씬 적은 시간이 들도록 최적화를 잘 하는 것이다.\
그러면 어쩌다 순간적인 랙이 발생하더라도 금방 물리 업데이트가 쫓아갈테니 괜찮다.

임시방편의 해결책도 있다.\
`frameTime` 에 상한을 둬서 최대 지연 시간을 제한하는 것이다.\
게임이 아예 멈추는 것보다야 잠깐씩 멈칫거리는 게 나을테니...

위 코드에서 주석 처리 해놓은 코드를 해제하면 이 임시방편 해결책이 적용된다.
 

# 3. Free the physics (Hard-fixed timestep)
## 필요성
위의 Semi-fixed timestep 도 실제로 자주 쓰이지만, 아쉬운 점이 있다.\
마지막 1번의 update 의 시간이 일정하지가 않아서, 같은 입력에 대해 항상 같은 결과를 보장하지 못한다는 점이다.\
만일 네트워크 동시 플레이를 지원해야 한다면, 물리 상태를 동기화하기 난해할 수 있다.

## 아이디어
위의 Semi-fixed timestep 에서 Semi- 에 해당하는 부분을 제거한다.

(N-1)번 업데이트 후 마지막 1번의 update 를 처리하기엔 잔여시간이 `SEC_PER_FRAME` 보다 적을 것이다.\
이 잔여시간을 Semi- 에서는 전부 소모했다면, 이번 Hard- 에서는 **남겨놓고 다음 루프에 누적하여 처리하게 한다.**\
간단히 말해 애매한 잔여시간을 다음 루프로 넘겨버리는 것이다.

같은 예시를 들면, `frameTime == 2.7/60` 를 진행시킬 때, 아래처럼 2번만 업데이트를 수행한다.
1. `deltaTime == 1/60 (SEC_PER_FRAME)` 주고 업데이트
1. `deltaTime == 1/60` 주고 업데이트

남은  `0.7/60` 초는 이번엔 진행을 포기하고, 다음 게임 루프로 누적해 넘겨서 처리하게 한다.

## 코드
```cpp
const int      FPS           = 60;
const sf::Time SEC_PER_FRAME = sf::seconds(1.f / FPS);

sf::Clock clock;
// 지연시간을 누적할 변수
sf::Time timeSinceLastUpdate = sf::Time::Zero;

while (game.isRunning()) {
    timeSinceLastUpdate += clock.restart();
    // 죽음의 소용돌이 임시방편 해결책
    timeSinceLastUpdate = std::min(timeSinceLastUpdate, sf::seconds(0.25f));
    while (timeSinceLastUpdate > SEC_PER_FRAME) {
        timeSinceLastUpdate -= SEC_PER_FRAME;
        game.processEvents();
        game.update(SEC_PER_FRAME);
    }
    game.render();
}
```

위 코드를 보면 `game.update(SEC_PER_FRAME)` 처럼, **완전히 고정된** *Hard-fixed* 델타 타임으로 업데이트 됨을 알 수 있다.

# The final touch

## 필요성
위의 Hard-fixed timestep 을 그대로 써도 무방하나 여전히 아쉬운 점이 있다.\
덜덜거리는 움직임 *stutter*

(수정 중)

## 아이디어

## 코드


# 참고 자료

* [Gaffer On Games - Fix Your Timestep!](https://gafferongames.com/post/fix_your_timestep/)

*마지막 수정 : {{ page.last_modified_at }}*
