---
layout: post
title: 픽셀 단위 떨림 셰이더 만들어보기 (feat. Gato Roboto)
tags: [Shader]
author: copyrat90
last_modified_at: 2022-12-11T12:00:00+09:00
---

최근 [Gato Roboto](https://www.gatoroboto.com/) 라는 [메트로배니아 게임](https://namu.wiki/w/메트로배니아)을 했는데,\
간단한 후처리로 그럴듯한 그래픽 효과를 준 게 나름 인상적이었다.

예를 들어 뜨거운 용암 지역에서, 아래와 같이 화면 전체를 픽셀 단위로 떨리게 하여\
열기가 올라오는 느낌을 살린 부분이 있었다.

![떨림 효과 있는 게임 캡쳐](/assets/img/posts/2022-12-11-heat-haze-frag-shader/01.gif)

같은 방을 용암을 식히고 오면 아래와 같이 떨림 효과가 사라진다.

![떨림 효과 없는 게임 캡쳐](/assets/img/posts/2022-12-11-heat-haze-frag-shader/02.gif)


# 궁금증

그런데 이런 효과는 어떻게 구현하는 걸까?

첫번째 gif 영상을 다시 보면,\
UI를 제외한 화면 내 모든 개체(플레이어, 타일맵)에 떨림 효과가 일괄 적용되고 있다.

두번째 gif 영상에 해당하는 프레임을 먼저 텍스처에 렌더링 한 후,\
실제 화면에 렌더링시에 그 텍스처를 읽으며,\
`gl_FragCoord.y`에 따라 프래그먼트를 좌측/우측으로 밀어주는 식으로 구현하면 어떨까?

# 기술

## 기술 스택

간단하게 프래그먼트 셰이더 후처리만 만들고 싶어서, C++ 멀티미디어 라이브러리인 [SFML](https://www.sfml-dev.org/)을 썼다.

[SFML](https://www.sfml-dev.org/)은 Simple and Fast Multimedia Library 의 약자로,\
여러 운영체제(Windows/MacOS/Linux...)의 GUI 윈도우(창)와 키보드/조이스틱 입력 등을 추상화한다.\
간단한 2D 게임 (및 게임 엔진)을 제작하는 데 쓸만하다.

비슷한 기능을 하는 것들로는 [SDL2](https://www.libsdl.org/), [raylib](https://www.raylib.com/) 등이 있지만 둘 다 C언어 라이브러리라 SFML을 사용.


## 렌더링 패스(초간단 요약)

SFML을 써서 C++ 코드로 화면에 Sprite를 배치하면,\
SFML이 알아서 적절한 내장 셰이더를 사용해, 요청한 위치에 그림을 그려준다.

1. 첫번째 패스는, SFML을 이용해 게임 장면을 1차 렌더링.
2. 두번째 패스는, 위에서 렌더링된 장면을 SFML을 이용해 화면 중앙에 배치.
   직접 작성한 프래그먼트 셰이더로 후처리.

거칠게 요약하면 아래의 렌더링 패스를 거치게 될 것이다.

||Vertex shader|→|Fragment shader|
|---|---|---|---|
|1st pass|SFML 내장 정점 셰이더|→|SFML 내장 프래그먼트 셰이더|
|2nd pass|SFML 내장 정점 셰이더|→|**직접 작성한 프래그먼트 셰이더(`heat_haze.frag`)**|

# 구현

## 그래픽 준비

위 gif 영상을 [`UI`, `Tilemap` 과 `Player` 의 3부분으로 쪼갰다.](https://github.com/copyrat90/sfml-practice/tree/699d262bd2baaef8377c04c9cf27c4ff1cfc04ca/assets/graphics)\
커스텀 셰이더는 `Tilemap`과 `Player`에만 적용돼야 할 것이다.

![](/assets/img/posts/2022-12-11-heat-haze-frag-shader/graphic-area-analysis.png)

`Player`의 각 애니메이션 프레임도 가져왔다.\
(실제 sprite sheet는 세로 방향이지만, 본문 스크롤을 줄이기 위해 가로 방향으로 표시.)

![](/assets/img/posts/2022-12-11-heat-haze-frag-shader/gato-roboto-sprite-sheet.png)

## 1st Pass

### 코드

SFML만 가지고 작업한 부분이다. ([GitHub 에서 commit 보려면 클릭](https://github.com/copyrat90/sfml-practice/tree/71ce65fa6065ae7590ac5e267f9ca9e1167c4ab7))

우선, [Texture를 로딩](https://github.com/copyrat90/sfml-practice/blob/71ce65fa6065ae7590ac5e267f9ca9e1167c4ab7/src/Game.cpp#L58-L60)한다. ([gr::TextureHolder 구현](https://github.com/copyrat90/sfml-practice/blob/71ce65fa6065ae7590ac5e267f9ca9e1167c4ab7/src/ResourceId.hpp#L26), [gr::ResourceHolder 구현](https://github.com/copyrat90/sfml-practice/blob/71ce65fa6065ae7590ac5e267f9ca9e1167c4ab7/src/ResourceHolder.hpp#L15))
```cpp
void Game::_loadResources() {
    _textureHolder.loadFromFile(TextureId::PLAYER, "assets/graphics/player.png");
    _textureHolder.loadFromFile(TextureId::TILEMAP, "assets/graphics/tilemap.png");
    _textureHolder.loadFromFile(TextureId::UI, "assets/graphics/ui.png");
}
```

[Texture로부터 Sprite를 생성](https://github.com/copyrat90/sfml-practice/blob/71ce65fa6065ae7590ac5e267f9ca9e1167c4ab7/src/Game.cpp#L21-L29)하고, Player를 적절한 위치로 이동시킨다.\
(애니메이션을 위해 [Player는 별도 클래스로 구현](https://github.com/copyrat90/sfml-practice/blob/71ce65fa6065ae7590ac5e267f9ca9e1167c4ab7/src/Player.hpp#L9)했다.)
```cpp
Game::Game() : _window(sf::VideoMode(WINDOW_WIDTH, WINDOW_HEIGHT), "Heat Haze Shader Example") {
    ...
    _loadResources();

    // 각 sprite의 texture 초기화
    _tilemapSprite.setTexture(_textureHolder.get(TextureId::TILEMAP), true);
    _uiSprite.setTexture(_textureHolder.get(TextureId::UI), true);
    _player.setTexture(_textureHolder.get(TextureId::PLAYER));

    // 플레이어 위치 이동 (내부 해상도 426 x 240 기준)
    _player.setPosition(132, 173);
}
```

매 프레임마다 Player 애니메이션을 [update()](https://github.com/copyrat90/sfml-practice/blob/71ce65fa6065ae7590ac5e267f9ca9e1167c4ab7/src/Game.cpp#L80) 하고, 모든 Sprite를 [draw()](https://github.com/copyrat90/sfml-practice/blob/71ce65fa6065ae7590ac5e267f9ca9e1167c4ab7/src/Game.cpp#L87-L93) 한다.\
(스프라이트는 426x240 공간인데, 윈도우는 그 3배인 1278x720 공간으로 업스케일한다.)
```cpp
// 윈도우의 해상도는 내부 해상도 x 3 으로 설정
static constexpr int WINDOW_SCALE_FACTOR = 3;

void Game::_update(const sf::Time deltaTime) {
    _player.update(deltaTime);
}

void Game::_render() {
    _window.clear();

    // 실제 윈도우는 3배 크기이므로, 3배 키우는 transform 적용
    auto states = sf::RenderStates::Default;
    states.transform.scale(WINDOW_SCALE_FACTOR, WINDOW_SCALE_FACTOR);

    _window.draw(_tilemapSprite, states);
    _window.draw(_player, states);
    _window.draw(_uiSprite, states);

    _window.display();
}
```

### 결과

첫번째 패스 렌더링이 완료됐다. ([GitHub commit 보기](https://github.com/copyrat90/sfml-practice/tree/71ce65fa6065ae7590ac5e267f9ca9e1167c4ab7))\
여기까지는 전부 SFML을 사용한 C++ 코드로만 구현할 수 있었다.

![](/assets/img/posts/2022-12-11-heat-haze-frag-shader/03.gif)



## 2nd Pass & 3rd Pass

### 설명

위의 1st Pass 에서는 `sf::RenderWindow` 로 렌더링했는데, 그러면 후처리를 적용할 수 없다.\
대신에 [중간 결과에 해당하는 `sf::RenderTexture` 를 둬서](https://github.com/copyrat90/sfml-practice/blob/699d262bd2baaef8377c04c9cf27c4ff1cfc04ca/src/Game.hpp#L37-L38), 텍스처에 렌더링 후,\
그 텍스처를 `sampler2D` 로 읽어 프래그먼트 셰이더를 적용하는 식으로 나머지 Pass 를 구현하겠다.

||1st pass|→|2nd pass|→|3rd pass|
|---|---|---|---|---|---|
|Texture| `_postEffectRequired`|→|`_postEffectApplied`|→|윈도우에 3x 크기로 그림|

> 3rd Pass 까지 추가로 둔 이유는, 셰이더 코드가 간단해지기 때문이다.
> 1. 셰이더에서 윈도우 크기를 몇배로 스케일링 했는지(`N`)를 변수로 받을 필요가 없다.
> 2. `N`픽셀 옆에 있는 픽셀의 색상이 아닌, 1px 옆 색상을 보면 된다.

### 코드

#### 렌더링 패스

셰이더 코드는 조금 이따 보도록 하고,\
먼저, 작성한 [프래그먼트 셰이더를 로딩](https://github.com/copyrat90/sfml-practice/blob/699d262bd2baaef8377c04c9cf27c4ff1cfc04ca/src/Game.cpp#L75)한다. ([gr::ShaderHolder 구현](https://github.com/copyrat90/sfml-practice/blob/699d262bd2baaef8377c04c9cf27c4ff1cfc04ca/src/ResourceId.hpp#L36), [gr::ResourceHolder 구현](https://github.com/copyrat90/sfml-practice/blob/699d262bd2baaef8377c04c9cf27c4ff1cfc04ca/src/ResourceHolder.hpp#L15))
```cpp
void Game::_loadResources() {
    /* ... */
    _shaderHolder.loadFromFile(ShaderId::HEAT_HAZE,
        "assets/shaders/heat_haze.frag", sf::Shader::Type::Fragment);
}
```

렌더링은 이제 [3단계의 Pass](https://github.com/copyrat90/sfml-practice/blob/699d262bd2baaef8377c04c9cf27c4ff1cfc04ca/src/Game.cpp#L108) 로 진행된다.
```cpp
void Game::_render() {
    _renderFirstPass();
    _renderSecondPass();
    _renderToWindowWithScaling();
}
```

[변경된 1st Pass](https://github.com/copyrat90/sfml-practice/blob/699d262bd2baaef8377c04c9cf27c4ff1cfc04ca/src/Game.cpp#L115) 에서는, **커스텀 셰이더를 적용할 개체들만** `_postEffectRequired` 텍스처에 그린다.\
즉, `Player`와 `Tilemap`만 그리고, **`UI`는 그리지 않는다.**

```cpp
void Game::_renderFirstPass() {
    _postEffectRequired.clear();

    // 커스텀 셰이더를 적용할 개체들만 그림.
    // UI 는 2nd Pass 에서 셰이더 적용 없이 그릴 예정.
    _postEffectRequired.draw(_tilemapSprite);
    _postEffectRequired.draw(_player);

    _postEffectRequired.display();
}
```

[2nd Pass](https://github.com/copyrat90/sfml-practice/blob/699d262bd2baaef8377c04c9cf27c4ff1cfc04ca/src/Game.cpp#L127) 에서는, 위 1st Pass 에서 그린 텍스처를 이용하여 `_postEffectApplied` 텍스처에 그린다.\
이 때, `UI`는 커스텀 셰이더를 적용하지 않고 따로 그린다.

```cpp
void Game::_renderSecondPass() {
    _postEffectApplied.clear();

    // 셰이더의 떨림 각도 전역변수 업데이트
    auto& customFragShader = _shaderHolder.get(ShaderId::HEAT_HAZE);
    customFragShader.setUniform("u_hazeRadians", _hazeRadians);

    // 커스텀 셰이더를 적용하는 `shaderStates`
    auto shaderStates = sf::RenderStates::Default;
    if (_isCustomShaderEnabled)
        shaderStates.shader = &customFragShader;

    sf::Sprite postEffectRequiredSprite(_postEffectRequired.getTexture());

    _postEffectApplied.draw(postEffectRequiredSprite, shaderStates);
    _postEffectApplied.draw(_uiSprite); // UI는 항상 셰이더 미적용

    _postEffectApplied.display();
}
```

위 코드를 보면 [셰이더에 전달하는 uniform 으로 `u_hazeRadians` 변수](https://github.com/copyrat90/sfml-practice/blob/699d262bd2baaef8377c04c9cf27c4ff1cfc04ca/src/Game.cpp#L133)가 있는데,\
이는 매 순간 변화하는 떨림 효과의 각도(phase)를 의미한다.

[떨림 각도 `_hazeRadians`는 다음과 같이 업데이트](https://github.com/copyrat90/sfml-practice/blob/699d262bd2baaef8377c04c9cf27c4ff1cfc04ca/src/Game.cpp#L100-L105)된다.
```cpp
void Game::_update(const sf::Time deltaTime) {
    _player.update(deltaTime);

    // 시간에 따른 떨림 각도 업데이트
    constexpr float PI = 3.1415926f;
    constexpr float WAVE_PER_SECOND = 0.25f * 2 * PI;
    _hazeRadians += WAVE_PER_SECOND * deltaTime.asSeconds();
    while (_hazeRadians >= 2 * PI)
        _hazeRadians -= 2 * PI;
}
```

[3rd Pass](https://github.com/copyrat90/sfml-practice/blob/699d262bd2baaef8377c04c9cf27c4ff1cfc04ca/src/Game.cpp#L148) 에서는, `_postEffectApplied` 텍스처를 3배 크기로 업스케일해 윈도우에 그린다.

```cpp
constexpr int WINDOW_SCALE_FACTOR = 3;

void Game::_renderToWindowWithScaling() {
    _window.clear();

    // 실제 윈도우 크기에 맞게 업스케일링 해서 그리기
    sf::Sprite finalGameFrame(_postEffectApplied.getTexture());
    finalGameFrame.scale({WINDOW_SCALE_FACTOR, WINDOW_SCALE_FACTOR});

    _window.draw(finalGameFrame);

    _window.display();
}

```




#### 커스텀 셰이더

이제 [프래그먼트 셰이더 코드](https://github.com/copyrat90/sfml-practice/blob/699d262bd2baaef8377c04c9cf27c4ff1cfc04ca/assets/shaders/heat_haze.frag)를 보자.

```c
#version 300 es
precision highp float;

out vec4 fragColor;

uniform sampler2D u_postEffectRequired;
uniform vec2 u_internalResolution;

uniform float u_hazeRadians;

void main() {
    // gl_FragCoord를 0 ~ 1 범위로 정규화해 UV 좌표 계산
    vec2 uv = gl_FragCoord.xy / u_internalResolution;
    float uvDiffX = 1. / u_internalResolution.x; // UV 좌표 기준 1px

    // 시시각각 달라지는 각도(u_hazeRadians)에 각 Y좌표별 Phase offset을 두어 떨림 구현
    float amplitude = sin(u_hazeRadians + gl_FragCoord.y / 3.);
    float direction = (amplitude > 0.5) ? 1. : (amplitude < -0.5) ? -1. : 0.;

    // `direction` 변수값에 따라 1px 왼쪽 or 1px 오른쪽 or 중앙 픽셀을 읽어 현재 픽셀 색칠
    fragColor = texture(u_postEffectRequired, vec2(uv.x + uvDiffX * direction, uv.y));
}
```

우선, `gl_FragCoord`는 렌더링할 텍스처 `postEffectApplied`의 해상도 기준 좌표이므로,\
범위가 `gl_FragCoord.x = 0.5 ~ 425.5` 이고 `gl_FragCoord.y = 0.5 ~ 239.5` 이다.

이 `gl_FragCoord`를 이용해 `u_postEffectRequired`로부터 색상을 sampling 하려면, `0 ~ 1` 범위로 정규화된 UV 좌표가 필요하다.\
그래서 `main()`의 첫 2줄에서는\
`0 ~ 1` 범위로 정규화된 UV 좌표 `uv`와, x축 기준 1픽셀의 거리 `uvDiffX`를 구하고 있다.

`main()`의 중간 2줄은 `direction`을 구하는 코드이다.\
`direction`은 현재 픽셀을 `u_postEffectRequired`로부터 가져올 때,\
1px 오른쪽에서 읽어와야 하는지, 왼쪽에서 읽어야하는지, 아니면 이동 없이 읽어와야 하는지를 결정한다.

현재 각도 `u_hazeRadians`와 `gl_FragCoord.y` 좌표의 합을 sine 함숫값을 취해 계산하고 있는데,\
이를 통해 *시간에 따른 픽셀 위치 이동* 과 *y좌표에 따른 픽셀 위치 이동*을 같이 표현할 수 있다.

그렇게 계산한 `uvDiffX`와 `direction`을 이용해, 이전 Pass의 텍스처 `u_postEffectRequired`로부터 샘플링하여 이번 프래그먼트를 칠한다.

```cpp
// 내부 해상도: 게임 픽셀아트 단위 해상도
static constexpr int INTERNAL_WIDTH = 426;
static constexpr int INTERNAL_HEIGHT = 240;

Game::Game() : _window(sf::VideoMode(WINDOW_WIDTH, WINDOW_HEIGHT), "Heat Haze Shader Example") {
    ...
    // 셰이더 전역 상수 설정
    auto& heatHazeShader = _shaderHolder.get(ShaderId::HEAT_HAZE);
    heatHazeShader.setUniform("u_postEffectRequired", _postEffectRequired.getTexture());
    heatHazeShader.setUniform("u_internalResolution", sf::Vector2f{INTERNAL_WIDTH, INTERNAL_HEIGHT});
}
```

위와 같이, 변하지 않는 [uniform 변수들은 Game 생성자에서 전달](https://github.com/copyrat90/sfml-practice/blob/699d262bd2baaef8377c04c9cf27c4ff1cfc04ca/src/Game.cpp#L39-L42)해주었다.

### 결과

실제 게임과 비슷한 결과를 얻었다.\
[GitHub commit 보기](https://github.com/copyrat90/sfml-practice/tree/699d262bd2baaef8377c04c9cf27c4ff1cfc04ca)\
[GitHub release 다운로드 페이지](https://github.com/copyrat90/sfml-practice/releases/tag/heat-haze-v1)

아무 키나 누르면 커스텀 셰이더를 껐다 켰다 할 수 있다.

![](/assets/img/posts/2022-12-11-heat-haze-frag-shader/04.gif)


# 연습문제

우선, [위 프로그램을 다운받아](https://github.com/copyrat90/sfml-practice/releases/tag/heat-haze-v1) 압축을 풀고 실행해보자.\
폴더 내 `assets/shaders/heat_haze.frag` 셰이더를 수정해서 아래의 결과를 얻어보자.

![](/assets/img/posts/2022-12-11-heat-haze-frag-shader/05.gif)

[정답 코드 보기](https://pastebin.com/qkGBdCJ0)

*마지막 수정 : {{ page.last_modified_at }}*
