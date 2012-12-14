This project provides a performant, flexible implementations of Colin Percival's [scrypt](http://www.tarsnap.com/scrypt.html).

# Features

## Modular Design

The code uses a modular (compile, not runtime) layout to allow new mixing & hash functions to be added easily. The base components (HMAC, PBKDF2, and scrypt) are static and will immediately work with any conforming mix or hash function.

## Supported Mix Functions

* [Salsa20/8](http://cr.yp.to/salsa20.html)
* [ChaCha20/8](http://cr.yp.to/chacha.html)
* [Salsa6420/8]()

I am not actually aware of any other candidates for a decent mix function. Salsa20/8 was nearly perfect, but its successor, ChaCha20/8, has better diffusion and is thus stronger, is potentially faster given advanced SIMD support (byte level shuffles, or a 32bit rotate), and is slightly cleaner to implement given that it requires no pre/post processing of data for SIMD implementations. 

64-byte blocks are no longer assumed! Salsa6420/8 is a 'proof of concept' 64-bit version of Salsa20/8 with a 128 byte block, and rotation constants chosen to allow 32-bit word shuffles instead of rotations for two of the rotations which put it on par with ChaCha in terms of SSE implementation shortcuts.

## Supported Hash Functions

* SHA256/512
* [BLAKE256/512](https://www.131002.net/blake/)
* [Skein512](http://www.skein-hash.info/)
* [Keccak256/512](http://keccak.noekeon.org/) (SHA-3)

Hash function implementations, unlike mix functions, are not optimized. The PBKDF2 computations are relatively minor in the scrypt algorithm, so including CPU specific versions, or vastly unrolling loops, would serve little purpose while bloating the code, both source and binary, and making it more confusing to implement correctly.

Most (now only two!) of the SHA-3 candidates fall in to the "annoying to read/implement" category and have not been included yet. This will of course be moot once ~~BLAKE is chosen as SHA-3~~ Keccak is chosen as SHA-3. Well shit.

## CPU Adaptation

The mixing function specialization is selected at runtime based on what the CPU supports (well, x86/x86-64 for now, but theoretically any). On platforms where this is not needed, e.g. where packages are usually compiled from source, it can also select the most suitable implementation at compile time, cutting down on binary size.

For those who are familiar with the scrypt spec, the code specializes at the ROMix level, allowing all copy, and xor calls to be inlined efficiently. ***Update***: This is actually not as important as I switched from specializing at the mix() level and letting the compiler somewhat inefficiently inline block_copy and block_xor to specializing at ChunkMix(), where they can be inlined properly. I thought about specializing at ROMix(), but it would increase the complexity per mix function even more and would not present many more opportunities than what is generated by the compiler presently.

MSVC uses SSE intrinsics as opposed to inline assembly for the mix functions to allow the compiler to fully inline properly. Also, Visual Studio is not smart enough to allow inline assembly in 64-bit code. 

## Self Testing

On first use, scrypt() runs a small series of tests to make sure the hash function, mix functions, and scrypt() itself, are generating correct results. It will exit() (or call a user defined fatal error function) should any of these tests fail. 

Test vectors for individual mix and hash functions are generated from reference implementations. The only "official" test vectors for the full scrypt() are for SHA256 + Salsa20/8 of course; other combinations are generated from this code (once it works with all reference test vectors) and subject to change if any implementation errors are discovered.

# Performance (on an E5200 2.5GHZ)

Benchmarks are run _without_ allocating memory, i.e. allocating enough memory before the trials are run. Different allocators can have different costs and non-deterministic effects, which is not the point of comparing implementations. The only hash function compared will be SHA-256 to be comparable to Colin's reference implementation, and the hash function will generally be a fraction of a percent of noise in the overall result.

Three different scrypt settings are tested (the last two are from the scrypt paper): 

* 'High Volume': N=4096, r=8, p=1, 4mb memory
* 'Interactive': N=16384, r=8, p=1, 16mb memory
* 'Non-Interactive': N=1048576, r=8, p=1, 1gb memory

Cycle counts are in millions of cycles. All versions compiled with gcc 4.6.3, -O3. Sorted from fastest to slowest.


<table>
<thead><tr><th>Implemenation</th><th>Algo</th><th>High Volume</th><th>Interactive</th><th>Non-Interactive</th></tr></thead>
<tbody>
<tr><td>scrypt-jane SSSE3 64bit</td><td>Salsa6420/8 </td><td>18.2m</td><td> 75.6m</td><td>5120.0m</td></tr>
<tr><td>scrypt-jane SSSE3 64bit</td><td>ChaCha20/8</td><td>19.6m</td><td> 79.6m</td><td>5296.7m</td></tr>
<tr><td>scrypt-jane SSSE3 32bit</td><td>ChaCha20/8</td><td>19.8m</td><td> 80.3m</td><td>5346.1m</td></tr>
<tr><td>scrypt-jane SSE2 64bit </td><td>Salsa6420/8 </td><td>19.8m</td><td> 82.1m</td><td>5529.2m</td></tr>
<tr><td>scrypt-jane SSE2 64bit </td><td>Salsa20/8 </td><td>22.1m</td><td> 89.7m</td><td>5938.8m</td></tr>
<tr><td>scrypt-jane SSE2 32bit </td><td>Salsa20/8 </td><td>22.3m</td><td> 90.6m</td><td>6011.0m</td></tr>
<tr><td>scrypt-jane SSE2 64bit </td><td>ChaCha20/8</td><td>23.9m</td><td> 96.8m</td><td>6399.7m</td></tr>
<tr><td>scrypt-jane SSE2 32bit </td><td>ChaCha20/8</td><td>24.2m</td><td> 98.3m</td><td>6500.7m</td></tr>
<tr><td>*Reference SSE2 64bit* </td><td>Salsa20/8 </td><td>32.9m</td><td>135.2m</td><td>8881.6m</td></tr>
<tr><td>*Reference SSE2 32bit* </td><td>Salsa20/8 </td><td>33.0m</td><td>134.4m</td><td>8885.2m</td></tr>
</tbody>
</table>

* scrypt-jane Salsa6420/8-SSSE3 is ~1.80x faster than reference Salsa20/8-SSE2 for High Volume, but drops to 1.73x faster for 'Non-Interactive' instead of remaining constant
* scrypt-jane ChaCha20/8-SSSE3 is ~1.67x faster than reference Salsa20/8-SSE2 
* scrypt-jane Salsa20/8-SSE2 is ~1.48x faster than reference Salsa20/8-SSE2 
* I decided to leave out the non-SIMD implemenations as nobody will be relying on them for performance

# Building

    [gcc,icc,clang] scrypt-jane.c -O3 -[m32,m64] -DSCRYPT_MIX -DSCRYPT_HASH -c

where SCRYPT_MIX is one of

* SCRYPT_SALSA
* SCRYPT_SALSA64 (no optimized 32-bit implementation)
* SCRYPT_CHACHA

and SCRYPT_HASH is one of

* SCRYPT_SHA256
* SCRYPT_SHA512
* SCRYPT_BLAKE256
* SCRYPT_BLAKE512
* SCRYPT_SKEIN512
* SCRYPT_KECCAK256
* SCRYPT_KECCAK512

e.g.

    gcc scrypt-jane.c -O3 -DSCRYPT_CHACHA -DSCRYPT_BLAKE512 -c
    gcc example.c scrypt-jane.o -o example

clang *may* need "-no-integrated-as" as some? versions don't support ".intel_syntax"

# Using

    #include "scrypt-jane.h"

    scrypt(password, password_len, salt, salt_len, Nfactor, pfactor, rfactor, out, want_bytes);

## scrypt parameters

* Nfactor: Increases CPU & Memory Hardness
* rfactor: Increases Memory Hardness
* pfactor: Increases CPU Hardness

In scrypt terms

* N = (1 << (Nfactor + 1)), which controls how many times to mix each chunk, and how many temporary chunks are used. Increasing N increases both CPU time and memory used. 
* r = (1 << rfactor), which controls how many blocks are in a chunk (i.e., 2 * r blocks are in a chunk). Increasing r increases how much memory is used.
* p = (1 << pfactor), which controls how many passes to perform over the set of N chunks. Increasing p increases CPU time used.

I chose to use the log2 of each parameter as it is the common way to communicate settings (e.g. 2^20, not 1048576).

# License

Public Domain, or MIT