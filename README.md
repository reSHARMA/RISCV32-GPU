# RISCV32 GPU ISA

## 64 bit pointers in RV32 using custom load/store in different address space.

* Update the data layout string in the clang front-end at clang/lib/Basic/Targets/RISCV.h to "e-m:e-p:32:32-p1:64:32-i64:64-n32-S128" from "e-m:e-p:32:32-i64:64-n32-S128"
