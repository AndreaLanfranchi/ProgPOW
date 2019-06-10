# ProgPoW - A Programmatic Proof of Work

[![Join the chat at https://gitter.im/ifdefelse/community](https://badges.gitter.im/ifdefelse/community.svg)](https://gitter.im/ifdefelse/community?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)[![Follow us](https://img.shields.io/twitter/follow/IfDefElse_.svg)](https://twitter.com/IfDefElse_)

ProgPoW is a proof-of-work algorithm designed to close the efficiency gap available to specialized ASICs. It utilizes almost all parts of commodity hardware (GPUs), and comes pre-tuned for the most common hardware utilized in the Ethereum network.

Ever since the first bitcoin mining ASIC was released, many new Proof of Work algorithms have been created with the intention of being “ASIC-resistant”. The goal of “ASIC-resistance” is to resist the centralization of PoW mining power such that these coins couldn’t be so easily manipulated by a few players. 

This document presents an overview of the algorithm and examines what it means to be “ASIC-resistant.” Next, we compare existing PoW designs by analyzing how each algorithm executes in hardware. Finally, we present the detailed implementation by walking through the code.

## ProgPoW Overview
The design goal of ProgPoW is to have the algorithm’s requirements match what is available on commodity GPUs:  If the algorithm were to be implemented on a custom ASIC there should be little opportunity for efficiency gains compared to a commodity GPU.

The main elements of the algorithm are:
* Changes keccak_f1600 (with 64-bit words) to keccak_f800 (with 32-bit words) to reduce impact on total power
* Increases mix state.
* Adds a random sequence of math in the main loop.
* Adds reads from a small, low-latency cache that supports random addresses.
* Increases the DRAM read from 128 bytes to 256 bytes.

The random sequence changes every `PROGPOW_PERIOD` (50 blocks or about 12.5 minutes).  When mining source code is generated for the random sequence and compiled on the host CPU.  The GPU will execute the compiled code where what math to perform and what mix state to use are already resolved.

While a custom ASIC to implement this algorithm is still possible, the efficiency gains available are minimal.  The majority of a commodity GPU is required to support the above elements. The only optimizations available are:
* Remove the graphics pipeline (displays, geometry engines, texturing, etc)
* Remove floating point math
* A few ISA tweaks, like instructions that exactly match the merge() function

These would result in minimal, roughly 1.1-1.2x, efficiency gains.  This is much less than the 2x for Ethash or 50x for Cryptonight.

## Rationale for PoW on Commodity Hardware
With the growth of large mining pools, the control of hashing power has been delegated to the top few pools to provide a steadier economic return for small miners. While some have made the argument that large centralized pools defeats the purpose of “ASIC resistance,” it’s important to note that ASIC based coins are even more centralized for several reasons.

1. No natural distribution: There isn’t an economic purpose for ultra-specialized hardware outside of mining and thus no reason for most people to have it. 
2. No reserve group: Thus, there’s no reserve pool of hardware or reserve pool of interested parties to jump in when coin price is volatile and attractive for manipulation. 
3. High barrier to entry: Initial miners are those rich enough to invest capital and ecological resources on the unknown experiment a new coin may be. Thus, initial coin distribution through mining will be very limited causing centralized economic bias. 
4. Delegated centralization vs implementation centralization: While pool centralization is delegated, hardware monoculture is not: only the limiter buyers of this hardware can participate so there isn’t even the possibility of divesting control on short notice.
5. No obvious decentralization of control even with decentralized mining: Once large custom ASIC makers get into the game, designing back-doored hardware is trivial. ASIC makers have no incentive to be transparent or fair in market participation.

While the goal of “ASIC resistance” is valuable, the entire concept of “ASIC resistance” is a bit of a fallacy.  CPUs and GPUs are themselves ASICs.  Any algorithm that can run on a commodity ASIC (a CPU or GPU) by definition can have a customized ASIC created for it with slightly less functionality. Some algorithms are intentionally made to be  “ASIC friendly” - where an ASIC implementation is drastically more efficient than the same algorithm running on general purpose hardware. The protection that this offers when the coin is unknown also makes it an attractive target for a dedicate mining ASIC company as soon as it becomes useful.

Therefore, ASIC resistance is: the efficiency difference of specialized hardware versus hardware that has a wider adoption and applicability.  A smaller efficiency difference between custom vs general hardware mean higher resistance and a better algorithm. This efficiency difference is the proper metric to use when comparing the quality of PoW algorithms.  Efficiency could mean absolute performance, performance per watt, or performance per dollar - they are all highly correlated.  If a single entity creates and controls an ASIC that is drastically more efficient, they can gain 51% of the network hashrate and possibly stage an attack.

## Review of Existing PoW Algorithms

### SHA256
* Potential ASIC efficiency gain ~ 1000X

The SHA algorithm is a sequence of simple math operations - additions, logical ops, and rotates.

To process a single op on a CPU or GPU requires fetching and decoding an instruction, reading data from a register file, executing the instruction, and then writing the result back to a register file.  This takes significant time and power.

A single op implemented in an ASIC takes a handful of transistors and wires.  This means every individual op takes negligible power, area, or time.  A hashing core is built by laying out the sequence of required ops.

The hashing core can execute the required sequence of ops in much less time, and using less power or area, than doing the same sequence on a CPU or GPU.  A bitcoin ASIC consists of a number of identical hashing cores and some minimal off-chip communication.

### Scrypt and NeoScrypt
* Potential ASIC efficiency gain ~ 1000X

Scrypt and NeoScrypt are similar to SHA in the arithmetic and bitwise operations used. Unfortunately, popular coins such as Litecoin only use a scratchpad size between 32kb and 128kb for their PoW mining algorithm. This scratch pad is small enough to trivially fit on an ASIC next to the math core. The implementation of the math core would be very similar to SHA, with similar efficiency gains.

### X11 and X16R
* Potential ASIC efficiency gain ~ 1000X

X11 (and similar X##) require an ASIC that has 11 unique hashing cores pipelined in a fixed sequence.  Each individual hashing core would have similar efficiency to an individual SHA core, so the overall design will have the same efficiency gains.

X16R requires the multiple hashing cores to interact through a simple sequencing state machine. Each individual core will have similar efficiency gains and the sequencing logic will take minimal power, area, or time.

The Baikal BK-X is an existing ASIC with multiple hashing cores and a programmable sequencer.  It has been upgraded to enable new algorithms that sequence the hashes in different orders.

### Equihash
* Potential ASIC efficiency gain ~ 100X

The ~150mb of state is large but possible on an ASIC. The binning, sorting, and comparing of bit strings could be implemented on an ASIC at extremely high speed.

### Cuckoo Cycle
* Potential ASIC efficiency gain ~ 100X

The amount of state required on-chip is not clear as there are Time/Memory Tradeoff attacks. A specialized graph traversal core would have similar efficiency gains to a SHA compute core.

### CryptoNight
* Potential ASIC efficiency gain ~ 50X

Compared to Scrypt, CryptoNight does much less compute and requires a full 2mb of scratch pad (there is no known Time/Memory Tradeoff attack).  The large scratch pad will dominate the ASIC implementation and limit the number of hashing cores, limiting the absolute performance of the ASIC.  An ASIC will consist almost entirely of just on-die SRAM.

### Ethash
* Potential ASIC efficiency gain ~ 2X

Ethash requires external memory due to the large size of the DAG.  However that is all that it requires - there is minimal compute that is done on the result loaded from memory.  As a result a custom ASIC could remove most of the complexity, and power, of a GPU and be just a memory interface connected to a small compute engine.

## ProgPoW Algorithm Walkthrough

The DAG is generated exactly as in Ethash.  All the parameters (epoch length, DAG size, etc) are unchanged.  See the original [Ethash](https://github.com/ethereum/wiki/wiki/Ethash) spec for details on generating the DAG.

ProgPoW can be tuned using the following parameters.  The proposed settings have been tuned for a range of existing, commodity GPUs:

* `PROGPOW_PERIOD`: Number of blocks before changing the random program
* `PROGPOW_LANES`: The number of parallel lanes that coordinate to calculate a single hash instance
* `PROGPOW_REGS`: The register file usage size
* `PROGPOW_DAG_LOADS`: Number of uint32 loads from the DAG per lane
* `PROGPOW_CACHE_BYTES`: The size of the cache
* `PROGPOW_CNT_DAG`: The number of DAG accesses, defined as the outer loop of the algorithm (64 is the same as Ethash)
* `PROGPOW_CNT_CACHE`: The number of cache accesses per loop
* `PROGPOW_CNT_MATH`: The number of math operations per loop

The value of these parameters has been tweaked between version 0.9.2 (live on the Gangnam testnet) and 0.9.3 (proposed for Ethereum adoption).  See [this medium post](https://medium.com/@ifdefelse/progpow-progress-da5bb31a651b) for details.

| Parameter             | 0.9.2 | 0.9.3 |
|-----------------------|-------|-------|
| `PROGPOW_PERIOD`      | `50`  | `10`  |
| `PROGPOW_LANES`       | `16`  | `16`  |
| `PROGPOW_REGS`        | `32`  | `32`  |
| `PROGPOW_DAG_LOADS`   | `4`   | `4`   |
| `PROGPOW_CACHE_BYTES` | `16x1024` | `16x1024` |
| `PROGPOW_CNT_DAG`     | `64`  | `64`  |
| `PROGPOW_CNT_CACHE`   | `12`  | `11`  |
| `PROGPOW_CNT_MATH`    | `20`  | `18`  |


The random program changes every `PROGPOW_PERIOD` blocks (default `50`, roughly 12.5 minutes) to ensure the hardware executing the algorithm is fully programmable.  If the program only changed every DAG epoch (roughly 5 days) certain miners could have time to develop hand-optimized versions of the random sequence, giving them an undue advantage.

Sample code is written in C++, this should be kept in mind when evaluating the code in the specification.

All numerics are computed using unsigned 32 bit integers.  Any overflows are trimmed off before proceeding to the next computation.  Languages that use numerics not fixed to bit lengths (such as Python and JavaScript) or that only use signed integers (such as Java) will need to keep their languages' quirks in mind.  The extensive use of 32 bit data values aligns with modern GPUs internal data architectures.

ProgPoW uses a 32-bit variant of **FNV1a** for merging data. The existing Ethash uses a similar variant of FNV1 for merging, but FNV1a provides better distribution properties.

Test vectors can be found [in the test vectors file](test-vectors.md#fnv1a).

```cpp
const uint32_t FNV_PRIME = 0x1000193;
const uint32_t FNV_OFFSET_BASIS = 0x811c9dc5;

uint32_t fnv1a(uint32_t h, uint32_t d)
{
    return (h ^ d) * FNV_PRIME;
}
```

ProgPow uses [KISS99](https://en.wikipedia.org/wiki/KISS_(algorithm)) for random number generation. This is the simplest (fewest instruction) random generator that passes the TestU01 statistical test suite.  A more complex random number generator like Mersenne Twister can be efficiently implemented on a specialized ASIC, providing an opportunity for efficiency gains.

Test vectors can be found [in the test vectors file](test-vectors.md#kiss99).

```cpp
typedef struct {
    uint32_t z, w, jsr, jcong;
} kiss99_t;

// KISS99 is simple, fast, and passes the TestU01 suite
// https://en.wikipedia.org/wiki/KISS_(algorithm)
// http://www.cse.yorku.ca/~oz/marsaglia-rng.html
uint32_t kiss99(kiss99_t &st)
{
    st.z = 36969 * (st.z & 65535) + (st.z >> 16);
    st.w = 18000 * (st.w & 65535) + (st.w >> 16);
    uint32_t MWC = ((st.z << 16) + st.w);
    st.jsr ^= (st.jsr << 17);
    st.jsr ^= (st.jsr >> 13);
    st.jsr ^= (st.jsr << 5);
    st.jcong = 69069 * st.jcong + 1234567;
    return ((MWC^st.jcong) + st.jsr);
}
```

The `fill_mix` function populates an array of `int32` values used by each lane in the hash calculations.

Test vectors can be found [in the test vectors file](test-vectors.md#fill_mix).

```cpp
void fill_mix(
    uint64_t seed,
    uint32_t lane_id,
    uint32_t mix[PROGPOW_REGS]
)
{
    // Use FNV to expand the per-warp seed to per-lane
    // Use KISS to expand the per-lane seed to fill mix
    kiss99_t st;
    st.z = fnv1a(FNV_OFFSET_BASIS, seed);
    st.w = fnv1a(st.z, seed >> 32);
    st.jsr = fnv1a(st.w, lane_id);
    st.jcong = fnv1a(st.jsr, lane_id);
    for (int i = 0; i < PROGPOW_REGS; i++)
            mix[i] = kiss99(st);
}
```

Like Ethash Keccak is used to seed the sequence per-nonce and to produce the final result.  The keccak-f800 variant is used as the 32-bit word size matches the native word size of modern GPUs.  The implementation is a variant of SHAKE with width=800, bitrate=576, capacity=224, output=256, and no padding.  The result of keccak is treated as a 256-bit big-endian number - that is result byte 0 is the MSB of the value.

As with Ethash the input and output of the keccak function are fixed and relatively small.  This means only a single "absorb" and "squeeze" phase are required.  For a pseudo-code implementation of the `keccak_f800_round` function see the `Round[b](A,RC)` function in the "Pseudo-code description of the permutations" section of the [official Keccak specs](https://keccak.team/keccak_specs_summary.html).

Test vectors can be found [in the test vectors file](test-vectors.md#keccak_f800_progpow).

```cpp
hash32_t keccak_f800_progpow(hash32_t header, uint64_t seed, hash32_t digest)
{
    uint32_t st[25];

    // Initialization
    for (int i = 0; i < 25; i++)
        st[i] = 0;

    // Absorb phase for fixed 18 words of input
    for (int i = 0; i < 8; i++)
        st[i] = header.uint32s[i];
    st[8] = seed;
    st[9] = seed >> 32;
    for (int i = 0; i < 8; i++)
        st[10+i] = digest.uint32s[i];

    // keccak_f800 call for the single absorb pass
    for (int r = 0; r < 22; r++)
        keccak_f800_round(st, r);

    // Squeeze phase for fixed 8 words of output
    hash32_t ret;
    for (int i=0; i<8; i++)
        ret.uint32s[i] = st[i];

    return ret;
}
```

The inner loop uses FNV and KISS99 to generate a random sequence from the `prog_seed`.  This random sequence determines which mix state is accessed and what random math is performed.

Since the `prog_seed` changes only once per `PROGPOW_PERIOD` (50 blocks or about 12.5 minutes) it is expected that while mining `progPowLoop` will be evaluated on the CPU to generate source code for that period's sequence.  The source code will be compiled on the CPU before running on the GPU.  You can see an example sequence and generated source code in [kernel.cu](test/kernel.cu).

Test vectors can be found [in the test vectors file](test-vectors.md#progPowInit).

```cpp
kiss99_t progPowInit(uint64_t prog_seed, int mix_seq_dst[PROGPOW_REGS], int mix_seq_src[PROGPOW_REGS])
{
    kiss99_t prog_rnd;
    prog_rnd.z = fnv1a(FNV_OFFSET_BASIS, prog_seed);
    prog_rnd.w = fnv1a(prog_rnd.z, prog_seed >> 32);
    prog_rnd.jsr = fnv1a(prog_rnd.w, prog_seed);
    prog_rnd.jcong = fnv1a(prog_rnd.jsr, prog_seed >> 32);
    // Create a random sequence of mix destinations for merge() and mix sources for cache reads
    // guarantees every destination merged once
    // guarantees no duplicate cache reads, which could be optimized away
    // Uses Fisher-Yates shuffle
    for (int i = 0; i < PROGPOW_REGS; i++)
    {
        mix_seq_dst[i] = i;
        mix_seq_src[i] = i;
    }
    for (int i = PROGPOW_REGS - 1; i > 0; i--)
    {
        int j;
        j = kiss99(prog_rnd) % (i + 1);
        swap(mix_seq_dst[i], mix_seq_dst[j]);
        j = kiss99(prog_rnd) % (i + 1);
        swap(mix_seq_src[i], mix_seq_src[j]);
    }
    return prog_rnd;
}
```

The math operations that merges values into the mix data are ones chosen to maintain entropy.

Test vectors can be found [in the test vectors file](test-vectors.md#math).

```cpp
// Merge new data from b into the value in a
// Assuming A has high entropy only do ops that retain entropy
// even if B is low entropy
// (IE don't do A&B)
uint32_t merge(uint32_t a, uint32_t b, uint32_t r)
{
    switch (r % 4)
    {
    case 0: return (a * 33) + b;
    case 1: return (a ^ b) * 33;
    // prevent rotate by 0 which is a NOP
    case 2: return ROTL32(a, ((r >> 16) % 31) + 1) ^ b;
    case 3: return ROTR32(a, ((r >> 16) % 31) + 1) ^ b;
    }
}
```

The math operations chosen for the random math are ones that are easy to implement in CUDA and OpenCL, the two main programming languages for commodity GPUs. The [mul_hi](https://www.khronos.org/registry/OpenCL/sdk/1.1/docs/man/xhtml/mul_hi.html), [min](https://www.khronos.org/registry/OpenCL/sdk/2.0/docs/man/xhtml/integerMax.html), [clz](https://www.khronos.org/registry/OpenCL/sdk/1.1/docs/man/xhtml/clz.html), and [popcount](https://www.khronos.org/registry/OpenCL/sdk/2.0/docs/man/xhtml/popcount.html) functions match the corresponding OpenCL functions.  ROTL32 matches the OpenCL [rotate](https://www.khronos.org/registry/OpenCL/sdk/1.0/docs/man/xhtml/rotate.html) function.  ROTR32 is rotate right, which is equivalent to `rotate(i, 32-v)`.

Test vectors can be found [in the test vectors file](test-vectors.md#math).

```cpp
// Random math between two input values
uint32_t math(uint32_t a, uint32_t b, uint32_t r)
{
    switch (r % 11)
    {
    case 0: return a + b;
    case 1: return a * b;
    case 2: return mul_hi(a, b);
    case 3: return min(a, b);
    case 4: return ROTL32(a, b);
    case 5: return ROTR32(a, b);
    case 6: return a & b;
    case 7: return a | b;
    case 8: return a ^ b;
    case 9: return clz(a) + clz(b);
    case 10: return popcount(a) + popcount(b);
    }
}
```


The flow of the inner loop is:
* Lane `(loop % LANES)` is chosen as the leader for that loop iteration
* The leader's `mix[0]` value modulo the number of 256-byte DAG entries is is used to select where to read from the full DAG
* Each lane reads `DAG_LOADS` sequential words, using `(lane ^ loop) % LANES` as the starting offset within the entry.
* The random sequence of math and cache accesses is performed
* The DAG data read at the start of the loop is merged at the end of the loop

`prog_seed` and `loop` come from the outer loop, corresponding to the current program seed (which is block_number/PROGPOW_PERIOD) and the loop iteration number.  `mix` is the state array, initially filled by `fill_mix`. `dag` is the bytes of the Ethash DAG grouped into 32 bit unsigned ints in litte-endian format.  On little-endian architectures this is just a normal int32 pointer to the existing DAG.

`DAG_BYTES` is set to the number of bytes in the current DAG, which is generated identically to the existing Ethash algorithm.  

Test vectors can be found [in the test vectors file](test-vectors.md#progPowLoop).

```cpp
void progPowLoop(
    const uint64_t prog_seed,
    const uint32_t loop,
    uint32_t mix[PROGPOW_LANES][PROGPOW_REGS],
    const uint32_t *dag)
{
    // dag_entry holds the 256 bytes of data loaded from the DAG
    uint32_t dag_entry[PROGPOW_LANES][PROGPOW_DAG_LOADS];
    // On each loop iteration rotate which lane is the source of the DAG address.
    // The source lane's mix[0] value is used to ensure the last loop's DAG data feeds into this loop's address.
    // dag_addr_base is which 256-byte entry within the DAG will be accessed
    uint32_t dag_addr_base = mix[loop%PROGPOW_LANES][0] %
        (DAG_BYTES / (PROGPOW_LANES*PROGPOW_DAG_LOADS*sizeof(uint32_t)));
    for (int l = 0; l < PROGPOW_LANES; l++)
    {
        // Lanes access DAG_LOADS sequential words from the dag entry
        // Shuffle which portion of the entry each lane accesses each iteration by XORing lane and loop.
        // This prevents multi-chip ASICs from each storing just a portion of the DAG
        size_t dag_addr_lane = dag_addr_base * PROGPOW_LANES + (l ^ loop) % PROGPOW_LANES;
        for (int i = 0; i < PROGPOW_DAG_LOADS; i++)
            dag_entry[l][i] = dag[dag_addr_lane * PROGPOW_DAG_LOADS + i];
    }

    // Initialize the program seed and sequences
    // When mining these are evaluated on the CPU and compiled away
    int mix_seq_dst[PROGPOW_REGS];
    int mix_seq_src[PROGPOW_REGS];
    int mix_seq_dst_cnt = 0;
    int mix_seq_src_cnt = 0;
    kiss99_t prog_rnd = progPowInit(prog_seed, mix_seq_dst, mix_seq_src);

    int max_i = max(PROGPOW_CNT_CACHE, PROGPOW_CNT_MATH);
    for (int i = 0; i < max_i; i++)
    {
        if (i < PROGPOW_CNT_CACHE)
        {
            // Cached memory access
            // lanes access random 32-bit locations within the first portion of the DAG
            int src = mix_seq_src[(mix_seq_src_cnt++)%PROGPOW_REGS];
            int dst = mix_seq_dst[(mix_seq_dst_cnt++)%PROGPOW_REGS];
            int sel = kiss99(prog_rnd);
            for (int l = 0; l < PROGPOW_LANES; l++)
            {
                uint32_t offset = mix[l][src] % (PROGPOW_CACHE_BYTES/sizeof(uint32_t));
                mix[l][dst] = merge(mix[l][dst], dag[offset], sel);
            }
        }
        if (i < PROGPOW_CNT_MATH)
        {
            // Random Math
            // Generate 2 unique sources 
            int src_rnd = kiss99(prog_rnd) % (PROGPOW_REGS * (PROGPOW_REGS-1));
            int src1 = src_rnd % PROGPOW_REGS; // 0 <= src1 < PROGPOW_REGS
            int src2 = src_rnd / PROGPOW_REGS; // 0 <= src2 < PROGPOW_REGS - 1
            if (src2 >= src1) ++src2; // src2 is now any reg other than src1
            int sel1 = kiss99(prog_rnd);
            int dst  = mix_seq_dst[(mix_seq_dst_cnt++)%PROGPOW_REGS];
            int sel2 = kiss99(prog_rnd);
            for (int l = 0; l < PROGPOW_LANES; l++)
            {
                uint32_t data = math(mix[l][src1], mix[l][src2], sel1);
                mix[l][dst] = merge(mix[l][dst], data, sel2);
            }
        }
    }
    // Consume the global load data at the very end of the loop to allow full latency hiding
    // Always merge into mix[0] to feed the offset calculation
    for (int i = 0; i < PROGPOW_DAG_LOADS; i++)
    {
        int dst = (i==0) ? 0 : mix_seq_dst[(mix_seq_dst_cnt++)%PROGPOW_REGS];
        int sel = kiss99(prog_rnd);
        for (int l = 0; l < PROGPOW_LANES; l++)
            mix[l][dst] = merge(mix[l][dst], dag_entry[l][i], sel);
    }
}
```

The flow of the overall algorithm is:
* A keccak hash of the header + nonce to create a seed
* Use the seed to generate initial mix data
* Loop multiple times, each time hashing random loads and random math into the mix data
* Hash all the mix data into a single 256-bit value
* A final keccak hash is computed
* When mining this final value is compared against a `hash32_t` target

```cpp
hash32_t progPowHash(
    const uint64_t prog_seed, // value is (block_number/PROGPOW_PERIOD)
    const uint64_t nonce,
    const hash32_t header,
    const uint32_t *dag // gigabyte DAG located in framebuffer - the first portion gets cached
)
{
    uint32_t mix[PROGPOW_LANES][PROGPOW_REGS];
    hash32_t digest;
    for (int i = 0; i < 8; i++)
        digest.uint32s[i] = 0;

    // keccak(header..nonce)
    hash32_t seed_256 = keccak_f800_progpow(header, nonce, digest);
    // endian swap so byte 0 of the hash is the MSB of the value
    uint64_t seed = ((uint64_t)bswap(seed_256.uint32s[0]) << 32) | bswap(seed_256.uint32s[1]);

    // initialize mix for all lanes
    for (int l = 0; l < PROGPOW_LANES; l++)
        fill_mix(seed, l, mix[l]);

    // execute the randomly generated inner loop
    for (int i = 0; i < PROGPOW_CNT_DAG; i++)
        progPowLoop(prog_seed, i, mix, dag);

    // Reduce mix data to a per-lane 32-bit digest
    uint32_t digest_lane[PROGPOW_LANES];
    for (int l = 0; l < PROGPOW_LANES; l++)
    {
        digest_lane[l] = FNV_OFFSET_BASIS;
        for (int i = 0; i < PROGPOW_REGS; i++)
            digest_lane[l] = fnv1a(digest_lane[l], mix[l][i]);
    }
    // Reduce all lanes to a single 256-bit digest
    for (int i = 0; i < 8; i++)
        digest.uint32s[i] = FNV_OFFSET_BASIS;
    for (int l = 0; l < PROGPOW_LANES; l++)
        digest.uint32s[l%8] = fnv1a(digest.uint32s[l%8], digest_lane[l]);

    // keccak(header .. keccak(header..nonce) .. digest);
    return keccak_f800_progpow(header, seed, digest);
}
```


## Example / Testcase

The random sequence generated for block 30,000 (prog_seed 600) can been seen in [kernel.cu](test/kernel.cu).

The algorithm run on block 30,000 produces the following digest and result:
```
header ffeeddccbbaa9988776655443322110000112233445566778899aabbccddeeff
nonce 123456789abcdef0

digest: 11f19805c58ab46610ff9c719dcf0a5f18fa2f1605798eef770c47219274767d
result: 5b7ccd472dbefdd95b895cac8ece67ff0deb5a6bd2ecc6e162383d00c3728ece
```

A full run showing some intermediate values can be seen in [result.log](test/result.log)

Additional test vectors can be found [in the test vectors file](test-vectors.md#progPowHash).



## Change History

- 0.9.3 (proposed) - Reduce parameters PERIOD, CNT_MATH, and CNT_CACHE. See [this medium post](https://medium.com/@ifdefelse/progpow-progress-da5bb31a651b) for details.
- [0.9.2](https://github.com/ifdefelse/ProgPOW/blob/0e39b62deb0c9ab14900fc404fcb19cac70240e1/README.md) - Unique sources for math() and prevent rotation by 0 in merge().  Suggested by [SChernykh](https://github.com/ifdefelse/ProgPOW/issues/19)
- [0.9.1](https://github.com/ifdefelse/ProgPOW/blob/60bba1c3fdad6a54539fc3e9f05727547de9c58c/README.md) - Shuffle what part of the DAG entry each lane accesses. Suggested by [mbevand](https://github.com/ifdefelse/ProgPOW/pull/13)
- [0.9.0](https://github.com/ifdefelse/ProgPOW/blob/a3f62349a1513f0393524683f9671cfe17cca895/README.md) - Unique cache address sources, re-tune parameters
- [0.8.0](https://github.com/ifdefelse/ProgPOW/blob/620b4c7aafe60391f863372814d7517e94386379/README.md) - Original spec

## License

The ProgPoW algorithm and this specification are a new work.  Copyright and related rights are waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

The reference ProgPoW mining implementation in this repository is a derivative of ethminer so retains the GPL license.
