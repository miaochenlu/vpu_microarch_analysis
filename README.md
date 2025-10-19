# VPU Microarchitecture Analysis

## 📖 Coral NPU Documentation Index

### Core Documentation

- **[Main Documentation Portal](coral/docs/README.md)** - Start here for complete navigation

### Architecture Overview

1. **[Introduction](coral/docs/01_introduction/README.md)**
   - Design goals and philosophy
   - System overview
   - Document scope

2. **[Top-Level Architecture](coral/docs/02_top_level/README.md)**
   - Triple-core fused architecture
   - System block diagram
   - High-level dataflow

### Frontend Pipeline (Scalar Core)

3. **[Frontend Overview](coral/docs/04_frontend/README.md)**
   - 4-way dispatch architecture
   - Pipeline stages
   - Hazard handling

4. **Frontend Components**
   - [Fetch Unit](coral/docs/04_frontend/fetch.md) - Instruction fetch with L0 cache
   - [Instruction Buffer](coral/docs/04_frontend/instruction_buffer.md) - Circular buffer
   - [Decode Unit](coral/docs/04_frontend/decode.md) - Instruction decoding
   - [Dispatch Unit](coral/docs/04_frontend/dispatch.md) - Scoreboarding and hazard detection
   - [Pipeline Walkthroughs](coral/docs/04_frontend/walkthrough.md) - Cycle-by-cycle examples

### Vector Core (VPU)

5. **[Vector Core Overview](coral/docs/06_vector_core/README.md)**
   - 128-bit SIMD architecture
   - Custom instruction set
   - CNN-optimized design

6. **Vector Core Components**
   - [VInst](coral/docs/06_vector_core/vinst.md) - Instruction frontend and slicing
   - [VDecode](coral/docs/06_vector_core/vdecode.md) - Deep decoder with out-of-order tagging
   - [VAlu](coral/docs/06_vector_core/valu.md) - Dual 128-bit SIMD ALU pipelines
   - [VRegfile](coral/docs/06_vector_core/vregfile.md) - Segmented multi-port register file
   - [VConv](coral/docs/06_vector_core/vconv.md) - 8×8 MAC array (256 MACs/cycle)
   - **[VLdSt - Vector Load/Store](coral/docs/06_vector_core/vldst/)** ⚠️
     - [Overview](coral/docs/06_vector_core/vldst/README.md)
     - [Operations](coral/docs/06_vector_core/vldst/operations.md) - vld/vst/vstq instructions
     - [Addressing Modes](coral/docs/06_vector_core/vldst/addressing.md) - Stride, stripmining, swizzle
     - [Microarchitecture](coral/docs/06_vector_core/vldst/microarchitecture.md) - Pipeline and control
     - [Limitations](coral/docs/06_vector_core/vldst/limitations.md) ⚠️ - Unsupported RISC-V features
     - [Exception Handling](coral/docs/06_vector_core/vldst/exceptions.md) ⚠️ - Safety risks
     - [RISC-V Comparison](coral/docs/06_vector_core/vldst/rvv_comparison.md) - Compliance analysis
   - [Vector Walkthroughs](coral/docs/06_vector_core/walkthrough.md) - Detailed examples