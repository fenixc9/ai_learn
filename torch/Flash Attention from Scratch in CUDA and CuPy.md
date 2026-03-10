@mohitwt_ (mohit):
📰 Flash Attention from Scratch in CUDA and CuPy

what attention actually is:

attention takes three matrices, queries, keys, and values, and computes a weighted blend of the value vectors where the weights come from how similar each query is to each key. the formula is softmax(Q @ K.T / sqrt(d)) @ V. the problem shows up when you think about what that NxN score matrix actually costs you in memory.

for a sequence length of 4096, that score matrix is 4096 by 4096 floats. at 4 bytes each that is 64 megabytes for a single layer, single head. multiply by batch size, number of heads, number of layers and you are looking at gigabytes just to store intermediate attention scores. and you have to write it to GPU memory, read it back for softmax, write it again, read it again for the V multiply. four full passes over a massive matrix.

why memory bandwidth is the real bottleneck:

GPUs are not slow at math. they are slow at moving data between fast on-chip memory (SRAM) and slow global memory (HBM). SRAM is tiny, maybe 48KB per streaming multiprocessor, but extremely fast. HBM is large but the bandwidth is the ceiling. most attention implementations spend more time moving the score matrix back and forth than actually computing anything.

this is the core insight behind Flash Attention. if you never write the NxN score matrix to HBM at all, you eliminate the bottleneck entirely. the question is how you compute softmax without materializing the full matrix first, because standard softmax needs to see every score in a row before it can normalize them.

online softmax:

it lets you compute exact softmax incrementally, one chunk at a time, without ever storing the full row. you maintain two running values per row, a running maximum m and a running sum l. when you see a new chunk of scores you update the maximum, then apply a correction factor to everything you computed before.

the correction factor is exp(m_old - m_new). when you find a new maximum, all your previous exp values were computed with the wrong max subtracted, so you rescale them. the running sum gets the same correction. after processing all chunks, dividing by the final l gives you the exact same result as computing softmax over the full row at once. mathematically identical, no full row needed.

tiling and the forward kernel:

Flash Attention applies online softmax to tiled matrix computation. the outer loop iterates over tiles of Q. the inner loop iterates over all K and V tiles for each Q tile. for each combination you compute a small score tile that fits entirely in SRAM, update the running max and sum, apply the correction to the accumulated output, and move to the next tile. the NxN matrix never exists anywhere. only the current tiles live in SRAM at any moment.

i implemented this as a CuPy RawKernel. the kernel is written in CUDA C and compiled at runtime. each thread block handles one row of the output. threads collaborate to load K and V tiles into shared memory, then each thread accumulates its own output dimension independently. the score tile lives in registers, the intermediate values never touch HBM.

the naive implementation first:

before writing any CUDA i built a pure CuPy naive implementation. this is the standard attention formula with no tricks, just matrix multiplies and softmax using CuPy ops. it uses cuBLAS under the hood which is highly optimized assembly. this gives you a correct reference to verify against and a baseline to benchmark.

the naive softmax is three lines. subtract the row max for numerical stability, take exp, divide by the row sum. subtracting the max before exp prevents overflow since exp of large numbers becomes infinity in float32. this numerically stable version gives identical results to the unstable version when values are small, and correct results when they are not.

forward pass results:

the kernel is mathematically correct, verified by comparing output against naive CuPy attention with max diff under 1e-6. performance against naive CuPy and PyTorch is slower at the sequence lengths tested. naive CuPy uses cuBLAS, PyTorch uses cuDNN, both are hand-tuned assembly. closing that gap requires warp-level reductions and shared memory score caching, which is the hard problem that the FlashAttention 2 paper solves.

the memory advantage is real regardless of the timing. the NxN score matrix is never written to global memory. this is what makes Flash Attention practical for long sequences where the naive approach runs out of memory entirely.

the backward pass:

training requires gradients. the backward pass computes dQ, dK, and dV, how the loss changes with respect to each input, so the optimizer can update the weights. this is chain rule applied backwards through every operation in the forward pass.

standard attention stores the full NxN softmax weight matrix P during forward so it can use it during backward. for long sequences this is another O(N²) memory cost on top of the forward pass. Flash Attention throws P away and recomputes it during backward using the original Q and K along with the scalars m and l that were saved from forward. two vectors of length N instead of an NxN matrix. the recomputation costs a bit of extra compute but the memory saving is enormous.

the backward math:

gradient through O = P @ V:

```python
dV = P.T @ dO
dP = dO @ V.T
```

dV tells you how each value vector contributed to the output. dP tells you how the loss flows back through the attention weights before softmax.

the softmax backward is the subtle step. softmax outputs are coupled, changing one score affects all probabilities in the row because they sum to 1. the gradient formula is:

```python
dS = P * (dP - rowsum(dP * P))
```

the rowsum term corrects for this coupling. without it the gradients are wrong in a way that is hard to notice because the output looks plausible. after dS you apply the scale:

```python
dS = dS / sqrt(d)
```

then Q and K gradients:

```python
dQ = dS @ K
dK = dS.T @ Q
```

verifying gradients against PyTorch:

i verified the backward pass by running the same inputs through both my implementation and PyTorch autograd and comparing dQ, dK, and dV. PyTorch computes its own gradients internally through its autograd engine, it has no knowledge of my backward code. matching to 1e-6 on all three gradients means the math is correct.

this is the right way to verify a custom backward implementation. you trust PyTorch as a reference, run identical inputs through both, and check the diff. if it passes, your chain rule is right. if it fails, you know exactly which gradient is wrong and can trace back to which operation has the bug.

the full implementation is on GitHub, custom CUDA RawKernel forward, CuPy backward, online softmax, tiling, recomputation trick, verified against PyTorch autograd to 1e-6 on dQ, dK, and dV.

if you are using transformers and have never looked at what happens below the abstraction, this is worth understanding once. the memory difference between naive and flash is not a micro optimization, it is what makes long sequence training possible at all.
📅 Sat Mar 07 06:34:32 +0000 2026
🔗 https://x.com/mohitwt_/status/2030170215439552860
❤️ 984  🔁 110  💬 7