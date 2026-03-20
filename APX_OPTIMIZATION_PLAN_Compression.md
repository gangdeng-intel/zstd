# Intel APX Optimization Plan for Zstd Compression

## 1. Analysis

Based on performance profiling of standard `zstd` compression at level 3, the `ZSTD_compressBlock_doubleFast` code paths consume approximately 70% of CPU time. The actual hot loops are embedded within the deeply nested, `FORCE_INLINE_TEMPLATE` generic functions (specifically `ZSTD_compressBlock_doubleFast_noDict_generic`, `ZSTD_compressBlock_doubleFast_dictMatchState_generic`, and `ZSTD_compressBlock_doubleFast_extDict_generic`). 

These loops handle bitstream parsing, maintaining multiple hash tables, managing dictionary sequences, and handling block constraints simultaneously. Consequently, they suffer from extreme register pressure, leading to frequent stack spilling on standard `x86_64`.

By applying Intel Advanced Performance Extensions (APX) via compiler multiversioning, we can leverage:
1. **Double General Purpose Registers (eGPRs):** Access to `r16`-`r31` significantly minimizes stack spilling.
2. **Non-Destructive Instructions (NDD):** 3-operand instruction templates avoid extra moves/copies.
3. **PUSH2/POP2:** Halves the instruction footprint for register preservation sequences.

## 2. Dynamic Dispatch Approach (Plan B strategy)
Consistent with `zstd`'s methodology (and mirroring the strategy laid out in `APX_OPTIMIZATION_PLAN.md` for decompression), we will use **Native Dynamic Dispatch**, avoiding explicit linker-level `IFUNC`s. 

### Why Native Dispatch?
The `doubleFast` logic is instantiated via macros to generate variants for different matching lengths (`mls` 4, 5, 6, 7). Attempting to use IFUNCs dynamically via macros across a matrix of multiple static inline templates causes symbol visibility and scoping collisions in older compilers. Explicit manual dispatch inside the primary jump tables cleanly separates APX logic from the default instruction set.

## 3. Implementation Steps

1. **APX Feature Tracking (`ZSTD_CCtx`):**
   We need to ensure the compression context caches the APX CPU capability to avoid invoking the `cpuid` hardware check iteratively for every block.
   - Modify `lib/compress/zstd_compress_internal.h` and attach an `apxf` flag to `struct ZSTD_CCtx_s`.
   - The flag should be initialized via `ZSTD_cpuSupportsApxf()` around context creation (e.g. `ZSTD_CCtx_resetParameters` and `ZSTD_createCCtx_advanced`).

2. **APX Macro Generators in `zstd_double_fast.c`:**
   Add APX-targeted macro variants right below the existing `ZSTD_GEN_DFAST_FN` macros. We will mark these generated wrapper functions with `APXF_TARGET_ATTRIBUTE`. Because the inner generic function is a `FORCE_INLINE_TEMPLATE`, compilers (GCC/Clang) will inject APX instructions deeply into the materialized inline code.

   ```c
   #if defined(DYNAMIC_APXF)

   #define ZSTD_GEN_DFAST_FN_APX(dictMode, mls)                                                             \
       APXF_TARGET_ATTRIBUTE                                                                                \
       static size_t ZSTD_compressBlock_doubleFast_##dictMode##_##mls##_apxf(                               \
               ZSTD_matchState_t* ms, seqStore_t* seqStore, U32 rep[ZSTD_REP_NUM],                          \
               void const* src, size_t srcSize)                                                             \
       {                                                                                                    \
           return ZSTD_compressBlock_doubleFast_##dictMode##_generic(ms, seqStore, rep, src, srcSize, mls); \
       }

   ZSTD_GEN_DFAST_FN_APX(noDict, 4)
   // ... apply to 4, 5, 6, 7 for noDict, dictMatchState, and extDict 
   #endif
   ```

3. **Update the Dispatchers (`zstd_double_fast.c`):**
   Update the three public dispatcher routines to intercept control flow and redirect to the `_apxf` routines when the context indicates APX capability. 

   ```c
   size_t ZSTD_compressBlock_doubleFast(...) {
       const U32 mls = ms->cParams.minMatch;
   #if defined(DYNAMIC_APXF)
       if (ms->cctx && ms->cctx->apxf) {
           switch(mls) {
               // Route to _apxf paths
           }
       }
   #endif
       // Fallback to standard x86 paths
   }
   ```
   *(Apply identical logic to `ZSTD_compressBlock_doubleFast_dictMatchState` and `ZSTD_compressBlock_doubleFast_extDict`.)*

4. **APX-optimize row-hash search in `zstd_lazy.c`:**
    After profiling higher compression levels such as level 12, the row-based match finder hotspot `RowFindBestMatch_noDict_5_6` showed up as another strong APX candidate. That path is reached through the generic row-hash dispatcher rather than through a standalone top-level compression entry point, so the optimization point is different from `doubleFast`.

    - The core matching logic remains centralized in `ZSTD_RowFindBestMatch(...)`.
    - We added APX-targeted outlined wrappers for every generated row-search specialization:
      - `ZSTD_RowFindBestMatch_noDict_5_6_apxf`
      - and the other `(dictMode, mls, rowLog)` row-hash combinations generated by macro expansion.
    - These wrappers are marked with `APXF_TARGET_ATTRIBUTE`, allowing the compiler to emit APX instructions for the hottest row-hash search bodies while preserving the original generic implementation as the fallback.

    Concretely, the row-search macro generator now has an APX form conceptually equivalent to:

    ```c
    #define GEN_ZSTD_ROW_SEARCH_FN_APXF(dictMode, mls, rowLog)                                 \
         APXF_TARGET_ATTRIBUTE ZSTD_SEARCH_FN_ATTRS size_t                                      \
         ZSTD_RowFindBestMatch_##dictMode##_##mls##_##rowLog##_apxf(                            \
                    ZSTD_MatchState_t* ms, const BYTE* ip, const BYTE* const iLimit,               \
                    size_t* offsetPtr)                                                              \
         {                                                                                       \
              return ZSTD_RowFindBestMatch(ms, ip, iLimit, offsetPtr, mls, ZSTD_##dictMode, rowLog); \
         }
    ```

    We then updated `ZSTD_searchMax(...)` so that when all of the following are true:
    - the selected search method is `search_rowHash`, and
    - the compression match state says APX is available (`ms->apxf`),

    it dispatches to the APX-generated row-search specialization instead of the default one.

    This means the level-12 hotspot `RowFindBestMatch_noDict_5_6` is now optimized in the same style as the APX `doubleFast` path:
    - keep the original generic algorithm unchanged,
    - generate APX-decorated outlined entry points,
    - and select them through existing runtime dispatch state.

5. **APX capability propagation for compression state:**
    To support both `doubleFast` and row-hash match finder dispatch, APX capability is now cached once in `ZSTD_CCtx` and propagated into `ZSTD_MatchState_t` during compression context reset.
    - `ZSTD_initCCtx()` initializes `cctx->apxf` using `ZSTD_cpuSupportsApxf()`.
    - `ZSTD_resetCCtx_internal()` copies that result into `cctx->blockState.matchState.apxf`.
    - This avoids repeated CPUID checks in the hot compression loops.

6. **APX-optimize checksum updates via `XXH64_update()`:**
   Profiling also showed checksum maintenance as a meaningful recurring cost in the compression pipeline. Zstd updates its frame checksum through `XXH64_update()`, so this became another good candidate for the same APX multiversioning pattern.

   The implementation follows the same structure used elsewhere:

   - The original logic was factored into a generic helper body: `XXH64_update_body()`.
   - An APX-specialized wrapper `XXH64_update_apxf()` was added and marked with `APXF_TARGET_ATTRIBUTE`.
   - The public `XXH64_update()` entry point now dispatches between the default and APX versions.

   Conceptually:

   ```c
   static XXH_errorcode XXH64_update_body(XXH64_state_t* state, const void* input, size_t len) {
       /* original generic implementation */
   }

   static APXF_TARGET_ATTRIBUTE XXH_errorcode
   XXH64_update_apxf(XXH64_state_t* state, const void* input, size_t len) {
       return XXH64_update_body(state, input, len);
   }

   XXH_errorcode XXH64_update(XXH64_state_t* state, const void* input, size_t len) {
       if (XXH_cpuSupportsApxf()) {
           return XXH64_update_apxf(state, input, len);
       }
       return XXH64_update_body(state, input, len);
   }
   ```

   To avoid paying feature-detection overhead in the hottest zstd checksum paths, the compression and decompression call sites were also updated to use cached APX state when available:

   - compression uses `cctx->apxf` in `zstd_compress.c`
   - decompression uses `dctx->apxf` in `zstd_decompress.c`

   This keeps the generic `xxhash` API APX-aware while still letting zstd's internal hot loops bypass repeated runtime feature queries.

## 4. Current APX Compression Coverage

The compression-side APX work now covers two important hotspot families:

1. **`zstd_double_fast.c`**
    - APX wrappers for `ZSTD_compressBlock_doubleFast_*`
    - Runtime dispatch for `noDict`, `dictMatchState`, and `extDict`

2. **`zstd_lazy.c` row-hash search**
    - APX wrappers for generated `ZSTD_RowFindBestMatch_*_*_*` specializations
    - Runtime dispatch inside `ZSTD_searchMax()` for `search_rowHash`
    - Includes the profiled hotspot `RowFindBestMatch_noDict_5_6`

3. **Checksum update path through `xxhash`**
    - APX wrapper for `XXH64_update()`
    - Generic-body plus APX-wrapper structure for multiversioning
    - Cached APX-based direct dispatch from zstd checksum hot paths
