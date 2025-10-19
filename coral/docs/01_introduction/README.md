# Introduction

## What is Coral NPU?

**Coral NPU** (Neural Processing Unit) is an open-source hardware accelerator designed by Google Research for ultra-low-power machine learning inference at the edge. Unlike traditional ML accelerators that function as separate coprocessors, Coral NPU is a **tightly-integrated processor** that fuses scalar, vector, and matrix processing capabilities into a single coherent architecture.

## Design Goals

### Primary Objectives

1. **Ultra-Low Power Consumption**
   - Target: Wearable devices (hearables, AR glasses, smartwatches)
   - Design principle: Minimize data movement, maximize computational efficiency
   - Features: Clock gating, tightly-coupled memories, single-cycle SRAM access

2. **High Computational Efficiency for ML Inference**
   - Outer-product MAC engine: 256 MAC operations per cycle (INT8)
   - Custom vector instructions optimized for neural network operations
   - Hardware support for quantized inference (INT8 weights/activations)

3. **Programmability and Flexibility**
   - Full RISC-V RV32IMF base ISA for control flow
   - Custom SIMD instructions for data preprocessing
   - Standard C/C++ toolchain support (GCC/Clang)
   - Ability to run non-ML workloads

4. **Deterministic Latency**
   - Tightly-coupled memories (TCM) with single-cycle access
   - In-order pipeline for predictable execution
   - No speculative execution or complex out-of-order logic

5. **Small Silicon Footprint**
   - Target: Integration into ultra-low-power SoCs
   - Efficient hardware implementation (< 1mm² in modern process nodes)
   - Minimal external memory requirements

### Secondary Objectives

- **Open Source**: Freely available for integration, modification, and study
- **Standard Interfaces**: AXI4 and TileLink bus support for easy SoC integration
- **Debug Support**: JTAG-based debugging, trace capabilities
- **Verification-friendly**: Built with Chisel for high-quality RTL generation

## Target Applications

Coral NPU is specifically designed for **always-on, ultra-low-power ML inference**:

### Wearable Devices

| Device Type | Use Cases | Requirements |
|-------------|-----------|--------------|
| **Hearables** | Wake word detection, voice activity detection, noise cancellation | < 1mW active power, < 10ms latency |
| **AR Glasses** | Gesture recognition, eye tracking, scene understanding | < 5mW active power, < 30fps processing |
| **Smartwatches** | Activity recognition, health monitoring, sensor fusion | < 2mW active power, low memory footprint |

### Edge AI Devices

- **Smart home sensors**: Presence detection, anomaly detection
- **Industrial IoT**: Predictive maintenance, quality inspection
- **Medical devices**: Continuous health monitoring, early warning systems

### Typical ML Workloads

1. **Convolutional Neural Networks** (CNNs)
   - MobileNet, EfficientNet architectures
   - 2D convolutions with 3×3, 5×5 kernels
   - Depthwise separable convolutions

2. **Fully-Connected Layers**
   - Matrix-vector multiplications
   - Small to medium sized matrices (< 1024×1024)

3. **Activation Functions**
   - ReLU, ReLU6, Sigmoid (via lookup tables)
   - Implemented using vector SIMD operations

4. **Pooling and Normalization**
   - Max pooling, average pooling
   - Batch normalization, layer normalization

## Architectural Approach: Domain-First Design

Coral NPU employs a **domain-first design methodology**, starting with ML inference requirements and building up from there:

### 1. Matrix/Convolution as Foundation

**Start with the compute kernel**:
- Identify that convolutions dominate ML inference workloads
- Design an efficient outer-product MAC engine
- Optimize for INT8 quantized operations

### 2. Add Vector Capabilities

**Extend beyond pure matrix operations**:
- Data preprocessing (normalization, type conversion)
- Post-processing (activation functions, pooling)
- General-purpose SIMD for flexibility

### 3. Include Scalar Control

**Lightweight RISC-V core for orchestration**:
- Loop control and address generation
- Branching and control flow
- Interface with host system

### 4. Fuse into Unified Design

**Tight coupling, not loose federation**:
- Shared memory space
- Integrated dispatch and scoreboard logic
- Unified register file access

This approach differs from traditional designs:

| Traditional Approach | Coral NPU Approach |
|---------------------|-------------------|
| Start with a CPU, add accelerator | Start with ML kernel, add CPU |
| Loose coupling via DMA/mailbox | Tight coupling via shared memory |
| Separate instruction streams | Unified instruction stream |
| Complex coherence protocols | Simplified memory model |

## Key Innovations

### 1. Triple-Core Fusion

**Scalar + Vector + Matrix** in a single pipeline:
- Single instruction fetch and decode
- Unified scoreboard for dependency tracking
- Shared memory subsystem

### 2. Outer-Product MAC Engine

**Broadcast-based multiply-accumulate**:
- Two-dimensional broadcast structure
- Wide broadcast for weights, narrow broadcast for activations
- Maximizes computation per memory access

### 3. Stripmining Mechanism

**Automatic instruction expansion**:
- Single vector instruction → 4 sequential operations
- Reduces dispatch pressure
- Native support for register tiling

### 4. Tightly-Coupled Memory

**Deterministic single-cycle access**:
- 8 KB ITCM for instructions
- 32 KB DTCM for data
- No cache miss unpredictability

### 5. Custom Vector ISA

**ML-optimized instruction set**:
- Reclaims RISC-V C-extension encoding space
- Flexible type encodings (8/16/32-bit)
- Instruction compression for reduced code size

## Architecture at a Glance

### ISA and Extensions

- **Base ISA**: RV32IMF (32-bit RISC-V with integer, multiply, and single-precision float)
- **Vector Extension**: Zve32x (32-bit vector element support)
- **Custom Extensions**: 
  - Custom SIMD instructions (using reclaimed C-extension space)
  - ML-specific operations (VDot, VConv)
- **Bit-manipulation**: Zbb (basic bit manipulation)

### Pipeline Overview

- **4-stage pipeline**: Fetch → Decode/Dispatch → Execute → Writeback
- **4-way superscalar**: Up to 4 instructions dispatched per cycle
- **In-order dispatch**: Instructions dispatched in program order
- **Out-of-order retire**: Instructions complete when ready

### Register Architecture

| Register File | Count | Width | Total Capacity |
|---------------|-------|-------|----------------|
| Scalar (x0-x31) | 32 | 32-bit | 128 bytes |
| Floating-point (f0-f31) | 32 | 32-bit | 128 bytes |
| Vector (v0-v63) | 64 | 256-bit | 2 KB |
| Accumulator | 64 | 32-bit | 256 bytes |

### Memory Hierarchy

```
┌─────────────────────────────────────────┐
│              Coral NPU                   │
│                                          │
│  ┌──────────┐         ┌──────────┐     │
│  │  L1 I$   │         │  L1 D$   │     │
│  │  8KB     │         │  16KB    │     │
│  │  4-way   │         │  4-way   │     │
│  └────┬─────┘         └────┬─────┘     │
│       │                    │            │
│  ┌────┴─────┐         ┌────┴─────┐     │
│  │  ITCM    │         │  DTCM    │     │
│  │  8KB     │         │  32KB    │     │
│  │  1-cycle │         │  1-cycle │     │
│  └──────────┘         └──────────┘     │
│                                         │
└─────────────┬───────────────────────────┘
              │
        ┌─────┴─────┐
        │  External │
        │  Memory   │
        │  (AXI4)   │
        └───────────┘
```

### Peak Performance Estimates

Assuming **500 MHz** clock frequency (process-dependent):

| Metric | Value | Calculation |
|--------|-------|-------------|
| **Scalar Throughput** | 2000 MIPS | 4 instructions/cycle × 500 MHz |
| **Vector Throughput** | 16 GOPS (32-bit) | 32 lanes × 500 MHz |
| **MAC Throughput** | 256 GOPS (INT8) | 256 MACs/cycle × 2 ops/MAC × 500 MHz |
| **Memory Bandwidth** | 24 GB/s | 3 buses × 128-bit × 500 MHz ÷ 8 |

**Note**: These are theoretical peaks. Actual sustained performance depends on workload characteristics and memory access patterns.

## Comparison with Other Architectures

### vs. Traditional CPUs (ARM Cortex-M)

| Aspect | ARM Cortex-M4 | Coral NPU |
|--------|---------------|-----------|
| ML Performance | ~10 GOPS (via DSP) | ~256 GOPS (INT8) |
| Power Efficiency | ~10 GOPS/W | ~500 GOPS/W (estimated) |
| Programmability | Standard C/C++ | Standard C/C++ + custom intrinsics |
| Latency | Variable (caches) | Deterministic (TCM) |

### vs. Dedicated ML Accelerators (Google Edge TPU)

| Aspect | Edge TPU | Coral NPU |
|--------|----------|-----------|
| Architecture | Separate accelerator | Fused processor |
| Host Dependency | Requires external CPU | Self-contained |
| Flexibility | Fixed operations | Fully programmable |
| Integration | PCIe/USB | On-chip SoC integration |

### vs. DSP Processors (Tensilica)

| Aspect | Tensilica DSP | Coral NPU |
|--------|---------------|-----------|
| ISA | Custom | RISC-V + custom |
| ML Optimization | Generic SIMD | Dedicated MAC engine |
| Ecosystem | Proprietary | Open source |
| Vector Width | 128-512 bit | 256 bit |

## Open Source and Ecosystem

### License

- **Apache 2.0 License**: Permissive, allows commercial use
- **Free for integration**: No licensing fees or restrictions

### Repository Structure

- **HDL Source**: Chisel hardware description (high-quality RTL)
- **Toolchain**: GCC/Clang with custom intrinsics
- **Simulation**: Verilator, VCS, and Cocotb testbenches
- **Examples**: Sample programs and ML kernels

### Supported Tools

| Category | Tools |
|----------|-------|
| **HDL** | Chisel 3.x, CIRCT/MLIR |
| **Simulation** | Verilator, VCS, Renode |
| **Verification** | Cocotb, UVM |
| **Synthesis** | Vivado, Genus |
| **Software** | GCC, Clang, Bazel |

## Document Structure

This documentation is organized into the following parts:

### Part I: System Overview (Chapters 1-3)

Foundation and high-level architecture

### Part II: Frontend (Chapters 4-5)

Instruction fetch, decode, and dispatch

### Part III: Scalar Core (Chapters 6-8)

RV32IMF execution units and register files

### Part IV: Vector Core (Chapters 9-12) ⭐

Custom SIMD backend with stripmining

### Part V: Matrix Core (Chapters 13-16) ⭐

Outer-product MAC engine for ML acceleration

### Part VI: Memory Subsystem (Chapters 17-20)

TCM, caches, LSU, and bus fabric

### Part VII: Advanced Features (Chapters 21-24)

Debug, retirement, clocking, fault management

### Part VIII: Performance and Integration (Chapters 25-27)

System integration and performance analysis

### Appendices

ISA reference, register maps, memory maps, parameters, glossary

## How to Read This Document

### For Hardware Engineers

1. Read Part I (Chapters 1-3) for overview
2. Study Part VI (Memory Subsystem) for integration details
3. Dive into specific parts based on your needs (frontend/execution/memory)
4. Reference Appendices for register maps and parameters

### For Software Engineers

1. Start with Chapter 3 (ISA and Programming Model)
2. Understand Chapter 6 (Vector Core Overview)
3. Review example programs in the repository
4. Use Appendix A (ISA Reference) as a quick reference

### For Researchers

1. Focus on **Part IV (Vector Core)** and **Part V (Matrix Core)**
2. Study the stripmining mechanism (Chapter 10)
3. Analyze the outer-product MAC engine (Chapter 14)
4. Review performance analysis (Chapter 26)

### For Students

1. Follow the document order from Chapter 1 onwards
2. Pay attention to the design decisions and trade-offs
3. Try to understand why certain choices were made
4. Run simulations with example programs to solidify understanding

## Document Conventions

### Code Listings

Chisel/Scala code from the repository:
```scala
val score = SCore(p)  // Scalar core instantiation
```

### Block Diagrams

Mermaid diagrams rendered directly in Markdown:
```mermaid
graph LR
    A[Input] --> B[Process]
    B --> C[Output]
```

### Register/Signal Notation

- `signal_name` - Signal wire
- `reg_name` - Register
- `MODULE.signal` - Hierarchical signal path

### Number Formats

- Decimal: `1234`
- Hexadecimal: `0xDEADBEEF`
- Binary: `0b1010`
- Sizes: `8 KB` (kilobytes), `1 GB/s` (gigabytes per second)

## Getting Started

Ready to dive in? Proceed to:

**[Chapter 2: Top-Level Architecture →](../02_top_level/README.md)**

Or jump directly to areas of interest:
- **[Vector Core →](../06_vector_core/README.md)** - Custom SIMD backend
- **[Matrix Core →](../07_matrix_core/README.md)** - ML acceleration engine
- **[Performance Analysis →](../10_performance/README.md)** - Benchmarks and optimization

---

**Last Updated**: 2025-10-19  
**Document Version**: 1.0  
**Based on**: Coral NPU main branch (commit: latest)

