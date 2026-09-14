# 实习经历：基于 DWT 的插桩模块

## 背景

在软件最差运行时间分析中，有三种分析方法。一种是静态分析，针对目标码进行反汇编，根据汇编指令、时钟周期等计算最差路径和最差运行时间，这种结果是悲观的，实际运行往往不会有这么慢，但这也能找出一条值得分析的最差路径；第二种是动态分析，这种方法旨在实际测量各种路径上的运行时间，但我们无法知道自己测量的是否是最差的，一般采用以大量的测量次数和尝试各种路径增加结果的可信度；

而第三种方法就是混合分析，这种方法融合了静态分析和动态分析的优点，先通过静态分析得到一条最差路径，再使用动态分析去实际测量最差路径的实际运行时间，使得得到的结果是基于实际的而且并不那么悲观。

本项目就是基于这种混合分析方法的软件最差运行时间分析工具的插桩模块，针对工具利用静态分析方法得出的最差路径去定点插桩，测量出真实的最差时间。

## 目标

设计并实现一个 **插桩模块**，为混合 WCET 分析提供分段实测数据。

1. **单一职责**：只测量非中断任务的耗时；中断干扰、CRPD、系统级响应时间一概不做。
2. **高精度**：基于 DWT_CYCCNT，精度达单个 CPU 周期。
3. **低开销**：最小化插桩本身的开销
4. **零依赖**：不依赖 OS、定时器、中断服务和标准库，可在裸机环境直接使用。
5. **可移植**：仅依赖 Cortex-M 通用 DWT 寄存器，适用于 M3/M4 等内核。

## DWT 计数器

DWT（Data Watchpoint and Trace）是ARM Cortex-M处理器的调试组件，提供：

- CYCCNT：周期计数器，记录CPU执行的时钟周期数
- CPICNT：CPI（Cycles Per Instruction）计数器
- EXCCNT：异常开销计数器
- SLEEPCNT：睡眠周期计数器
- LSUCNT：加载/存储单元计数器
- FOLDCNT：指令折叠计数器

### DWT 寄存器映射

DWT基地址：0xE0001000

关键寄存器：
- DWT_CTRL     (0xE0001000): 控制寄存器
- DWT_CYCCNT   (0xE0001004): 周期计数器
- DWT_CPICNT   (0xE0001008): CPI计数器
- DWT_EXCCNT   (0xE000100C): 异常计数器
- DWT_SLEEPCNT (0xE0001010): 睡眠计数器
- DWT_LSUCNT   (0xE0001014): LSU计数器
- DWT_FOLDCNT  (0xE0001018): 折叠计数器

本模块只用 **CYCCNT** 一个。其余几个是 DWT 的完整能力，列出仅供参照。

## 架构

驱动层：初始化 DWT，配置，读取
测量：热路径为宏（展开在被测代码里，无函数调用）；隔离测量负责把中断挡在窗口外
输出层：串口

## 详细说明

### 数据结构与接口提供
````c
/**
 * @file wcet_profiler.h
 * @brief 最坏运行时间（WCET）插桩模块
 *
 */

#ifndef WCET_PROFILER_H
#define WCET_PROFILER_H

/* ===================== 基础类型定义 ===================== */

/*
 * 本模块不包含任何标准库头文件（<stdint.h> / <stdbool.h> / <stddef.h>），
 * 所需的定宽类型在此自行给出。
 *
 * 注意：定宽类型无法只靠标准 C 语法"推导"，必须依赖平台整数模型。
 * 本文档面向 Cortex-M（ARM 32 位 ABI）：
 *     char = 8 bit, short = 16 bit, int = 32 bit, long long = 64 bit
 */
typedef unsigned char      uint8_t;
typedef unsigned int       uint32_t;
typedef unsigned long long uint64_t;

/* <stdbool.h>：C99 之前没有 _Bool，用无符号字符模拟（true/false 是宏，不是关键字） */
typedef unsigned char bool;
#define true   1
#define false  0

/* <stddef.h> 的 NULL */
#define NULL ((void *)0)

/* ===================== 配置参数 ===================== */

#define WCET_MAX_COUNTERS       32      // 最大计数器数量
#define WCET_NAME_MAX_LEN       32      // 名称最大长度

/* ===================== DWT 寄存器定义 ===================== */

#define DWT_CTRL                (*(volatile uint32_t *)0xE0001000)
#define DWT_CYCCNT              (*(volatile uint32_t *)0xE0001004)
#define CoreDebug_DEMCR         (*(volatile uint32_t *)0xE000EDFC)     //调试异常与监控控制寄存器

/* ===================== 控制位 ===================== */

/* CoreDebug->DEMCR：跟踪总开关 */
#define CoreDebug_DEMCR_TRCENA  (1u << 24)

/* DWT->CTRL：CYCCNT 使能位 */
#define DWT_CTRL_CYCCNTENA      (1u << 0)

/* ===================== 数据结构 ===================== */

/**
 * @brief 单个 WCET 计数器
 *
 * 只有一份数据。测量在屏蔽中断的条件下进行，样本不存在"干净 / 被抢占"之分，
 * 也就不需要维护第二组统计量。字段按宽度排列消除填充，整体 48 字节。
 */
typedef struct {
    char     name[WCET_NAME_MAX_LEN]; // 计数器名称，即软件测量点的名称
    uint32_t addr;                    // 测量段的标签地址，供静态工具按地址对齐
    uint32_t start;                   // 进入时的 CYCCNT——测量窗口的起点
    uint32_t max_cycles;              // 观测到的最大耗时——替换静态时间模型的输入
    uint32_t call_count;              // 样本总数——判断上面那个最大值是否可信
} wcet_counter_t;

/**
 * @brief WCET 分析器
 */
typedef struct {
    wcet_counter_t counters[WCET_MAX_COUNTERS];  // 所有测量点
    uint32_t offset_cycles;           // 复核出的插桩固定开销（周期）
    uint32_t cpu_freq_mhz;            // CPU 主频，用于周期换算时间
    uint8_t  counter_count;           // 已注册数量
    bool     initialized;             // 初始化标志
    bool     calibrated;              // 是否已完成开销复核
} wcet_analyzer_t;

/* ===================== 基础接口 ===================== */

/**
 * @brief 初始化 DWT CYCCNT
 * @param analyzer    分析器句柄
 * @param cpu_freq_mhz CPU 主频（MHz）
 * @return true 成功；false 表示内核不支持 CYCCNT
 */
bool wcet_init(wcet_analyzer_t *analyzer, uint32_t cpu_freq_mhz);

/**
 * @brief 注册一个 WCET 计数器
 * @return 计数器指针；分析器未初始化或槽位已满时返回 NULL
 *
 * 返回指针而不是索引，这样调用点拿到后一直用它，热路径上不再有索引换算。
 */
wcet_counter_t *wcet_register(wcet_analyzer_t *analyzer, const char *name);

/**
 * @brief 按名称查找计数器索引
 * @return 索引，>=0 成功；-1 未找到
 */
int wcet_find(wcet_analyzer_t *analyzer, const char *name);

/**
 * @brief 按名称取计数器指针（= wcet_find + 取地址）
 * @return 指针；未找到时返回 NULL
 */
wcet_counter_t *wcet_get(wcet_analyzer_t *analyzer, const char *name);

/* ===================== 混合分析接口 ===================== */

/**
 * @brief 为某个测量点登记段地址
 *
 * 地址供静态分析工具把实测数据与它的基本块对齐。建议直接传标签地址，例如：
 *     wcet_set_addr(pid, (uint32_t)&&lbl_pid);
 *
 * 注意：addr 属于配置而非统计量——初始化时登记一次，之后不再改动。
 */
void wcet_set_addr(wcet_counter_t *c, uint32_t addr);

/**
 * @brief 复核插桩固定开销
 *
 * 空窗读两次 CYCCNT，差值就是插桩自身的固定开销。取最小值：任何外部干扰只会
 * 抬高结果，取最小最接近真实值。
 *
 * @param iterations 采样次数，建议 >= 1000
 * @return true 至少取到一个有效样本；false 表示参数非法
 */
bool wcet_calibrate(wcet_analyzer_t *analyzer, uint32_t iterations);

/* ===================== 结果查询 ===================== */

/** 获取观测到的最大耗时（周期）——WCET 分析的主输出 */
uint32_t wcet_get_max_cycles(const wcet_analyzer_t *analyzer, int index);

/** 获取样本总数——判断上面那个最大值是否可信的依据 */
uint32_t wcet_get_call_count(const wcet_analyzer_t *analyzer, int index);

/** 获取复核出的插桩固定开销（周期） */
uint32_t wcet_get_offset_cycles(const wcet_analyzer_t *analyzer);

/* ===================== 隔离测量：把中断挡在窗口外 ===================== */

/*
 * 在 ARM Cortex-M 中，中断优先级数值越小，优先级越高。
 * BASEPRI 的作用是：屏蔽掉优先级数值大于等于 threshold 的中断。也就是说，只有优先级数值比 threshold 小的中断（即更高优先级的中断）才能打断当前执行。
 * 如果传入 threshold = 0x20，那么优先级为 0x20 到 0xFF 的中断都会被屏蔽。但优先级 0（如 NMI、HardFault、部分高优先级中断）不受 BASEPRI 影响，依然可以打断。
 */
static inline uint32_t wcet_irq_disable(uint32_t threshold)
{
    uint32_t saved;
     // 1. 读取当前的 BASEPRI 寄存器值，保存到 saved 变量中
    __asm volatile("mrs %0, basepri" : "=r"(saved));
     // 2. 将传入的 threshold（阈值）写入 BASEPRI 寄存器
    __asm volatile("msr basepri, %0" :: "r"(threshold) : "memory");
    // 3. 返回备份的旧 BASEPRI 值，以便后续恢复现场
    return saved;
}

/** 还原 BASEPRI 现场 */
static inline void wcet_irq_restore(uint32_t saved)
{
    // 将之前备份的 BASEPRI 值重新写回寄存器，恢复原有的中断屏蔽状态
    __asm volatile("msr basepri, %0" :: "r"(saved) : "memory");
}

/* ===================== 便捷宏（热路径） ===================== */

/* 编译器屏障，防止编译器重排内存访问指令 */
#define WCET_COMPILER_BARRIER()   __asm volatile("" ::: "memory")

/* 起始点，打下时间戳 */
#define WCET_BEGIN_P(c)                                            \
    do {                                                           \
        (c)->start = DWT_CYCCNT;                  /* ← 窗口起点 */  \
        WCET_COMPILER_BARRIER();                                   \
    } while (0)

/* 结算时间戳，算出耗时，并更新统计信息。*/
#define WCET_END_P(c)                                              \
    do {                                                           \
        WCET_COMPILER_BARRIER();                                   \
        uint32_t _wcet_d = DWT_CYCCNT - (c)->start;  /* ← 窗口终点 */\
        if (_wcet_d > (c)->max_cycles) {                           \
            (c)->max_cycles = _wcet_d;                             \
        }                                                          \
        (c)->call_count++;                                         \
    } while (0)

/*
 * 只计数、不计时的标记点。
 *
 * 用来验证一次运行里"想测的那条路径到底跑没跑"。放在最差路径的入口，跑完看
 * wcet_get_call_count()——是 0 就说明这条路径一次都没走到，同一次运行里采到的
 * 段数据全部作废。
 *
 * 段计时的 call_count 只说明那段代码进了多少次，不说明走的是哪条分支，回答不了
 * 这个问题。
 *
 * 不读 CYCCNT，只有一次自增。给标记点单独用一个计数器，不要和计时的混用。
 */
#define WCET_COUNT_P(c)     do { (c)->call_count++; } while (0)

#endif /* WCET_PROFILER_H */

````

### 辅助函数

````c
#include "wcet_profiler.h"

/* ===================== 内部辅助 ===================== */

static void copy_name(char *dst, const char *src)
{
    uint32_t i = 0;
    if (src == NULL) {
        dst[0] = '\0';
        return;
    }
    while (src[i] != '\0' && i < (WCET_NAME_MAX_LEN - 1)) {
        dst[i] = src[i];
        i++;
    }
    dst[i] = '\0';
}

/**
 * @brief 内存填充（等价于标准库 memset）
 * @param dst   目标内存首地址，为 NULL 时直接返回 NULL
 * @param value 填充值（取低 8 位）
 * @param len   要填充的字节数
 * @return dst
 */
static void *wcet_memset(void *dst, int value, uint32_t len)
{
    uint8_t *p = (uint8_t *)dst;

    if (dst == NULL) {
        return NULL;
    }

    while (len-- > 0u) {
        *p++ = (uint8_t)value;
    }
    return dst;
}

/**
 * @brief 字符串比较（等价于标准库 strncmp）
 *
 * 语义与标准库一致：
 *   - 最多比较 n 个字符；
 *   - 遇到 '\0' 或不同字符即停止；
 *   - 返回 <0 / 0 / >0 表示 s1 小于 / 等于 / 大于 s2。
 *
 * NULL 保护：将 NULL 视为空字符串，避免解引用崩溃。
 *
 * @param s1 字符串 1
 * @param s2 字符串 2
 * @param n  最大比较字符数
 * @return 比较结果
 */
static int wcet_strncmp(const char *s1, const char *s2, uint32_t n)
{
    /* NULL 保护 */
    if (s1 == NULL && s2 == NULL) {
        return 0;
    }
    if (s1 == NULL) {
        return (*s2 == '\0') ? 0 : -1;
    }
    if (s2 == NULL) {
        return (*s1 == '\0') ? 0 : 1;
    }

    while (n-- > 0u) {
        unsigned char c1 = (unsigned char)*s1++;
        unsigned char c2 = (unsigned char)*s2++;

        if (c1 != c2) {
            return (c1 < c2) ? -1 : 1;
        }
        if (c1 == '\0') {
            return 0;   /* 同时到达字符串结尾 */
        }
    }
    return 0;           /* 前 n 个字符完全相同 */
}

````

### 驱动层

````c
/* ===================== 初始化 ===================== */

bool wcet_init(wcet_analyzer_t *analyzer, uint32_t cpu_freq_mhz)
{
    if (analyzer == NULL) {
        return false;
    }

    wcet_memset(analyzer, 0, sizeof(wcet_analyzer_t));
    analyzer->cpu_freq_mhz = cpu_freq_mhz;

    /* 1. 打开跟踪总开关 */
    CoreDebug_DEMCR |= CoreDebug_DEMCR_TRCENA;

    /* 2. 清零 CYCCNT 并使能 */
    DWT_CYCCNT = 0;
    DWT_CTRL  |= DWT_CTRL_CYCCNTENA;

    /* 3. 检测 CYCCNT 是否工作（部分 M0/M0+ 无此计数器） */
    DWT_CYCCNT = 0;
    if (DWT_CYCCNT == 0) {
        volatile uint32_t i;
        for (i = 0; i < 100; i++) { __asm volatile("nop"); }
        if (DWT_CYCCNT == 0) {
            analyzer->initialized = false;
            return false;
        }
    }

    /* 计数器数组已由上面的 memset 整体清零：call_count / max_cycles 等统计量
     * 全部归零。没有 min 类统计需要预置成最大值，因此这里不再需要逐个初始化
     * 的循环。 */

    analyzer->offset_cycles = 0u;
    analyzer->calibrated    = false;

    analyzer->initialized = true;
    return true;
}

/* ===================== 注册 / 查找 ===================== */

wcet_counter_t *wcet_register(wcet_analyzer_t *analyzer, const char *name)
{
    if (analyzer == NULL || !analyzer->initialized) {
        return NULL;
    }
    if (analyzer->counter_count >= WCET_MAX_COUNTERS) {
        return NULL;
    }

    wcet_counter_t *c = &analyzer->counters[analyzer->counter_count];

    copy_name(c->name, name);

    c->start      = 0u;
    c->max_cycles = 0u;
    c->call_count = 0u;
    c->addr       = 0u;                   /* 待 wcet_set_addr 填写 */

    analyzer->counter_count++;
    return c;
}

int wcet_find(wcet_analyzer_t *analyzer, const char *name)
{
    if (analyzer == NULL || name == NULL) {
        return -1;
    }
    for (uint8_t i = 0; i < analyzer->counter_count; i++) {
        if (wcet_strncmp(analyzer->counters[i].name, name,
                    WCET_NAME_MAX_LEN) == 0) {
            return (int)i;
        }
    }
    return -1;
}

wcet_counter_t *wcet_get(wcet_analyzer_t *analyzer, const char *name)
{
    int index = wcet_find(analyzer, name);
    return (index < 0) ? NULL : &analyzer->counters[index];
}

/* ===================== 混合分析：开销复核 / 地址 ===================== */

bool wcet_calibrate(wcet_analyzer_t *analyzer, uint32_t iterations)
{
    if (analyzer == NULL || !analyzer->initialized) return false;
    if (iterations == 0u) return false;

    /* 空窗复核：两次相邻的 CYCCNT 读之间不写任何用户代码，差值就是插桩自身的
     * 固定开销。取最小值而非均值——外部干扰只会抬高结果，取最小最接近真实值。
     *
     * 改造后窗口内已不含任何记账代码，正常结果应当只有几个周期；若它明显偏大，
     * 说明有编译器 / 链接器层面的东西插进了窗口——这正是保留本函数的意义：
     * 它是窗口干净与否的体检。 */
    uint32_t best = 0xFFFFFFFFu;

    for (uint32_t i = 0; i < iterations; i++) {
        uint32_t t0 = DWT_CYCCNT;
        uint32_t t1 = DWT_CYCCNT;       /* 两条 volatile 读不会被合并 */
        uint32_t d  = t1 - t0;

        if (d < best) { best = d; }
    }

    analyzer->offset_cycles = best;
    analyzer->calibrated    = true;
    return true;
}

void wcet_set_addr(wcet_counter_t *c, uint32_t addr)
{
    if (c == NULL) return;

    c->addr = addr;
}
````

**注册负责给每个桩点分配槽位并返回它的指针，查找负责按名字把同一个指针找回来；二者共同管理桩，
提升代码的可读性可维护性。热路径在注册后就把指针存好，此后测量点连索引换算都省掉。**
````txt
┌─────────────────────────────────────────────────┐
│ 1. wcet_init()          初始化 DWT + 清空分析器  │
│    可选的 wcet_calibrate()  复核插桩开销接近 0   │
└──────────────────────┬──────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────┐
│ 2. wcet_counter_t *pid =                        │
│        wcet_register(&wcet, "PID");             │
│    wcet_counter_t *spi =                        │
│        wcet_register(&wcet, "SPI");             │
│    wcet_counter_t *adc =                        │
│        wcet_register(&wcet, "ADC");             │
│    （测量点在此刻被"登记"，名字与槽位绑定，        │
│      返回的就是它的指针，后续热路径一直用它）      │
└──────────────────────┬──────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────┐
│ 3. 运行阶段（隔离测量，中断挡在窗口外）：          │
│                                                 │
│    uint32_t s = wcet_irq_disable(0x40u);        │
│    WCET_BEGIN_P(pid);        ← 窗口起点          │
│    PID_Calc();                                  │
│    WCET_END_P(pid);          ← 窗口终点          │
│    wcet_irq_restore(s);                         │
└──────────────────────┬──────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────┐
│ 4. 查询（getter 收索引，索引用名字换）：          │
│    int k = wcet_find(&wcet, "PID");             │
│    wcet_get_max_cycles(&wcet, k);               │
│    wcet_get_call_count(&wcet, k);               │
│    （导给工具另需 addr 与 offset_cycles）         │
└─────────────────────────────────────────────────┘
````

### 结果查询

````c
uint32_t wcet_get_max_cycles(const wcet_analyzer_t *analyzer, int index)
{
    if (analyzer == NULL || index < 0 ||
        index >= (int)analyzer->counter_count) return 0;
    return analyzer->counters[index].max_cycles;
}

uint32_t wcet_get_call_count(const wcet_analyzer_t *analyzer, int index)
{
    if (analyzer == NULL || index < 0 ||
        index >= (int)analyzer->counter_count) return 0;
    return analyzer->counters[index].call_count;
}

uint32_t wcet_get_offset_cycles(const wcet_analyzer_t *analyzer)
{
    if (analyzer == NULL) return 0;
    return analyzer->offset_cycles;
}
````

### 完整流程

  1. 静态工具给出最差路径，你从中挑出要实测的段（通常是怀疑静态模型估偏的那些：Flash 取指、cache 未命中、除法）
  2. 建一个特性化固件（单独的构建或单独的入口），wcet_init + 逐段 wcet_register / wcet_set_addr，wcet_calibrate 确认 offset_cycles
     接近 0
  3. 在最差路径的入口再登记一个标记点，用 WCET_COUNT_P 统计这条路径被进入的次数
  4. 驱动最差路径执行 N 次，每次经过测量点时自动隔离 + 计时
  5. 导出：先查标记点的 call_count——为 0 说明这条路径一次都没走到，本次数据作废；
     否则遍历各段读 addr / max_cycles / call_count，导成 CSV 交给静态工具
  6. 工具逐块替换时间模型，沿原路径重新求和

## 插桩开销

### 时间开销

  1. 窗口内：≈ 2 周期

  设计的核心是让 CYCCNT 的读取贴在窗口边界上：

  /* BEGIN：读是最后一步，后面的 store 落在窗口外 */
  (c)->start = DWT_CYCCNT;        /* ← 窗口起点 */
  /* END：读是第一步，后面所有记账落在窗口外 */
  uint32_t _wcet_d = DWT_CYCCNT - (c)->start;   /* ← 窗口终点 */

  所以两个读取点之间没有一条插桩指令，只有用户代码。

  但"零"是不到达的：窗口的物理定义是"BEGIN 那条 ldr 被 CYCCNT 采样的那一刻"到"END 那条 ldr
  被采样的那一刻"，这两次采样之间至少隔着一条指令的取指/执行。这个不可约地板就是 wcet_calibrate 测的东西——它连续读两次
  CYCCNT，差值就是这个下限：

  asm
  ldr r2, [r3]     @ 2 周期（PPB 读）
  ldr r1, [r3]     @ 2 周期
  sub r0, r1, r2   @ 1

  offset_cycles ≈ 2，也就是报出来的 max_cycles 系统性偏高约 2 个周期。

### 内存开销

 1. RAM：1 548 字节（精确，可以从结构体布局算死）
这 1.5 KB 与测量次数无关。跑 10 次和跑 1000 万次占同样的 RAM，热路径的所有局部量都是单个uint32_t，全在寄存器里，一个字节栈都不占。模块里没有动态分配。
 2. Flash：2K以内
