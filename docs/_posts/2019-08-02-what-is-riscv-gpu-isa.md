---
layout: post
title:  "What is RISCV GPU ISA?"
author: Reshabh Sharma
---

Bespoke group is developing a version two of their RISCV (RV32) based open-source GPGPU ([bjump.org/manycore/](http://bjump.org/manycore/)). RISCV GPU ISA is a custom extension which is designed to make use of hardware features unique to the GPGPU.

We started this by adding 64 bit pointers in RV32 at address space 1.

### Instructions
##### LDW - Load from Double Word address
This instruction loads the value from a 64 bit address by concating the i32 values in two registers. 
Syntax: LDW rd, rs1, rs2 

##### SDW - Store from Double Word address
This instruction stores the value to a 64 bit address by concating the i32 values in two registers. 
Syntax: SDW rd, rs1, rs2 

Many more to be designed and added ... 
