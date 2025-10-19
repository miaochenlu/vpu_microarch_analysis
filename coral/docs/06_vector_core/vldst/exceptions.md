# VLdSt Exception Handling and Safety

## ⚠️ Critical Safety Issue

**Coral NPU VLdSt has NO exception handling mechanism.**

This document explains the risks, current behavior, and recommendations for production use.

---

## Executive Summary

### Current Implementation

**VLdSt.scala has ZERO exception-related signals**:
```scala
// ❌ All missing:
val exception = Output(Bool())           // No exception signal
val exception_cause = Output(UInt(4.W))  // No cause code
val exception_addr = Output(UInt(32.W))  // No faulting address
val dbus_error = Input(Bool())           // No bus error input
val fault_detected = Output(Bool())      // No fault status
```

**Consequence**: **All memory accesses are assumed to succeed.**

---

## What Happens on Memory Errors

### Scenario 1: Invalid Address (Unmapped Region)

**Code**:
```assembly
li x10, 0xFFFFFFFF    # Invalid address
vld.w v0, (x10)       # Attempt load
```

**Expected Behavior** (proper system):
1. DBus transaction issued to 0xFFFFFFFF
2. Memory system returns error (SLVERR on AXI)
3. VLdSt captures error
4. Exception raised to scalar core
5. Software handles error (trap handler)

**Actual Behavior** (Coral NPU):
```
1. DBus transaction issued to 0xFFFFFFFF
2. Memory system returns error (SLVERR)
3. io.dbus.ready never asserts ← DEADLOCK!
4. VLdSt waits forever for ready
5. Command queue stalls
6. Entire Vector Core hangs ❌
```

**Code Evidence** (VLdSt.scala lines 267-271):
```scala
ctrl.io.out.ready := io.dbus.ready &&  // Waits for ready
                     (data.io.out.valid || !ctrl.io.out.bits.write)
io.dbus.valid := ctrl.io.out.valid &&   // Valid set
                 (data.io.out.valid || !ctrl.io.out.bits.write)

// If io.dbus.ready never asserts → infinite wait
// No timeout, no error check, no recovery
```

---

### Scenario 2: Page Fault (Virtual Memory)

**Code**:
```assembly
li x10, 0x10000FF0    # Near page boundary
vld.w v8, (x10)       # Load 64 bytes (stripmining)
```

**Expected Behavior**:
1. Transaction 0: 0x10000FF0-0x10000FFF ✅ Success (16 bytes, in page)
2. Transaction 1: 0x10001000-0x1000100F ❌ Page fault (crosses to unmapped page)
3. Fault-only-first: Update vl=16, raise exception
4. Software maps page, resumes

**Actual Behavior** (Coral NPU):
```
1. Transaction 0: 0x10000FF0 ✅ Success
2. Transaction 1: 0x10001000 ❌ Page fault
3. ??? Undefined behavior (likely deadlock)
4. No partial completion
5. Registers v8-v11 may be corrupted
```

---

### Scenario 3: Bus Error (Hardware Fault)

**Code**:
```assembly
li x10, 0x20000000    # External device address
vst.w v0, (x10)       # Store to device
```

**Expected Behavior**:
1. DBus write transaction to 0x20000000
2. Device returns SLVERR (write failed)
3. Exception raised
4. Software logs error, retries, or aborts

**Actual Behavior** (Coral NPU):
```
1. DBus write transaction to 0x20000000
2. Device returns SLVERR
3. No error propagation to VLdSt
4. Store completes "successfully" (silently fails) ❌
5. Data corruption undetected
```

---

### Scenario 4: Misaligned Address (Strict Alignment Required)

**Code**:
```assembly
li x10, 0x10000001    # Misaligned (not 4-byte aligned)
vld.w v0, (x10)       # Load word
```

**Expected Behavior**:
- Swizzle logic handles misalignment (Coral NPU does this ✅)
- OR: Raise alignment exception if unsupported

**Actual Behavior** (Coral NPU):
- Swizzle logic rotates data correctly ✅
- BUT: If memory system rejects misaligned → deadlock (same as Scenario 1)

---

## Root Cause Analysis

### Missing Components

#### 1. No Error Input Signal

**Missing** (VLdSt.scala):
```scala
class VLdSt(p: Parameters) extends Module {
  val io = IO(new Bundle {
    // ... existing signals ...
    
    // ❌ MISSING:
    val dbus_error = Input(Bool())  // Bus error from memory system
  })
}
```

**Impact**: Cannot detect when DBus transaction fails.

---

#### 2. No Exception Output Signal

**Missing** (VLdSt.scala):
```scala
// ❌ MISSING:
val exception_valid = Output(Bool())     // Exception occurred
val exception_cause = Output(UInt(4.W))  // Cause code
val exception_vaddr = Output(UInt(32.W)) // Virtual address
val exception_pc = Output(UInt(32.W))    // PC of faulting instruction
```

**Impact**: Cannot inform scalar core of errors.

---

#### 3. No State Recovery

**Missing**: Logic to:
- Save command queue state before error
- Flush pipeline on exception
- Resume from partial completion
- Rollback register writes

**Current Behavior**: State is undefined after error.

---

#### 4. No Timeout/Watchdog

**Missing**:
```scala
// ❌ MISSING:
val timeout_counter = RegInit(0.U(16.W))
when (io.dbus.valid && !io.dbus.ready) {
  timeout_counter := timeout_counter + 1.U
  when (timeout_counter > TIMEOUT_THRESHOLD) {
    // Raise timeout exception
  }
}
```

**Impact**: Infinite wait on unresponsive bus.

---

## Comparison with Proper Implementation

### Standard Vector LSU with Exception Handling

**Signals** (typical implementation):
```scala
class VLdStWithExceptions extends Module {
  val io = IO(new Bundle {
    // Memory interface
    val dbus = new DBusIO
    val dbus_error = Input(Bool())           // ✅ Error input
    val dbus_error_cause = Input(UInt(2.W))  // ✅ Error type
    
    // Exception interface
    val exception = Output(Bool())           // ✅ Exception valid
    val exception_cause = Output(UInt(5.W))  // ✅ Cause code
    val exception_vaddr = Output(UInt(32.W)) // ✅ Faulting address
    val exception_pc = Output(UInt(32.W))    // ✅ Instruction PC
    
    // Partial completion
    val vl_update = Output(UInt())           // ✅ Elements completed
    val partial_done = Output(Bool())        // ✅ Partial success
  })
}
```

**Error Handling Logic**:
```scala
// Detect error
val error_detected = RegInit(false.B)
val error_addr = Reg(UInt(32.W))
val error_cause = Reg(UInt(5.W))

when (io.dbus.valid && io.dbus.ready && io.dbus_error) {
  error_detected := true.B
  error_addr := io.dbus.addr
  error_cause := Mux(io.dbus.write, CAUSE_STORE_FAULT, CAUSE_LOAD_FAULT)
}

// Stop pipeline
q.io.out.ready := !error_detected && ... 

// Report exception
io.exception := error_detected
io.exception_vaddr := error_addr
io.exception_cause := error_cause

// Clear on acknowledgement
when (io.exception_ack) {
  error_detected := false.B
}
```

**Area Cost**: ~10-15% of VLdSt logic

---

## Real-World Risk Assessment

### Risk Level: **HIGH** 🔴

**Failure Modes**:

1. **System Deadlock**: Invalid address → hang (requires hard reset)
2. **Silent Data Corruption**: Bus error ignored → wrong results
3. **Security Vulnerability**: Unvalidated addresses → potential exploits
4. **Debugging Nightmare**: No error indication → hard to diagnose

**Probability**:
- Development: **VERY HIGH** (bugs, typos, wrong pointers)
- Production (validated): **MEDIUM** (hardware faults, cosmic rays)

**Impact**:
- Development: Lost debugging time
- Production: System failure, potential safety hazard

---

## Mitigation Strategies

### Strategy 1: Software Address Validation (Mandatory)

**Principle**: **Never issue vector load/store without validating address first.**

**Implementation**:
```c
// Address validation function
bool is_valid_vld_address(void *addr, size_t length) {
    // Check alignment (optional for Coral NPU, but good practice)
    if ((uintptr_t)addr % 4 != 0) {
        return false;
    }
    
    // Check bounds (must be in valid memory region)
    if (addr < DTCM_BASE || addr >= DTCM_END) {
        if (addr < SRAM_BASE || addr >= SRAM_END) {
            return false;  // Not in any valid region
        }
    }
    
    // Check length doesn't overflow
    if ((uintptr_t)addr + length < (uintptr_t)addr) {
        return false;  // Overflow
    }
    
    // Check end address
    void *end_addr = (char *)addr + length;
    if (end_addr > DTCM_END && end_addr > SRAM_END) {
        return false;  // Crosses region boundary
    }
    
    return true;
}

// Safe vector load macro
#define VLD_W_SAFE(vd, addr) \
    do { \
        assert(is_valid_vld_address(addr, 16)); \
        vld_w(vd, addr); \
    } while(0)
```

**Coverage**: Prevents most invalid address errors ✅

**Limitation**: Cannot prevent hardware faults (bus errors) ❌

---

### Strategy 2: Watchdog Timer (Hardware)

**Principle**: If VLdSt doesn't complete within timeout, reset it.

**Implementation** (system-level, NOT in VLdSt):
```verilog
// Watchdog counter
reg [15:0] vldst_timeout_counter;
wire vldst_active = vldst_io_in_valid;

always @(posedge clk) begin
    if (rst) begin
        vldst_timeout_counter <= 0;
    end else if (vldst_active) begin
        vldst_timeout_counter <= vldst_timeout_counter + 1;
        if (vldst_timeout_counter > 16'hFFFF) begin
            // Timeout! Force reset VCore
            vcore_reset <= 1;
        end
    end else begin
        vldst_timeout_counter <= 0;
    end
end
```

**Coverage**: Prevents deadlock (forces recovery) ✅

**Limitation**: Loses inflight data, not graceful ❌

---

### Strategy 3: Memory Region Constraints (Software)

**Principle**: Only allow vector operations in "safe" memory regions.

**Implementation**:
```c
// Whitelist safe regions
const struct MemRegion {
    void *base;
    size_t size;
} safe_regions[] = {
    { DTCM_BASE, DTCM_SIZE },  // 32KB DTCM (always safe)
    { ITCM_BASE, ITCM_SIZE },  // 8KB ITCM (always safe)
    // AXI regions: NOT safe (may have bus errors)
};

bool is_safe_region(void *addr) {
    for (int i = 0; i < ARRAY_SIZE(safe_regions); i++) {
        if (addr >= safe_regions[i].base &&
            addr < (char *)safe_regions[i].base + safe_regions[i].size) {
            return true;
        }
    }
    return false;
}
```

**Coverage**: Limits exposure to safe, known memory ✅

**Limitation**: Restricts flexibility ⚠️

---

### Strategy 4: Simulation-Only Assertions

**Principle**: Catch errors during development, not production.

**Implementation** (already in VLdSt.scala):
```scala
// Address validity assertions (simulation only)
assert(!(ctrl.io.out.valid && ctrl.io.out.bits.addr(31)))
assert(!(ctrl.io.out.valid && ctrl.io.out.bits.adrx(31)))
assert(!(io.dbus.valid && io.dbus.addr(31)))

// These catch address MSB errors in simulation
// But are disabled in synthesis (no runtime protection)
```

**Coverage**: Development only ✅

**Limitation**: NO protection in production ❌

---

## Recommended Production Configuration

### Minimal Safety Setup

**For production deployment**:

1. **Software validation**: Mandatory for ALL vector memory ops
2. **Watchdog timer**: System-level timeout (10,000 cycles)
3. **Region constraints**: Restrict to DTCM/ITCM only
4. **Extensive testing**: Validate all code paths in simulation

**Code Template**:
```c
// Production-safe vector load
void vld_w_production(int vd, void *addr) {
    // 1. Validate address
    if (!is_safe_region(addr)) {
        log_error("VLD: Invalid address %p", addr);
        abort();  // Fail-safe
    }
    
    // 2. Check alignment (optional)
    if ((uintptr_t)addr % 4 != 0) {
        log_warning("VLD: Misaligned address %p", addr);
        // Proceed anyway (swizzle handles it)
    }
    
    // 3. Issue vector load
    vld_w(vd, addr);
    
    // 4. Wait with timeout
    uint32_t timeout = 1000;
    while (!vld_complete(vd) && timeout--) {
        // Spin wait
    }
    if (timeout == 0) {
        log_error("VLD: Timeout on address %p", addr);
        reset_vcore();  // Last resort
    }
}
```

---

## Future Recommendations

### If Redesigning VLdSt

**Priority 1: Add Minimal Exception Handling** (HIGH)

```scala
// Add to VLdSt.scala
val io = IO(new Bundle {
  // ... existing ...
  
  // NEW: Error inputs
  val dbus_error = Input(Bool())
  
  // NEW: Exception outputs
  val exception = Output(Bool())
  val exception_addr = Output(UInt(32.W))
})

// Detect and report errors
val error_reg = RegInit(false.B)
when (io.dbus.valid && io.dbus.ready && io.dbus_error) {
  error_reg := true.B
}
io.exception := error_reg
```

**Benefit**: Prevents deadlock, enables graceful error handling

**Cost**: ~5-10% area increase

---

**Priority 2: Add Timeout Counter** (MEDIUM)

```scala
val timeout_counter = RegInit(0.U(16.W))
when (io.dbus.valid && !io.dbus.ready) {
  timeout_counter := timeout_counter + 1.U
  when (timeout_counter > 1000.U) {
    // Force error
    error_reg := true.B
  }
}
```

**Benefit**: Prevents infinite wait

**Cost**: ~2-3% area increase

---

**Priority 3: Add Partial Completion** (LOW)

```scala
val elements_completed = RegInit(0.U())
io.vl_update := elements_completed  // Report to scalar core
```

**Benefit**: Fault-only-first support

**Cost**: ~10-15% area increase

---

## Summary

### Current Status

❌ **NO exception handling**  
❌ **NO bus error detection**  
❌ **NO timeout/watchdog**  
❌ **NO partial completion**  
❌ **NO state recovery**  

**Risk**: **System deadlock on invalid address or bus error**

---

### Mitigation

✅ **Software address validation** (mandatory)  
✅ **Watchdog timer** (system-level)  
✅ **Memory region constraints**  
✅ **Extensive simulation**  
⚠️ **Accept risk** for validated, controlled environments  

---

### Production Readiness

**Suitable for**:
- ✅ Embedded systems with controlled memory layout
- ✅ Single-application (no OS, no virtual memory)
- ✅ Thoroughly validated code paths
- ✅ Internal development/research

**NOT suitable for**:
- ❌ General-purpose OS
- ❌ Multi-process environment
- ❌ Safety-critical systems (automotive, medical)
- ❌ Untrusted code execution

---

**Key Takeaway**: **Coral NPU VLdSt requires careful software discipline to avoid catastrophic failures.**

---

**Next**: [RISC-V Vector Comparison](rvv_comparison.md)

---

**References**:
- VLdSt.scala: Lines 267-285 (DBus transaction logic, no error handling)
- DBusIO.scala: Memory interface definition (no error signals)

