# VLdSt Addressing Modes and Stripmining

## Overview

VLdSt supports **two primary addressing modes** optimized for CNN memory patterns:
1. **Unit-stride**: Contiguous memory access (most common)
2. **Constant-stride**: Fixed offset between elements (for dilated convolution, matrix transpose)

Additionally, **automatic stripmining** subdivides large transfers into hardware-manageable chunks.

---

## Addressing Modes

### 1. Unit-Stride (Contiguous Access)

**Definition**: Load/store consecutive memory locations with no gaps.

**Encoding**:
```scala
// VLdSt.scala line 110
val stride = in.f2(1)  // f2[1] = 0 for unit-stride
```

**Address Pattern**:
```
addr[0] = base
addr[1] = base + 16  (register size)
addr[2] = base + 32
addr[3] = base + 48
```

**Assembly**:
```assembly
vld.w v0, (x10)              # Unit-stride, single register
vld.w v0, (x10), 0, 64       # Unit-stride, 64 bytes (4 registers)
```

**Use Cases**:
- Sequential array access
- Dense tensor loading
- Contiguous filter weights
- Activation maps

**Performance**: ✅ **Best** (cache-friendly, potential burst transfers)

**Example**:
```c
float activations[64];  // 64 floats = 256 bytes

// Load 64 bytes (16 floats, 4 registers)
vld_w(v0, activations);  // m=1, unit-stride

// Memory access pattern:
// v0 ← activations[0:15]   (addr 0x00-0x0F)
// v1 ← activations[16:31]  (addr 0x10-0x1F)
// v2 ← activations[32:47]  (addr 0x20-0x2F)
// v3 ← activations[48:63]  (addr 0x30-0x3F)
```

---

### 2. Constant-Stride (Non-Contiguous Access)

**Definition**: Load/store with fixed offset between consecutive accesses.

**Encoding**:
```scala
// VLdSt.scala lines 110, 132
val stride = in.f2(1)  // f2[1] = 1 for strided
out.offset := Mux(stride, data(31,0), ...)  // Use rs2 value
```

**Address Pattern**:
```
addr[0] = base
addr[1] = base + stride
addr[2] = base + stride*2
addr[3] = base + stride*3
```

**Assembly**:
```assembly
li x11, 32                   # Stride = 32 bytes
vld.w v0, (x10), x11         # Strided load
```

**Stride Calculation** (VLdSt.scala lines 119-121):
```scala
val data = MuxOR(in.sz(0), in.sv.data) |          // 8-bit: × 1
           MuxOR(in.sz(1), Cat(in.sv.data, 0.U(1.W))) |  // 16-bit: × 2
           MuxOR(in.sz(2), Cat(in.sv.data, 0.U(2.W)))    // 32-bit: × 4
```

**Meaning**: Stride is specified in **elements**, hardware converts to **bytes**.

**Example**:
```assembly
li x11, 8                    # Stride = 8 elements
vld.w v0, (x10), x11         # 32-bit elements

# Hardware calculation:
# stride_bytes = 8 × 4 = 32 bytes
# addr[0] = x10
# addr[1] = x10 + 32
# addr[2] = x10 + 64
# addr[3] = x10 + 96
```

**Use Cases**:
- Dilated convolution (stride > 1)
- Matrix column access (row-major layout)
- Downsampling (every N-th element)
- Interleaved data (skip unwanted fields)

**Performance**: ⚠️ **Moderate to Poor** (cache-unfriendly if stride > cache line)

---

## Stride Calculation Details

### Element-to-Byte Conversion

**Code** (VLdSt.scala lines 119-121):
```scala
val data = MuxOR(in.sz(0), in.sv.data) |          // sz=001: byte
           MuxOR(in.sz(1), Cat(in.sv.data, 0.U(1.W))) |  // sz=010: halfword
           MuxOR(in.sz(2), Cat(in.sv.data, 0.U(2.W)))    // sz=100: word
```

**Translation Table**:

| Element Size | sz Encoding | Multiplier | Example |
|--------------|-------------|------------|---------|
| 8-bit (byte) | 0b001 | ×1 | stride=10 → 10 bytes |
| 16-bit (halfword) | 0b010 | ×2 | stride=10 → 20 bytes |
| 32-bit (word) | 0b100 | ×4 | stride=10 → 40 bytes |

**Example Calculation**:
```
Instruction: vld.h v0, (x10), x11
  x11 = 5 (stride in 16-bit elements)
  sz = 0b010 (16-bit)

Calculation:
  data = Cat(5, 0.U(1.W)) = 5 << 1 = 10 bytes
  offset = 10 bytes

Address sequence:
  Transaction 0: addr = x10 + 0  = x10
  Transaction 1: addr = x10 + 10 = x10 + 0x0A
  Transaction 2: addr = x10 + 20 = x10 + 0x14
  Transaction 3: addr = x10 + 30 = x10 + 0x1E
```

---

## Stripmining (Automatic Loop Subdivision)

### Purpose

**Problem**: Vector registers are 128 bits (16 bytes), but operations may need to process **multiple registers** (e.g., 64 bytes = 4 registers).

**Solution**: Hardware automatically **subdivides** large transfers into 16-byte chunks and **iterates** until complete.

### Stripmining Control

**m Bit** (VLdSt.scala line 117):
```scala
val limit = Mux(in.m, maxvlbm, maxvlb)
// m=0: Single register (16 bytes)
// m=1: 4 registers (64 bytes)
```

**Encoding**:
- `m=0`: Single-register operation (no stripmining)
- `m=1`: Multi-register operation (4-register stripmining)

### Iteration Logic

**Fin Function** (Initial Setup, VLdSt.scala lines 108-141):
```scala
def Fin(in: VDecodeBits): VLdStCmdq = {
  val limit = Mux(in.m, maxvlbm, maxvlb)
  // maxvlb  = 16 bytes (1 register)
  // maxvlbm = 64 bytes (4 registers)
  
  out.addr := in.sv.addr           // Start address
  out.offset := stride_value       // Address increment
  out.remain := total_bytes        // Bytes left to transfer
  out.vd := in.vd                  // Current destination register
  out.last := !in.m && ...         // Last flag
  
  out
}
```

**Fout Function** (Iteration Update, VLdSt.scala lines 143-172):
```scala
def Fout(in: VLdStCmdq, m: Bool, step: UInt, valid: Bool): (VLdStCmdq, Bool) = {
  val fmaxvlb = Mux(vstq, maxvlb >> 2, maxvlb)  // 4 or 16 bytes
  
  // Update for next iteration
  out.vd.addr := in.vd.addr + 1.U       // Next register
  out.addr := in.addr + in.offset       // Next address
  out.remain := in.remain - fmaxvlb     // Decrement remaining
  
  // Check if done
  val last = !m || step === 3.U  // 4 iterations for m=1
  
  (out, last)
}
```

### Stripmining Example

**Instruction**:
```assembly
vld.w v8, (x10), x11, x12
# x10 = 0x10000000 (base address)
# x11 = 32 (stride in bytes, assuming 32-bit elements → 8 elements)
# x12 = 48 (total length in bytes)
# m = 1 (stripmining enabled)
```

**Execution Trace**:

```
Fin(): Initialize
  addr = 0x10000000
  offset = 32
  remain = 48
  vd = v8
  last = false

Iteration 0 (step=0):
  DBus transaction: addr=0x10000000, size=16
  Load 16 bytes → v8
  
  Fout(): Update
    addr = 0x10000020 (0x10000000 + 32)
    vd = v9 (v8 + 1)
    remain = 32 (48 - 16)
    last = false

Iteration 1 (step=1):
  DBus transaction: addr=0x10000020, size=16
  Load 16 bytes → v9
  
  Fout(): Update
    addr = 0x10000040 (0x10000020 + 32)
    vd = v10 (v9 + 1)
    remain = 16 (32 - 16)
    last = false

Iteration 2 (step=2):
  DBus transaction: addr=0x10000040, size=16
  Load 16 bytes → v10
  
  Fout(): Update
    addr = 0x10000060 (0x10000040 + 32)
    vd = v11 (v10 + 1)
    remain = 0 (16 - 16)
    last = true (step=2, remain=0)

Iteration 3 (step=3):
  NOT EXECUTED (remain=0, last=true in previous iteration)

Result:
  v8  ← mem[0x10000000:0x1000000F]
  v9  ← mem[0x10000020:0x1000002F]
  v10 ← mem[0x10000040:0x1000004F]
  Total: 48 bytes loaded with stride=32
```

---

## Special Case: vstq (Quadrant Store)

**Purpose**: Store 4 registers with **4-byte stride** (not 16-byte).

**Why**: Convolution output generates 4 registers, but each register only has **4 bytes of valid data** (32 bits per output element).

**Encoding** (VLdSt.scala lines 132, 149):
```scala
out.offset := Mux(in.op === e.vstq.U, maxvlb >> 2, maxvlb)
// vstq: offset = 16 >> 2 = 4 bytes
// others: offset = 16 bytes

val fmaxvlb = Mux(in.op === e.vstq.U, maxvlb >> 2, maxvlb)
// vstq: chunk size = 4 bytes (not 16!)
```

**Iteration Pattern** (VLdSt.scala lines 161-162, 169):
```scala
out.vd.addr := Mux(vstq && step(1,0) =/= 3.U, 
                   in.vd.addr,           // Keep same register
                   in.vd.addr + 1.U)     // Increment register

out.quad := Mux(in.op === e.vstq.U, step + 1.U, 0.U)  // Track quadrant
```

**Execution**:
```
vstq v48, (x10)

Step 0: Store v48[0:3]   → addr=0x10000000, quad=0
Step 1: Store v48[4:7]   → addr=0x10000004, quad=1
Step 2: Store v48[8:11]  → addr=0x10000008, quad=2
Step 3: Store v48[12:15] → addr=0x1000000C, quad=3, vd→v49

Step 4: Store v49[0:3]   → addr=0x10000010, quad=0
Step 5: Store v49[4:7]   → addr=0x10000014, quad=1
Step 6: Store v49[8:11]  → addr=0x10000018, quad=2
Step 7: Store v49[12:15] → addr=0x1000001C, quad=3, vd→v50

... (repeat for v50, v51)

Step 15: Store v51[12:15] → addr=0x1000003C, last=true

Total: 16 iterations, 64 bytes, 4 registers
Memory layout: Contiguous 64 bytes (4 bytes per register slice)
```

---

## Swizzle Logic (Misalignment Handling)

### Purpose

**Problem**: Memory addresses may not be aligned to 16-byte boundaries, but DBus is 128-bit (16-byte) wide.

**Solution**: **Swizzle** (rotate) data to align with bus address.

### Swizzle Function

**Implementation** (VLdSt.scala lines 66-83):
```scala
def Swizzle(positive: Boolean, size: Int, addr: UInt, data: UInt): UInt = {
  val bytes = 16  // 128-bit bus = 16 bytes
  val index = addr(3, 0)  // Lower 4 bits = byte offset
  
  // Split data into bytes
  val datain = Wire(Vec(bytes, UInt(size.W)))
  for (i <- 0 until bytes) {
    datain(i) := data(size * i + (size - 1), size * i)
  }
  
  // Rotate bytes
  val dataout = Wire(Vec(bytes, UInt(size.W)))
  for (i <- 0 until bytes) {
    val idx = if (positive) i.U + index else i.U - index
    dataout(i) := VecAt(datain, idx)  // Circular indexing
  }
  
  dataout.asUInt
}
```

**Parameters**:
- `positive`: Rotation direction (true=left, false=right)
- `size`: Byte size (always 8 for byte-level rotation)
- `addr`: Memory address (lower 4 bits used)
- `data`: Input data (128 bits)

**Usage**:

**Store Path** (VLdSt.scala line 263):
```scala
data.io.in.bits.wdata := Swizzle(false, 8, rdataAshf, io.read.data)
// Rotate RIGHT to align with bus address
```

**Load Path** (VLdSt.scala line 315):
```scala
io.write.data := Swizzle(true, 8, wrega.io.out.bits.addr, wregd.io.out.bits)
// Rotate LEFT to restore register alignment
```

### Swizzle Example

**Scenario**: Store v5 to address 0x10000004 (4-byte offset)

**Input**:
```
Register v5: [D15 D14 D13 D12 D11 D10 D9 D8 D7 D6 D5 D4 D3 D2 D1 D0]
             (16 bytes, byte-indexed)

Address: 0x10000004
  Lower 4 bits: 0x4 (index = 4)
```

**Swizzle Operation** (rotate right by 4):
```
Input:  [D15 D14 D13 D12 D11 D10 D9 D8 D7 D6 D5 D4 D3 D2 D1 D0]
Index = 4

Rotate right by 4:
  dataout[0] = datain[0 - 4 mod 16] = datain[12] = D12
  dataout[1] = datain[1 - 4 mod 16] = datain[13] = D13
  ...
  dataout[11] = datain[11 - 4 mod 16] = datain[7] = D7
  dataout[12] = datain[12 - 4 mod 16] = datain[8] = D8  (wraps)
  ...

Output: [D11 D10 D9 D8 D7 D6 D5 D4 D3 D2 D1 D0 D15 D14 D13 D12]
```

**Bus Transaction**:
```
Address: 0x10000000 (aligned to 16-byte boundary)
Data:    [D11 D10 D9 D8 D7 D6 D5 D4 D3 D2 D1 D0 X X X X]
Mask:    [1   1   1  1  1  1  1  1  1  1  1  1  0 0 0 0]
         (bytes 0-3 masked, bytes 4-15 valid)

Memory result:
  0x10000000: [original data, unchanged]
  0x10000004: D0
  0x10000005: D1
  ...
  0x1000000F: D11
```

**Performance**: Swizzle is **combinational logic** (0 cycle penalty) ✅

---

## Address Generation Performance

### Best Case (Unit-Stride, Aligned)

**Scenario**:
```assembly
vld.w v0, (x10)    # x10 = 0x10000000 (aligned)
```

**Latency**:
- Address generation: 0 cycles (combinational)
- Swizzle: 0 cycles (not needed, aligned)
- DBus transaction: 1 cycle (best case, cache hit)
- **Total: 1 cycle per transaction**

---

### Worst Case (Strided, Misaligned, Cache Miss)

**Scenario**:
```assembly
li x11, 128
vld.w v0, (x10), x11    # x10 = 0x10000003 (misaligned), stride=128
```

**Latency**:
- Address generation: 0 cycles (combinational)
- Swizzle: 0 cycles (combinational)
- DBus transaction: 50-200 cycles (cache miss, DRAM access)
- **Total: 50-200 cycles per transaction**

**Cache Impact**:
- Stride=128 → Each transaction hits different cache line
- 100% cache miss rate if stride > cache size
- Memory bandwidth bottleneck

---

## Addressing Mode Comparison

| Mode | Cache Locality | Bandwidth Efficiency | Use Case |
|------|----------------|---------------------|----------|
| **Unit-stride** | ✅ Excellent | ✅ Excellent (burst) | Dense arrays, CNNs |
| **Small stride** (< cache line) | ✅ Good | ⚠️ Moderate | Dilated conv (stride=2-4) |
| **Large stride** (> cache line) | ❌ Poor | ❌ Poor | Matrix transpose, downsampling |
| **vstq** | ✅ Excellent | ✅ Excellent | Convolution output |

---

## Software Optimization Tips

### Tip 1: Prefer Unit-Stride

```c
// BAD: Strided access ❌
for (int i = 0; i < N; i += 4) {
    vld_w(v0, &data[i]);  // Stride = 4 elements = 16 bytes
}

// GOOD: Unit-stride ✅
vld_w(v0, data);  // Load contiguously
```

---

### Tip 2: Align Data to 16-Byte Boundaries

```c
// BAD: Misaligned ❌
float *data = malloc(256);  // May not be aligned

// GOOD: Aligned ✅
float *data = aligned_alloc(16, 256);  // 16-byte aligned
```

**Benefit**: Avoids swizzle overhead (though it's combinational, alignment helps cache)

---

### Tip 3: Minimize Stride

```c
// BAD: Large stride ❌
for (int col = 0; col < COLS; col++) {
    vld_w(v0, &matrix[0][col]);  // Stride = row_size (large)
}

// GOOD: Transpose data first ✅
transpose_matrix(matrix);
for (int row = 0; row < ROWS; row++) {
    vld_w(v0, &matrix[row][0]);  // Unit-stride
}
```

---

### Tip 4: Use vstq for Convolution Output

```c
// BAD: 4 separate stores ❌
vst_w(v48, output + 0);
vst_w(v49, output + 16);
vst_w(v50, output + 32);
vst_w(v51, output + 48);

// GOOD: Single vstq ✅
vstq(v48, output);  // Stores v48-v51 efficiently
```

---

## Summary

**Supported Addressing Modes**:
- ✅ Unit-stride (contiguous)
- ✅ Constant-stride (fixed offset)
- ✅ Automatic stripmining (4-register chunks)
- ✅ Swizzle for misalignment (0-cycle penalty)
- ✅ Special vstq for convolution

**NOT Supported**:
- ❌ Indexed (gather/scatter)
- ❌ Segment (multi-field structs)
- ❌ Variable stride

**Performance Keys**:
- Unit-stride: Best (cache-friendly)
- Small stride: Good (< cache line)
- Large stride: Poor (cache-unfriendly)
- Swizzle: Free (combinational)

---

**Next**: [Microarchitecture](microarchitecture.md) - Pipeline and control logic

---

**Source**: `VLdSt.scala` lines 66-83 (swizzle), 108-172 (stripmining)

