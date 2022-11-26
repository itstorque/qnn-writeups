---
title: "Resources for writing GPU kernels"
date: "2022-09-15"
categories: 
  - "general"
author_name: 
  - "Torque (Tareq) Dandachi"
author_email: 
  - "tareq@mit.edu"
---

These are a bunch of resources I found helpful when I first started writing custom GPU kernels in Julia. I sorted them out based on category and put them here for posterity. There are a lot of them and they go in differing amounts of depth with a lot of repetition between them. I think the first two resources are a good starting point for anyone who wants to implement any custom simple GPU kernel. Then they can explore the other section when performance and allocation start to matter.

#### Best 2 resources in my opinion

- [https://github.com/maleadt/juliacon21-gpu\_workshop/blob/main/deep\_dive/CUDA.ipynb](https://github.com/maleadt/juliacon21-gpu_workshop/blob/main/deep_dive/CUDA.ipynb)
- [https://developer.download.nvidia.com/assets/cuda/files/reduction.pdf](https://developer.download.nvidia.com/assets/cuda/files/reduction.pdf)

#### In-depth tutorial for basics of CUDA.jl (juliacon 2021)

- [https://www.youtube.com/watch?v=Hz9IMJuW5hU](https://www.youtube.com/watch?v=Hz9IMJuW5hU)

#### CUDA.jl

- (Notebook working thru optimiazations) [https://github.com/maleadt/juliacon21-gpu\_workshop/blob/main/deep\_dive/CUDA.ipynb](https://github.com/maleadt/juliacon21-gpu_workshop/blob/main/deep_dive/CUDA.ipynb)
- (CuArrays) [https://cuda.juliagpu.org/stable/usage/array/](https://cuda.juliagpu.org/stable/usage/array/)
- [http://www.cs.unb.ca/~aubanel/JuliaGPUNotesCUDA.html](http://www.cs.unb.ca/~aubanel/JuliaGPUNotesCUDA.html)
- [https://cuda.juliagpu.org/stable/usage/workflow/](https://cuda.juliagpu.org/stable/usage/workflow/)
- (mention of CUDA.scan here) [https://discourse.julialang.org/t/parallel-prefix-sum-with-cuda-jl/62316](https://discourse.julialang.org/t/parallel-prefix-sum-with-cuda-jl/62316)

#### Allocation and Shared Memory

- [https://developer.nvidia.com/blog/using-shared-memory-cuda-cc/](https://developer.nvidia.com/blog/using-shared-memory-cuda-cc/)
- [https://discourse.julialang.org/t/initializing-custaticsharedmem-array/10861](https://discourse.julialang.org/t/initializing-custaticsharedmem-array/10861)
- [https://github.com/JuliaGPU/CUDA.jl/issues/555](https://github.com/JuliaGPU/CUDA.jl/issues/555)

#### Occupancy (how to allocate cores)

- [https://juliagpu.org/post/2020-07-07-cuda\_1.1/index.html](https://juliagpu.org/post/2020-07-07-cuda_1.1/index.html)
- [https://discourse.julialang.org/t/the-most-general-way-to-estimate-the-optimal-arguments-for-cuda-macro/39342](https://discourse.julialang.org/t/the-most-general-way-to-estimate-the-optimal-arguments-for-cuda-macro/39342)

#### Misc

- GPUArrays.neutral\_element: [https://discourse.julialang.org/t/what-is-a-neutral-element-for-accumulate-on-gpu/58845](https://discourse.julialang.org/t/what-is-a-neutral-element-for-accumulate-on-gpu/58845)

Some GPU operations need a neutral element, such as XOR. You will need to import GPUArrays and set neutral element outside of CUDA.jl

#### Algorithmic Theory

- (for some reason I found this very helpful) [https://developer.download.nvidia.com/assets/cuda/files/reduction.pdf](https://developer.download.nvidia.com/assets/cuda/files/reduction.pdf)
- (I liked this a lot) [https://developer.nvidia.com/gpugems/gpugems3/part-vi-gpu-computing/chapter-39-parallel-prefix-sum-scan-cuda](https://developer.nvidia.com/gpugems/gpugems3/part-vi-gpu-computing/chapter-39-parallel-prefix-sum-scan-cuda)
- [https://nyu-cds.github.io/python-numba/05-cuda/](https://nyu-cds.github.io/python-numba/05-cuda/)
- [https://sodocumentation.net/cuda/topic/6566/parallel-reduction--e-g--how-to-sum-an-array-](https://sodocumentation.net/cuda/topic/6566/parallel-reduction--e-g--how-to-sum-an-array-)
- [https://developer.nvidia.com/gpugems/gpugems2/part-iv-general-purpose-computation-gpus-primer/chapter-33-implementing-efficient](https://developer.nvidia.com/gpugems/gpugems2/part-iv-general-purpose-computation-gpus-primer/chapter-33-implementing-efficient)
- [https://jenni-westoby.github.io/Julia\_GPU\_examples/dev/Vector\_addition/](https://jenni-westoby.github.io/Julia_GPU_examples/dev/Vector_addition/)

#### Mindfulness about GPU Hardware

- [https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html)
- [https://core.vmware.com/resource/exploring-gpu-architecture#section1](https://core.vmware.com/resource/exploring-gpu-architecture#section1)

#### Debugging

- [https://cuda.juliagpu.org/stable/development/debugging/](https://cuda.juliagpu.org/stable/development/debugging/)
- [http://www0.cs.ucl.ac.uk/staff/W.Langdon/ftp/papers/debug\_cuda.pdf](http://www0.cs.ucl.ac.uk/staff/W.Langdon/ftp/papers/debug_cuda.pdf)
- (cmd line tool) `nvidia-smi`
- (cmd line tool) `deviceQuery`

#### Useful Discourse

- [https://discourse.julialang.org/t/map-performance-with-cuarrays/33497/2](https://discourse.julialang.org/t/map-performance-with-cuarrays/33497/2)

#### Implementation Examples

- [https://github.com/JuliaGPU/CuArrays.jl/blob/master/src/accumulate.jl](https://github.com/JuliaGPU/CuArrays.jl/blob/master/src/accumulate.jl)
- [https://github.com/JuliaGPU/CuArrays.jl/blob/master/src/mapreduce.jl](https://github.com/JuliaGPU/CuArrays.jl/blob/master/src/mapreduce.jl)
- [https://github.com/mgula/fibCUDA/blob/master/fib\_cuda.cu](https://github.com/mgula/fibCUDA/blob/master/fib_cuda.cu)
