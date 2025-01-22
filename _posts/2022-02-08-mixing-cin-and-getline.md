---
layout: post
title: std::cin과 getline()을 섞어쓸 때 주의점
tags: [C++]
author: copyrat90
last_modified_at: 2022-02-08T12:00:00+09:00
---

개행문자(`'\n'`)를 읽느냐 마느냐가 다르다!

> 1. `std::cin >>` 은 `'\n'` **직전**까지만 읽는다. *(버퍼에 `'\n'`이 남아있음)*
> 2. `getline()`은 `'\n'`**까지** 읽고, *`'\n'`을 버린* 문자열을 저장한다.
> 3. 따라서, 그냥 1 -> 2 순서로 사용하면 2에서 빈 문자열을 저장하게 되니,
> 사용 직전에 `std::cin.ignore(4096, '\n');` or `std::cin >> std::ws;` 를 써주자.
> (사실 `4096` 대신 [이렇게 긴 걸 적어주는 게 정석.](https://en.cppreference.com/w/cpp/string/basic_string/getline#Notes))

`getline()`은 아래 2가지가 있는데, 둘 다 위 설명처럼 작동한다.

1. [`std::cin.getline(char*, std::streamsize)`](https://en.cppreference.com/w/cpp/io/basic_istream/getline)
2. [`<string>` 헤더의 `std::getline(std::cin, std::string&)`](https://en.cppreference.com/w/cpp/string/basic_string/getline)

사족으로, 둘 다 `std::cin`을 참조로 반환한다.

- - -

### 출처

1. <https://en.cppreference.com/w/cpp/string/basic_string/getline#Notes>
2. <https://stackoverflow.com/questions/21567291/why-does-stdgetline-skip-input-after-a-formatted-extraction>

*마지막 수정 : {{ page.last_modified_at }}*
