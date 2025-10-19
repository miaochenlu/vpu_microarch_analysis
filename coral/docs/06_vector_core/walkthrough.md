# Vector Core Walkthrough Examples

## Overview

This document provides **detailed step-by-step examples** of vector instruction execution through Coral NPU's Vector Core, from scalar dispatch to register writeback. We trace specific instructions cycle-by-cycle to illustrate:

1. **Simple vector arithmetic** (vadd)
2. **Stripmining operation** (large vector processing)
3. **Convolution acceleration** (vdwconv)
4. **Vector load/store** (vld/vst)
5. **Hazard handling** (RAW dependencies)

---

## Example 1: Vector Add (No Stripmining)

### Instruction: `vadd.w v4, v1, v2`

**Operation**: Add two 256-bit vectors element-wise (8 × 32-bit elements)

**Encoding**:
```
inst[31:26] = 0x00 (func2: vadd)
inst[25:20] = 2    (vt: v2)
inst[19:14] = 1    (vs: v1)
inst[13:12] = 2    (sz: 32-bit)
inst[11:6]  = 4    (vd: v4)
inst[5]     = 0    (m: no stripmining)
inst[4:2]   = 0    (func1)
inst[1]     = 0    (x: not scalar)
inst[0]     = 0    (v: vv format)
```

### Initial State

```
v1 = [0x00000001, 0x00000002, 0x00000003, 0x00000004, 
      0x00000005, 0x00000006, 0x00000007, 0x00000008]

v2 = [0x00000010, 0x00000020, 0x00000030, 0x00000040,
      0x00000050, 0x00000060, 0x00000070, 0x00000080]

vrfsb = 0x00000000_00000000  (no pending writes)
```

### Cycle-by-Cycle Execution

#### Cycle 0: Scalar Dispatch

**Scalar Decode** identifies vector instruction:
```
d.viop = 1  (vector operation detected)
Dispatch to VCore via io.score.vinst
```

**VInst receives**:
```
io.in(0).valid = 1
io.in(0).bits.inst = encoded instruction
io.in(0).bits.op = VInstOp.VIOP
```

#### Cycle 1: VInst Slice

**VInst decodes and slices**:
```
VInst.io.out.valid = 1
VInst.io.out.lane(0).valid = 1
VInst.io.out.lane(0).bits.inst = instruction
VInst.io.out.lane(0).bits.addr = (not used for ALU ops)
VInst.io.out.lane(0).bits.data = (not used for ALU ops)
```

**Slice unit** buffers instruction for decoupling.

#### Cycle 2: VDecode FIFO Enqueue

**VDecode FIFO**:
```
f.io.in.valid = 1
f.io.in.bits(0).valid = 1
f.io.in.bits(0).inst = instruction

FIFO state:
  count = 1/16
  depth = 16 + 8 (guard)
```

#### Cycle 3: VDecodeInstruction Module

**Instruction decode** (VDecodeInstruction.scala):
```
inst = instruction word
v = 0, x = 0, m = 0, sz = 2, func1 = 0, func2 = 0

Opcode match: DecodeFmt(0, 0) → vadd
  enc.vadd = 0x01

Operand extraction:
  vd = inst[11:6] = 4 (v4)
  vs = inst[19:14] = 1 (v1)
  vt = inst[25:20] = 2 (v2)
  
Active mask calculation:
  RActiveVsVt(2):
    vs_active = UIntToOH(1, 64) = 0x0000000000000002 (bit 1)
    vt_active = UIntToOH(2, 64) = 0x0000000000000004 (bit 2)
    ractive = 0x0000000000000006 (bits 1-2)
    
  WActiveVd():
    vd_active = UIntToOH(4, 64) = 0x0000000000000010 (bit 4)
    wactive = 0x0000000000000010 (bit 4)
```

**Output** (VDecodeBits):
```
op = 0x01 (vadd)
sz = 2 (32-bit)
m = 0 (no stripmining)
vd.addr = 4
vs.addr = 1, vs.valid = 1, vs.tag = 0
vt.addr = 2, vt.valid = 1, vt.tag = 0
```

#### Cycle 4: Tag Assignment

**Tag register** (VDecode.scala lines 75-82):
```
tagReg = 0x0000000000000000 (initial, 64-bit for 16 groups)

wactive from this instruction:
  Writes v4 → group 1 (v4[5:2] = 0b0001)
  wactive = 0x0000000000000010

XOR update:
  tags(0) = tagReg = 0x0000000000000000
  tags(1) = tagReg ^ wactive(group1_bit)
         = 0x0000000000000000 ^ 0x0002  (group 1, bit 1)
         = 0x0000000000000002
         
Apply tags to operands:
  vs.tag = tagReg[vs.addr[5:2]] = tagReg[0] = 0b0000 → tag bit 0 = 0
  vt.tag = tagReg[vt.addr[5:2]] = tagReg[0] = 0b0000 → tag bit 0 = 0
  vd implicitly gets tag from tags(1)
  
tagReg := tags(1) = 0x0000000000000002
```

#### Cycle 5: Scoreboard Check

**Dependency check** (VDecode.scala lines 143-152):
```
io.vrfsb.data = 0x00000000_00000000 (no pending operations)
io.active = 0x0000000000000000 (no active executions)

wactive = io.vrfsb.data[63:0] | io.vrfsb.data[127:64] | io.active
        = 0x0000000000000000

ractive = 0x0000000000000000 (no pending writes from earlier lanes)

depends(0) = (wactive & this.wactive) != 0 || (ractive & this.ractive) != 0
           = (0 & 0x10) != 0 || (0 & 0x06) != 0
           = FALSE ✅

Instruction ready to dispatch!
```

**Command queue routing**:
```
io.cmdq(0).alu = 1  (route to VAlu)
io.cmdq(0).conv = 0
io.cmdq(0).ldst = 0
```

#### Cycle 6: VAlu Command Queue

**VAlu receives command**:
```
io.in.valid = 1
io.in.bits(0).valid = 1
io.in.bits(0).bits = VDecodeBits {
  op = 0x01,
  sz = 2,
  m = 0,
  step = 0,
  vd = 4,
  vs = {addr:1, tag:0, valid:1},
  vt = {addr:2, tag:0, valid:1}
}

VAlu command queue (depth 8):
  Enqueue command
  count = 1/8
```

#### Cycle 7: VAlu Read Phase

**VAlu issues reads**:
```
io.read(0).valid = 1
io.read(0).addr = 1  (v1)
io.read(0).tag = 0

io.read(1).valid = 1
io.read(1).addr = 2  (v2)
io.read(1).tag = 0
```

**VRegfile responds** (1-cycle latency):
```
// Cycle 7: Read request
// Cycle 8: Data available

io.read(0).data = v1 data (256-bit from 8 segments)
io.read(1).data = v2 data (256-bit from 8 segments)
```

#### Cycle 8: VAlu Compute

**VAluInt performs SIMD addition**:

```
For each of 8 × 32-bit lanes:
  
Lane 0: result[0] = v1[0] + v2[0] = 0x00000001 + 0x00000010 = 0x00000011
Lane 1: result[1] = v1[1] + v2[1] = 0x00000002 + 0x00000020 = 0x00000022
Lane 2: result[2] = v1[2] + v2[2] = 0x00000003 + 0x00000030 = 0x00000033
Lane 3: result[3] = v1[3] + v2[3] = 0x00000004 + 0x00000040 = 0x00000044
Lane 4: result[4] = v1[4] + v2[4] = 0x00000005 + 0x00000050 = 0x00000055
Lane 5: result[5] = v1[5] + v2[5] = 0x00000006 + 0x00000060 = 0x00000066
Lane 6: result[6] = v1[6] + v2[6] = 0x00000007 + 0x00000070 = 0x00000077
Lane 7: result[7] = v1[7] + v2[7] = 0x00000008 + 0x00000080 = 0x00000088

All additions happen in parallel!
```

**Result vector**:
```
v4 = [0x00000011, 0x00000022, 0x00000033, 0x00000044,
      0x00000055, 0x00000066, 0x00000077, 0x00000088]
```

#### Cycle 9: VAlu Write Phase

**VAlu issues write**:
```
io.write(0).valid = 1
io.write(0).addr = 4
io.write(0).data = 0x00000088_00000077_00000066_00000055_
                     00000044_00000033_00000022_00000011
```

**VRegfile receives write** (Cycle 9):
```
writevalid(0) = 1
writebits(0).addr = 4
writebits(0).data = result

// Register write internally
writevalidReg := writevalid  (Cycle 10)
writebitsReg(0) := writebits(0)
```

**Scoreboard update**:
```
// Set scoreboard on write initiation
vrfsbSet = UIntToOH(4, 64) = 0x0000000000000010
vrfsbSet_128 = Cat(vrfsbSet, vrfsbSet) = 0x0000000000000010_0000000000000010

vrfsb := (vrfsb & ~vrfsbClr) | vrfsbSet
       = (0 & ~0) | 0x0000000000000010_0000000000000010
       = 0x0000000000000010_0000000000000010 (v4 marked busy)
```

#### Cycle 10: VRegfile Write

**Write to SRAM segments**:
```
For each segment j (0-7):
  vreg(j).io.write(0).valid = 1
  vreg(j).io.write(0).addr = 4
  vreg(j).io.write(0).data = result[32*j+31 : 32*j]
  
Segment 0 writes: 0x00000011
Segment 1 writes: 0x00000022
...
Segment 7 writes: 0x00000088
```

**Scoreboard clear**:
```
vrfsbClr on write completion:
  vrfsbClr = 0x0000000000000010_0000000000000010
  
vrfsb := (vrfsb & ~vrfsbClr) | 0
       = (0x0000000000000010_0000000000000010 & ~0x0000000000000010_0000000000000010) | 0
       = 0x00000000_00000000 ✅ v4 available
```

#### Cycle 11: Complete

**Instruction complete!**
- Total latency: **11 cycles** (dispatch to writeback)
- Throughput: **1 inst/cycle** (if no dependencies)

---

## Example 2: Stripmining Operation

### Instruction: `vadd.w.m v8, v16, v20` (m=1, processes 4 strips)

**Operation**: Add vectors with stripmining (processes v8-v11 ← v16-v19 + v20-v23)

### Encoding

```
m = 1 (stripmining enabled)
vd = 8, vs = 16, vt = 20

Hardware automatically processes:
  Strip 0: v8  = v16 + v20
  Strip 1: v9  = v17 + v21
  Strip 2: v10 = v18 + v22
  Strip 3: v11 = v19 + v23
```

### Active Mask Calculation

**From RegActive function** (VCommon.scala lines 24-114):

```
regnum = 8 (0b001000)
  regnum[5:2] = 0b0010 = 2 (group 2)
  regnum[1:0] = 0b00 = 0 (strip 0)
  
step = 3 (process 4 strips: 0,1,2,3)

UIntToOH(2, 16) = 0x0004 (group 2 bit set)

For strip 0 (addr[1:0] = 00):
  oh0 pattern sets bits: 0, 4, 8, 12, 16, ... for group 2
  Result: bit 8 set

For strip 1 (addr[1:0] = 01):
  oh1 pattern sets bits: 1, 5, 9, 13, 17, ... for group 2
  Result: bit 9 set

For strip 2 (addr[1:0] = 10):
  oh2 pattern sets bits: 2, 6, 10, 14, 18, ... for group 2
  Result: bit 10 set

For strip 3 (addr[1:0] = 11):
  oh3 pattern sets bits: 3, 7, 11, 15, 19, ... for group 2
  Result: bit 11 set

Combined (step <= 3):
  active = oh0 | oh1 | oh2 | oh3
         = 0x0000000000000F00 (bits 8-11 set)
         = registers v8, v9, v10, v11
```

### Execution

**VAlu processes strips sequentially**:

```
Cycle N:   Read v16, v20 → Compute → Write v8
Cycle N+1: Read v17, v21 → Compute → Write v9
Cycle N+2: Read v18, v22 → Compute → Write v10
Cycle N+3: Read v19, v23 → Compute → Write v11
```

**Total**: 4 cycles for 4 strips = **4× throughput multiplier**

**Scoreboard**:
```
All 4 registers marked busy initially:
  vrfsb[8]  = 1
  vrfsb[9]  = 1
  vrfsb[10] = 1
  vrfsb[11] = 1

Cleared as each write completes
```

---

## Example 3: Convolution Acceleration

### Instruction: `vdwconv.b v48, v0, v4`

**Operation**: Depthwise 3×3 convolution, 8-bit elements, accumulate to v48-v55

### Setup

**Kernel** (3×3 filter in v0):
```
v0[0:8] = [k00, k01, k02, k10, k11, k12, k20, k21, k22, ...]
```

**Input** (feature map in v4-v7, 4 strips for 256-bit config):
```
v4  = strip 0 data
v5  = strip 1 data
v6  = strip 2 data
v7  = strip 3 data
```

**Accumulator** (v48-v63 reserved):
```
Initial: v48-v55 = 0 (8 registers for 8 output channels)
```

### Execution Phases

#### Phase 1: Initialize Accumulator

**Instruction**: `vdwconv.init v48, v0`

```
io.conv.valid = 1
io.conv.op.init = 1
io.conv.addr1 = 0  (source: v0)

VConvAlu:
  Load v0 into accumulator array
  acc[0:7][0:7] = v0 data (repeated/broadcasted)
```

#### Phase 2: Transpose Kernel

**Automatic transpose**:

```
io.transpose.valid = 1
io.transpose.addr = 0  (v0)
io.transpose.index = 0  (first 8×8 block)

VRegfileSegment transposes:
  Input (row-major):  [k00, k01, k02, ...]
  Output (col-major): [k00, k10, k20, ...]
  
This prepares kernel for outer-product multiplication
```

#### Phase 3: Convolution Accumulation (4 Strips)

**Strip 0**: Process v4

```
Cycle M: Convolution Operation
  io.conv.valid = 1
  io.conv.op.conv = 1
  io.conv.addr1 = 0   (kernel in v0, transposed)
  io.conv.addr2 = 4   (input strip 0 in v4)
  io.conv.mode = 0    (8-bit mode)
  io.conv.abias = 0   (no bias)
  io.conv.bbias = 0
  io.conv.asign = 0   (unsigned)
  io.conv.bsign = 0

VConvAlu performs outer-product:
  For each of 8×8 positions:
    acc[i][j] += kernel[i] * input[j]
    
  Example (one MAC):
    acc[0][0] += k00 * v4[0]
    acc[0][1] += k00 * v4[1]
    ...
    acc[7][7] += k77 * v4[7]
    
  256 MACs in parallel! ✅
```

**Strip 1-3**: Repeat for v5, v6, v7

```
Each strip adds another 256 MACs to accumulator
Total MACs: 256 × 4 strips = 1024 MACs
```

#### Phase 4: Writeback and Clear

**Instruction**: `vdwconv.wclr v48`

```
io.conv.valid = 1
io.conv.op.wclr = 1

VConvAlu:
  Write accumulator to v48-v55 (8 registers)
  Clear accumulator for next operation
  
VRegfile:
  convClear = 1
  For each segment i:
    vreg(i).io.conv.valid = 1
    vreg(i).io.conv.data(j) = vconv.io.out(j)[32*i+31:32*i]
    
Scoreboard clear:
  vrfsbClr for v48-v55 (8 bits)
```

### Performance

**Total cycles**: ~20-30 cycles (including all phases)  
**Total MACs**: 1024 (256/strip × 4 strips)  
**Effective throughput**: **34-51 MACs/cycle** sustained  
**Peak throughput**: **256 MACs/cycle** (during accumulation)

---

## Example 4: Vector Load

### Instruction: `vld.w v8, 0(x10)`

**Operation**: Load 256 bits from memory address in x10 to vector register v8

### Execution

#### Cycle 0-2: Dispatch & Decode

(Similar to vadd example)

```
VInst identifies VLD operation
Address calculation: addr = x10 + 0
```

#### Cycle 3: VLdSt Read Request

**VLdSt issues DBus read**:

```
io.dbus.valid = 1
io.dbus.addr = x10  (from scalar register)
io.dbus.op = Load
io.dbus.size = 256 bits (32 bytes)
```

#### Cycle N+4 to N+8: Memory Access

**Depends on memory hierarchy**:

```
Best case (DTCM hit):  +1 cycle
L1 D$ hit:             +3 cycles
L1 D$ miss:            +50+ cycles (external memory)
```

**Assuming DTCM hit**:

```
Cycle 4: DBus request
Cycle 5: DTCM responds
  io.dbus.rdata = 256-bit data
```

#### Cycle 6: VLdSt Write to VRegfile

**VLdSt issues write**:

```
io.write(0).valid = 1
io.write(0).addr = 8
io.write(0).data = memory data
```

**VRegfile** processes write (Cycles 6-8, as in vadd example)

**Total latency**: 
- DTCM: ~8-10 cycles
- L1 hit: ~10-12 cycles
- Miss: 50+ cycles

---

## Example 5: RAW Hazard

### Instructions:
```
vadd.w v4, v1, v2    # Lane 0: Write v4
vsub.w v5, v4, v3    # Lane 1: Read v4 (RAW hazard!)
```

### Execution

#### Cycle 0-5: First Instruction (vadd)

```
Processes normally as in Example 1
```

#### Cycle 3: Second Instruction Enters VDecode

```
VDecodeInstruction for vsub:
  vs.addr = 4 (v4)
  vt.addr = 3 (v3)
  vd.addr = 5 (v5)
  
ractive = 0x0000000000000010 (v4 pending write from lane 0)
```

#### Cycle 4: Dependency Check

```
depends(1) = (wactive & this.wactive) != 0 || (ractive & this.ractive) != 0
           = (0 & 0x20) != 0 || (0x10 & 0x10) != 0
           = TRUE ❌ RAW HAZARD!

Action: Stall lane 1 in VDecode
  io.out(1).valid = 0  (don't dispatch yet)
```

#### Cycle 9: First Instruction Completes

```
vadd writes v4 to VRegfile
Scoreboard clears v4:
  vrfsb[4] = 0 ✅
```

#### Cycle 10: Retry Second Instruction

```
VDecode re-checks dependencies:
  ractive = 0x0000000000000000 (v4 write complete)
  depends(1) = FALSE ✅
  
Dispatch vsub to VAlu
```

#### Cycle 11-19: Second Instruction Executes

```
Read v4 (now available), v3
Compute v5 = v4 - v3
Write v5
```

**Stall penalty**: ~5-6 cycles (waiting for v4 writeback)

---

## Performance Summary

| Scenario | Latency | Throughput | Bottleneck |
|----------|---------|------------|------------|
| **Simple VADD** | 11 cycles | 1 inst/cycle | None |
| **Stripmining (4 strips)** | 14 cycles | 0.25 inst/cycle | Sequential processing |
| **Convolution** | 25 cycles | 256 MACs/cycle | Memory bandwidth |
| **Vector Load (DTCM)** | 10 cycles | 32 bytes/cycle | Memory latency |
| **RAW Hazard** | +6 cycles | Stalled | Dependency |

---

## Key Insights

1. **Decoupling works**: Deep FIFO (16-entry) hides scalar dispatch variability
2. **Stripmining efficient**: 4× throughput for large vectors with minimal overhead
3. **Forwarding critical**: VRegfile bypassing eliminates many RAW stalls
4. **Convolution fast**: 256 MACs/cycle peak via outer-product engine
5. **Dependencies hurt**: RAW hazards add 5-10 cycle penalties

---

## Optimization Strategies

### Software (Compiler)

1. **Reorder instructions**: Place independent ops between dependent ones
2. **Loop unrolling**: Expose more parallelism for stripmining
3. **Prefetch**: Issue loads early to hide memory latency
4. **Register allocation**: Minimize register reuse to reduce WAW/RAW

### Hardware (Future)

1. **More read ports**: Reduce register file contention
2. **Out-of-order execution**: Dispatch past stalled instructions
3. **Larger FIFOs**: Increase decoupling capacity
4. **Multiple VAlu**: Parallel SIMD lanes

---

**See Also**:
- [Vector Core Overview](README.md) - Architecture and components
- [VRegfile](vregfile.md) - Register file details
- [VAlu](valu.md) - SIMD execution unit

---

**Source Files**:
- `coral/codes/coralnpu/hdl/chisel/src/coralnpu/vector/VCore.scala` (172 lines)
- `coral/codes/coralnpu/hdl/chisel/src/coralnpu/vector/VDecode.scala` (326 lines)
- `coral/codes/coralnpu/hdl/chisel/src/coralnpu/vector/VAlu.scala` (370+ lines)
- `coral/codes/coralnpu/hdl/chisel/src/coralnpu/vector/VConvAlu.scala`

