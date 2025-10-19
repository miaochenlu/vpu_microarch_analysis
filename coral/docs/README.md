# Coral NPU Microarchitecture Documentation

> **专业微架构文档 - 对标 Saturn Vectors 质量标准**

Google Coral NPU 是一个面向边缘 AI 推理的开源硬件加速器，采用独特的**三核融合设计**（Scalar + Vector + Matrix）。本文档基于 Chisel HDL 源码进行深度分析，提供完整的微架构视图。

## 架构亮点

- **三核融合**: RV32IMF 标量核心 + 自定义 SIMD 向量核心 + 外积 MAC 矩阵核心
- **高效流水线**: 4-way 解码派遣，顺序发射，乱序退休
- **低延迟内存**: 8KB ITCM + 32KB DTCM，单周期访问
- **灵活向量**: 64x 256-bit 向量寄存器，支持 8/16/32-bit 数据类型
- **ML 优化**: 专用外积引擎，每周期 256 MAC 操作

## 文档导航

### Part I: 系统概览
- [1. Introduction](01_introduction/README.md) - 设计目标与核心特性
- [2. Top-Level Architecture](02_top_level/README.md) - 顶层架构与数据流
- [3. ISA and Programming Model](03_isa/README.md) - 指令集与编程模型

### Part II: 前端 (Frontend Pipeline)
- [Frontend Overview](04_frontend/README.md) - 前端概览
- [Fetch Unit](04_frontend/fetch.md) - 取指单元与 L0 缓存
- [Instruction Buffer](04_frontend/instruction_buffer.md) - 指令缓冲
- [Decode Unit](04_frontend/decode.md) - 解码单元
- [Dispatch Unit](04_frontend/dispatch.md) - 派遣单元与记分板
- [Pipeline Walkthrough](04_frontend/walkthrough.md) ⭐ - 流水线示例

### Part III: 标量核心 (Scalar Core)
- [Scalar ALU](05_scalar_core/alu.md) - 算术逻辑单元 (待完成)
- [MLU/DVU](05_scalar_core/mlu.md) - 乘除单元 (待完成)
- [LSU](05_scalar_core/lsu.md) - 加载存储单元 (待完成)

### Part IV: 向量核心 (Vector Core - Custom SIMD)
- [Vector Core Overview](06_vector_core/README.md) ⭐ - 向量核心概览
- [VInst - Instruction Frontend](06_vector_core/vinst.md) ⭐ - 指令前端与切片
- [VDecode - Deep Decoder](06_vector_core/vdecode.md) ⭐ - 深度解码器与依赖追踪
- [VAlu - SIMD ALU](06_vector_core/valu.md) ⭐ - 双路 128-bit SIMD ALU
- [VRegfile - Register File](06_vector_core/vregfile.md) ⭐ - 64×256-bit 寄存器文件
- [VLdSt - Load/Store Unit](06_vector_core/vldst.md) ⭐ - 向量加载存储单元
- [VConv - Convolution Accelerator](06_vector_core/vconv.md) ⭐ - 256 MACs/周期卷积加速器
- [Vector Walkthrough](06_vector_core/walkthrough.md) ⭐ - 向量指令执行示例

### Part V: 矩阵核心 (Matrix Core)
- [13. MAC Engine Overview](07_matrix_core/README.md) - MAC 引擎概览
- [14. Outer Product Engine](07_matrix_core/outer_product.md) - 外积引擎
- [15. Convolution ALU](07_matrix_core/conv_alu.md) - 卷积加速器
- [16. MAC Pipeline](07_matrix_core/pipeline.md) - MAC 流水线

### Part VI: 内存子系统
- [17. Load/Store Unit](08_memory_subsystem/lsu.md) - 加载存储单元
- [18. TCM](08_memory_subsystem/tcm.md) - 紧耦合内存
- [19. Cache System](08_memory_subsystem/cache.md) - 缓存系统
- [20. Memory Fabric](08_memory_subsystem/fabric.md) - 内存互联

### Part VII: 高级特性
- [21. Retirement](09_advanced_features/retirement.md) - 乱序退休
- [22. Debug](09_advanced_features/debug.md) - 调试架构
- [23. Clock and Reset](09_advanced_features/clocking.md) - 时钟与复位
- [24. Fault Management](09_advanced_features/fault.md) - 故障管理

### Part VIII: 性能分析
- [25. System Integration](10_performance/integration.md) - 系统集成
- [26. Performance Analysis](10_performance/analysis.md) - 性能分析
- [27. Power and Area](10_performance/power_area.md) - 功耗与面积

### Appendices
- [A. ISA Reference](appendices/isa_reference.md) - 指令集参考
- [B. CSR Register Map](appendices/csr_map.md) - CSR 寄存器映射
- [C. Memory Map](appendices/memory_map.md) - 内存映射
- [D. Design Parameters](appendices/parameters.md) - 设计参数
- [E. Glossary](appendices/glossary.md) - 术语表

---

## 使用指南

### 阅读建议

**初学者**:
1. 从 [Introduction](01_introduction/README.md) 开始，了解设计背景
2. 阅读 [Top-Level Architecture](02_top_level/README.md)，建立全局视图
3. 根据兴趣选择深入的模块（标量/向量/矩阵）

**硬件工程师**:
1. 快速浏览 Introduction 和 Top-Level
2. 深入阅读你关注的子系统（如内存子系统）
3. 参考 Appendices 中的寄存器映射和参数配置

**研究人员**:
1. 重点关注 [Vector Core](06_vector_core/README.md) 和 [Matrix Core](07_matrix_core/README.md)
2. 阅读 [Performance Analysis](10_performance/analysis.md) 了解性能特征
3. 研究 Stripmining 和外积 MAC 等创新设计

### AI 辅助分析工作流

本文档基于 AI 辅助的系统化代码分析方法生成。如果你想复现或扩展分析：

1. **克隆仓库**: 
   ```bash
   git clone https://github.com/google-coral/coralnpu.git
   ```

2. **使用 AI 提问模板** (见 [MICROARCH_ANALYSIS_PLAN.md](../MICROARCH_ANALYSIS_PLAN.md)):
   - 模块分析模板
   - 时序分析模板
   - 互联分析模板

3. **迭代改进**:
   - 基于 AI 生成的初稿
   - 人工审查和校正
   - 交叉验证多个代码文件

---

## 图表说明

- **框图**: 使用 Mermaid 生成，可在 GitHub 和支持 Mermaid 的编辑器中直接渲染
- **时序图**: 部分使用 WaveDrom JSON 格式
- **PNG 图片**: 复杂图表和原始设计文档中的截图

---

## 贡献指南

欢迎贡献改进和补充：

1. **纠错**: 如果发现技术错误，请提交 Issue 或 PR
2. **补充**: 添加更多的示例、Walkthrough 或性能数据
3. **优化**: 改进图表清晰度或文字表述

---

## 许可证

本文档基于 [Apache 2.0 License](../LICENSE) 发布，与 Coral NPU 项目保持一致。

---

## 致谢

- **Google Research**: 开源 Coral NPU 项目
- **RISC-V 社区**: 提供标准 ISA 基础
- **Saturn Vectors**: 提供优秀的微架构文档范例

---

**开始阅读**: [Introduction →](01_introduction/README.md)

