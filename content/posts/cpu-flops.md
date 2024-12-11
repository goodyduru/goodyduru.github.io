+++
date = '2024-12-11T15:52:25+01:00'
title = 'CPU FLOPs'
draft = false
+++
My friend and I have been taking some performance engineering courses. One thing we've come to realize is how important calculating the maximum Floating-point Operations per second (FLOPs) of a CPU is. It turns out this is a little trickier than it looks. 

This is a formula I use to calculate this value. The formula is `number of cores * peak cpu speed * (max SIMD register size/32) * (fma throughput * 2)`. Let me break it down.
* Number of cores: The amount of cores in its CPU.
* Peak CPU speed: The highest speed the CPU can achieve. On Intel, it's Turbo Boost. On AMD, it's Max. Boost Clock.
* Max SIMD register size: Single Instruction Multiple Data (SIMD) allows a program to carry out vectorized operations using special registers. Some of the extensions are SSE, AVX, AVX2, and AVX-512. There are other extensions on [Wikipedia](https://en.wikipedia.org/wiki/Streaming_SIMD_Extensions#Later_versions). You can find a CPU's supported extension on its web page. CPUs that only support SSE have a max SIMD register size of 128 bits. Those that support AVX or AVX2 have a max SIMD register size of 256 bits. Those that support AVX-512 have a max SIMD register size of 512 bits. 

    The size of a single-precision float type is 4 bytes. To check the maximum number of floats a SIMD register can support, the formula is `register size / 4 bytes * 8 bits in a byte`.
* FMA throughput: This is the tricky one. Fused Multiply Add (FMA) allows a program to carry out this operation `A*B + C` in a single instruction ([VFMADD...](https://www.felixcloutier.com/x86/vfmadd132ps:vfmadd213ps:vfmadd231ps)). Getting a CPU's microarchitecture is the first step in figuring out its FMA throughput. Once that's done, we can get its throughput of the FMA instruction in the _Reciprocal throughput_ column of the _Arithmetic_ subsection within the _Floating point XMM and YMM instructions_ table of the CPU's microarchitecture chapter in the excellent Agner Fog's [Instruction Tables](https://agner.org/optimize/instruction_tables.pdf) doc. We divide 1 by this reciprocal throughput to get the throughput. This throughput defines the number of FMA operations that the CPU can execute in a single cycle. 

 Finally, we multiply this number by 2 because FMA contains 2 operations. We should note that the FMA instruction was only introduced in 2012. Processors made before that year will not support it. In their case, we use the [VMULPS](https://uops.info/html-instr/VMULPS_YMM_YMM_YMM.html) instruction and multiply by 1 instead of 2 in the equation.

#### A Simple Example
The cheapest AMD dedicated server on Hetzner is the [AX42](https://www.hetzner.com/dedicated-rootserver/matrix-ax/). This has an AMD Ryzen™ 7 PRO 8700GE CPU. From its [page](https://www.amd.com/en/products/processors/desktops/ryzen-pro/8000-series/amd-ryzen-7-pro-8700ge.html), we can see it's a Zen 4 microarchitecture CPU. Let's calculate its FLOPs.
```
Number of cores: 8
Peak CPU speed: 5.1 GHz
Max SIMD register size: 512 bits (It supports AVX-512). This means we can do vectorized operations on 16 floats at a time.
FMA throughput: 1 (For smaller vector register sizes, it's 2). 
Its peak FLOPs: 8 * 5.1 * 10^9 * 16 * 1 * 2 = 1305.6 GFLOPs.
```
It's surreal that we can get all that power for just €46.00/month.
