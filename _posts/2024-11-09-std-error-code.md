---
layout: post
title: C++ 오류 코드 `std::error_code` 써보기
tags: [C++]
author: copyrat90
last_modified_at: 2024-11-09T12:00:00+09:00
---

프로그래밍을 함에 있어, 오류 처리는 피할 수 없는 숙명이다.\
이를 위해 C++에서는 보통 아래 3가지 메커니즘을 사용하게 된다.

1. 예외
2. 오류 코드
    * `std::error_code` (C++11)
3. `std::expected` (C++23)

이번 시간엔, C에서 흔히 쓰던 방법인 2. 오류 코드를 래핑한 표준 C++ 클래스 `std::error_code`를 써보자.


# 오류 코드

오류 코드를 C 스타일로 작성한다면 이런 식으로 작성할 수 있을 것이다.
```c
// 온라인 쇼핑몰에서 상품을 구매할 때 발생할 수 있는 오류
enum PurchaseErrc {
    /* 정상 처리 (오류 아님) */
    PE_OK = 0,
    /* 판매자 측 오류 */
    PE_STORE_NOT_OPEN = 100,       // 상점이 열리지 않은 경우
    PE_OUT_OF_STOCK,               // 주문할 제품의 재고가 부족한 경우
    /* 사용자 측 입력 오류 */
    PE_INVALID_ITEMS_AMOUNT = 200, // 잘못된 수량을 입력한 경우
    PE_INVALID_CARD_NUMBER,        // 잘못된 카드 번호를 입력한 경우
    /* 쇼핑몰 측 오류 */
    PE_INTERNAL_ERROR = 5000,      // 쇼핑몰 측 내부 오류
};

// 온라인 쇼핑몰에 상품을 등록할 때 발생할 수 있는 오류
enum RegisterProductErrc {
    /* 정상 처리 (오류 아님) */
    RPE_OK = 0,
    /* 사용자 측 입력 오류 */
    RPE_INVALID_STOCK_AMOUNT = 5000, // 잘못된 재고량을 입력한 경우
    RPE_NO_NAME_PROVIDED,            // 상품 이름을 입력하지 않은 경우
    /* 쇼핑몰 측 오류 */
    RPE_INTERNAL_ERROR = 9999,       // 쇼핑몰 측 내부 오류
};
```

물론 이대로 써도 잘 작동하지만, 아쉬운 점이 몇가지 있다.

* 똑같은 오류가 enum 별로 값이 다를 수 있다.
    * `PE_INTERNAL_ERROR` 와 `RPE_INTERNAL_ERROR`
    * `PE_INVALID_ITEMS_AMOUNT` 와 `RPE_INVALID_STOCK_AMOUNT`
* 오류 조건을 묶기 위해 정수값 범위에 의존하고 있는데, 이 범위도 다르다.
    * `사용자 측 오류`, `쇼핑몰 측 오류`라는 오류 조건 묶음은 같은데 값 범위가 다르다.

이런 이유로, 서로 다른 오류 enum 에 대한 오류 처리 코드를 따로 작성해야 하는 단점이 있다.\
이걸 일원화 할 수는 없을까?


# `std::error_code`

요컨대, `std::error_code`는 오류 enum 종류와 값을 type-erased 방식으로 저장하는 컨테이너이다.\
안에 저장되는 값은 2가지이다.

* `std::error_code::category()` : enum 종류를 구분하기 위한 사용자 정의 Category 싱글턴 참조
* `std::error_code::value()` : enum 값을 나타내는 `int`

`std::error_code::category()`의 반환형은 `const std::error_category&`로, 우리는 이 `std::error_category`를 상속받은 Category를 따로 싱글턴으로 만들어, 에러 코드 초기화시에 레퍼런스를 전달하게 된다.\
싱글턴이어야 하는 이유는, 에러 종류를 구분하는 방법이 위 `category` 참조의 포인터 주소값을 비교하는 것이기 때문이다.

## enum 연동하기

아니 그러면, `std::error_code` 객체 하나 만들자고 `category`와 `value`를 매번 따로 전달해줘야 하는걸까?\
당연히 그러면 쓰기가 너무 불편하므로, 오류 enum 변수를 받아 `std::error_code`를 반환하는 `make_error_code(Errc)` 함수를 직접 정의하면, `std::error_code` 생성자에서 그걸 사용해 자동 변환되도록 API가 설계되어 있다.
```cpp
template <typename Errc>
    requires is_error_code_enum<Errc>::value
error_code(Errc e) noexcept : error_code(make_error_code(e)) {
}
```

위 생성자 덕분에, 아래와 같은 에러 코드의 초기화 및 비교가 가능하다.
```cpp
std::error_code ec1 = PurchaseErrc::PE_OUT_OF_STOCK;
std::error_code ec2 = RegisterProductErrc::RPE_NO_NAME_PROVIDED;
assert(ec1 != RegisterProductErrc::RPE_INTERNAL_ERROR);
```

그런데, 아무 enum 이나 들어갈 수 있다면, 실수로 에러 코드가 아닌 enum 이 초기화에 사용될 수 있는 위험성이 있을 것이다.\
그래서 위 생성자에서 보이는 것처럼, `std::is_error_code_enum<Errc>::value`가 `true`인 enum 만 쓸 수 있도록 템플릿 제약조건이 걸려 있다.\
따라서, 위 초기화가 작동하기 위해서는 아래와 같은 템플릿 특수화도 필요하다.
```cpp
namespace std {
    // 1. `value`를 `true`로 직접 설정
    template <>
    struct is_error_code_enum<PurchaseErrc> {
        static constexpr bool value = true;
    };
    // 2. 같은 일을 하는 `std::true_type` 상속받기
    template <>
    struct is_error_code_enum<RegisterProductErrc> : true_type {};
}
```

참고로, 에러 코드에 사용되는 **enum 의 `0` 값은 오류가 아닌 정상 상황**을 의미해야 한다.\
그렇지 않으면 `operator bool`로 오류가 있는지 검사하는 코드를 짤 수 없다.

## Category 싱글턴 만들기

그래서, 커스텀 Category 싱글턴은 어떻게 만들까?\
아래와 같이 `std::error_code` 내부에서 이 Category 싱글턴을 이용하는 부분이 좀 있다.

* `std::error_code::message()` : `category().message(value())` 로 에러 코드에 해당하는 메시지 문자열을 얻어 옴
* `operator<<(ostream, ec)` : `ostream << ec.category().name() << ':' << ec.value()` 로 출력을 수행함

위로 인해, 우리의 Category는 `std::error_category`를 상속받고, 아래처럼 가상함수를 overriding 해줘야 한다.

```cpp
struct RegisterProductErrorCategory : std::error_category {
    auto name() const noexcept -> const char* override {
        return "RegisterProduct";
    }
    auto message(int ev) const -> std::string override {
        switch (static_cast<RegisterProductErrc>(ev)) {
            case RegisterProductErrc::RPE_OK:
                return "OK";
            case RegisterProductErrc::RPE_INVALID_STOCK_AMOUNT:
                return "Invalid stock amount provided";
            case RegisterProductErrc::RPE_NO_NAME_PROVIDED:
                return "No item name provided";
            case RegisterProductErrc::RPE_INTERNAL_ERROR:
                return "Internal system error";
            default:
                return "(Unknown error)";
        }
    }
};
```

이제 위 Category를 적절히 싱글턴으로 만들고, `make_error_code(Errc)` 에서 `value`와 `category`의 참조를 전달해주면 된다.
```cpp
auto make_error_code(RegisterProductErrc e) -> std::error_code {
    return std::error_code(static_cast<int>(e), RegisterProductErrorCategory::instance());
}
```


# `std::error_condition`

개별 오류 코드를 enum 과 비교하는 것은 위 구현만으로 가능하지만, 아직 오류 조건을 묶어서 처리할 수는 없다.\
오류 조건을 묶으려면 `std::error_condition`이라는 별도의 클래스를 이용할 수 있다.

## enum 연동하기 + Category 싱글턴 만들기

`std::error_condition`은 `std::error_code`와 API가 거의 유사하다.\
그래서 `std::error_condition`을 만드는 방법도 위에서 다룬 것과 거의 같다.
```cpp
enum FailReasonErrc {
    /* 오류값은 `0`이 아님에 유의 */
    FRE_SELLER_NOT_READY = 1, // 판매자 측 오류
    FRE_BAD_USER_INPUT,       // 사용자 측 입력 오류
    FRE_SYSTEM_ERROR,         // 쇼핑몰 측 오류
};

namespace std {
    // 거의 비슷하나, `std::is_error_condition_enum`을 사용하는 게 차이점
    template <>
    struct is_error_condition_enum<FailReasonErrc> : true_type {};
}

struct FailReasonErrorCategory : std::error_category {
	static auto instance() -> const FailReasonErrorCategory& {
        static FailReasonErrorCategory category;
        return category;
    }
    auto name() const noexcept -> const char* override {
        return "FailReason";
    }
    auto message(int ev) const -> std::string override {
        switch (static_cast<FailReasonErrc>(ev)) {
            case FailReasonErrc::FRE_SELLER_NOT_READY:
                return "Seller is not ready";
            case FailReasonErrc::FRE_BAD_USER_INPUT:
                return "Bad user input provided";
            case FailReasonErrc::FRE_SYSTEM_ERROR:
                return "Internal system error";
            default:
                return "(Unknown error)";
        }
    }
private:
    FailReasonErrorCategory() = default;
};

// 거의 비슷하나, `make_error_condition()`을 만드는 게 차이점
auto make_error_condition(FailReasonErrc e) -> std::error_condition {
    return std::error_condition(static_cast<int>(e), FailReasonErrorCategory::instance());
}
```

## `std::error_code`와 매칭시키기

### `std::error_category::equivalent()`

`std::error_condition`은 enum 종류도 값도 다른 `std::error_code`를 묶는데 활용될 수 있다.\
왜냐하면 `std::error_code`와 `std::error_condition`은 서로 다른 타입이지만, 아래와 같이 `operator==`이 정의되어 있기 때문이다.
```cpp
bool operator==(const std::error_code& code, const std::error_condition& cond) {
    return code.category().equivalent(code.value(), cond) ||
           cond.category().equivalent(code, cond.value());
}
```

따라서 `category`의 `equivalent()`를 overriding 하여, 특정 오류 코드가 특정 오류 조건과 매칭됨을 표현할 수 있다.
```cpp
struct FailReasonErrorCategory : std::error_category {
    ...
    bool equivalent(const std::error_code& code, int condition_value) const noexcept override {
        const std::error_category& purchase_category = PurchaseErrorCategory::instance();
        const std::error_category& register_product_category = RegisterProductErrorCategory::instance();
        
        switch (static_cast<FailReasonErrc>(condition_value)) {
            case FailReasonErrc::FRE_SELLER_NOT_READY:
                if (code.category() == purchase_category)
                    return code.value() / 100 == 1; // 100번대 오류
                return false;
            case FailReasonErrc::FRE_BAD_USER_INPUT:
                if (code.category() == purchase_category)
                    return code.value() / 100 == 2; // 200번대 오류
                else if (code.category() == register_product_category)
                    return code.value() / 1000 == 5; // 5000번대 오류
                return false;
            case FailReasonErrc::FRE_SYSTEM_ERROR:
                return code == PurchaseErrc::PE_INTERNAL_ERROR ||
                       code == RegisterProductErrc::RPE_INTERNAL_ERROR;
            default:
                break;
        }
        return false;
    }
};
```

이제 아래와 같이 특정 에러 코드가 특정 에러 조건에 해당하는 지를 확인할 수 있다.
```cpp
std::error_code ec = PurchaseErrc::PE_INVALID_CARD_NUMBER;
if (ec == FailReasonErrc::FRE_BAD_USER_INPUT)
    std::cout << "Your input was invalid; Please try again." << std::endl;
```

오류 처리가 아주 깔끔해졌다.\
내부적으로 가상함수 호출이 일어난다는 건 성능상 아쉬울 수는 있겠지만.

### `std::error_category::default_error_condition()`

에러 코드를 잘 나타내는 에러 조건이 있는 경우, 매칭을 에러 코드의 Category의 `default_error_condition()`을 overriding 해 표현할 수도 있다.

```cpp
struct PurchaseErrorCategory : std::error_category {
    ...
    auto default_error_condition(int ev) const noexcept -> std::error_condition {
        if (ev % 100 == 1) // 100번대 오류
            return FailReasonErrc::FRE_SELLER_NOT_READY;
        else if (ev % 100 == 2) // 200번대 오류
            return FailReasonErrc::FRE_BAD_USER_INPUT;
        else if (ev == static_cast<int>(PurchaseErrc::PE_INTERNAL_ERROR))
            return FailReasonErrc::FRE_SYSTEM_ERROR;
        else
            assert(false);
        return {};
    }
};
```

이게 작동하는 이유는 부모 `equivalent()` 가상함수가 아래와 같이 정의되어 있기 때문이다.
```cpp
bool equivalent(int code, const std::error_condition& cond) const noexcept {
    return default_error_condition(code) == cond;
}
```
따라서, `equivalent()`를 재정의할 경우, 당연히 `default_error_condition()`은 동작하지 않는다.

# 최종 예제 코드

[완성된 예제 코드](https://gist.github.com/copyrat90/caf2ea729852b01726ac4e615d684ebd)

# 실제 사용 케이스

## `<filesystem>` (C++17)

아무래도 파일시스템 조작을 하다 보면 정말 다양한 오류가 발생하기 마련이다.\
파일이 존재하지 않거나, 권한이 부족하거나, 파일이 아니라 디렉터리거나...\
보통 이런 파일시스템 오류가 발생하면 `errno`등의 OS 오류 코드가 설정된다.

[C++17 표준 `<filesystem>` API](https://en.cppreference.com/w/cpp/filesystem) 대부분은 이 오류 코드를 직접 얻어올 수 있도록, `std::error_code& ec`를 마지막 인자로 받는 오버로드를 제공한다.

예를 들어 [`std::filesystem::copy_file()`](https://en.cppreference.com/w/cpp/filesystem/copy_file)을 보면, 일반 오버로드인 (1)과 (3) 아래에 대응되는 `std::error_code& ec`가 있는 (2)와 (4) 오버로드가 존재한다.\
(2)와 (4) 오버로드를 호출하면, 오류 발생 시 `ec`에 에러 코드가 설정되고, 예외를 throw 하지 않는다.\
반면, (1)과 (3) 오버로드를 호출하면, [`std::filesystem_error`](https://en.cppreference.com/w/cpp/filesystem/filesystem_error)를 throw 한다.\
이 예외는 [`std::system_error`](https://en.cppreference.com/w/cpp/error/system_error)를 상속받는데, 거기에 있는 [`code()` 멤버함수](https://en.cppreference.com/w/cpp/error/system_error/code)로 에러 코드 객체를 얻을 수 있다.

또한, POSIX API 에러 조건에 대응되는 [`std::errc`](https://en.cppreference.com/w/cpp/error/errc) enum 과 [`std::system_category`](https://en.cppreference.com/w/cpp/error/system_category) 가 있어, 크로스 플랫폼 조건 비교를 쉽게 만들어줄 것으로 보이... 지만, Windows 전용 에러 코드가 제대로 매핑되어 있는지는 확인이 필요할 것 같다.

## asio 라이브러리

소켓 API 조작 또한 다양한 오류를 유발하는 대표적인 예시이다.\
유효하지 않은 주소를 사용하거나, 연결이 끊어졌거나, 바인딩되지 않은 소켓이거나...

asio 라이브러리 또한 `<filesystem>`과 같은 패턴을 사용한다.\
`ec`를 받는 오버로드와 받지 않는 오버로드를 같이 제공하고, `ec`를 받지 않는 버전은 `ec`를 포함하는 `asio::system_error`를 throw 한다.

(참고로, asio 라이브러리 작성자가 `std::error_code` 표준화에 기여한 사람이다.)

## 직접 만든 소켓 라이브러리

간단하게 [BSD socket 래퍼 라이브러리](https://github.com/copyrat90/DirtySocks)를 짜봤는데, 거기서도 매개변수로 `ec`를 받는 식으로 응용했다.

다만, 내 라이브러리는 `ec`를 반드시 써야하고, 예외를 던지는 API는 따로 제공하진 않았다.

# 참고자료

* Andrzej's C++ blog
    1. [Your own error code](https://akrzemi1.wordpress.com/2017/07/12/your-own-error-code/)
    1. [Your own error condition](https://akrzemi1.wordpress.com/2017/08/12/your-own-error-condition/)
    1. [Using error codes effectively](https://akrzemi1.wordpress.com/2017/09/04/using-error-codes-effectively/)
    1. [error codes - some clarifications](https://akrzemi1.wordpress.com/2017/10/14/error-codes-some-clarifications/)
* System error support in C++0x
    * [1편](http://blog.think-async.com/2010/04/system-error-support-in-c0x-part-1.html) / [2편](http://blog.think-async.com/2010/04/system-error-support-in-c0x-part-2.html) / [3편](http://blog.think-async.com/2010/04/system-error-support-in-c0x-part-3.html) / [4편](http://blog.think-async.com/2010/04/system-error-support-in-c0x-part-4.html) / [5편](http://blog.think-async.com/2010/04/system-error-support-in-c0x-part-5.html)
    * 위는 asio 라이브러리 개발자가 쓴 글로, 그분이 `std::error_code` 표준화에도 관여했으므로 읽어볼만하다.

*마지막 수정 : {{ page.last_modified_at }}*
