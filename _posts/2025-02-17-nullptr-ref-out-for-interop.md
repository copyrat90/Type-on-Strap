---
layout: post
title: C# Interop 중 nullptr인 `ref`, `out` 처리
tags: [C#]
author: copyrat90
last_modified_at: 2025-02-17T18:11:00+09:00
---

P/Invoke할 native 함수의 매개변수로 pointer가 있는데, 그게 nullptr이 가능하다면?

한 줄 요약: 넘길 때는 `Unsafe.NullRef<T>`, 반환받을 때는 `Unsafe.IsNullRef<T>`로 nullptr인지 체크.

# 배경

[GameNetworkingSockets](https://github.com/ValveSoftware/GameNetworkingSockets)을 [C#에서 P/Invoke로 감싸는 라이브러리를 fork](https://github.com/copyrat90/ValveSockets-CSharp)해서 수정하고 있다.

# 설명

[LibraryImport로 P/Invoke 처리](https://learn.microsoft.com/en-us/dotnet/standard/native-interop/pinvoke-source-generation#basic-usage)를 할 때, C++에서 pointer나 reference로 된 부분은 `ref`, `in` 이나 `out`, 혹은 배열이면 `Span<T>`로 수정하면, `Microsoft.Interop.LibraryImportGenerator`가 알아서 적절히 `fixed` pointer로 pinning 및 변환을 해 준다.

이를테면 [`ISteamNetworkingSockets::CreateListenSocketIP()`](https://partner.steamgames.com/doc/api/ISteamnetworkingSockets#CreateListenSocketIP)는 다음과 같다.
```cpp
HSteamListenSocket CreateListenSocketIP( const SteamNetworkingIPAddr &localAddress, int nOptions, const SteamNetworkingConfigValue_t *pOptions );
```

이걸 C# 쪽에서는 이런 식으로 LibraryImport 처리할 수 있다.
```cs
[LibraryImport("GameNetworkingSockets")]
[UnmanagedCallConv(CallConvs=new [] { typeof(CallConvCdecl) })]
public static partial uint SteamAPI_ISteamNetworkingSockets_CreateListenSocketIP(IntPtr self, in SteamNetworkingIPAddr localAddress, int nOptions, ReadOnlySpan<SteamNetworkingConfigValue> pOptions);
```

그러면 LibraryImportGenerator에 의해, 아래와 같은 코드가 자동 생성된다.
```cs
// 클래스 생략, 클래스에 unsafe가 붙어 있다.
[System.CodeDom.Compiler.GeneratedCodeAttribute("Microsoft.Interop.LibraryImportGenerator", "7.0.10.26716")]
[System.Runtime.CompilerServices.SkipLocalsInitAttribute]
public static partial uint SteamAPI_ISteamNetworkingSockets_CreateListenSocketIP(nint self, in global::Valve.Sockets.SteamNetworkingIPAddr localAddress, int nOptions, global::System.ReadOnlySpan<global::Valve.Sockets.SteamNetworkingConfigValue> pOptions)
{
    uint __retVal;
    // Pin - Pin data in preparation for calling the P/Invoke.
    fixed (global::Valve.Sockets.SteamNetworkingIPAddr* __localAddress_native = &localAddress)
    fixed (void* __pOptions_native = &global::System.Runtime.InteropServices.Marshalling.ReadOnlySpanMarshaller<global::Valve.Sockets.SteamNetworkingConfigValue, global::Valve.Sockets.SteamNetworkingConfigValue>.ManagedToUnmanagedIn.GetPinnableReference(pOptions))
    {
        __retVal = __PInvoke(self, __localAddress_native, nOptions, (global::Valve.Sockets.SteamNetworkingConfigValue*)__pOptions_native);
    }

    return __retVal;
    // Local P/Invoke
    [System.Runtime.InteropServices.DllImportAttribute("GameNetworkingSockets", EntryPoint = "SteamAPI_ISteamNetworkingSockets_CreateListenSocketIP", ExactSpelling = true)]
    [System.Runtime.InteropServices.UnmanagedCallConvAttribute(CallConvs = new System.Type[] { typeof(global::System.Runtime.CompilerServices.CallConvCdecl) })]
    static extern unsafe uint __PInvoke(nint self, global::Valve.Sockets.SteamNetworkingIPAddr* localAddress, int nOptions, global::Valve.Sockets.SteamNetworkingConfigValue* pOptions);
}
```
알아서 전달한 값을 Native 쪽에서 볼 수 있는 pointer로 변환 및 pinning하여 전달해주고 있다.

그런데 API에 따라서는 pointer 변수가 optional이라서, 상황에 따라 `nullptr`을 집어넣어야 할 수도 있다.\
이걸 `in`, `ref`나 `out` keyword로 선언해버리면 Null reference를 어떻게 넣을까?\
답은 간단한데, [`Unsafe.NullRef<T>()`](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.unsafe.nullref?view=net-9.0)를 쓰면 된다.

또한, native 쪽에 원본이 있고 그게 `in`, `ref`로 콜백돼 넘어오는 경우라면, [`Unsafe.IsNullRef<T>()`](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.unsafe.isnullref?view=net-9.0)로 체크할 수 있다.

# 문제 상황

하지만 pointer의 배열이 넘어오는 상황은 어떻게 처리해야 할까?

이를테면, [`ISteamnetworkingSockets::ReceiveMessagesOnPollGroup()`](https://partner.steamgames.com/doc/api/ISteamnetworkingSockets#ReceiveMessagesOnPollGroup)은\
아래와 같이 `ppOutMessages`로 여러 개의 메시지를 여러 pointer를 배열에 넣어 주는 식으로 받도록 돼 있다.
```cpp
int ReceiveMessagesOnPollGroup( HSteamNetPollGroup hPollGroup, SteamNetworkingMessage_t **ppOutMessages, int nMaxMessages );
```

안타깝게도 `Span<ref T>`는 불가능하므로, 이 때는 `Span<IntPtr>`로 `IntPtr`로 받아온 후에, 호출한 측에서 알아서 잘 `T`로 변환하는 방법밖에는 없을 것 같다.\
(이 라이브러리에 국한된 얘기지만, 어차피 이 함수로 받은 메시지는 [`SteamNetworkingMessage_t::Release()` 호출해서 지워야해서](https://partner.steamgames.com/doc/api/ISteamNetworkingSockets#ReceiveMessagesOnConnection) `IntPtr`을 들고 있긴 해야한다. )
```cs
[LibraryImport("GameNetworkingSockets")]
[UnmanagedCallConv(CallConvs=new [] { typeof(CallConvCdecl) })]
public static partial int SteamAPI_ISteamNetworkingSockets_ReceiveMessagesOnPollGroup(IntPtr self,uint hPollGroup,Span<IntPtr> ppOutMessages,int nMaxMessages);
```

참고로, 변환 시에 [`Marshal.PtrToStructure<T>`](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.marshal.ptrtostructure?view=net-9.0)는 managed object로 복사를 하므로,\
복사가 필요없다면 성능을 생각해 [배열을 돌면서 1칸짜리 `Span<T>`를 읽어서 처리하면 좋겠다.](https://github.com/nxrighthere/ValveSockets-CSharp/blob/abe9031a25b2f5d6d6e2530fbd908faade6eb896/ValveSockets.cs#L580-L593)\
(사족으로, 방금 건 링크의 코드에선 `ArrayPool`을 사용하는데, 배열 크기가 작다면 `stackalloc`도 괜찮은 방법일 듯하다.)


# 참고 자료

* [dotnet/runtime GitHub Issue - A better way to PInvoke nullable pointers?](https://github.com/dotnet/runtime/issues/4392)
* [dotnet/csharplang GitHub Discussions - Proposal: For static extern methods, C# should support passing `null` to any `out` or `ref` parameters marked with `[Optional]`.](https://github.com/dotnet/csharplang/discussions/79)

*마지막 수정 : {{ page.last_modified_at }}*
