# StaticTools

[![Dev](https://img.shields.io/badge/docs-dev-blue.svg)](https://brenhinkeller.github.io/StaticTools.jl/dev)
[![Build Status](https://github.com/brenhinkeller/StaticTools.jl/actions/workflows/CI.yml/badge.svg?branch=main)](https://github.com/brenhinkeller/StaticTools.jl/actions/workflows/CI.yml?query=branch%3Amain)
[![Coverage](https://codecov.io/gh/brenhinkeller/StaticTools.jl/branch/main/graph/badge.svg)](https://codecov.io/gh/brenhinkeller/StaticTools.jl)

Tools to enable [StaticCompiler.jl](https://github.com/tshort/StaticCompiler.jl)-based static compilation of Julia code to standalone native binaries by eliding GC allocations and `llvmcall`-ing all the things.

This package currently requires Julia 1.8+

### Examples
```julia
# This is all StaticCompiler-friendly
using StaticTools

function print_args(argc::Int, argv::Ptr{Ptr{UInt8}})
    # c"..." lets you construct statically-sized, stack allocated `StaticString`s
    # We also have m"..." and MallocString if you want the same thing but on the heap
    printf(c"Argument count is %d:\n", argc)
    for i=1:argc
        # iᵗʰ input argument string
        pᵢ = unsafe_load(argv, i) # Get pointer
        strᵢ = MallocString(pᵢ) # Can wrap to get high-level interface
        println(strᵢ)
        # No need to `free` since we didn't allocate this memory
    end
    println(c"That was fun, see you next time!")
    return 0
end

# Compile executable
using StaticCompiler # `] add https://github.com/tshort/StaticCompiler.jl` to get latest master
filepath = compile_executable(print_args, (Int64, Ptr{Ptr{UInt8}}), "./")
```
and...
```
shell> ./print_args 1 2 3 4 5.0 foo
Argument count is 7:
./print_args
1
2
3
4
5.0
foo
That was fun, see you next time!

shell> hyperfine './print_args hello there'
Benchmark 1: ./print_args hello there
  Time (mean ± σ):       2.2 ms ±   0.5 ms    [User: 0.8 ms, System: 0.0 ms]
  Range (min … max):     1.5 ms …   5.5 ms    564 runs

  Warning: Command took less than 5 ms to complete. Results might be inaccurate.

shell> ls -lh print_args
  -rwxr-xr-x  1 user  staff   8.5K Feb 10 02:36 print_args
```