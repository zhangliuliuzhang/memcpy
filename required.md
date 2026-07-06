# Specification: RISC-V + io_uring Memcached 重构

## Overview

将 Memcached 1.6.42 针对 RISC-V 架构和 Linux 6.0 的 io_uring 进行深度性能优化重构，融合 RISC-V 指令集扩展（V/B/A/Zicbom/Zihintpause）、Slab 机制深度优化、io_uring 异步网络引擎与内核协同技术，目标是吞吐量提升 3-5 倍，P99 延迟降低 50% 以上。

## Clarifications

### Session 2026-07-06

- Q: CXL 分层内存是否纳入本次规格？ → A: 不纳入，作为后续独立功能规划。
- Q: 8 项 Slab 优化 + 混合分配模式如何纳入？ → A: 第 1-8 项全部纳入（混合分配不纳入）。第 8 项（显式大页管理）强化原 F1.2。
- Q: 混合分配模式（Slab + Log-Structured）是否纳入本次？ → A: 不纳入，作为后续独立规格，本次聚焦 RISC-V + io_uring 主线。

## Problem Statement

原生 Memcached 设计于 x86 架构和传统 epoll 时代，存在以下性能瓶颈：

1. **内存布局未对齐**：item/slab 结构体未按 RISC-V cache line（64字节）对齐，导致伪共享和跨缓存行惩罚
2. **Slab 页尺寸不匹配**：默认 1MB 无法利用 RISC-V Sv39/Sv48 的 2MB Mega-page 巨页优化
3. **内存清零低效**：重用 Slab 块时的 `memset()` 无法利用 RISC-V Zicbom 扩展的 `cbo.zero` 硬件指令
4. **计算热点未向量化**：哈希计算、协议解析、位图扫描等热点未利用 RISC-V V 扩展和 B 扩展
5. **网络 I/O 开销大**：libevent/epoll 模型存在频繁系统调用、用户态-内核态数据拷贝、事件重注册开销
6. **锁竞争与尾延迟**：高并发下全局锁竞争激烈，缺乏 RISC-V Zihintpause 指令优化自旋等待
7. **Slab 内存钙化**：各 Slab Class 之间内存无法弹性流转，长生命周期对象"锁死"整个 Page，引发系统颠簸
8. **置换策略次优**：LRU 局限在单个 Slab Class 内部，无法实现全局最优化置换
9. **原子操作低效**：refcount/time 字段更新依赖传统锁机制或 CAS 循环，无法利用 RV64A 原子指令的硬件优势
10. **TTL 混合污染**：短命与长命对象混在同一 Page，无法实现整页秒级回收

## User Stories

### 作为基础设施运维工程师

1. 我希望在 RISC-V 服务器上部署高性能 Memcached，以便支撑高并发缓存服务
2. 我希望在不修改应用代码的前提下获得 3-5 倍吞吐提升，以便降低硬件成本
3. 我希望看到详细的性能基准报告，以便验证优化效果并向上级汇报

### 作为性能工程师

4. 我希望网络 I/O 延迟尽可能低，以便提升系统整体吞吐量
5. 我希望哈希计算和协议解析占用的 CPU 时间更少，以便释放更多资源给业务处理
6. 我希望看到分阶段的性能对比数据，以便评估每个优化阶段的 ROI

### 作为系统开发者

7. 我希望代码保持可读性和可维护性，以便后续迭代和调试
8. 我希望重构分阶段进行，以便在出现问题时能快速定位回退
9. 我希望 Slab 子系统的"钙化"问题得到根治，以便长生命周期对象不再霸占内存页
10. 我希望热点路径的并发控制开销最小化，以便消除高并发下的总线争用

## Dependencies

### Phase 依赖关系图

```
Phase 1（内存基底）
    ↓ 提供 2MB 大页 + NUMA 亲和 + Zicbom 清零
Phase 2（计算向量化）
    ↓ 提供快速哈希 + 协议解析
Phase 3（io_uring 网络引擎）← 必须在 Phase 2.5 之前完成
    ↓ 提供零拷贝 DMA + SQE/CQE 派发
Phase 4（尾延迟优化）
    ↓ 提供 pause + MADV_COLLAPSE
Phase 2.5（Slab 深度优化）← 依赖 Phase 3 的内存布局稳定
    ↓ 提供 Defrag + 借调 + 全局置换
Phase 4.5（RV64A 原子操作）
    ↓ 提供 amoadd + amomax + lr/sc
```

### 关键依赖说明

| Phase | 依赖项 | 依赖原因 |
|-------|--------|---------|
| Phase 2.5 → Phase 3 | Phase 3 必须先完成 | Defrag 移动 Item 会改变 Slab 地址，而 PBUF_RING 的 DMA 目标地址必须稳定。若顺序颠倒，DMA 写入可能指向已迁移的旧地址。 |
| Phase 4.5 → Phase 1 | Phase 1 的 item 结构体对齐完成 | RV64A 原子操作（lr.d/sc.d）要求操作地址 8 字节对齐，Phase 1 的 item 字段对齐是前置条件。 |
| Phase 3 → Phase 1 | Phase 1 的 2MB 大页完成 | PBUF_RING 注册要求缓冲区物理连续且对齐 2MB，Phase 1 的显式大页管理是前置条件。 |

### 并行执行建议

- **Phase 1、Phase 2、Phase 4** 可以并行开发（无相互依赖）
- **Phase 3** 必须在 Phase 2.5 之前完成并稳定
- **Phase 4.5** 可与 Phase 2.5 并行，但需等待 Phase 1 的 item 对齐完成

## Functional Requirements

### Phase 1: 内存基底与指令对齐

| ID | 需求 | 验收标准 |
|----|------|----------|
| F1.1 | item 结构体两级对齐 | **结构体级**：`sizeof(item) % 64 == 0`（cache line 对齐）；**字段级**：`next/prev/refcount/time` 均 8 字节对齐（适配 lr.d/sc.d）。无伪共享警告。 |
| F1.2 | Slab 页尺寸调整 + 显式大页管理 | **配置层**：`settings.slab_page_size == 2MB`；**实现层**：通过 `mmap(MAP_HUGETLB)` 显式申请大页，无锁队列管理空闲 Chunk。THP 内核停顿率 < 1%。 |
| F1.3 | 内存清零使用 Zicbom `cbo.zero` | 在支持 Zicbom 的 CPU 上，清零带宽 > 20GB/s（对比基线提升 2x+） |
| F1.4 | 创建 RISC-V 内联汇编封装头文件 | `riscv_intrinsics.h` 包含 cbo.zero/ctz/clz/pause |
| F1.5 | NUMA 亲和性 Slab 分配 | 空闲页池按 NUMA 节点拆分，Worker 线程优先从本地节点分配，跨节点访问率 < 5% |

### Phase 2: 计算热点矢量化

| ID | 需求 | 验收标准 |
|----|------|----------|
| F2.1 | MurmurHash3 使用 V 扩展向量化 | 单线程哈希吞吐 > 500MB/s（对比基线提升 2x+） |
| F2.2 | 协议解析 `\r\n` 扫描向量化 | ASCII 协议解析吞吐 > 1GB/s |
| F2.3 | Slab 空闲块扫描使用 B 扩展 ctz/clz | 位图扫描延迟 < 10ns/op |
| F2.4 | 编译选项包含 RVV 和 B 扩展 | `-march=rv64gcv_zba_zbb_zbs_zicbom` |

### Phase 2.5: Slab 机制深度优化

> **前置依赖**：Phase 3（io_uring）必须完成并稳定，确保 PBUF_RING 的 DMA 目标地址不再变化。

| ID | 需求 | 验收标准 |
|----|------|----------|
| F2.5.1 | 效用函数驱动重平衡 | 综合淘汰率/命中率/空闲率计算迁移增益，盲目迁移率 < 5% |
| F2.5.2 | 页压缩与条目迁移（Defrag） | 残余活跃 Item 复制到同类 Page 空槽，钉子户 Page 回收率 > 90%。**并发安全**：Defrag 移动期间引用计数正确性验证通过（无 Use-After-Free）。 |
| F2.5.3 | 运行时自适应 Slab Class 创建 | 动态采样 Value 大小分布，热点非标尺寸自动适配。**限制**：动态创建 Class 数量 <= 63（保持 `slabs_clsid` 为 uint8），内碎片率降低 30%+ |
| F2.5.4 | 相邻 Slab Class 空间借调 | A 满时存入 B 空槽并打标签，后台线程异步搬回，突发流量淘汰率降低 50%+ |
| F2.5.5 | 跨 Slab 全局借道置换 | 全局冷数据队列，A 满时回收其他 Slab 中极度寒冷/已过期 Item，全局命中率提升 5%+ |
| F2.5.6 | 基于 TTL 预分类 | 短命/长命对象隔离到不同 Page，短命 Page 整页秒级回收，过期清理 CPU 开销降低 80%+ |

### Phase 3: 异步网络引擎重写

| ID | 需求 | 验收标准 |
|----|------|----------|
| F3.1 | 用 io_uring 替换 libevent/epoll | 所有网络 I/O 通过 io_uring SQE/CQE 派发 |
| F3.2 | 启用 IORING_OP_RECV Multi-Shot | 内核持续推送套接字事件，无用户态 Re-arm |
| F3.3 | 注册 PBUF_RING 实现零拷贝 | 网卡 DMA 直接写入 Slab，无用户态拷贝（通过 perf/eBPF 验证） |
| F3.4 | 每个 worker 线程独立 io_uring 实例 | 无跨线程 ring 访问，锁竞争最小化 |

### Phase 4: 内核高级协同与尾延迟优化

| ID | 需求 | 验收标准 |
|----|------|----------|
| F4.1 | 自旋锁使用 Zihintpause 指令 | 自旋期间 CPU 功耗降低 15%+（通过 `perf stat -e power/energy-pkg/` 测量），超线程资源让出 |
| F4.2 | 冷 Slab 调用 madvise(MADV_COLLAPSE) | THP 聚合成功率 > 80% |
| F4.3 | 精简不必要的内存屏障 | fence 指令数量从 N 条减少到 M 条（通过 objdump 反汇编统计，基线 N 由 Phase 1 测量），无正确性问题 |

### Phase 4.5: RV64A 原子操作深度优化

> **前置依赖**：Phase 1 的 F1.1（item 字段 8 字节对齐）必须完成。

| ID | 需求 | 验收标准 |
|----|------|----------|
| F4.5.1 | 引用计数字段类型变更 | `refcount` 从 `unsigned short`（uint16）改为 `uint32_t`，以适配 `amoadd.w`（32 位原子加）。item 结构体布局变更已验证（sizeof 增加 2 字节，对齐目标不变）。 |
| F4.5.2 | 引用计数 RV64A 优化 | `incr_refcount` 用 `amoadd.w`（Relaxed）；`decr_refcount` 用 `amoadd.w.aqrl`（Acq-Rel），仅在计数归零时同步释放 |
| F4.5.3 | LRU 时间戳 amomax 优化 | `time` 字段（已为 uint32）用 `amomax.w.rl` 单指令完成"比较并写入"，热点 Key 写冲突在硬件层退化为并发读 |
| F4.5.4 | LRU 自旋锁重构 | `amoswap.w.aq` + `pause` 自旋，替代 `pthread_mutex`，自旋开销降低 60%+ |
| F4.5.5 | Bump Queue 无锁化 | `lr.d`/`sc.d` 独占预约实现无 ABA 问题的无锁入队，无需版本号计数器 |

## Non-Functional Requirements

### 性能指标

| 指标 | 基线目标 | 优化目标 |
|------|----------|----------|
| 吞吐量（GET/SET QPS） | 记录基线 | 提升 3-5 倍 |
| P99 延迟 | 记录基线 | 降低 50%+ |
| P999 延迟 | 记录基线 | 降低 60%+ |
| CPU Cache Miss 率 | 记录基线 | 降低 40%+ |
| 内存带宽利用率 | 记录基线 | 提升 2x+ |
| Slab 内碎片率 | 记录基线 | 降低 30%+ |
| 过期对象回收 CPU 开销 | 记录基线 | 降低 80%+ |

### 兼容性

- 保持 Memcached ASCII/Binary 协议完全兼容
- 保持现有配置参数语义（新增参数可选）
- 支持在非 RISC-V / 非 Linux 6.0 环境回退到传统实现

### 可维护性

- 每个阶段独立可测试、可回退
- 代码注释说明 RISC-V 特定优化点
- 性能基准脚本可重复执行

## Success Criteria

### 量化指标

1. **吞吐量提升**：在相同硬件配置下，GET/SET 混合负载 QPS 达到基线的 3-5 倍
2. **延迟改善**：P99 延迟降低 50% 以上，P999 延迟降低 60% 以上
3. **资源效率**：相同 QPS 下 CPU 占用降低 30% 以上
4. **零拷贝达成**：网络 I/O 路径无用户态-内核态数据拷贝（通过 perf/eBPF 验证）
5. **Slab 钙化治理**：钉子户 Page 回收率 > 90%，全局命中率提升 5%+
6. **原子操作优化**：热点路径原子操作延迟降低 60%+，无总线争用告警

### 质性指标

7. **协议兼容**：现有 Memcached 客户端无需修改即可使用
8. **渐进交付**：每个 Phase 完成后可独立部署验证
9. **可观测性**：提供详细的性能计数器输出（Slab 命中率、io_uring 深度、向量指令利用率、原子操作统计等）

## Prerequisites

### 硬件环境

| 前置条件 | 要求 | 降级策略 |
|---------|------|---------|
| RISC-V 64-bit | 支持 V/B/A/Zicbom/Zihintpause 扩展 | 检测缺失扩展时禁用对应优化，回退到 C 标准库实现 |
| Linux 内核 | >= 6.0 | < 6.0 时禁用 io_uring Multi-Shot/PBUF_RING，回退到 epoll |
| NUMA 拓扑 | >= 2 个节点 | 单节点时禁用 NUMA 亲和性优化，使用全局内存池 |
| 大页预留 | `vm.nr_hugepages >= N`（N 由内存容量计算） | 大页不足时回退到 4KB 页，性能降级 15-20% |

### 编译环境

| 前置条件 | 要求 | 降级策略 |
|---------|------|---------|
| GCC | >= 14.0 或 LLVM >= 18.0 | 低版本时禁用 RISC-V intrinsics，回退到 C 标准库 |
| io_uring 库 | liburing >= 2.4 | 缺失时回退到 libevent/epoll |

### 工作负载假设

- 以中小对象（< 1KB）为主的高并发读写场景（Memcached 典型用例）
- 使用 mutilate 或 memtier_benchmark 进行压测，perf 进行性能剖析

## Out of Scope

1. **Memcached 协议扩展**：不新增自定义命令或协议字段
2. **持久化功能**：不涉及外部存储（extstore）或重启恢复优化
3. **分布式特性**：不涉及集群、分片、复制等分布式功能
4. **安全功能**：不涉及 SASL、TLS 性能优化（保持现有实现）
5. **Proxy 模式**：不涉及 Memcached proxy 模式优化
6. **非 RISC-V 架构优化**：x86/ARM 特定优化不在本次重构范围
7. **Windows/FreeBSD 支持**：仅针对 Linux 6.0+ 平台
8. **CXL 分层内存**：冷数据分层路由作为后续独立功能规划
9. **混合分配模式（Slab + Log-Structured）**：大对象日志追加写入 + Copy-GC 子系统作为后续独立规格规划，本次仅保留纯 Slab 架构
10. **动态 Slab Class 数量突破 64**：保持 `slabs_clsid` 为 uint8，动态创建 Class 数量上限为 63（预留 1 个全局池）。如需突破，需独立规划 `slabs_clsid` 改为 uint16 的兼容性迁移。

## Risk Analysis

### 跨 Phase 并发风险

| 风险场景 | 涉及 Phase | 缓解措施 |
|---------|-----------|---------|
| Defrag 移动 Item 时，其他线程并发 decr_refcount | Phase 2.5 + Phase 4.5 | Defrag 移动前先 `incr_refcount`，移动完成后 `decr_refcount`，确保引用计数 >= 2 期间不会被释放 |
| PBUF_RING DMA 写入时，Defrag 迁移目标 Slab | Phase 3 + Phase 2.5 | **Phase 2.5 必须在 Phase 3 稳定后启动**；DMA 目标 Slab 在 io_uring CQE 返回后才可参与 Defrag |
| 动态 Slab Class 创建时，哈希表扩容 | Phase 2.5 + Phase 2 | 动态 Class 创建使用独立的 `slabclass_dyn[]` 数组，不修改原有 `slabclass[]` |

### 前置条件不满足风险

| 风险场景 | 缓解措施 |
|---------|---------|
| 大页预留不足 | 启动时检测大页可用量，不足时打印警告并回退到 4KB 页 |
| NUMA 节点数为 1 | 编译时检测 `libnuma`，运行时检测节点数，单节点时跳过 NUMA 亲和性初始化 |
| RISC-V 扩展缺失 | 通过 `getauxval(AT_HWCAP)` 检测，缺失时回退到 C 标准库实现 |
