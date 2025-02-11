---
layout: post
title: GameNetworkingSockets 빌드하기
tags: [GameNetworkingSockets]
author: copyrat90
last_modified_at: 2025-02-11T21:27:00+09:00
---

`BUILDING.md` 읽고 그대로 따라하면 될 줄 알았는데... 이상한 오류가 나서 한참 헤맸다.

# GameNetworkingSockets?

[Valve](https://www.valvesoftware.com/en)에서 오픈소스로 공개한 UDP 소켓을 게임용으로 추상화한 라이브러리.

[GitHub: ValveSoftware/GameNetworkingSockets](https://github.com/ValveSoftware/GameNetworkingSockets)

TCP처럼 connection 기반으로 돌아가나, message 단위로 자르는 작업을 알아서 해 준다.\
그리고 Reliable과 Unreliable한 메시지 둘 다 보낼 수 있다.

# 빌드 문제

미리 컴파일된 버전을 써도 되지만, ICE 등의 P2P 기능을 쓰려면 직접 빌드해야 하는 것 같아서 빌드를 하기로 했다.

일단 64비트 Windows에서 [`BUILDING.md`](https://github.com/ValveSoftware/GameNetworkingSockets/blob/master/BUILDING.md)를 읽으며 그대로 따라해봤는데... 참 많은 오류를 겪었다.\
미래에 같은 실수를 반복하지 않도록 겪었던 문제들을 적어놓는다.

* 서브모듈 가져왔는지 확인할 것
    * 해결책: `git submodule update --init`
* 외부 폴더에 있는 vcpkg를 사용할 시, 툴체인 위치 명시가 필요
    * 해결책: `-DCMAKE_TOOLCHAIN_FILE=path/to/vcpkg/scripts/buildsystems/vcpkg.cmake`
* 반드시 64비트 버전의 Visual C++ 컴파일러를 사용하도록 하기 (`cl.exe`)
    * `Developer PowerShell for VS 2022`를 썼는데, 여기 `cl.exe`는 x86 컴파일러가 사용됨
        * 임시적인 해결책
            1. 위 `Developer Powershell for VS 2022` 바로 가기 링크를 확인해, `Enter-VsDevShell` 뒤에 있는 hex문자 6글자를 기억 (e.g. `f174b73f`)
            1. 일반 Powershell을 실행
            1. 아래 명령을 실행하되, 위에서 기억한 hex문자 6글자로 대체해서 실행
                ```ps
                &{Import-Module "C:/Program Files/Microsoft Visual Studio/2022/Community/Common7/Tools/Microsoft.VisualStudio.DevShell.dll"; Enter-VsDevShell f174b73f -SkipAutomaticLocation -DevCmdArguments "-arch=x64 -host_arch=x64"}
                ```
                * 안타깝게도, 이 명령을 그냥 `Developer Powershell for VS 2022` 바로가기 링크에 넣을 수가 없음. 명령이 너무 길어서 잘림...
            1. 이제 64비트 컴파일러가 제대로 잡힘.
        * 영구적인 해결책
            * 아예 VSCode `settings.json`에다가 64비트 Developer Powershell 터미널을 실행할 수 있도록 넣어 놓기
                ```json
                "terminal.integrated.profiles.windows": {
                    "PowerShell (VS Dev)": {
                    "args": [
                        "-noe",
                        "-c",
                        "&{Import-Module \"C:/Program Files/Microsoft Visual Studio/2022/Community/Common7/Tools/Microsoft.VisualStudio.DevShell.dll\"; Enter-VsDevShell f174b73f -SkipAutomaticLocation -DevCmdArguments \"-arch=x64 -host_arch=x64\"}"
                    ],
                    "source": "PowerShell",
                    "icon": "terminal-powershell"
                    }
                }
                ```
    * 개발 환경에 있는 MinGW-w64 GCC가 대신 사용되어버림
        * 해결책: `-DCMAKE_C_COMPILER=cl -DCMAKE_CXX_COMPILER=cl`
        * 그런데 이건 위에서 64비트 VS Developer Powershell 설정을 하면 나는게 비정상.\
          혹시 일반 Powershell에서 잘못 실행하는 게 아닌지 확인할 것.
    * Could not find OpenSSL, Protobuf...
        * CMake 툴체인 파일 명시하는 데 오타 났는지 다시 한번 확인할 것.


*마지막 수정 : {{ page.last_modified_at }}*
