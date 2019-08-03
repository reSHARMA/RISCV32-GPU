---
layout: post
title:  "Custom 64 bit pointers in RV32"
author: Reshabh Sharma
---

32 bit addressable memory is very limited to the present GPGPU use cases. We added 64 bit pointers at address space 1.
We support 64 bit pointers by adding two new instructions in the RV32 backend.
1. LDW Load from Double Word address
2. SDW Store to Double Word address

We divide the whole process in the following phases:

### Phase 1: Defining instruction syntax in the backend
We fixed the syntax of LDW and SDW as:
- LDW rd, rs1, rs2
- SDW rd, rs1. rs2

We started with defining instruction format for store instruction in RISCVInstrInfo.td
```cpp
let hasSideEffects = 0, mayLoad = 0, mayStore = 1 in
class Store_rrr<bits<7> funct7, bits<3> funct3, string opcodestr>
    : RVInstR<funct7, funct3, OPC_STORE, (outs), (GPR:$rd, ins GPR:$rs1, GPR:$rs2), opcodestr, "$rd, $rs1, $rs2">;
```

We use the above defined format to define the custom store instruction
```cpp
def SDW	 : Store_rrr<0b0000010, 0b000, "sdw">;
```
### Phase 2: Update data layout string 
Data layout string tells the front-end not to truncate 64 bit pointer addresses in RV32. We updated the data layout string to "e-m:e-p:32:32-p1:64:32-i64:64-n32-S128"

-p1:64:32- denotes pointer in address space 1 of size 64 bits and alignment 32 bits.

### Phase 3: Lowering to custom load and store
In this phase we lower the ISD::LOAD and ISD::STORE into RISCV::LWD and RISCV::SWD respectively.

**Problem 1:** 
ISD::STORE has 3 input node (chain, rd, rs) and 1 output node (chain) but our desired result RISCV::SWD has 4 input node (chain, rd, rs1, rs2) and 1 output node (chain). 

**Plan 1:** 
We though of introducing a custom ISD node which has similar format to RISCV::SDW and directly lower IR to that node. 
[ ] We were sure from the start that we should avoid this, so this never went ahead.

**Plan 2:**
Abstract two rs1 and rs2 into one, so that it is in sync with the format of ISD::LOAD node
[x] We went ahead to explore options for this abstraction.

**Problem 2:**
Abstract rs1 and rs2 into one such that it passes the legalizer

**Plan 1:**
Inspired from ADMGPU and SPARC register pair, we thought to abstract out 4 inputs into 3 by combining two with a register pair. 
```sh
SDW rd, rs1, rs2 -> SDW rd, [rs1, rs2]
```
[ ] Though it sounds promising but I was a bit hesitant as I felt it unnecessary to add register pairs to support one feature.

**Plan 2:**
Inspired from AMDGPU, we thought of converting the GlobalAddress in address space 1 into v2i32. This worked well and GlobalAddress passed the legalizer but legalizing store node failed as one of the operand was was v2i32.

[ ] We were stuck on this for some time, this could have worked though with a limitation that it would not be able to differentiate between a real v2i32 and a global address in address space 1. We went keenly as the next approached looked very promising. 

**Plan 3:**
AVR supports i16 and i32 on register size 8 bits. They do it by using register pairs. We thought we can do the same and register pairs from Plan 1 are back. We realized that if we say i64 is legal through register pairs then no i64 result will be expanded and we would have to support all of it on our own.

[ ] This was the backup plan.

**Plan 4:**
We thought to introduce a WRAPPER node, that would wrap the illegal i64 global address and make it pass the legalizer. We planned to keep the return type of WRAPPER node to i32 to easily pass the legalization phase.

We could foresee that at the end we might need to lower 
```
STORE(rd, WRAPPER(GlobalAddress(64 bit))) => SDW rd, rs1, rs2
```

[ ] We questioned, do we really need a wrapper node? and went ahead to explore plan 5. 

**Plan 5:**
Break the global address into two i32 global addresses, send the lower bit address to the store node and glue the high bit address to the low bit address

```
GlobalAddress 0xABCD -> GlobalAddress 0xCD | Glue | GlobalAddress  0xAB
```

This should pass the legalizer and we planned that at the instruction selection we will look for GlobalAddress in address space 1 and lower it to our custom load/store.

- Why do we want to break i64 GlobalAddress into two i32 GlobalAddress?
I struggled to find a way to do this so this question arose. I wanted to use MO_HI and MO_LO without adding any new target flags. 

In the end the GlobalAddress breaks down to LUI and ADDI pairs. So instead of breaking i64 global address into two global address, lets break i64 global address into two pairs of ADDI and LUI.

```
LUI             LUI
|                |
ADDI -- Glue -- ADDI
|
STORE
```

After Tim's reply on our [llvm-dev thread](http://lists.llvm.org/pipermail/llvm-dev/2019-July/133828.html) we realized that Glue is not a great solution for this. On his suggestion we moved to Plan 6.

[ ] Plan 6 was similar to the WRAPPER node from plan 4

**Plan 6**
Using ISA::BUILD_PAIR to pair high and low ADDI. This failed while legalizing the store node as one operand was of type i64 which is illegal (BUILD_PAIR).

- Can't we lower the store before the legalizer?
Indeed! we should.

[x] We lowered the store into ISD::SDW before the legalizer and everything worked

**Problem 3**
We failed when we tried to lower load using the same approach. Load unlike store have a integer return type. Legalizer tried to expand it. In that process it tries to expand the BUILD_PAIR operand over the LOAD node. On that it fails.

[x] We moved the lowering of load node one phase up at the DAG Combine I phase and it worked.

Finally we were able to support 64 bit pointer in address space 1 on RV32.
