# Frontend Pipeline Walkthrough

## Overview

This document provides **detailed step-by-step examples** of how instructions flow through the Coral NPU frontend, from fetch to dispatch. We'll trace specific instruction sequences cycle-by-cycle to illustrate:

1. **Normal 4-way dispatch** (best case)
2. **Hazard detection and stalling** (RAW/WAW)
3. **Branch prediction and misprediction**
4. **Resource conflicts** (single MLU)
5. **Complex scenarios** (mixed instruction types)

---

## Example 1: Perfect 4-Way Dispatch

### Instruction Sequence

```assembly
0x1000: add  x5, x1, x2    # Lane 0: x5 = x1 + x2
0x1004: sub  x6, x3, x4    # Lane 1: x6 = x3 - x4
0x1008: or   x7, x8, x9    # Lane 2: x7 = x8 | x9
0x100C: and  x10, x11, x12 # Lane 3: x10 = x11 & x12
```

### Assumptions
- All source registers are ready (no dependencies)
- L0 I-Cache hit
- No resource conflicts

### Cycle-by-Cycle Execution

#### Cycle N: Fetch

**Fetch Unit**:
```
Input:  PC = 0x1000
Action: L0 cache lookup
        - Index = 0x1000[9:5] = 0x00
        - Tag   = 0x1000[31:10] = 0x00004
        - Hit! (assume cached)
Output: 256-bit line containing 8 instructions
        Extract instructions [0:3]
```

**Instruction Buffer**:
```
Enqueue: 4 instructions
State:   nEnqueued = 4, nSpace = 12
```

**Output to Decode**:
```
Lane 0: valid=1, inst=0x003100B3, addr=0x1000
Lane 1: valid=1, inst=0x40420333, addr=0x1004
Lane 2: valid=1, inst=0x009463B3, addr=0x1008
Lane 3: valid=1, inst=0x00C5F533, addr=0x100C
```

#### Cycle N: Decode (Same Cycle)

**Decode Unit** (parallel for all 4 lanes):

```
Lane 0: inst = 0x003100B3
        Pattern match: add (opcode=0110011, funct3=000, funct7=0000000)
        ✅ d.add = 1
        rs1 = inst[19:15] = 1 (x1)
        rs2 = inst[24:20] = 2 (x2)
        rd  = inst[11:7]  = 5 (x5)
        readsRs1() = true
        readsRs2() = true
        isAlu() = true

Lane 1: inst = 0x40420333
        Pattern match: sub (opcode=0110011, funct3=000, funct7=0100000)
        ✅ d.sub = 1
        rs1 = 3 (x3), rs2 = 4 (x4), rd = 6 (x6)
        
Lane 2: inst = 0x009463B3
        Pattern match: or (opcode=0110011, funct3=110, funct7=0000000)
        ✅ d.or = 1
        rs1 = 8 (x8), rs2 = 9 (x9), rd = 7 (x7)
        
Lane 3: inst = 0x00C5F533
        Pattern match: and (opcode=0110011, funct3=111, funct7=0000000)
        ✅ d.and = 1
        rs1 = 11 (x11), rs2 = 12 (x12), rd = 10 (x10)
```

#### Cycle N: Scoreboard Check (Same Cycle)

**Input Scoreboard**:
```
io.scoreboard.regd = 0x00000000  (no pending writes)
io.scoreboard.comb = 0x00000000  (no pending writes)
```

**Generate Write Scoreboard for This Cycle**:
```
Lane 0 writes x5  → rdScoreboard[0] = 0x00000020  (bit 5)
Lane 1 writes x6  → rdScoreboard[1] = 0x00000040  (bit 6)
Lane 2 writes x7  → rdScoreboard[2] = 0x00000080  (bit 7)
Lane 3 writes x10 → rdScoreboard[3] = 0x00000400  (bit 10)

scoreboardScan:
  [0] = 0x00000000                    (initial)
  [1] = 0x00000020                    (lane 0)
  [2] = 0x00000060                    (lanes 0+1)
  [3] = 0x000000E0                    (lanes 0+1+2)
  [4] = 0x000004E0                    (lanes 0+1+2+3)
```

**Check RAW Hazards**:
```
Lane 0: reads x1, x2
        readScoreboard = 0x00000006 (bits 1,2)
        comb[0] & readScoreboard = 0x00000000 & 0x00000006 = 0
        ✅ No RAW

Lane 1: reads x3, x4
        readScoreboard = 0x00000018 (bits 3,4)
        comb[1] & readScoreboard = 0x00000020 & 0x00000018 = 0
        ✅ No RAW

Lane 2: reads x8, x9
        readScoreboard = 0x00000300 (bits 8,9)
        comb[2] & readScoreboard = 0x00000060 & 0x00000300 = 0
        ✅ No RAW

Lane 3: reads x11, x12
        readScoreboard = 0x00001800 (bits 11,12)
        comb[3] & readScoreboard = 0x000000E0 & 0x00001800 = 0
        ✅ No RAW
```

**Check WAW Hazards**:
```
Lane 0: writes x5, comb[0] = 0x00000000 → No conflict ✅
Lane 1: writes x6, comb[1] = 0x00000020 → No conflict ✅
Lane 2: writes x7, comb[2] = 0x00000060 → No conflict ✅
Lane 3: writes x10, comb[3] = 0x000000E0 → No conflict ✅
```

**Dispatch Decision**:
```
canDispatch[0] = !halted && !interlock && valid && !jumped && 
                 !RAW && !WAW && ... = ✅ TRUE
canDispatch[1] = ✅ TRUE
canDispatch[2] = ✅ TRUE
canDispatch[3] = ✅ TRUE
```

#### Cycle N: Dispatch (Same Cycle)

**Try-Dispatch Loop**:
```
Lane 0: tryDispatch = lastReady[0] && canDispatch[0] = true && true = ✅
        ALU op = ADD
        io.alu(0).valid = 1
        io.alu(0).bits = {addr: 5, op: ADD}
        dispatched = ✅ (ALU fired)
        lastReady[1] = true

Lane 1: tryDispatch = lastReady[1] && canDispatch[1] = true && true = ✅
        ALU op = SUB
        io.alu(1).valid = 1
        io.alu(1).bits = {addr: 6, op: SUB}
        dispatched = ✅ (ALU fired)
        lastReady[2] = true

Lane 2: tryDispatch = lastReady[2] && canDispatch[2] = true && true = ✅
        ALU op = OR
        io.alu(2).valid = 1
        io.alu(2).bits = {addr: 7, op: OR}
        dispatched = ✅ (ALU fired)
        lastReady[3] = true

Lane 3: tryDispatch = lastReady[3] && canDispatch[3] = true && true = ✅
        ALU op = AND
        io.alu(3).valid = 1
        io.alu(3).bits = {addr: 10, op: AND}
        dispatched = ✅ (ALU fired)
        lastReady[4] = true
```

**Register File Operations**:
```
Read Ports:
  Lane 0: rs1Read(0) = x1, rs2Read(0) = x2
  Lane 1: rs1Read(1) = x3, rs2Read(1) = x4
  Lane 2: rs1Read(2) = x8, rs2Read(2) = x9
  Lane 3: rs1Read(3) = x11, rs2Read(3) = x12

Write Marks (for scoreboard update):
  Lane 0: rdMark(0) = x5
  Lane 1: rdMark(1) = x6
  Lane 2: rdMark(2) = x7
  Lane 3: rdMark(3) = x10
```

**Result**: ✅ **All 4 instructions dispatched to ALU in 1 cycle!**

---

## Example 2: RAW Hazard Stall

### Instruction Sequence

```assembly
0x2000: add  x5, x1, x2    # Lane 0: x5 = x1 + x2
0x2004: sub  x6, x5, x4    # Lane 1: x6 = x5 - x4  ⚠️ RAW on x5!
0x2008: or   x7, x8, x9    # Lane 2: x7 = x8 | x9
0x200C: and  x10, x11, x12 # Lane 3: x10 = x11 & x12
```

### Cycle N: Fetch & Decode

(Same as Example 1 - fetch and decode proceed normally)

### Cycle N: Scoreboard Check

**Input Scoreboard**:
```
io.scoreboard.regd = 0x00000000
io.scoreboard.comb = 0x00000000
```

**Generate Write Scoreboard**:
```
Lane 0 writes x5  → rdScoreboard[0] = 0x00000020
Lane 1 writes x6  → rdScoreboard[1] = 0x00000040
Lane 2 writes x7  → rdScoreboard[2] = 0x00000080
Lane 3 writes x10 → rdScoreboard[3] = 0x00000400

scoreboardScan:
  [0] = 0x00000000
  [1] = 0x00000020  ← Lane 0 will write x5
  [2] = 0x00000060
  [3] = 0x000000E0
  [4] = 0x000004E0
```

**Check RAW Hazards**:
```
Lane 0: reads x1, x2
        readScoreboard = 0x00000006
        comb[0] & readScoreboard = 0x00000000 & 0x00000006 = 0
        ✅ No RAW

Lane 1: reads x5, x4  ⚠️ x5 is pending write from Lane 0!
        readScoreboard = 0x00000030 (bits 4,5)
        comb[1] & readScoreboard = 0x00000020 & 0x00000030 = 0x00000020 ≠ 0
        ❌ RAW HAZARD DETECTED!
        readAfterWrite[1] = TRUE

Lane 2: reads x8, x9
        readScoreboard = 0x00000300
        comb[2] & readScoreboard = 0x00000060 & 0x00000300 = 0
        ✅ No RAW

Lane 3: reads x11, x12
        readScoreboard = 0x00001800
        comb[3] & readScoreboard = 0x000000E0 & 0x00001800 = 0
        ✅ No RAW
```

**Dispatch Decision**:
```
canDispatch[0] = ✅ TRUE  (no hazards)
canDispatch[1] = ❌ FALSE (RAW hazard on x5)
canDispatch[2] = ✅ TRUE  (no hazards, but...)
canDispatch[3] = ✅ TRUE  (no hazards, but...)
```

### Cycle N: Dispatch

**Try-Dispatch Loop**:
```
Lane 0: tryDispatch = lastReady[0] && canDispatch[0] = true && true = ✅
        io.alu(0).valid = 1
        dispatched = ✅
        lastReady[1] = true

Lane 1: tryDispatch = lastReady[1] && canDispatch[1] = true && false = ❌
        io.alu(1).valid = 0
        dispatched = ❌
        lastReady[2] = false  ← BLOCKS LANE 2!

Lane 2: tryDispatch = lastReady[2] && canDispatch[2] = false && true = ❌
        (blocked by Lane 1 due to in-order constraint)
        io.alu(2).valid = 0
        lastReady[3] = false

Lane 3: tryDispatch = lastReady[3] && canDispatch[3] = false && true = ❌
        (blocked by Lane 2 due to in-order constraint)
        io.alu(3).valid = 0
        lastReady[4] = false
```

**Result**: Only **1 instruction dispatched** (Lane 0). Lanes 1-3 stalled due to hazard and in-order constraint.

### Cycle N+1: Retry

**Input Scoreboard** (updated after Cycle N):
```
io.scoreboard.regd = 0x00000020  (x5 marked pending)
io.scoreboard.comb = 0x00000020  (x5 marked pending)
```

**Instructions in Decode**:
```
Lane 0: sub x6, x5, x4   (was Lane 1 in Cycle N)
Lane 1: or  x7, x8, x9   (was Lane 2 in Cycle N)
Lane 2: and x10, x11, x12 (was Lane 3 in Cycle N)
Lane 3: (next instruction from fetch)
```

**Check RAW**:
```
Lane 0: reads x5, x4
        readScoreboard = 0x00000030
        comb[0] & readScoreboard = 0x00000020 & 0x00000030 = 0x00000020 ≠ 0
        ❌ STILL RAW! (x5 write hasn't completed)
        
Result: Lane 0 still blocked, entire dispatch stalled
```

### Cycle N+2: ALU Completes

**ALU Unit** completes `add x5, x1, x2`:
```
io.alu.rd.valid = 1
io.alu.rd.addr = 5
io.alu.rd.data = (result)
```

**Register File** clears scoreboard for x5:
```
io.scoreboard.regd = 0x00000000  (x5 write complete)
io.scoreboard.comb = 0x00000000
```

### Cycle N+3: Hazard Resolved

**Instructions in Decode**:
```
Lane 0: sub x6, x5, x4   (retry)
Lane 1: or  x7, x8, x9
Lane 2: and x10, x11, x12
Lane 3: (next instruction)
```

**Check RAW**:
```
Lane 0: reads x5, x4
        readScoreboard = 0x00000030
        comb[0] & readScoreboard = 0x00000000 & 0x00000030 = 0
        ✅ No RAW! (x5 is now available)
```

**Dispatch Decision**:
```
canDispatch = [TRUE, TRUE, TRUE, TRUE]
```

**Result**: ✅ **All 4 instructions dispatch successfully!**

**Summary**: RAW hazard caused **3-cycle bubble** (N+1, N+2, N+3 to resolve).

---

## Example 3: Branch Misprediction

### Instruction Sequence

```assembly
0x3000: add  x5, x1, x2    # Lane 0
0x3004: beq  x5, x0, +16   # Lane 1: if (x5 == 0) goto 0x3014
0x3008: sub  x6, x3, x4    # Lane 2: (should NOT execute if branch taken)
0x300C: or   x7, x8, x9    # Lane 3: (should NOT execute if branch taken)
...
0x3014: and  x10, x11, x12 # Branch target
```

### Cycle N: Fetch & Decode

**Fetch Unit Predecode**:
```
Lane 1: inst = beq (backward/forward check)
        imm = +16 (forward branch)
        Prediction: NOT TAKEN ✅
        Continue fetching sequentially
```

**Decode**:
```
Lane 0: add x5, x1, x2
Lane 1: beq x5, x0, +16
Lane 2: sub x6, x3, x4
Lane 3: or  x7, x8, x9
```

### Cycle N: Scoreboard Check

```
Lane 0: ✅ No hazards
Lane 1: reads x5 → RAW on Lane 0's write!
        ❌ RAW hazard
Lane 2: ✅ No hazards (but blocked by in-order)
Lane 3: ✅ No hazards (but blocked by in-order)
```

**Dispatch Decision**:
```
canDispatch = [TRUE, FALSE, TRUE, TRUE]  (but in-order blocks 2-3)
```

**Dispatch Result**: Only Lane 0 dispatches.

### Cycle N+1: Branch Executes

**BRU Unit** evaluates branch:
```
Input: rs1 = x5 (value from Lane 0's ALU result, forwarded)
       rs2 = x0 = 0
       op = BEQ
       
Compute: x5 == 0?
         Assume x5 = 5 (not zero)
         Result: Branch NOT TAKEN ✅
         
Prediction was correct! No flush needed.
```

**Continue Normal Dispatch**:
```
Lane 0: sub x6, x3, x4  (was Lane 2)
Lane 1: or  x7, x8, x9  (was Lane 3)
Lane 2: (next from fetch)
Lane 3: (next from fetch)
```

### Alternative: Branch Taken (Misprediction)

**BRU Unit** evaluates:
```
Compute: x5 == 0?
         Assume x5 = 0 (zero!)
         Result: Branch TAKEN ❌
         
Prediction was WRONG! (predicted not-taken)
Target: PC + 16 = 0x3014
```

**Flush Pipeline**:
```
io.branchTaken = 1
io.branch(0).valid = 1
io.branch(0).value = 0x3014

Instruction Buffer: FLUSH
  io.flush = 1
  All buffered instructions invalidated
  
Fetch Unit: REDIRECT
  PC = 0x3014
  Start fetching from branch target
```

**Penalty**: **3 cycles**
- Cycle N: Branch executes
- Cycle N+1: Flush + redirect
- Cycle N+2: Fetch from target (L0 miss assumed)
- Cycle N+3: Decode from target

---

## Example 4: Resource Conflict (Single MLU)

### Instruction Sequence

```assembly
0x4000: mul  x5, x1, x2    # Lane 0: multiply ⚠️
0x4004: mul  x6, x3, x4    # Lane 1: multiply ⚠️ (conflicts with Lane 0)
0x4008: add  x7, x8, x9    # Lane 2: add
0x400C: sub  x10, x11, x12 # Lane 3: subtract
```

### Cycle N: Decode

```
Lane 0: d.mul = 1
Lane 1: d.mul = 1  ⚠️ Both lanes want MLU!
Lane 2: d.add = 1
Lane 3: d.sub = 1
```

### Cycle N: Scoreboard Check

```
All lanes: ✅ No RAW/WAW hazards
```

### Cycle N: Dispatch

**Try-Dispatch Loop**:
```
Lane 0: tryDispatch = true && true = ✅
        mlu.valid = true
        io.mlu(0).valid = 1  ← MLU accepts Lane 0
        io.mlu(0).bits = {addr: 5, op: MUL}
        dispatched = ✅ (MLU fired)
        lastReady[1] = true

Lane 1: tryDispatch = true && true = ✅
        mlu.valid = true
        io.mlu(1).valid = 1  ← Try to dispatch to MLU
        io.mlu(1).ready = 0  ❌ MLU busy (Lane 0 using it)
        io.mlu(1).fire = 0
        dispatched = ❌ (MLU rejected)
        lastReady[2] = false  ← BLOCKS REMAINING LANES

Lane 2: tryDispatch = false && true = ❌
        (blocked by in-order constraint)
        
Lane 3: tryDispatch = false && true = ❌
        (blocked by in-order constraint)
```

**Result**: Only **1 instruction dispatched** (Lane 0 MUL).

**Note**: Even though ALU is available for Lanes 2-3, **in-order constraint** blocks them.

### Cycle N+1: Retry

```
Lane 0: mul x6, x3, x4  (was Lane 1)  ← Retry
Lane 1: add x7, x8, x9  (was Lane 2)
Lane 2: sub x10, x11, x12 (was Lane 3)
Lane 3: (next instruction)
```

**Dispatch**: Lane 0's MUL dispatches, then Lanes 1-3 (ALU) dispatch.

**Throughput**: **1 multiply per cycle** (MLU bottleneck).

---

## Example 5: Complex Mixed Scenario

### Instruction Sequence

```assembly
0x5000: lw   x5, 0(x1)     # Lane 0: load
0x5004: add  x6, x5, x2    # Lane 1: use loaded value (RAW!)
0x5008: mul  x7, x3, x4    # Lane 2: multiply
0x500C: csrrw x8, mstatus, x9  # Lane 3: CSR (slot-0-only!)
```

### Cycle N: Decode

```
Lane 0: d.lw = 1, rd=5, rs1=1, imm=0
Lane 1: d.add = 1, rs1=5, rs2=2, rd=6
Lane 2: d.mul = 1, rs1=3, rs2=4, rd=7
Lane 3: d.csrrw = 1, csr=mstatus, rs1=9, rd=8
```

### Cycle N: Scoreboard & Rules Check

**Scoreboard**:
```
Lane 0: writes x5
Lane 1: reads x5 → RAW on Lane 0! ❌
Lane 2: no hazards ✅
Lane 3: no hazards ✅
```

**Slot-0 Rule**:
```
Lane 3: CSR instruction
        forceSlot0Only() = TRUE
        slot0Interlock[3] = FALSE  ❌ (CSR not in slot 0)
```

**Dispatch Decision**:
```
canDispatch[0] = TRUE  (no hazards)
canDispatch[1] = FALSE (RAW hazard)
canDispatch[2] = TRUE  (but blocked by in-order)
canDispatch[3] = FALSE (slot-0-only violation)
```

### Cycle N: Dispatch

```
Lane 0: LSU.valid = 1  ✅ Dispatched
Lane 1-3: Blocked
```

**Result**: Only **load** dispatches.

### Cycle N+1: Reorder

```
Lane 0: csrrw x8, mstatus, x9  (move to slot 0)
Lane 1: add x6, x5, x2
Lane 2: mul x7, x3, x4
Lane 3: (next instruction)
```

**Scoreboard**:
```
Lane 0: CSR in slot 0 ✅
        No hazards
Lane 1: reads x5 → still RAW (load not complete)
Lane 2-3: blocked by in-order
```

**Dispatch**: Only CSR dispatches.

### Cycle N+2: Load Completes

**LSU** completes:
```
io.lsu.rd.valid = 1
io.lsu.rd.addr = 5
io.lsu.rd.data = (loaded value)
```

**Scoreboard** clears x5.

### Cycle N+3: Full Dispatch

```
Lane 0: add x6, x5, x2  ✅ (x5 now available)
Lane 1: mul x7, x3, x4  ✅
Lane 2-3: (next instructions) ✅
```

**Summary**: Complex dependencies and rules caused **staggered dispatch**.

---

## Performance Summary Table

| Scenario | Dispatch Width | Cycles | IPC | Bottleneck |
|----------|---------------|--------|-----|------------|
| **Perfect 4-way** | 4, 4, 4, 4 | 4 | 4.0 | None |
| **RAW Hazard** | 1, 0, 0, 4 | 5 | 1.0 | Dependency |
| **Branch Misp** | 1, (flush), 4, 4 | 6 | 1.5 | Branch |
| **MLU Conflict** | 1, 4, 4, 4 | 4 | 3.25 | Single MLU |
| **Mixed Complex** | 1, 1, 0, 4 | 6 | 1.0 | Multiple |

**Key Insights**:
- **Dependencies kill IPC**: RAW/WAW hazards are the main bottleneck
- **In-order constraint**: Blocks independent instructions behind stalled ones
- **Resource limits**: Single MLU/DVU limit multiply/divide throughput
- **Branch prediction**: Mispredictions flush pipeline (3-cycle penalty)

---

## Optimization Strategies

### 1. Software (Compiler)
- **Instruction scheduling**: Reorder instructions to avoid dependencies
- **Loop unrolling**: Expose more parallelism
- **Register allocation**: Minimize register reuse (reduce WAW)

### 2. Hardware (Future)
- **Out-of-order execution**: Dispatch past stalled instructions
- **More MLUs**: Increase multiply throughput (2+ MLUs)
- **Better branch prediction**: Dynamic predictors (reduce mispredictions)
- **Larger instruction buffer**: Hide longer fetch latencies

---

**See Also**:
- [Fetch Unit](fetch.md) - Instruction fetch details
- [Decode Unit](decode.md) - Instruction decoding
- [Dispatch Unit](dispatch.md) - Dispatch rules and scoreboard

