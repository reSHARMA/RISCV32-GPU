---
title: Getting Started
permalink: /docs/home/
redirect_from: /docs/index.html
---


Start with cloning the llvm-project 
```sh
$ git clone https://github.com/llvm/llvm-project.git -b llvmorg-8.0.0 
```

Replace the RISCV target backend with our fork
```sh
$ cd llvm-project/llvm/lib/Target
$ rm -rf RISCV
$ git clone https://github.com/reSHARMA/RISCV32-GPU.git
```

Update the updated data layout string to "e-m:e-p:32:32-p:64:32-i64:64-n32-S128" in clang to match the backend [here](https://github.com/llvm/llvm-project/blob/llvmorg-8.0.0/clang/lib/Basic/Targets/RISCV.h#L81)

Build LLVM with Clang, till version 8.x RISCV was an experimental target!

```sh
$ cd llvm-project
$ mkdir build & cd build
$ cmake -DLLVM_ENABLE_PROJECTS=clang  -DCMAKE_BUILD_TYPE=Debug -DLLVM_TARGETS_TO_BUILD="X86" -DLLVM_EXPERIMENTAL_TARGETS_TO_BUILD="RISCV" -DLLVM_USE_LINKER=gold -DLLVM_ENABLE_Z3_SOLVER=OFF ../llvm
$ make 
```
