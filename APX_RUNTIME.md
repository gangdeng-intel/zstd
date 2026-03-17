# Zstd Dynamic Dispatch for Intel APX

This document explains how the Zstd framework dynamically checks for Intel Advanced Performance Extensions (APX) support and safely transitions code execution to the APX-optimized code loops. The process relies on "Dynamic Dispatch" in four chronological steps.

## 1. Hardware Detection via CPUID (The Probing)

Before running optimized code, Zstd checks your CPU hardware and OS configuration for capability. This is defined in `lib/common/cpu.h`.

First, an empty state is populated by a native `cpuid` call gathering processor information from several "leaves". The APX specific checks are:

1. **Does the hardware base support APX?**
   It probes `CPUID 0x07` (Subleaf 1, ECX = 1). The base APX support is mapped to EDX bit 21. Zstd macros read this through `ZSTD_cpuid_apx_f_hw()`.
2. **Does the hardware support APX extended sub-features?**
   It probes `CPUID 0x29` (Subleaf 0, ECX = 0). The `NDD` (Non-Destructive Destination) and `NF` (No Flags) capabilities are listed on EBX bit 0. This is checked using `ZSTD_cpuid_apx_nci_ndd_nf_hw()`.
3. **Is the OS Kernel aware of APX?**
   Hardware support isn't enough; the OS must save/restore the Extended GPRs (r16-r31) upon thread context switching. Zstd runs `XGETBV` to read `XCR0` and checks bit 19.

Zstd consolidates these three tests into a single master boolean:

```c
MEM_STATIC int ZSTD_cpuid_apx_f(ZSTD_cpuid_t const cpuid) {
    return ZSTD_cpuid_apx_f_hw(cpuid) && 
           ZSTD_cpuid_apx_nci_ndd_nf_hw(cpuid) && 
           ((cpuid.xcr0_eax & (1U << 19)) != 0);
}
```

*If this function returns `1`, it is completely safe to run Intel APX instructions.*

## 2. State Initialization (The Caching)

Constantly issuing `CPUID` and `XGETBV` operations inside a fast decompression loop is slow. Instead, strings are evaluated exactly *once* during context creation.

In `lib/decompress/zstd_decompress.c` inside `ZSTD_initDCtx_internal`:

```c
#if DYNAMIC_APXF
    dctx->apxf = ZSTD_cpuSupportsApxf();
#endif
```

The result of the CPUID check is cached cleanly inside the `dctx` (Decompression Context) state object.

## 3. Loop Dispatch (The Routing)

During decompression, the core routine asks for the standard decoding functions (like `ZSTD_decompressSequences`). At this point, the application uses the cached state to dynamically branch to different sub-functions (located in `lib/decompress/zstd_decompress_block.c`):

```c
#if DYNAMIC_APXF
    if (ZSTD_DCtx_get_apxf(dctx)) {
        return ZSTD_decompressSequences_apxf(dctx, dst, maxDstSize, seqStart, seqSize, nbSeq, isLongOffset);
    }
#endif
#if DYNAMIC_BMI2
    if (ZSTD_DCtx_get_bmi2(dctx)) {
        return ZSTD_decompressSequences_bmi2(dctx, dst, maxDstSize, seqStart, seqSize, nbSeq, isLongOffset);
    }
#endif
    return ZSTD_decompressSequences_default(dctx, dst, maxDstSize, seqStart, seqSize, nbSeq, isLongOffset);
```

* Notice the fallback chain: If your system supports APX, it routes down the `_apxf` tunnel. If it lacks APX but has BMI2, it uses the `_bmi2` tunnel. If it has neither, it falls all the way down to `_default`.

## 4. Optimized Generation (The Code Generation)

Because standard functions or C files don't automatically become extended hardware code, Zstd explicitly tells the compiler using function attributes. The function `ZSTD_decompressSequences_apxf` is declared with the `APXF_TARGET_ATTRIBUTE` configured in `lib/common/compiler.h`:

```c
__attribute__((__target__("egpr,push2pop2,ppx,ndd,ccmp,nf,cf,zu,lzcnt,bmi,bmi2")))
ZSTD_decompressSequences_apxf(...) { 
    /* decompression logic */ 
}
```

When LLVM/GCC hits this block, it natively translates the inner C logic using zero-upper structures (`zu`), non-destructive destinations (`ndd`), extended GPRs (`egpr`), and bit manipulation tracking (`bmi2`).
