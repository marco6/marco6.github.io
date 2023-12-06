---
layout: post
title:  "Endianness madness"
date:   2023-10-06 10:00:00 +0100
tags: programming, endianness, C, C++
---

<style>pre { max-height: 350px }</style>

[A](https://github.com/postgres/postgres/blob/master/src/backend/utils/adt/network.c#L652) [lot](https://github.com/xiph/flac/blob/master/src/flac/encode.c#L2301) [of](https://github.com/sqlite/sqlite/blob/master/src/wal.c#L869) `C` code in the wild does something on the lines of:

```C
void write32(char *buffer, uint32_t n) {
    memcpy(buffer, (char*)&n, 4);
}
```

This code, however, is clearly not portable as the order of the bytes of `n` is architecture-dependent. This is especially clear in lazy-oriented programming, when an entire structure is just written to a file:

```C
struct state {
    ... 
    // Many fields here
};

int write_state(FILE* f, struct state* s) {
    return fwrite(s, sizeof(*s), 1, f);
}
```

Which is even a worse idea since `struct state` might be:

 - different in size, depending on the architecture;
 - different in *size and layout* on the same architecture, but under a different *data model* [^2].

[^2]: an example for this is x86-64: LLP64 data model uses 4-byte `long`s (used on Windows) while in LP64 (*nix) the same type is twice that size (i.e. 8 bytes). 

So, the Good Programmer<sup>TM</sup> will understand this and will:

 - define the size of each field beforehand (or use a stricter type like `uint32_t`);
 - swap the bytes, if needed, before writing to disk/network.

However, the challenge with byte swapping arises from the necessity to determine the system's endianness. If the byte order of your architecture aligns with the output order, swapping becomes unnecessary.

There are plenty of bad ways to do that, but what I found most intriguing was:

```C
// Pre-C99, invalid
#define IS_BIG_ENDIAN ((*(uint16_t*)("\1\2")) == 0x0102)

// C99-compliant
#define IS_BIG_ENDIAN (!*(unsigned char *)&(uint16_t){1})
```

The first is not valid neither C nor in C++, the second is valid only in `C99` and above. However, they still work on all major compilers. Speaking of which, relying on compiler-specific stuff is also an option. `gcc` has had `__BYTE_ORDER__` macro for quite a while now. In C++, you could use `std::endian`. Another viable option is to rely on OS-specific stuff (like POSIX's `<endian.h>` header).

Nevertheless, writing conditional logic to consider this factor is necessary. I remember I once read [an article](https://commandcenter.blogspot.com/2012/04/byte-order-fallacy.html) talking about this exact problem and providing a simple solution: writing each individual byte as you expect it to be. It seemed like very solid advice to be honest. In detail, he was proposing something on the lines of:

```C
void write32le(char* buffer, int value) {
    buffer[0] = (char)(value);
    buffer[1] = (char)(value >> 8);
    buffer[2] = (char)(value >> 16);
    buffer[3] = (char)(value >> 24);
}

void write32be(char* buffer, int value) {
    buffer[0] = (char)(value >> 24);
    buffer[1] = (char)(value >> 16);
    buffer[2] = (char)(value >> 8);
    buffer[3] = (char)(value);
}
```

Now, this is what I like: simple code, always work, no compiler-specific, no undefined-behavior.

## C++ to the rescue 

Admittedly, this article might seem too *boring* and *simple*. I might as well wrote the same in Go, where those properties are idolatrised.

But not on my whatch (or on my personal blog). Let's embrace some C++ madness and write the generic version of that. 

```C++
namespace little_endian {
    template <size_t S, std::integral T>
    requires (0 < S && S <= sizeof(T))
    constexpr void write(std::span<char> output, T value) {
        if constexpr (std::is_signed_v<T>) {
            using unsigned_T = std::make_unsigned_t<T>;
            return write<S, unsigned_T>(output, static_cast<unsigned_T>(value));
        }

        for (size_t i = 0; i < S; i++) {
            output[i] = static_cast<char>(value >> i*8);
        }
    }
}
```

This interface is more complex compared to the original C version, but adds some nice features:

 - it lets you specify the byte count to write. This way it is possible to write a number with bit-count which is not part of the C++ type system (e.g. `write<24>(buffer, 10)`)
 - it supports both signed and unsigned numbers;
 - it supports *all* sizes, so if by chanche you have access to `int128_t`, you can use it.

Still, it bothers me that for simple cases I can't just write `write(buffer, 10)`. To account for that, it is possible to add an overload to simplify things:

```C++
namespace little_endian {
    ...
    template <std::integral T>
    constexpr void write(std::span<char> output, T value) {
        write<sizeof(T), T>(output, value);
    }
}
```

Sweet! Now I can have my cake and eat it, too.

## Performance

How about the performance of the solution above? Some might assume that byte-order swapping would execute faster due to being a compiler intrinsic, especially when compared to the C++ version, which contains a loop.

So, let's inspect the [assembly generated](https://godbolt.org/z/G3WK44Wqa) by:

**GCC 13**

Using `-std=c++20 -O3` [^3]:

[^3]: Notice that the `64-bit` version is not this beautiful whith `-O2`.

```asm
void little_endian::write<unsigned short>(std::span<char, 18446744073709551615ul>, unsigned short):
        mov     WORD PTR [rdi], dx
        ret
void little_endian::write<unsigned int>(std::span<char, 18446744073709551615ul>, unsigned int):
        mov     DWORD PTR [rdi], edx
        ret
void little_endian::write<unsigned long>(std::span<char, 18446744073709551615ul>, unsigned long):
        mov     QWORD PTR [rdi], rdx
        ret
```

Just perfect.

**Clang 17**

Using `-std=c++20 -O2`.

```C++
void little_endian::write<unsigned short>(std::span<char, 18446744073709551615ul>, unsigned short): # @void little_endian::write<unsigned short>(std::span<char, 18446744073709551615ul>, unsigned short)
        mov     word ptr [rdi], dx
        ret
void little_endian::write<unsigned int>(std::span<char, 18446744073709551615ul>, unsigned int): # @void little_endian::write<unsigned int>(std::span<char, 18446744073709551615ul>, unsigned int)
        mov     dword ptr [rdi], edx
        ret
void little_endian::write<unsigned long>(std::span<char, 18446744073709551615ul>, unsigned long): # @void little_endian::write<unsigned long>(std::span<char, 18446744073709551615ul>, unsigned long)
        mov     qword ptr [rdi], rdx
        ret
```

On par with GCC. Awesome.

**MSVC**

Using `/std:c++20 /O2` (and tried with `/std:c++20 /Os`).

```asm
output$ = 8
value$ = 16
void little_endian::write<unsigned __int64>(std::span<char,-1>,unsigned __int64) PROC ; little_endian::write<unsigned __int64>, COMDAT
        mov     rcx, QWORD PTR [rcx]
        mov     rax, rdx
        shr     rax, 8
        mov     BYTE PTR [rcx], dl
        mov     BYTE PTR [rcx+1], al
        mov     rax, rdx
        shr     rax, 16
        mov     BYTE PTR [rcx+2], al
        mov     rax, rdx
        shr     rax, 24
        mov     BYTE PTR [rcx+3], al
        mov     rax, rdx
        shr     rax, 32                             ; 00000020H
        mov     BYTE PTR [rcx+4], al
        mov     rax, rdx
        shr     rax, 40                             ; 00000028H
        mov     BYTE PTR [rcx+5], al
        mov     rax, rdx
        shr     rax, 48                             ; 00000030H
        shr     rdx, 56                             ; 00000038H
        mov     BYTE PTR [rcx+6], al
        mov     BYTE PTR [rcx+7], dl
        ret     0
void little_endian::write<unsigned __int64>(std::span<char,-1>,unsigned __int64) ENDP ; little_endian::write<unsigned __int64>

output$ = 8
value$ = 16
void little_endian::write<unsigned int>(std::span<char,-1>,unsigned int) PROC ; little_endian::write<unsigned int>, COMDAT
        mov     rcx, QWORD PTR [rcx]
        mov     eax, edx
        shr     eax, 8
        mov     BYTE PTR [rcx], dl
        mov     BYTE PTR [rcx+1], al
        mov     eax, edx
        shr     eax, 16
        shr     edx, 24
        mov     BYTE PTR [rcx+2], al
        mov     BYTE PTR [rcx+3], dl
        ret     0
void little_endian::write<unsigned int>(std::span<char,-1>,unsigned int) ENDP ; little_endian::write<unsigned int>

$T1 = 0
output$ = 32
value$ = 40
void little_endian::write<unsigned short>(std::span<char,-1>,unsigned short) PROC ; little_endian::write<unsigned short>, COMDAT
$LN16:
        sub     rsp, 24
        movups  xmm0, XMMWORD PTR [rcx]
        xor     eax, eax
        movaps  XMMWORD PTR $T1[rsp], xmm0
        mov     r9, QWORD PTR $T1[rsp]
$LL6@write:
        movzx   ecx, al
        movzx   r8d, dx
        shl     cl, 3
        shr     r8w, cl
        mov     BYTE PTR [r9+rax], r8b
        inc     rax
        cmp     rax, 2
        jb      SHORT $LL6@write
        add     rsp, 24
        ret     0
void little_endian::write<unsigned short>(std::span<char,-1>,unsigned short) ENDP ; little_endian::write<unsigned short>

output$ = 8
value$ = 16
void little_endian::write<2,unsigned short>(std::span<char,-1>,unsigned short) PROC ; little_endian::write<2,unsigned short>, COMDAT
        mov     r9, QWORD PTR [rcx]
        xor     eax, eax
        npad    11
$LL4@write:
        movzx   ecx, al
        movzx   r8d, dx
        shl     cl, 3
        shr     r8w, cl
        mov     BYTE PTR [r9+rax], r8b
        inc     rax
        cmp     rax, 2
        jb      SHORT $LL4@write
        ret     0
void little_endian::write<2,unsigned short>(std::span<char,-1>,unsigned short) ENDP ; little_endian::write<2,unsigned short>

output$ = 8
value$ = 16
void little_endian::write<4,unsigned int>(std::span<char,-1>,unsigned int) PROC ; little_endian::write<4,unsigned int>, COMDAT
        mov     r8, QWORD PTR [rcx]
        mov     eax, edx
        shr     eax, 8
        mov     BYTE PTR [r8+1], al
        mov     eax, edx
        mov     BYTE PTR [r8], dl
        shr     eax, 16
        shr     edx, 24
        mov     BYTE PTR [r8+2], al
        mov     BYTE PTR [r8+3], dl
        ret     0
void little_endian::write<4,unsigned int>(std::span<char,-1>,unsigned int) ENDP ; little_endian::write<4,unsigned int>

output$ = 8
value$ = 16
void little_endian::write<8,unsigned __int64>(std::span<char,-1>,unsigned __int64) PROC ; little_endian::write<8,unsigned __int64>, COMDAT
        mov     r8, QWORD PTR [rcx]
        mov     rax, rdx
        shr     rax, 8
        mov     BYTE PTR [r8+1], al
        mov     rax, rdx
        shr     rax, 16
        mov     BYTE PTR [r8+2], al
        mov     rax, rdx
        shr     rax, 24
        mov     BYTE PTR [r8+3], al
        mov     rax, rdx
        shr     rax, 32                             ; 00000020H
        mov     BYTE PTR [r8+4], al
        mov     rax, rdx
        shr     rax, 40                             ; 00000028H
        mov     BYTE PTR [r8+5], al
        mov     rax, rdx
        mov     BYTE PTR [r8], dl
        shr     rax, 48                             ; 00000030H
        shr     rdx, 56                             ; 00000038H
        mov     BYTE PTR [r8+6], al
        mov     BYTE PTR [r8+7], dl
        ret     0
void little_endian::write<8,unsigned __int64>(std::span<char,-1>,unsigned __int64) ENDP ; little_endian::write<8,unsigned __int64>
```

~~Beautiful as well~~.

No, wait! What? MSVC apparently is not understanding this simple piece of code.

To be honest, this is not the first time I see MSVC generating sub-par assembly. As sometimes it gets confused when using templates. I attempted the same with the C version (`write32le`), but unfortunately had no luck there either.

Nevertheless, I would pretty much prefer this code, even if slower on Windows, as it is portable and simple. Sometimes performance is not the first aim. At times, if performance is the main priority, considering a change in the compiler could be a viable option.

## Reading

Reading follows the same ideas.

```C++
namespace byte_order::little_endian {
    template <std::integral T, size_t S = sizeof(T)>
    requires (0 < S && S <= sizeof(T))
    constexpr T read(std::span<const char> output) {
        if constexpr (std::is_signed_v<T>) {
            return static_cast<T>(read<S, std::make_unsigned_t<T>>(output));
        } else {
            T result = 0;
            for (size_t i = 0; i < S; i++) {
                result |= static_cast<T>(static_cast<unsigned char>(output[i])) << (i*8);
            }
            return result;
        }
    }
}
```

Notice how it's required to first cast to `unsigned char` and then to `T` to properly handle `char` signedness[^4].

[^4]: directly converting `char` to `T` would have sign-extended the register leading to a wrong result.

This time, results are a bit different though: Clang and MSVC don't change their behavior, leading to similar code.

**Clang**

Using `-std=c++20 -O2`.

```asm
unsigned short byte_order::little_endian::read<unsigned short, 2ul>(std::span<char const, 18446744073709551615ul>): # @unsigned short byte_order::little_endian::read<unsigned short, 2ul>(std::span<char const, 18446744073709551615ul>)
        movzx   eax, word ptr [rdi]
        ret
unsigned int byte_order::little_endian::read<unsigned int, 4ul>(std::span<char const, 18446744073709551615ul>): # @unsigned int byte_order::little_endian::read<unsigned int, 4ul>(std::span<char const, 18446744073709551615ul>)
        mov     eax, dword ptr [rdi]
        ret
unsigned long byte_order::little_endian::read<unsigned long, 8ul>(std::span<char const, 18446744073709551615ul>): # @unsigned long byte_order::little_endian::read<unsigned long, 8ul>(std::span<char const, 18446744073709551615ul>)
        mov     rax, qword ptr [rdi]
        ret
```

**MSVC**

```C++
output$ = 8
unsigned __int64 byte_order::little_endian::read<unsigned __int64,8>(std::span<char const ,-1>) PROC ; byte_order::little_endian::read<unsigned __int64,8>, COMDAT
        mov     rdx, QWORD PTR [rcx]
        movzx   eax, BYTE PTR [rdx+6]
        movzx   ecx, BYTE PTR [rdx+2]
        movzx   r8d, BYTE PTR [rdx+7]
        shl     r8, 8
        or      r8, rax
        movzx   eax, BYTE PTR [rdx+5]
        shl     r8, 8
        or      r8, rax
        movzx   eax, BYTE PTR [rdx+4]
        shl     r8, 8
        or      r8, rax
        movzx   eax, BYTE PTR [rdx+3]
        shl     r8, 8
        or      rax, r8
        shl     rax, 8
        or      rax, rcx
        movzx   ecx, BYTE PTR [rdx+1]
        shl     rax, 8
        or      rax, rcx
        movzx   ecx, BYTE PTR [rdx]
        shl     rax, 8
        or      rax, rcx
        ret     0
unsigned __int64 byte_order::little_endian::read<unsigned __int64,8>(std::span<char const ,-1>) ENDP ; byte_order::little_endian::read<unsigned __int64,8>

output$ = 8
unsigned short byte_order::little_endian::read<unsigned short,2>(std::span<char const ,-1>) PROC ; byte_order::little_endian::read<unsigned short,2>, COMDAT
        mov     rdx, QWORD PTR [rcx]
        movzx   eax, BYTE PTR [rdx+1]
        movzx   ecx, BYTE PTR [rdx]
        shl     ax, 8
        or      ax, cx
        ret     0
unsigned short byte_order::little_endian::read<unsigned short,2>(std::span<char const ,-1>) ENDP ; byte_order::little_endian::read<unsigned short,2>

output$ = 8
unsigned int byte_order::little_endian::read<unsigned int,4>(std::span<char const ,-1>) PROC ; byte_order::little_endian::read<unsigned int,4>, COMDAT
        mov     rdx, QWORD PTR [rcx]
        movzx   ecx, BYTE PTR [rdx+2]
        movzx   eax, BYTE PTR [rdx+3]
        shl     eax, 8
        or      eax, ecx
        movzx   ecx, BYTE PTR [rdx+1]
        shl     eax, 8
        or      eax, ecx
        movzx   ecx, BYTE PTR [rdx]
        shl     eax, 8
        or      eax, ecx
        ret     0
unsigned int byte_order::little_endian::read<unsigned int,4>(std::span<char const ,-1>) ENDP ; byte_order::little_endian::read<unsigned int,4>
```


This time, however, GCC doesn't seem to handle the generation properly, generating results similar to MSVC. However, hand-writing the routine, gets things better.

```C++
uint32_t read32le(unsigned char* data) {
    return  (data[0]<<0) | (data[1]<<8) | (data[2]<<16) | (data[3]<<24);
}
```

Generates (in GCC, *not in MSVC*):

```asm
read32le(unsigned char*):
        mov     eax, DWORD PTR [rdi]
        ret
```

So, most likely it's the loop confusing GCC. What if I remove it and merge everything in a single embeddable expression? It's possible to get there by recusively loading the high part and the low part, each of them half the size of the result.

```C++
namespace byte_order::little_endian {
    template <std::integral T, size_t S = sizeof(T)>
    requires (0 < S && S <= sizeof(T))
    constexpr T read(std::span<const char> buffer) {
        if constexpr (std::is_signed_v<T>) {
            return static_cast<T>(read<S, std::make_unsigned_t<T>>(buffer));
        } else if constexpr (S == 1) {
            return static_cast<uint8_t>(buffer[0]);
        } else {
            const auto low_part = read<T, S/2>(buffer);
            const auto high_part = read<T, (S+1)/2>(buffer.subspan(S/2));

            return low_part | (high_part << ((S/2)*8));
        }
    }
}
```

And this solves for GCC as well, leaving (again) MSVC the only one generating sub-par assembly.

```asm
unsigned short byte_order::little_endian::read<unsigned short, 2ul>(std::span<char const, 18446744073709551615ul>):
        movzx   eax, WORD PTR [rdi]
        ret
unsigned int byte_order::little_endian::read<unsigned int, 4ul>(std::span<char const, 18446744073709551615ul>):
        mov     eax, DWORD PTR [rdi]
        ret
unsigned long byte_order::little_endian::read<unsigned long, 8ul>(std::span<char const, 18446744073709551615ul>):
        mov     rax, QWORD PTR [rdi]
        ret
```

You can find the complete code [here](https://godbolt.org/z/zo57hExf8).

----------------

