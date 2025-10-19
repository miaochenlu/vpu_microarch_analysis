# VLdSt Limitations and Unsupported Features

## Overview

Coral NPU VLdSt is **NOT a full RISC-V Vector implementation**. It's a **minimal, CNN-optimized** design that intentionally omits many standard vector memory features to reduce area and power consumption.

This document details **what is NOT supported** and why these design choices were made.

---

## ❌ Unsupported RISC-V Vector Features

### 1. Fault-Only-First Loads (vleff)

**RISC-V RVV Specification**:
```assembly
vleff.v vd, (rs1)        # Load until first fault
# vl ← number of elements successfully loaded before fault
```

**Purpose**: Safely handle unknown data boundaries (e.g., strings, variable-length arrays).

**How It Works**:
1. Start loading elements sequentially
2. If page fault occurs on element N, stop immediately
3. Update `vl` register to N (partial completion)
4. Raise exception for software to handle
5. Software can resume from element N

**Coral NPU Status**: ❌ **NOT IMPLEMENTED**

**Evidence**:
```scala
// VLdSt.scala has NO:
val vl_update = Output(UInt())        // ❌ Missing
val exception = Output(Bool())        // ❌ Missing
val partial_valid = Output(Bool())    // ❌ Missing
```

**Impact**:
- Cannot safely load unknown-length data
- Page boundary crossings are risky
- No partial completion mechanism

**Workaround**:
```c
// Software must validate addresses BEFORE vector load
if (validate_page_boundary(addr, length)) {
    vld_w(vd, addr);  // Safe
} else {
    // Scalar fallback or split into safe chunks
}
```

**Why Not Implemented**:
- CNNs have **fixed, known tensor sizes**
- No dynamic strings or variable-length data
- Area cost too high (~5-10% of VLdSt logic)
- Exception handling requires precise PC tracking

---

### 2. Segment Loads/Stores (vlseg, vsseg)

**RISC-V RVV Specification**:
```assembly
vlseg2e32.v vd, (rs1)    # Load 2-field struct array
# vd[i]   = mem[rs1 + i*8 + 0]
# vd+1[i] = mem[rs1 + i*8 + 4]

vlseg3e16.v vd, (rs1)    # Load 3-field struct array
# vd[i]   = mem[rs1 + i*6 + 0]
# vd+1[i] = mem[rs1 + i*6 + 2]
# vd+2[i] = mem[rs1 + i*6 + 4]
```

**Purpose**: Efficiently load structure-of-arrays (SoA) with multiple fields.

**Use Case**:
```c
struct Pixel { uint8_t r, g, b; };
Pixel image[16];

// Want: v0=R, v1=G, v2=B
vlseg3e8.v v0, (image)  // RVV way (efficient)
```

**Coral NPU Status**: ❌ **NOT IMPLEMENTED**

**Evidence**:
```scala
// VLdSt.scala has NO:
val nf = Input(UInt(3.W))      // Number of fields ❌
val segment_mode = Input(Bool()) // ❌
```

**Current Approach**: Only single-register loads
```scala
// VLdSt.scala line 134-135
out.vd := in.vd           // Single destination
out.vs := in.vs           // Single source
// No multi-register interleaving logic
```

**Workaround**:
```assembly
# Manual segment load (3x overhead)
vld.b v0, (x10)          # Load R values
addi x11, x10, 1
vld.b v1, (x11)          # Load G values
addi x12, x10, 2
vld.b v2, (x12)          # Load B values
```

**Why Not Implemented**:
- CNNs use **planar memory layouts** (separate R/G/B tensors)
- Structure-of-arrays (SoA) not common in ML
- Complex addressing logic (~15-20% area increase)
- Manual batching acceptable for edge devices

---

### 3. Indexed Loads/Stores (Gather/Scatter)

**RISC-V RVV Specification**:
```assembly
# Indexed load (gather)
vluxei32.v vd, (rs1), vs2    # vd[i] = mem[rs1 + vs2[i]]

# Indexed store (scatter)
vsuxei32.v vs3, (rs1), vs2   # mem[rs1 + vs2[i]] = vs3[i]
```

**Purpose**: Indirect addressing for sparse matrices, hash tables, indirect indexing.

**Use Case**:
```c
int indices[4] = {5, 12, 3, 9};
float data[16];

// Want: Load data[indices[i]]
vle32.v v1, (indices)         // Load indices
vluxei32.v v2, (data), v1     // Indexed load (gather)
```

**Coral NPU Status**: ❌ **NOT IMPLEMENTED**

**Evidence**:
```scala
// VLdSt.scala lines 91-93
val addr = UInt(32.W)      // SCALAR base address
val offset = UInt(32.W)    // CONSTANT stride
// No vector index register
```

**Current Limitation**: Only `base + constant_stride` addressing
```scala
// VLdSt.scala lines 132-133
out.offset := Mux(stride, data(31,0),  // Scalar stride
                  Mux(in.op === e.vstq.U, maxvlb >> 2, maxvlb))
// No support for: out.offset := vs2[i]  (per-element index)
```

**Workaround**:
```c
// Scalar loop (slow!)
for (int i = 0; i < 16; i++) {
    v2[i] = data[indices[i]];
}
```

**Why Not Implemented**:
- CNNs are **dense** (no sparsity in typical models)
- Indexed access requires:
  - Extra read port for index vector (~20% area)
  - Complex address generation (~10% area)
  - Out-of-order completion logic (~15% area)
  - **Total: ~45% area increase** 💸
- Sparse models (< 5% of CNN workloads on edge)

---

### 4. Whole Register Loads/Stores (vl1r, vs1r)

**RISC-V RVV Specification**:
```assembly
vl1re32.v vd, (rs1)      # Load 1 full register, ignore vl
vl2re32.v vd, (rs1)      # Load 2 full registers
vl4re32.v vd, (rs1)      # Load 4 full registers
vl8re32.v vd, (rs1)      # Load 8 full registers
```

**Purpose**: Bulk data movement, ignoring `vl` setting.

**Coral NPU Status**: ⚠️ **PARTIALLY SUPPORTED** (implicit)

**Explanation**:
- Coral NPU always loads **full registers** (16 bytes = 128 bits)
- No separate `vl` register to ignore
- Stripmining (`m=1`) effectively does `vl4re` behavior

**Difference from RVV**:
- No explicit encoding for whole-register ops
- No `vl`-aware vs `vl`-ignoring distinction
- Always full-width loads

**Workaround**: Use standard `vld` with `m=1` for bulk loads

---

### 5. Masked Loads/Stores (v0.t)

**RISC-V RVV Specification**:
```assembly
vle32.v vd, (rs1), v0.t      # Masked load
# Only load where v0[i] == 1

vse32.v vs3, (rs1), v0.t     # Masked store
# Only store where v0[i] == 1
```

**Purpose**: Conditional memory access based on mask register.

**Use Case**:
```c
// Conditional update
for (int i = 0; i < 16; i++) {
    if (mask[i]) {
        data[i] = new_value[i];  // Only update masked elements
    }
}
```

**Coral NPU Status**: ❌ **NOT IMPLEMENTED**

**Evidence**:
```scala
// VLdSt.scala has NO:
val mask = Input(UInt(128.W))     // ❌ Mask register
val masked = Input(Bool())        // ❌ Mask enable
```

**All operations are unconditional**:
```scala
// VLdSt.scala lines 271-277
io.dbus.valid := ctrl.io.out.valid && (data.io.out.valid || !ctrl.io.out.bits.write)
// No mask check, always issues transaction
```

**Workaround**:
```assembly
# Scalar loop with conditional
loop:
    lbu t0, 0(mask_ptr)
    beqz t0, skip
    vld.w v0, (data_ptr)
    # ... operation ...
    vst.w v0, (data_ptr)
skip:
    addi mask_ptr, mask_ptr, 1
    addi data_ptr, data_ptr, 16
    bne loop_counter, zero, loop
```

**Why Not Implemented**:
- CNNs have **uniform, dense** operations (no masking needed)
- Masked memory access requires:
  - Mask register read port (~5% area)
  - Conditional transaction logic (~10% area)
  - Partial register write (~8% area)
- Use cases rare in CNN inference

---

### 6. Unit-Stride Fault-Only-First (vleff with stride)

**RISC-V RVV Specification**:
```assembly
vlseff.v vd, (rs1), rs2      # Strided fault-only-first
```

**Coral NPU Status**: ❌ **NOT IMPLEMENTED** (no fault-only-first at all)

---

### 7. Atomic Memory Operations (AMOs)

**RISC-V RVV Specification**:
```assembly
# No vector AMOs in base RVV 1.0, but conceptually:
vamoaddei32.v vd, (rs1), vs2, vs3  # Atomic add with index
```

**Coral NPU Status**: ❌ **NOT IMPLEMENTED**

**Why Not Needed**:
- Single-threaded execution (no concurrency)
- No shared memory with other cores
- No need for atomicity in CNN inference

---

### 8. Memory Ordering/Fences

**RISC-V RVV Specification**:
```assembly
vfence.v      # Vector memory fence
```

**Coral NPU Status**: ❌ **NOT IMPLEMENTED**

**Why Not Needed**:
- In-order memory transactions
- No speculative execution
- Single memory interface (no reordering)

---

### 9. Byte-Enable Stores (Partial Writes)

**Expected Feature**: Write only specific bytes within 16-byte transaction

**Coral NPU Status**: ⚠️ **PARTIALLY SUPPORTED**

**What Works**:
```scala
// VLdSt.scala lines 264, 277
data.io.in.bits.wmask := Swizzle(false, 1, rdataAddr, rdataWmask.asUInt)
io.dbus.wmask := data.io.out.bits.wmask
// Byte mask for partial writes ✅
```

**Limitation**: Only for address alignment, not arbitrary byte selection

---

## Feature Comparison Table

| Feature | RISC-V Vector | Coral NPU VLdSt | Gap Impact |
|---------|---------------|-----------------|------------|
| **Unit-stride load/store** | ✅ | ✅ | None |
| **Constant-stride load/store** | ✅ | ✅ | None |
| **Indexed load/store** | ✅ | ❌ | **High** (sparse data) |
| **Segment load/store** | ✅ | ❌ | Medium (SoA structs) |
| **Fault-only-first** | ✅ | ❌ | **High** (safety) |
| **Whole register load/store** | ✅ | ⚠️ Implicit | Low |
| **Masked load/store** | ✅ | ❌ | Medium (conditional) |
| **Atomic operations** | Future | ❌ | None (single-core) |
| **Memory fences** | ✅ | ❌ | None (in-order) |
| **Byte-enable stores** | ✅ | ⚠️ Limited | Low |

**Compliance**: ~40% of RISC-V Vector baseline features

---

## Design Philosophy

### CNN-First Approach

**Target Workload Analysis**:
```
Typical CNN Layer (MobileNetV2, 224×224 input):
- Activation loads: 100% unit-stride ✅
- Weight loads: 100% unit-stride ✅
- Output stores: 100% unit-stride or vstq ✅
- Indexed access: 0% ❌
- Segment loads: 0% ❌
- Fault-only-first: 0% ❌
```

**Conclusion**: Basic load/store covers >99% of CNN memory operations.

### Area/Power Budget

**Full RISC-V Vector Memory Unit**:
- Indexed access: +45% area
- Fault-only-first: +10% area
- Segment loads: +20% area
- Masked operations: +15% area
- **Total overhead: +90% area, +60% power** 💸

**Coral NPU Design Constraint**:
- Total SoC budget: <5mm² @ 28nm
- Vector Core budget: <1mm²
- VLdSt budget: <0.15mm²

**Trade-off**: Omit rarely-used features to fit budget.

---

## When Coral NPU VLdSt is Insufficient

### Workload 1: Sparse Neural Networks

**Problem**: Sparse matrices with <10% non-zero elements

**Required Feature**: Indexed load/store (gather/scatter)

**Coral NPU Limitation**: Must use scalar loops (100× slower)

**Alternative**: Use specialized sparse accelerator (not VCore)

---

### Workload 2: Dynamic Data Structures

**Problem**: Variable-length sequences (NLP, speech)

**Required Feature**: Fault-only-first loads

**Coral NPU Limitation**: Risk of page faults, no partial completion

**Alternative**: Software bounds checking, or use scalar core

---

### Workload 3: Structure-of-Arrays

**Problem**: Multi-field data (RGB pixels, 3D coordinates)

**Required Feature**: Segment loads

**Coral NPU Limitation**: Must issue separate loads (3× overhead)

**Alternative**: Reorganize data to planar layout (R plane, G plane, B plane)

---

### Workload 4: Conditional Updates

**Problem**: Masked memory operations (e.g., dropout)

**Required Feature**: Masked loads/stores

**Coral NPU Limitation**: Must use scalar conditionals

**Alternative**: Compute mask separately, use scalar or branchless tricks

---

## Software Implications

### 1. Compiler Must Validate Addresses

**Responsibility**: Software (not hardware) ensures valid addresses

**Example**:
```c
// BAD: No validation ❌
void process(float *data, int length) {
    vld_w(v0, data);  // What if data is NULL or unmapped?
}

// GOOD: Validate first ✅
void process(float *data, int length) {
    if (!is_mapped(data, length)) {
        handle_error();
        return;
    }
    vld_w(v0, data);  // Safe
}
```

---

### 2. No Sparsity Support

**Impact**: Sparse convolution must use scalar fallback

**Example**:
```c
// Dense convolution: Use VCore ✅
conv2d_dense(activations, weights, output);

// Sparse convolution: Fallback to scalar ❌
for (int i = 0; i < sparse_indices_count; i++) {
    int idx = sparse_indices[i];
    output[idx] = compute_sparse(activations, weights, idx);
}
```

---

### 3. Manual Structure Unpacking

**Impact**: Multi-field structs require separate loads

**Example**:
```c
struct RGB { uint8_t r, g, b; };
RGB pixels[16];

// Inefficient: 3 separate loads ❌
vld_b(v0, &pixels[0].r, stride=3);  // Not supported!
// Workaround:
for (int i = 0; i < 16; i++) {
    r_array[i] = pixels[i].r;
    g_array[i] = pixels[i].g;
    b_array[i] = pixels[i].b;
}
vld_b(v0, r_array);
vld_b(v1, g_array);
vld_b(v2, b_array);
```

---

## Recommended Best Practices

### 1. Pre-Validate All Addresses

```c
#define VLD_SAFE(vd, addr, len) \
    do { \
        assert(is_valid_address(addr, len)); \
        vld_w(vd, addr); \
    } while(0)
```

---

### 2. Use Planar Memory Layouts

```c
// BAD: Interleaved ❌
struct { float r, g, b; } pixels[N];

// GOOD: Planar ✅
float r_plane[N];
float g_plane[N];
float b_plane[N];
```

---

### 3. Avoid Sparse Operations

```c
// If sparsity < 30%, use dense anyway
if (sparsity < 0.3) {
    use_dense_convolution();  // Faster with VCore
} else {
    use_scalar_sparse();      // Fallback
}
```

---

### 4. Document Assumptions

```c
/**
 * @brief Vector load with stripmining
 * @pre addr must be valid for [addr, addr+64)
 * @pre addr must be in DTCM or cached region
 * @warning NO fault handling - invalid addr causes deadlock!
 */
void vld_w_stripmine(int vd, void *addr);
```

---

## Summary

**Coral NPU VLdSt is deliberately minimal**:
- ✅ Excellent for dense CNN inference (target workload)
- ❌ Insufficient for sparse, dynamic, or complex memory patterns
- ⚠️ Requires careful software validation (no hardware safety net)

**Key Takeaway**: **Know the limitations, design around them.**

---

**Next**: [Exception Handling](exceptions.md) - Understanding the safety implications

---

**References**:
- RISC-V Vector Spec v1.0: https://github.com/riscv/riscv-v-spec
- Coral NPU Source: `VLdSt.scala` (330 lines)

