## 性能指标（Performance Metrics）

收集各种性能事件在性能分析中非常有帮助。然而，这里有一个注意点。假设你运行了一个程序并收集了 `MEM_LOAD_RETIRED.L3_MISS` 事件（用于统计 LLC 未命中次数），结果显示为十亿次。这听起来确实很多，于是你决定调查这些缓存未命中来自哪里。等等！你确定这是一个问题吗？如果程序只执行了二十亿次加载，那么确实是个问题，因为一半的加载都在 LLC 中未命中。但相反，如果程序执行了一万亿次加载，那么每一千次加载中只有一次导致 L3 缓存未命中。

这就是为什么除了硬件性能事件之外，性能工程师（performance engineer）还经常使用指标（metrics）——这些指标是在原始事件基础上构建的。表 perf_metrics 展示了 Intel 第 12 代 Golden Cove 架构的指标列表，以及描述和公式。该列表并不详尽，但展示了最重要的指标。Intel CPU 完整指标列表及其公式可以在 [TMA_metrics.xlsx](https://github.com/intel/perfmon/blob/main/TMA_Metrics.xlsx) 中找到。[^1] [PerfMetricsCaseStudy] 展示了如何在实践中使用性能指标。


--------------------------------------------------------------------------
指标名称  描述                          公式
------- -------------------------- ---------------------------------------
L1MPKI  每千条指令的 L1 缓存真实       1000 * MEM_LOAD_RETIRED.L1_MISS_PS /
        未命中次数（针对已退休的        INST_RETIRED.ANY
        按需加载）。

L2MPKI  每千条指令的 L2 缓存真实       1000 * MEM_LOAD_RETIRED.L2_MISS_PS /
        未命中次数（针对已退休的        INST_RETIRED.ANY
        按需加载）。

L3MPKI  每千条指令的 L3 缓存真实       1000 * MEM_LOAD_RETIRED.L3_MISS_PS /
        未命中次数（针对已退休的        INST_RETIRED.ANY
        按需加载）。

Branch  所有分支中预测错误的比率        BR_MISP_RETIRED.ALL_BRANCHES /
Mispr.                               BR_INST_RETIRED.ALL_BRANCHES
Ratio

Code    每千条指令的 STLB（第二级      1000 * ITLB_MISSES.WALK_COMPLETED
STLB    TLB）推测性代码未命中次数      / INST_RETIRED.ANY
MPKI    （完成页表遍历的任意页大小
        未命中）

Load    每千条指令的 STLB 数据加载     1000 * DTLB_LD_MISSES.WALK_COMPLETED
STLB    推测性未命中次数               / INST_RETIRED.ANY
MPKI

Store   每千条指令的 STLB 数据存储     1000 * DTLB_ST_MISSES.WALK_COMPLETED
STLB    推测性未命中次数               / INST_RETIRED.ANY
MPKI

Load    L1 D-cache 未命中按需加载操    L1D_PEND_MISS.PENDING /
Miss    作的平均延迟（以核心周期计）    MEM_LD_COMPLETED.L1_MISS_ANY
Real
Latency

ILP     每核心的指令级并行度           UOPS_EXECUTED.THREAD /
        （有执行时平均执行的            UOPS_EXECUTED.CORE_CYCLES_GE1,
        $\mu$ops 数量）               若启用 SMT 则除以 2

MLP     每线程的内存级并行度           L1D_PEND_MISS.PENDING /
        （至少有一个 L1 未命中按需     L1D_PEND_MISS.PENDING_CYCLES
        加载时的平均未命中数量）

DRAM    读写操作的平均外部内存带宽      ( 64 * ( UNC_M_CAS_COUNT.RD +
BW Use  使用量（GB/s）                          UNC_M_CAS_COUNT.WR )
GB/sec                               / 1GB ) / Time

IpCall  每次近调用（near call）的       INST_RETIRED.ANY /
        指令数（数字越小表示发生率      BR_INST_RETIRED.NEAR_CALL
        越高）

Ip      每个分支的指令数               INST_RETIRED.ANY /
Branch                               BR_INST_RETIRED.ALL_BRANCHES

IpLoad  每次加载的指令数               INST_RETIRED.ANY /
                                     MEM_INST_RETIRED.ALL_LOADS_PS

IpStore 每次存储的指令数               INST_RETIRED.ANY /
                                     MEM_INST_RETIRED.ALL_STORES_PS

IpMisp  每次非推测性分支预测错误的      INST_RETIRED.ANY /
redict  指令数                        BR_MISP_RETIRED.ALL_BRANCHES

IpFLOP  每次 FP（浮点）操作的指令数    参见 TMA_metrics.xlsx

IpArith 每次 FP 算术指令的指令数       参见 TMA_metrics.xlsx

IpArith 每次 FP 算术标量单精度         INST_RETIRED.ANY /
Scalar  指令的指令数                   FP_ARITH_INST.SCALAR_SINGLE
SP

IpArith 每次 FP 算术标量双精度         INST_RETIRED.ANY /
Scalar  指令的指令数                   FP_ARITH_INST.SCALAR_DOUBLE
DP

Ip      每次算术 AVX/SSE              INST_RETIRED.ANY / (
Arith   128 位指令的指令数             FP_ARITH_INST.128B_PACKED_DOUBLE+
AVX128                               FP_ARITH_INST.128B_PACKED_SINGLE)

Ip      每次算术 AVX*                 INST_RETIRED.ANY / (
Arith   256 位指令的指令数             FP_ARITH_INST.256B_PACKED_DOUBLE+
AVX256                               FP_ARITH_INST.256B_PACKED_SINGLE)

Ip      每次软件预取指令（任意类型）    INST_RETIRED.ANY /
SWPF    的指令数                      SW_PREFETCH_ACCESS.T0:u0xF
--------------------------------------------------------------------------

表：Intel Golden Cove 架构的性能指标列表（不完整）及其描述和公式。

关于这些指标，有几点说明。首先，ILP 和 MLP 指标并不代表应用程序的理论最大值；它们衡量的是应用程序在特定机器上的实际 ILP 和 MLP。在拥有无限资源的理想机器上，这些数字会更高。其次，除"DRAM BW Use"和"Load Miss Real Latency"之外的所有指标都是分数；我们可以对每个指标进行相当直接的推理，判断某个指标是高还是低。但要理解"DRAM BW Use"和"Load Miss Real Latency"指标，我们需要结合上下文。对于前者，我们想知道程序是否使内存带宽饱和。后者给出了缓存未命中平均代价的概念，单独来看意义不大，除非你了解缓存层次结构中每个组件的延迟。我们将在下一节讨论如何确定缓存延迟和峰值内存带宽。

一些工具可以自动报告性能指标。如果工具没有这个功能，你始终可以手动计算这些指标，因为你知道公式和必须收集的相应性能事件。表 perf_metrics 提供了 Intel Golden Cove 架构的公式，但只要底层性能事件可用，你就可以在其他平台上构建类似的指标。

[^1]: TMA 指标 - [https://github.com/intel/perfmon/blob/main/TMA_Metrics.xlsx](https://github.com/intel/perfmon/blob/main/TMA_Metrics.xlsx)。
