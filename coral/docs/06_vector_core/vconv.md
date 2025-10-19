# Convolution Accelerator (VConvAlu & VConvCtrl)

## Overview

The **Convolution Accelerator** is Coral NPU's dedicated hardware for accelerating 2D convolution and depthwise convolution operations. It achieves **256 MACs (Multiply-Accumulate) per cycle** through an outer-product matrix multiplication engine with integrated accumulator array.

**Key Features**:
- **Peak throughput**: 256 MACs/cycle (8×8 matrix multiply)
- **Accumulator array**: 8×8 × 32-bit registers (2KB total)
- **Operations**: conv (accumulate), init (initialize), tran (transpose), wclr (writeback & clear)
- **Data types**: 8-bit quantized (INT8/UINT8) with bias support
- **Sparse convolution**: 3 modes for depthwise operations
- **Integration**: Direct connection to VRegfile (v48-v63 reserved)

---

## Architecture

### Block Diagram

```mermaid
graph TD
    VConvCtrl[VConvCtrl<br/>Command Queue] -->|control| VConvAlu[VConvAlu<br/>8×8 MAC Array]
    
    VRF[VRegfile] -->|transpose| VConvAlu
    VRF -->|read data| VConvAlu
    VConvAlu -->|write accum| VRF
    
    subgraph VConvAlu
        ACC[Accumulator<br/>8×8×32-bit]
        VDOT[VDot Units<br/>64 parallel]
        INTER[Interleave Logic]
    end
    
    VDot -->|256 MACs| ACC
    ACC -->|writeback| INTER
    
    style VConvCtrl fill:#e1f5ff
    style VConvAlu fill:#d4edda
    style ACC fill:#f8d7da
    style VDOT fill:#fff4e1
```

### Component Hierarchy

```
VConvCtrl (Control & Scheduling)
  ├── Command Queue (4-deep FIFO)
  ├── Scoreboard tracking
  └── VRegfile interface

VConvAlu (Execution Engine)
  ├── Accumulator Array (8×8×32-bit)
  ├── VDot Units (64 parallel MACs)
  ├── Interleave Logic (output reordering)
  └── Bias & Sign Support
```

---

## VConvAlu - MAC Array Engine

**File**: `VConvAlu.scala` (127 lines)

### Accumulator Array

**Structure** (VConvAlu.scala lines 56):
```scala
val acc = Reg(Vec(tcnt, Vec(tcnt, UInt(32.W))))
// tcnt = vectorBits / 32 = 8 for 256-bit config
// Total: 8×8 = 64 registers × 32-bit = 2048 bits = 256 bytes
```

**Layout**:
```
Accumulator Array (8×8 matrix):
┌─────────────────────────────────────────────────┐
│ acc[0][0] acc[0][1] acc[0][2] ... acc[0][7]    │
│ acc[1][0] acc[1][1] acc[1][2] ... acc[1][7]    │
│ acc[2][0] acc[2][1] acc[2][2] ... acc[2][7]    │
│ ...                                              │
│ acc[7][0] acc[7][1] acc[7][2] ... acc[7][7]    │
└─────────────────────────────────────────────────┘

Each element: 32-bit signed integer (accumulates INT8×INT8 products)
```

### Operations

#### 1. Initialize (init)

**Purpose**: Load initial values into accumulator

**Code** (VConvAlu.scala lines 87-88):
```scala
val pload = MuxOR(io.op.tran, io.adata) |
            MuxOR(io.op.init, io.bdata)
```

**Operation**:
```
For each position (i,j):
  acc[i][j] = io.bdata[segment]
  
Loads one 256-bit vector into accumulator row/column
```

**Use case**: Initialize accumulator to bias values before convolution

#### 2. Transpose (tran)

**Purpose**: Load transposed data for outer-product setup

**Operation**:
```
For each position (i,j):
  acc[i][j] = io.adata[transposed_segment]
  
Prepares kernel data in column-major format
```

**Use case**: Transpose 3×3 kernel before convolution

#### 3. Convolution (conv)

**Purpose**: Perform outer-product MAC and accumulate

**Code** (VConvAlu.scala lines 74-83):
```scala
val dpa = Wire(Vec(tcnt, Vec(tcnt, UInt(32.W))))

for (i <- 0 until tcnt) {
  for (j <- 0 until tcnt) {
    val accum = MuxOR(io.op.conv, acc(i)(j))
    dpa(i)(j) := accum + VDot(io.op.conv,
        io.adata(i * 32 + 31, i * 32),  // A operand (kernel)
        io.bdata(j * 32 + 31, j * 32),  // B operand (input)
        io.abias, io.bbias, io.asign, io.bsign)
  }
}
```

**Matrix Multiply Concept** (VConvAlu.scala lines 49-54):
```
MatMul: Outer Product
     B B B B B B B B  (input data, 8 elements)
   ┌─────────────────┐
 A │ . . . . . . . . │
 A │ . . . . . . . . │
 A │ . . . . . . . . │  Each dot = MAC operation
 A │ . . . . . . . . │  Total: 8×8 = 64 MACs
 A │ . . . . . . . . │
 A │ . . . . . . . . │
 A │ . . . . . . . . │
 A │ . . . . . . . . │
   └─────────────────┘
 (kernel, 8 elements)
```

**Detailed Operation**:
```
For each position (i,j):
  acc[i][j] += VDot(kernel[i], input[j], bias_a, bias_b, sign_a, sign_b)
  
VDot performs 4 × INT8 multiplies + accumulate:
  result = (a[0]+bias_a)×(b[0]+bias_b) + 
           (a[1]+bias_a)×(b[1]+bias_b) + 
           (a[2]+bias_a)×(b[2]+bias_b) + 
           (a[3]+bias_a)×(b[3]+bias_b)
```

**Parallelism**:
- **64 VDot units** (one per accumulator element)
- Each VDot: **4 INT8 multiplies** = 4 MACs
- **Total**: 64 × 4 = **256 MACs per cycle**

#### 4. Writeback & Clear (wclr)

**Purpose**: Write accumulator to VRegfile and reset

**Code** (VConvAlu.scala lines 96-105):
```scala
for (i <- 0 until tcnt) {
  for (j <- 0 until tcnt) {
    val (si, sj) = Interleave(i, j)
    
    val aclr = io.op.clear || reset.asBool
    val conv = io.op.conv
    val load = (io.op.init || io.op.tran) && si.U === io.index
    
    when (aclr || conv || load) {
      acc(i)(j) := Mux(conv, dpa(i)(j),
                       pload(sj * 32 + 31, sj * 32))
    }
  }
}
```

**Operation**:
```
Write accumulator to v48-v55 (8 registers for 256-bit config)
Clear accumulator to 0
```

---

## VDot Unit - INT8 MAC Engine

**File**: `VDot.scala` (191 lines)

### Standard Conv2D Mode

**Function** (VDot.scala lines 23-59):
```scala
def apply(en: Bool, adata: UInt, bdata: UInt,
    abias: UInt, bbias: UInt, asign: Bool, bsign: Bool): UInt
```

**Operation**: 4 × INT8 multiplies + accumulate

**Detailed Logic**:
```
Input: adata[31:0] = [a3, a2, a1, a0]  (4 × 8-bit)
       bdata[31:0] = [b3, b2, b1, b0]  (4 × 8-bit)
       abias[8:0]  = bias for A
       bbias[8:0]  = bias for B
       asign       = signed/unsigned for A
       bsign       = signed/unsigned for B

For each lane i (0-3):
  1. Sign extend:
     as = adata[8*i+7] & asign  (MSB if signed)
     bs = bdata[8*i+7] & bsign
     
  2. Add bias:
     aval = Cat(as, adata[8*i+7:8*i]).asSInt + abias.asSInt  (10-bit)
     bval = Cat(bs, bdata[8*i+7:8*i]).asSInt + bbias.asSInt  (10-bit)
     
  3. Multiply:
     mul[i] = aval × bval  (20-bit)
     
  4. Accumulate:
     dotp = (mul[0] + mul[1]) + (mul[2] + mul[3])  (22-bit)
     
  5. Sign extend to 32-bit:
     result = Cat(sign_extend(dotp[21]), dotp)  (32-bit)
```

**Example**:
```
adata = 0x04030201  (a0=1, a1=2, a2=3, a3=4)
bdata = 0x01010101  (b0=1, b1=1, b2=1, b3=1)
abias = 0 (no bias)
bbias = 0
asign = 0 (unsigned)
bsign = 0

Lane 0: (1+0) × (1+0) = 1
Lane 1: (2+0) × (1+0) = 2
Lane 2: (3+0) × (1+0) = 3
Lane 3: (4+0) × (1+0) = 4

dotp = (1+2) + (3+4) = 10
result = 0x0000000A (10 in decimal)
```

### Depthwise Convolution Mode

**Function** (VDot.scala lines 62-147):
```scala
def apply(alu: Int, en: Bool, adata: Vec[UInt], bdata: Vec[UInt],
    scalar: UInt): (UInt, UInt)
```

**Purpose**: Accelerate depthwise convolution (3×3 kernel)

**Sparse Modes**:
```
Mode 0 (Dense): [n, n, n]
  adata[0] → kernel row 0
  adata[1] → kernel row 1
  adata[2] → kernel row 2
  
Mode 1 (Sparse): [n-1, n, n+1]
  Shift input by 1 element
  
Mode 2 (Sparse): [n, n+1, n+2]
  Shift input by 2 elements
```

**Operation**:
```
For each of 8 output channels (cnt=8):
  For each of 3 kernel positions:
    Perform 3 × INT8 MACs
    
  Total per channel: 3×3 = 9 MACs
  Total all channels: 8 × 9 = 72 MACs per VDot call
  
With 2 ALUs (alu=0, alu=1):
  Total: 72 × 2 = 144 MACs per cycle (depthwise mode)
```

---

## VConvCtrl - Control & Scheduling

**File**: `VConvCtrl.scala` (220 lines)

### Command Queue

**Structure** (VConvCtrl.scala lines 50-66):
```scala
class VConvCtrlCmdq extends Bundle {
  val conv   = Bool()  // convolution operation
  val init   = Bool()  // initialize accumulator
  val tran   = Bool()  // transpose operation
  val wclr   = Bool()  // writeback and clear
  val addr1  = UInt(6.W)  // kernel address
  val addr2  = UInt(6.W)  // input address
  val base2  = UInt(6.W)  // base input address
  val mode   = UInt(2.W)  // convolution mode
  val mark2  = UInt((vectorBits / 32).W)  // active mask
  val index  = UInt(log2Ceil(vectorBits / 32).W)  // current index
  val end    = UInt(log2Ceil(vectorBits / 32).W)  // end index
  val abias  = UInt(9.W)  // A operand bias
  val bbias  = UInt(9.W)  // B operand bias
  val asign  = Bool()  // A operand signed
  val bsign  = Bool()  // B operand signed
}
```

**Depth**: 4 entries (VConvCtrl.scala line 44)

### Instruction Decoding

**Supported Instructions** (VConvCtrl.scala lines 71-74):
```scala
val vcget  = in.op === e.vcget.U  // writeback & clear
val acset  = in.op === e.acset.U  // initialize
val actr   = in.op === e.actr.U   // transpose
val aconv  = in.op === e.aconv.U  // convolution
```

**Encoding** (from scalar value):
```
sv.data[1:0]   = mode (0=dense, 1/2=sparse)
sv.data[6:2]   = start index
sv.data[11:7]  = stop index
sv.data[20:12] = abias (9-bit)
sv.data[21]    = asign
sv.data[30:22] = bbias (9-bit)
sv.data[31]    = bsign
```

### Scoreboard Integration

**3-cycle pipeline tracking** (VConvCtrl.scala lines 172-179):
```scala
val vrfsb0 = io.vrfsb(63,0) | io.vrfsb(127,64)  // Current scoreboard
val vrfsb1 = RegInit(0.U(64.W))  // 1 cycle ago
val vrfsb2 = RegInit(0.U(64.W))  // 2 cycles ago
val vrfsb = vrfsb0 | vrfsb1 | vrfsb2  // Combined

vrfsb1 := vrfsb0
vrfsb2 := vrfsb1

val ready = (active & vrfsb) === 0.U  // Check dependencies
```

**Purpose**: Account for VRegfile write pipeline (2 cycles) + forwarding (1 cycle)

---

## Interleave Logic

**Purpose**: Reorder accumulator output to match VRegfile segment layout

**Function** (VConvAlu.scala lines 62-70):
```scala
def Interleave(i: Int, j: Int): (Int, Int) = {
  val interleave = Seq(0, 2, 1, 3)
  val rbase = i & ~3
  val rquad = i & 3
  val word  = j
  val si = rbase + interleave(word & 3)
  val sj = rquad * (tcnt / 4) + (word / 4)
  (si, sj)
}
```

**Example** (8×8 accumulator):
```
Accumulator layout (row-major):
  acc[0][0], acc[0][1], acc[0][2], acc[0][3], acc[0][4], ...
  
VRegfile layout (column-major, interleaved):
  v48[0] = acc[0][0]
  v48[1] = acc[1][0]
  v48[2] = acc[2][0]
  ...
  v49[0] = acc[0][2]  (note: interleaved, not acc[0][1])
  v49[1] = acc[1][2]
  ...
```

**Interleave pattern**: `[0, 2, 1, 3]` reorders columns for efficient memory layout

---

## Complete Operation Example

### Instruction: `vdwconv.b v48, v0, v4`

**Operation**: Depthwise 3×3 convolution, 8-bit, accumulate to v48-v55

### Phase 1: Initialize Accumulator

```
Instruction: acset v48, v0

VConvCtrl:
  op.init = 1
  addr1 = 0  (source: v0)
  
VConvAlu:
  Load v0 into accumulator
  acc[i][j] = v0[segment(i,j)]
```

### Phase 2: Transpose Kernel

```
Instruction: actr v48, v0

VConvCtrl:
  op.tran = 1
  addr1 = 0  (kernel)
  
VRegfile:
  io.transpose.valid = 1
  io.transpose.addr = 0
  io.transpose.data = transposed v0
  
VConvAlu:
  Load transposed kernel
  acc[i][j] = transpose(v0)[segment(i,j)]
```

### Phase 3: Convolution (4 strips)

**Strip 0**: Process v4

```
Instruction: aconv v48, v0, v4

VConvCtrl:
  op.conv = 1
  addr1 = 0  (kernel)
  addr2 = 4  (input strip 0)
  index = 0
  end = 3
  
VConvAlu:
  For each (i,j) in 8×8:
    acc[i][j] += VDot(kernel[i], input[j], bias_a, bias_b, sign_a, sign_b)
    
  256 MACs executed in parallel! ✅
```

**Strips 1-3**: Repeat for v5, v6, v7

```
Each strip:
  addr2 increments: 4 → 5 → 6 → 7
  index increments: 0 → 1 → 2 → 3
  
Total MACs: 256 × 4 strips = 1024 MACs
```

### Phase 4: Writeback & Clear

```
Instruction: vcget v48

VConvCtrl:
  op.wclr = 1
  
VConvAlu:
  Write accumulator to VRegfile
  Clear accumulator to 0
  
VRegfile:
  v48[0:7] = acc[0][0], acc[1][0], ..., acc[7][0]  (column 0)
  v49[0:7] = acc[0][2], acc[1][2], ..., acc[7][2]  (column 2, interleaved)
  ...
  v55[0:7] = acc[0][7], acc[1][7], ..., acc[7][7]  (column 7)
```

### Performance

**Total cycles**: ~20-25 cycles (all phases)  
**Total MACs**: 1024  
**Effective throughput**: **41-51 MACs/cycle** (sustained)  
**Peak throughput**: **256 MACs/cycle** (during conv phase)

---

## Register File Integration

### Reserved Registers

**Accumulator Output**: v48-v63 (16 registers)

**Allocation** (for 256-bit config):
```
v48-v55: Accumulator writeback (8 registers × 256-bit)
v56-v63: Reserved for future use or larger VLEN
```

**Constraints** (VRegfile.scala lines 363-371):
```scala
// No writes to v48-v63 during convolution
val convmaska = 0xffff.U << 48.U
assert(!(convClear0 && (writeCurr & convmaska) =/= 0.U))

// No reads of v48-v63 during convolution accumulation
assert(!(convRead0 && writeCurr(io.conv.addr2)))
```

### Interface Signals

**From VConvCtrl to VRegfile** (VRegfile.scala lines 92-109):
```scala
io.conv.valid  // Operation valid
io.conv.ready  // Scoreboard ready
io.conv.op.conv  // Convolution accumulate
io.conv.op.init  // Initialize accumulator
io.conv.op.tran  // Transpose load
io.conv.op.wclr  // Writeback and clear
io.conv.addr1  // Kernel address (transpose source)
io.conv.addr2  // Input address (conv source)
io.conv.mode   // Convolution mode
io.conv.index  // Current strip index
io.conv.abias  // A operand bias
io.conv.bbias  // B operand bias
io.conv.asign  // A operand sign
io.conv.bsign  // B operand sign
```

---

## Performance Analysis

### Throughput

| Operation | MACs/Cycle | Utilization |
|-----------|------------|-------------|
| **Conv2D (outer-product)** | 256 | 100% (peak) |
| **Depthwise Conv** | 144 | 56% |
| **Init/Transpose** | 0 | Setup phase |
| **Writeback** | 0 | Commit phase |

**Sustained Throughput** (typical workload):
```
Phases:
  Init:      1 cycle  (0 MACs)
  Transpose: 1 cycle  (0 MACs)
  Conv:      4 cycles (256 MACs/cycle × 4 = 1024 MACs)
  Writeback: 1 cycle  (0 MACs)
  
Total: 7 cycles, 1024 MACs
Average: 1024 / 7 = 146 MACs/cycle
```

### Comparison with Software

**Software (scalar core)**:
```
3×3 convolution, 8 channels:
  9 multiplies × 8 channels = 72 operations
  Scalar: ~3 cycles/multiply = 216 cycles
  
Hardware (VConvAlu):
  4 strips × 1 cycle/strip = 4 cycles (conv phase only)
  
Speedup: 216 / 4 = 54× faster! ✅
```

### Bottlenecks

1. **Memory bandwidth**: Loading input data (v4-v7)
2. **Setup overhead**: Init + transpose (2 cycles)
3. **Writeback**: Clearing accumulator (1 cycle)

**Optimization**: Pipeline multiple convolutions to amortize setup cost

---

## Design Insights

### Why Outer-Product?

**Alternative**: Inner-product (dot-product per output)
```
Inner-product:
  For each output element:
    result = Σ(kernel[i] × input[i])
  Requires sequential accumulation
```

**Outer-product** (chosen):
```
For all output elements simultaneously:
  result[i][j] = kernel[i] × input[j]
  All 64 products computed in parallel!
```

**Benefit**: **64× parallelism** vs. sequential

### Why 8×8 Array?

**Trade-off**:
- Larger array (16×16): 4× more MACs, but 4× more area/power
- Smaller array (4×4): 1/4 MACs, but underutilizes 256-bit VLEN

**8×8 is optimal** for 256-bit vector width:
- Matches 8 × 32-bit elements per vector
- Efficient outer-product (8 kernel × 8 input = 64 products)
- Reasonable area/power budget

### Why Interleave?

**Problem**: Accumulator row-major, VRegfile column-major

**Solution**: Interleave pattern `[0, 2, 1, 3]` reorders output

**Benefit**: Efficient writeback without extra transpose

---

## Summary

The Convolution Accelerator achieves **256 MACs/cycle peak** through:

✅ **Outer-product parallelism**: 64 simultaneous MACs  
✅ **Integrated accumulator**: 2KB on-chip storage  
✅ **VDot units**: 4 INT8 multiplies per unit  
✅ **Bias & sign support**: Flexible quantization  
✅ **Sparse modes**: Optimized depthwise convolution  
✅ **VRegfile integration**: Zero-copy accumulator writeback  

**Next**: [VAlu - Vector ALU](valu.md)

---

**Source Files**:
- `coral/codes/coralnpu/hdl/chisel/src/coralnpu/vector/VConvAlu.scala` (127 lines)
- `coral/codes/coralnpu/hdl/chisel/src/coralnpu/vector/VConvCtrl.scala` (220 lines)
- `coral/codes/coralnpu/hdl/chisel/src/coralnpu/vector/VDot.scala` (191 lines)

