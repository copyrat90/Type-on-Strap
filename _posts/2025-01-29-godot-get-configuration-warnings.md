---
layout: post
title: (Godot) 직접 만든 Scene이 에디터 warning 표시하게 하기
tags: [Godot]
author: copyrat90
last_modified_at: 2025-01-30T12:08:00+09:00
---

어떤 노드들은 특정 타입의 자식이 없으면 오류 메시지가 뜬다.\
내가 만든 Scene에 대해서도 이런 식으로 설정이 제대로 안 됐음을 알릴 수 없을까?

![](/assets/img/posts/2025-01-29-godot-get-configuration-warnings/default_warn.png)

# Tool script

이걸 하려면 우선 C# 스크립트를 Tool script 로 만들어야 한다.

간단한데, 클래스 앞에 `[Tool]` 만 붙이면 된다.\
그러면 [에디터에서도 코드가 돌기 시작한다.](https://docs.godotengine.org/en/stable/tutorials/plugins/running_code_in_the_editor.html)

> **조심하자. 에디터에서 코드가 돈다는 말은, 잘못 짜면 에디터가 튕긴다는 말이다.**\
> null reference 등 없도록 조심 또 조심.

현재 게임 내에서 실행되고 있는지, 아니면 에디터에서 실행되고 있는지 판단은\
`Engine.IsEditorHint()`를 호출해서 할 수 있다.

# `_GetConfigurationWarnings()`

Node에 들어있는 가상 메서드 [`_GetConfigurationWarnings()`](https://docs.godotengine.org/en/stable/classes/class_node.html#class-node-private-method-get-configuration-warnings)가 `null`이 아닌 `string[]`을 반환한다면, 오류 메시지가 표시된다.\
다만, 이 메서드는 Scene을 처음 열 때만 평가되므로, 더 이상 오류가 아니게 되거나 하면 [`UpdateConfigurationWarnings()`](https://docs.godotengine.org/en/stable/classes/class_node.html#class-node-method-update-configuration-warnings)를 호출해야 한다.

# 사용 예시

아래는 자식으로 `Marker2D` 타입의 노드를 갖고 있지 않으면 경고를 발생시키는 예시다.

```cs
using Godot;
using System.Linq;

[Tool] // tool script
public partial class RequireMarker2dAsChild : Node
{
    // 오류 문자열 반환하는 메서드
    public override string[] _GetConfigurationWarnings()
    {
        // 자식 중에 Marker2D 타입의 노드가 하나라도 있다면, 오류 문자열 없음.
        if (GetChildren().Any(child => child is Marker2D))
            return null;

        // Marker2D 없다면, 오류 문자열을 반환.
        return ["Marker2D 타입의 자식 노드가 없음.", "자식으로 Marker2D 타입의 노드를 추가해 주세요!"];
    }

    // 위 오류 문자열 평가는 처음 씬을 열 때만 되므로,
    // 노드를 자식으로 추가했을 때 `UpdateConfigurationWarnings()`를 호출하도록 Signal handling을 해 줘야 함
    public override void _Ready()
    {
        // 에디터라면, 자식 노드 추가/삭제 Signal handler 추가
        if (Engine.IsEditorHint())
        {
            // 자식 추가/삭제 시, 오류 문자열 재평가
            ChildEnteredTree += (child) => UpdateConfigurationWarnings();
            ChildExitingTree += (child) => UpdateConfigurationWarnings();
            return;
        }
    }

    public override void _Process(double delta)
    {
        // 게임 내 로직이 에디터에서 돌면 안 될 테니,
        // 모든 게임 내에서만 돌릴 notification에 대해서 이걸 앞에 달아줘야 할 듯.
        // 좀 귀찮아지긴 한다.
        if (Engine.IsEditorHint())
            return;
    }
}
```

![](/assets/img/posts/2025-01-29-godot-get-configuration-warnings/custom_warn.png)
![](/assets/img/posts/2025-01-29-godot-get-configuration-warnings/no_warn.png)

보다시피 잘 작동된다.

이렇게 tool script 를 작성하게 되면, 위 스샷처럼 스크립트 아이콘이 파란색이 된다.\
그 말은, 현재 스크립트가 에디터에서 실행 중이라는 뜻.

*마지막 수정 : {{ page.last_modified_at }}*
