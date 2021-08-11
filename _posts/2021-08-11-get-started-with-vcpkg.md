---
layout: post
title: vcpkg 로 C++ 오픈소스 라이브러리 간단 설치하기 (feat. Manifest mode)
tags: [vcpkg]
author: copyrat90
last_modified_at: 2021-08-11T20:23:00+09:00
---


vcpkg 가 업데이트되어 프로젝트별로 의존성을 관리할 수 있게 됐다.

vcpkg 를 처음 접하는 사람도 있을테니, *(사실 필자도 처음이다.)*\
개요와 설치법부터 사용까지 간단히 다루겠다.

## vcpkg 란?

[vcpkg](https://vcpkg.io/) 는 Microsoft 에서 시작된 오픈소스 C/C++ 패키지 매니저다.\
C++ 라이브러리를 간단히 설치하고 링크할 수 있게 만들어졌다.\
파이썬의 `pip`, Node.js 의 `npm`, Rust 의 `cargo` 등과 비슷한 기능을 한다.

운영체제는 Windows, Linux, macOS 를 지원한다.\
빌드 시스템은 MSBuild 와 CMake 를 지원하고, 다른 빌드 시스템과도 부분적으로 호환된다.

내부적으로 라이브러리의 소스코드를 다운받아서 직접 컴파일하는 방식을 사용한다.\
왜 미리 빌드된 바이너리를 제공하지 않는지가 궁금하다면 [이 영상](https://www.youtube.com/watch?v=3vXOKkv3ND0&t=337s)을 확인해보면 된다.

## vcpkg 설치하기

[공식 가이드는 여기서 볼 수 있다.](https://vcpkg.io/en/getting-started.html)\
이 글은 Windows 10 와 Visual Studio 2019 를 사용한다는 가정 하에 진행하겠다.

우선 [Git](https://git-scm.com/) 이 설치되어 있어야 한다. 없다면 먼저 Git 부터 설치하자.\
우선 설치하고 싶은 위치에 폴더를 하나 만든다. (편의상 `{util_dir}` 라고 부르겠다.)\
[명령줄](https://ko.wikipedia.org/wiki/명령_줄_인터페이스)에서 [`cd`](https://ko.wikipedia.org/wiki/Cd_(명령어)) 로 `{util_dir}`로 이동 후, 아래 명령어로 vcpkg 레포지토리를 복제하자.

```powershell
git clone https://github.com/microsoft/vcpkg
```

복제했으면 이제 아래 명령어로 vcpkg 를 설치하자.

```powershell
.\vcpkg\bootstrap-vcpkg.bat
```

그러면 생성된 `vcpkg` 폴더 안에 `vcpkg.exe` 파일이 설치된다.\
이제 Path 환경 변수에 `{util_dir}\vcpkg` 를 추가하면 된다.

## vcpkg 와 Visual Studio 연동하기

마소에서 만든 프로그램이라 그런지, Visual Studio 하고 연동하기가 아주 쉽다.\
그냥 명령줄에 아래 한 줄만 입력하면 된다.
```powershell
vcpkg integrate install
```
그러면 이제 MSVC 가 vcpkg 에 설치된 라이브러리를 자동으로 인식한다.

## 패키지 검색 및 설치

`vcpkg search [검색어]` 를 입력하면 vcpkg 공개 저장소에 등록된 패키지를 검색해준다.\
검색 결과에서 왼쪽 열에 있는 패키지명을 기억해놓으면 된다.

```powershell
PS> vcpkg search xml
...
rapidxml                 1.13-4           RapidXml is an attempt to create the fastest XML parser possible, while re...
rapidxml-ns              1.13.2           RapidXML with added XML namespaces support.
tidy-html5               5.7.28-2         Tidy tidies HTML and XML. It can tidy your documents by itself, and develo...
tinyxml                  2.6.2-7          A simple, small, minimal, C++ XML parser that can be easily integrating in...
tinyxml2                 8.0.0-1          A simple, small, efficient, C++ XML parser
...
The search result may be outdated. Run `git pull` to get the latest results.

If your library is not listed, please open an issue at and/or consider making a pull request:
    https://github.com/Microsoft/vcpkg/issues
```

아니면 [이 페이지](https://vcpkg.io/en/packages.html)에서 검색할 수도 있다.

![](/assets/img/posts/2021-08-11-get-started-with-vcpkg/vcpkg-browse-packages-page.png)

왼쪽 위에 보이는 볼드체의 패키지명을 기억해놓으면 된다.

패키지를 설치할 때는 두 가지 방법이 있다.
> 1. **Classic Mode:** 모든 프로젝트에 사용 가능하도록 전역으로 설치
> 2. **Manifest Mode:** 특정 프로젝트에 의존성을 명시하여, 그 프로젝트 폴더 안에 설치

두 방법을 순서대로 살펴보도록 하겠다.

그런데 어떤 경우든 **Visual Studio 영어 언어 팩이 없으면 패키지 설치가 안된다.**\
먼저 Visual Studio Installer 를 켜서 Visual Studio 에 대고 `수정(M)`을 누른 후, `언어 팩`에서 `영어`를 찾아 설치해주자.

### Classic Mode: 전역 라이브러리 설치

패키지를 전역에 설치하는 건 간단하다.
```powershell
vcpkg install [패키지명]
또는
vcpkg install [패키지명]:[Triplet]
```
*Triplet* 에는 실행 환경을 명시하면 되는데, 명시하지 않으면 기본적으로 `x86-windows` 로 지정돼 패키지가 설치되게 된다.\
이건 64bit Windows 를 사용하고 있다고 해도 마찬가지다.\
64bit Windows 를 기본값으로 하고 싶으면 환경 변수에 `VCPKG_DEFAULT_TRIPLET` 를 `x64-windows` 값으로 추가해주자.[^1]

### Manifest Mode: 프로젝트별 라이브러리 설치

프로젝트 내에 의존성을 명시해, 빌드시 사용될 패키지가 프로젝트 디렉토리에 설치되도록 할 수도 있다.\
이 글에서는 기초적인 사용법만 다루므로, [자세한 내용은 여기를 참고하자.](https://vcpkg.readthedocs.io/en/latest/users/manifests/)

프로젝트 디렉토리(MSBuild) or 솔루션 디렉토리(MSBuild, CMake)에 아래와 같은 형식의 `vcpkg.json` 파일을 만들자.
```json
{
  "$schema": "https://raw.githubusercontent.com/microsoft/vcpkg/master/scripts/vcpkg.schema.json",
  "name": "my-app",
  "version": "0.0.1",
  "dependencies": [
    "fmt",
    "pugixml",
    "nlohmann-json"
  ]
}
```
위와 같이 `dependencies` 필드 안에 설치할 라이브러리를 명시하면 된다.\
참고로 `name`, `version` 은 필수 필드라, 없으면 빌드시 오류가 발생한다.

추가로 MSBuild 기반 프로젝트의 경우, `프로젝트 속성(P) -> 구성 속성 -> vcpkg` 탭에서 `Use Vcpkg Manifest` 를 `Yes` 로 바꿔줘야 한다.\
만일 `구성 속성`에서 `vcpkg` 탭이 안 보일 경우, 명령줄에서 `vcpkg integrate install` 를 실행해 Visual Studio 와 연동부터 하자.

참고로, [vcpkg 공식 Docs 에서는 가능하면 Manifest Mode 를 사용할 것을 권장한다.](https://vcpkg.readthedocs.io/en/latest/users/manifests/)

## 패키지 사용

설치를 했으니 사용을 해보자.\
MSBuild 기반 프로젝트가 더 간단하므로 먼저 살펴보고, CMake 기반 프로젝트를 살펴보겠다.\
[자세한 내용은 여기를 참고하자.](https://vcpkg.readthedocs.io/en/latest/users/integration/)

### MSBuild 에서 패키지 사용

MSBuild 기반 프로젝트에서는, *대부분의 경우* 이 이상 아무것도 하지 않아도 해당 라이브러리의 헤더를 `#include` 해서 사용할 수 있다.\
`#include` 하는 순간 MSBuild 가 알아서 해당 패키지의 정적 라이브러리를 링크를 해준다.\
게다가 **해당 라이브러리에 사용되는 DLL 파일이 자동으로 빌드 폴더로 복사된다!** 정말 편리하다.

하지만 [가끔 수동으로 링크해야 하는 경우](https://vcpkg.readthedocs.io/en/latest/users/integration/#integrate-command)도 있는데, 대표적으로 라이브러리에서 `main` 을 건드린다거나 하는 경우가 있다.\
그런 경우, `manual-link` 디렉토리를 찾아서 라이브러리 디렉토리에 추가하고, 그 안에 있는 `*.lib` 파일을 링커 입력에 직접 넣어주면 된다.

예를 들어 SDL2 라이브러리의 경우 `SDL2main.lib` 이 [main 을 건드리기 때문에]({{ site.url }}{{ site.baseurl }}/2021/08/08/why-sdl-does-not-allow-int-main-void.html) 수동으로 링크해야 한다.\
`프로젝트 속성(P) -> 구성 속성 -> VC++ 디렉터리 -> 라이브러리 디렉터리`에서 플랫폼에 맞는 `lib\manual-link` 디렉터리를 추가하자. (여기서는 *x64-windows*)
```
$(VcpkgManifestRoot)\vcpkg_installed\x64-windows\lib\manual-link
```

그리고 `프로젝트 속성(P) -> 구성 속성 -> 링커 -> 입력 -> 추가 종속성`에 필요한 lib 파일을 추가하자.
```
SDL2main.lib
```

이렇게 설정하고 다시 빌드해보면 정상적으로 빌드될 것이다.

### CMake 에서 패키지 사용

CMake 기반 프로젝트의 경우, `vcpkg.json` 파일이 [최상단 `CMakeLists.txt` 가 있는 디렉토리에 있어야 한다.](https://vcpkg.readthedocs.io/en/latest/users/manifests/#cmake-integration)

사용하는 건 [*CMake를 쓸 줄 안다는 가정 하에*](https://modoocode.com/332) 의외로 간단한데, `CMakeLists.txt`에 `find_package()`와 `target_link_libraries()`를 작성할 줄만 알면 된다.
```cmake
# 예시로 SDL2 라이브러리를 사용
find_package(SDL2 REQUIRED)
target_link_libraries(main PRIVATE SDL2::SDL2 SDL2::SDL2main)
```

그리고 `cmake` 를 명령줄에서 호출할 때 [툴체인 파일을 명시해주면 된다.](https://vcpkg.readthedocs.io/en/latest/examples/installing-and-using-packages/#cmake-toolchain-file)
```powershell
cmake {project_dir} -DCMAKE_TOOLCHAIN_FILE={util_dir}\vcpkg\scripts\buildsystems\vcpkg.cmake
```

Visual Studio 사용한다면 빌드할 때 툴체인 파일 명시는 IDE 가 알아서 해준다.

이제 라이브러리의 헤더를 라이브러리의 헤더를 `#include` 해서 사용하면 된다.\
MSBuild 와 마찬가지로 **DLL 파일이 자동으로 복사되는 것으로 보인다.**

## 맺음말

기존에 C++ 로 작업할때는 패키지 매니저 없이 일일히 라이브러리를 다운받아서 사용했었다.\
vcpkg 를 사용해보니 이걸 자동화하는게 얼마나 편리한 일인지 깨닫게 됐다.

이번에 소개한 vcpkg 사용법은 아주 기초적인 수준에 불과하다.\
직접 써보면서 궁금한 게 생기면 [공식 홈페이지](https://vcpkg.io/)와 [GitHub Repository](https://github.com/microsoft/vcpkg) 를 오가는 걸 추천한다.\
필자는 앞으로 이걸 주로 쓰면서 괜찮은 활용법이 있으면 추가로 포스팅할 계획이다.

*마지막 수정 : {{ page.last_modified_at }}*

* * *

[^1]: <https://github.com/microsoft/vcpkg/issues/1254>

