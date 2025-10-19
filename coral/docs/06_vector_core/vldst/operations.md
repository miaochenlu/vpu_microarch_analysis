# VLdSt Operations

## Overview

VLdSt supports **three basic vector memory operations**, all optimized for CNN inference patterns.

---

## Supported Instructions

### 1. vld - Vector Load

**Syntax**:
```assembly
vld.{b|h|w} vd, (rs1)              # Unit-stride
vld.{b|h|w} vd, (rs1), rs2         # Strided
vld.{b|h|w} vd, (rs1), rs2, rs3    # Custom length
```

**Encoding** (VInst.scala lines 127-128):
```scala
val isVLD  = func3 === 0.U         // Vector Load
vld_o(i) := reqvalid(i) && (op === VInstOp.VLD) && !p
vld_u(i) := reqvalid(i) && (op === VInstOp.VLD) &&  p
```

**Fields**:
- `.b/.h/.w`: Element size (8-bit, 16-bit, 32-bit)
- `vd`: Destination vector register (v0-v63)
- `rs1`: Base address (scalar register)
- `rs2`: Stride in elements (optional)
- `rs3`: Length in bytes (optional)
- `m` bit: Stripmining enable (4-register block)

**Operation** (VLdSt.scala lines 99-101):
```scala
def IsLoad(): Bool = {
  op === e.vld.U
}
```

**Behavior**:
1. Calculate effective address: `addr = rs1 + offset`
2. Load data from memory (16 bytes per transaction)
3. Write to vector register file
4. If stripmining (`m=true`): Repeat for next register

---

### 2. vst - Vector Store

**Syntax**:
```assembly
vst.{b|h|w} vs, (rs1)              # Unit-stride
vst.{b|h|w} vs, (rs1), rs2         # Strided
vst.{b|h|w} vs, (rs1), rs2, rs3    # Custom length
```

**Encoding** (VInst.scala lines 129-130):
```scala
val isVST  = func3 === 1.U         // Vector Store
vst_o(i) := reqvalid(i) && (op === VInstOp.VST) && !p
vst_u(i) := reqvalid(i) && (op === VInstOp.VST) &&  p && !q
```

**Fields**:
- `.b/.h/.w`: Element size (8-bit, 16-bit, 32-bit)
- `vs`: Source vector register (v0-v63)
- `rs1`: Base address (scalar register)
- `rs2`: Stride in elements (optional)
- `rs3`: Length in bytes (optional)
- `m` bit: Stripmining enable

**Operation** (VLdSt.scala lines 103-105):
```scala
def IsStore(): Bool = {
  op === e.vst.U || op === e.vstq.U
}
```

**Behavior**:
1. Read data from vector register file
2. Calculate effective address: `addr = rs1 + offset`
3. Apply swizzle if misaligned
4. Write to memory (16 bytes per transaction)
5. If stripmining: Repeat for next register

---

### 3. vstq - Vector Store Quadrant

**Syntax**:
```assembly
vstq vs, (rs1)                     # Always 64 bytes (4 registers)
```

**Encoding** (VInst.scala line 131):
```scala
vst_q(i) := reqvalid(i) && (op === VInstOp.VST) &&  p &&  q
```

**Purpose**: **Special operation for convolution output**

**Problem**: Convolution generates 4 adjacent registers, but each register only has **4 bytes of valid data** (not full 16 bytes).

**Memory Layout Requirement**:
```
Register: [v48  ][v49  ][v50  ][v51  ]
          16B    16B    16B    16B     (full registers)
          
But only need:
          [4B   ][4B   ][4B   ][4B   ]
          
Memory: [v48(0:3)][v49(0:3)][v50(0:3)][v51(0:3)]
        0x00      0x04      0x08      0x0C       (16 bytes total, packed)
```

**Operation**:
1. Store 4 bytes from vs → addr
2. Store 4 bytes from vs (offset +4) → addr+4
3. Store 4 bytes from vs (offset +8) → addr+8
4. Store 4 bytes from vs (offset +12) → addr+12
5. Increment vs, repeat for next register (vs+1, vs+2, vs+3)

**Total**: 16 iterations, 64 bytes written (4 registers × 4 bytes/register × 4 slices)

**Code** (VLdSt.scala lines 136, 169):
```scala
out.last := !in.m && in.op =/= e.vstq.U  // vstq always stripmines
out.quad := Mux(in.op === e.vstq.U, step + 1.U, 0.U)  // Track quadrant
```

---

## Instruction Encoding

### f2 Field (Function Modifier)

**Bits** (VLdSt.scala line 110-111):
```scala
val stride = in.f2(1)  // Bit 1: Stride enable
val length = in.f2(0)  // Bit 0: Length enable
```

**Encoding**:
| f2[1] | f2[0] | Mode | Description |
|-------|-------|------|-------------|
| 0 | 0 | Default | Unit-stride, maxvl length |
| 0 | 1 | Custom length | Unit-stride, rs3 length |
| 1 | 0 | Strided, maxvl | Stride=rs2, maxvl length |
| 1 | 1 | Strided, custom | Stride=rs2, rs3 length |

### sz Field (Size)

**One-hot encoding** (VLdSt.scala line 112):
```scala
assert(PopCount(in.sz) <= 1.U)  // Only one bit set
```

| sz[2:0] | Element Size | Elements per Register |
|---------|--------------|----------------------|
| 001 | 8-bit (byte) | 16 elements |
| 010 | 16-bit (halfword) | 8 elements |
| 100 | 32-bit (word) | 4 elements |

### m Bit (Stripmining)

**Meaning** (VLdSt.scala line 117):
```scala
val limit = Mux(in.m, maxvlbm, maxvlb)
// m=0: Single register (16 bytes)
// m=1: 4 registers (64 bytes)
```

---

## Detailed Operation Breakdown

### vld: Load Execution Flow

**Step-by-step** (VLdSt.scala lines 108-141):

```
1. VInst receives instruction from scalar core
2. VInst extracts operands: vd, rs1, rs2, rs3, f2, sz, m
3. VInst forwards to VDecode FIFO

4. VDecode routes to VLdSt command queue
5. Fin() function processes:
   - Calculate stride: 
     * If f2[1]=0: stride = 16 bytes (unit-stride)
     * If f2[1]=1: stride = rs2 × element_size
   - Calculate length:
     * If f2[0]=0: length = 64 bytes (if m=1) or 16 bytes (if m=0)
     * If f2[0]=1: length = min(rs3, maxvl)
   - Initialize: addr=rs1, remain=length, vd=vd

6. Loop (Fout() function):
   a. Issue DBus read: addr, size=min(remain, 16)
   b. Wait for data: io.dbus.rdata
   c. Swizzle data if addr misaligned
   d. Write to register: vd
   e. Update: addr += stride, vd += 1, remain -= 16
   f. If remain > 0: goto 6a
   
7. Done: Scoreboard clears dependency
```

**Code Path**:
```
VInst.scala → VDecode.scala → VLdSt.scala (Fin) → 
VCmdq (iteration) → VLdSt.scala (Fout) → 
DBus transaction → Register write
```

### vst: Store Execution Flow

**Similar to vld, but**:
1. Read from register file FIRST
2. Swizzle data BEFORE bus write
3. No register writeback (memory is destination)

**Key Difference** (VLdSt.scala lines 239-265):
```scala
// Store path has data FIFO
when (rdataEnNxt) {
  rdataSize := qsize
  rdataAddr := q.io.out.bits.addr
  rdataAshf := q.io.out.bits.addr - (quad * (maxvlb >> 2.U))
}
rdataEn := rdataEnNxt  // Delayed by 1 cycle
data.io.in.valid := rdataEn
data.io.in.bits.wdata := Swizzle(false, 8, rdataAshf, io.read.data)
```

### vstq: Quadrant Store Execution Flow

**Special stripmining** (VLdSt.scala lines 147-172):
```scala
val vstq = in.op === e.vstq.U
val fmaxvlb = Mux(vstq, maxvlb >> 2, maxvlb)  // 4 bytes (not 16!)

// Update destination register only every 4 iterations
out.vd.addr := Mux(vstq && step(1,0) =/= 3.U, 
                   in.vd.addr,           // Keep same
                   in.vd.addr + 1.U)     // Increment

// 16 iterations total for 4 registers
val last = Mux(vstq, step === 15.U, step === 3.U)
```

**Iteration Pattern**:
```
Step 0-3:   v48[0:3], v48[4:7], v48[8:11], v48[12:15]  → 0x00-0x0C
Step 4-7:   v49[0:3], v49[4:7], v49[8:11], v49[12:15]  → 0x10-0x1C
Step 8-11:  v50[0:3], v50[4:7], v50[8:11], v50[12:15]  → 0x20-0x2C
Step 12-15: v51[0:3], v51[4:7], v51[8:11], v51[12:15]  → 0x30-0x3C
```

---

## Example Walkthroughs

### Example 1: Simple Unit-Stride Load

**Assembly**:
```assembly
li x10, 0x10000000
vld.w v0, (x10)        # Load 16 bytes
```

**Encoding**:
- vd = v0
- rs1 = x10 = 0x10000000
- f2 = 0b00 (unit-stride, maxvl)
- sz = 0b100 (32-bit)
- m = 0 (single register)

**Execution**:
```
Cycle N: VInst dispatches to VDecode
Cycle N+1: VDecode routes to VLdSt command queue
Cycle N+2: Fin() initializes: addr=0x10000000, remain=16, vd=0
Cycle N+3: Scoreboard check passes
Cycle N+4: DBus read issued: addr=0x10000000, size=16
Cycle N+5: DBus returns data
Cycle N+6: Swizzle (none needed, aligned)
Cycle N+7: Write to v0
Cycle N+8: Scoreboard clears v0 dependency

Total latency: ~8 cycles (best case)
```

### Example 2: Strided Load with Stripmining

**Assembly**:
```assembly
li x10, 0x10000000
li x11, 8              # Stride = 8 elements = 32 bytes (32-bit)
vld.w v8, (x10), x11   # Load 64 bytes with stride (m=1)
```

**Encoding**:
- vd = v8
- rs1 = x10 = 0x10000000
- rs2 = x11 = 8 elements
- f2 = 0b10 (strided, maxvl)
- sz = 0b100 (32-bit)
- m = 1 (4 registers)

**Execution**:
```
Fin(): addr=0x10000000, offset=32 (8×4), remain=64, vd=8

Iteration 0:
  DBus read: addr=0x10000000, size=16 → v8
  Fout(): addr=0x10000020 (+32), vd=9, remain=48

Iteration 1:
  DBus read: addr=0x10000020, size=16 → v9
  Fout(): addr=0x10000040 (+32), vd=10, remain=32

Iteration 2:
  DBus read: addr=0x10000040, size=16 → v10
  Fout(): addr=0x10000060 (+32), vd=11, remain=16

Iteration 3:
  DBus read: addr=0x10000060, size=16 → v11
  last = true (step=3)

Result: v8-v11 loaded from addresses 0x00, 0x20, 0x40, 0x60
```

### Example 3: Quadrant Store (Convolution Output)

**Assembly**:
```assembly
li x12, 0x20000000
vstq v48, (x12)        # Store v48-v51 (64 bytes, packed)
```

**Execution**:
```
Fin(): addr=0x20000000, offset=4, remain=64, vs=48

Step 0: Read v48[0:3],   addr=0x20000000, quad=0
Step 1: Read v48[4:7],   addr=0x20000004, quad=1
Step 2: Read v48[8:11],  addr=0x20000008, quad=2
Step 3: Read v48[12:15], addr=0x2000000C, quad=3, vs→v49

Step 4: Read v49[0:3],   addr=0x20000010, quad=0
Step 5: Read v49[4:7],   addr=0x20000014, quad=1
Step 6: Read v49[8:11],  addr=0x20000018, quad=2
Step 7: Read v49[12:15], addr=0x2000001C, quad=3, vs→v50

... (repeat for v50, v51)

Step 15: Read v51[12:15], addr=0x2000003C, last=true

Memory layout: 64 bytes contiguous, 4 bytes per register slice
```

---

## Performance Characteristics

| Operation | Best-case Latency | Worst-case Latency | Throughput |
|-----------|-------------------|-------------------|------------|
| **vld (hit)** | 3-5 cycles | 5-7 cycles | 1/cycle |
| **vld (miss)** | 10-50 cycles | 50-200 cycles | Depends on cache |
| **vst** | 2-4 cycles | 4-8 cycles | 1/cycle |
| **vstq** | 8-12 cycles | 12-20 cycles | 1/4 cycles (16 iterations) |

**Notes**:
- Latency includes: command queue, scoreboard, pipeline, bus transaction
- Cache miss dominates worst-case latency
- Stripmining adds ~1 cycle overhead per iteration
- Misaligned addresses: no penalty (swizzle is combinational)

---

## Operation Constraints

### Assertions (VLdSt.scala lines 113-115)

```scala
assert(!(in.op === e.vst.U  && ( in.vd.valid || !in.vs.valid)))
// Store must have source (vs), not destination (vd)

assert(!(in.op === e.vstq.U && ( in.vd.valid || !in.vs.valid)))
// vstq same as vst

assert(!(in.op === e.vld.U  && (!in.vd.valid ||  in.vs.valid)))
// Load must have destination (vd), not source (vs)
```

### Size Constraints (VLdSt.scala line 112)

```scala
assert(PopCount(in.sz) <= 1.U)
// Only one size bit can be set (8b XOR 16b XOR 32b)
```

### vstq Constraints (VLdSt.scala lines 151-157)

```scala
assert(in.op === e.vdwconv.U || op === e.adwconv.U)
assert(in.m === false.B)       // vstq doesn't use m bit
assert(in.sz === 4.U)          // Always 32-bit
assert(in.sv.valid === false.B) // No scalar operand
```

---

## See Also

- **[Addressing Modes](addressing.md)** - Stride calculation and stripmining details
- **[Microarchitecture](microarchitecture.md)** - Pipeline implementation and FIFOs
- **[Limitations](limitations.md)** - Unsupported RISC-V Vector features

---

**Source**: `VLdSt.scala` lines 87-184, `VInst.scala` lines 127-131

