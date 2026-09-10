# 基于 DWT 的插桩模块

## 背景

在实时嵌入式系统中，控制环、中断服务程序等关键代码段的执行时间直接影响系统的实时性与安全性。仅靠平均值或经验估算，无法保证系统在最坏情况下仍能满足截止时间要求。Cortex-M 内核自带 DWT 单元，其中的 CYCCNT 周期计数器可提供 1 个 CPU 周期精度 的计时能力，无需额外定时器或中断，是实现 WCET 测量最轻量的硬件基础。

## 目标

设计并实现一个 **WCET 插桩模块**，用于精确测量关键代码段的最坏运行时间：

1. **高精度**：基于 DWT_CYCCNT，精度达单个 CPU 周期。
2. **低开销**：插桩仅读写一个寄存器，不干扰被测量代码的时序特性。
3. **极简接口**：`WCET_BEGIN` / `WCET_END` 一对宏即可完成一次测量。
4. **统计能力**：记录调用次数、单次耗时、历史最大/最小/平均耗时，重点输出 **max_cycles**。
5. **防回绕**：利用无符号减法天然处理 32 位计数器回绕。
6. **零依赖**：不依赖 OS、定时器、中断，可在裸机环境直接使用。
7. **可移植**：仅依赖 Cortex-M 通用 DWT 寄存器，适用于 M3/M4 等内核。

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

## 架构

驱动层：初始化 DWT，配置，读取
测量：提供开始、停止、统计的接口
输出层：RTT or 串口

## 详细说明

### 数据结构与接口提供
````c
/**
 * @file wcet_profiler.h
 * @brief 最坏运行时间（WCET）插桩模块
 *
 * 仅使用 Cortex-M DWT 的 CYCCNT 周期计数器，
 * 用于精确测量代码段的最坏运行时间。
 */

#ifndef WCET_PROFILER_H
#define WCET_PROFILER_H

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
 * 只记录与"耗时"相关的量，不含 CPI/异常/睡眠等分析字段。
 */
typedef struct {
    char     name[WCET_NAME_MAX_LEN]; // 计数器名称，即软件测量点的名称
    uint32_t start;                   // 进入时的 CYCCNT
    uint32_t cur_cycles;              // 本次耗时（周期）
    uint32_t max_cycles;              // 历史最大耗时（周期）—— WCET 核心输出
    uint32_t min_cycles;              // 历史最小耗时（周期）
    uint64_t total_cycles;            // 累计耗时（周期，64 位防溢出）
    uint32_t call_count;              // 调用次数
    bool     running;                 // 是否正在测量（防重入）
    bool     enabled;                 // 是否启用
} wcet_counter_t;

/**
 * @brief WCET 分析器
 */
typedef struct {
    wcet_counter_t counters[WCET_MAX_COUNTERS];    //所有测量点
    uint8_t  counter_count;             //已注册数量
    bool     initialized;               //初始化标志
    uint32_t cpu_freq_mhz;            // CPU 主频，用于周期换算时间
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
 * @return 计数器索引，>=0 成功；-1 失败
 */
int wcet_register(wcet_analyzer_t *analyzer, const char *name);

/**
 * @brief 按名称查找计数器索引
 * @return 索引，>=0 成功；-1 未找到
 */
int wcet_find(wcet_analyzer_t *analyzer, const char *name);

/* ===================== 测量接口 ===================== */

/** 开始测量（记录起始 CYCCNT） */
void wcet_start(wcet_analyzer_t *analyzer, int index);

/** 结束测量并更新统计（cur / max / min / total / call_count） */
void wcet_stop(wcet_analyzer_t *analyzer, int index);

/** 复位某个计数器的统计值 */
void wcet_reset(wcet_analyzer_t *analyzer, int index);

/** 复位所有计数器 */
void wcet_reset_all(wcet_analyzer_t *analyzer);

/* ===================== 结果查询 ===================== */

/** 获取最近一次耗时（周期） */
uint32_t wcet_get_cur_cycles(const wcet_analyzer_t *analyzer, int index);

/** 获取历史最大耗时（周期）—— 即 WCET 的核心输出 */
uint32_t wcet_get_max_cycles(const wcet_analyzer_t *analyzer, int index);

/** 获取历史最小耗时（周期） */
uint32_t wcet_get_min_cycles(const wcet_analyzer_t *analyzer, int index);

/** 获取平均耗时（周期） */
uint32_t wcet_get_avg_cycles(const wcet_analyzer_t *analyzer, int index);

/** 周期 -> 纳秒 */
uint32_t wcet_cycles_to_ns(const wcet_analyzer_t *analyzer, uint32_t cycles);

/* ===================== 便捷宏 ===================== */

#define WCET_BEGIN(an, idx)   wcet_start(&(an), (idx))
#define WCET_END(an, idx)     wcet_stop(&(an), (idx))

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

    /* min_cycles 初始化为最大值，方便第一次 stop 时更新 */
    for (uint8_t i = 0; i < WCET_MAX_COUNTERS; i++) {
        analyzer->counters[i].min_cycles = 0xFFFFFFFFu;
        analyzer->counters[i].enabled    = false;
    }

    analyzer->initialized = true;
    return true;
}

/* ===================== 注册 / 查找 ===================== */

int wcet_register(wcet_analyzer_t *analyzer, const char *name)
{
    if (analyzer == NULL || !analyzer->initialized) {
        return -1;
    }
    if (analyzer->counter_count >= WCET_MAX_COUNTERS) {
        return -1;
    }

    uint8_t idx = analyzer->counter_count;
    wcet_counter_t *c = &analyzer->counters[idx];

    copy_name(c->name, name);
    c->start        = 0;
    c->cur_cycles   = 0;
    c->max_cycles   = 0;
    c->min_cycles   = 0xFFFFFFFFu;
    c->total_cycles = 0;
    c->call_count   = 0;
    c->running      = false;
    c->enabled      = true;

    analyzer->counter_count++;
    return (int)idx;
}

int wcet_find(wcet_analyzer_t *analyzer, const char *name)
{
    if (analyzer == NULL || name == NULL) {
        return -1;
    }
    for (uint8_t i = 0; i < analyzer->counter_count; i++) {
        if (strncmp(analyzer->counters[i].name, name,
                    WCET_NAME_MAX_LEN) == 0) {
            return (int)i;
        }
    }
    return -1;
}

````

**注册为每个桩点分配id，查找负责按身份找回id，二者共同管理桩，提升代码的可读性可维护性。**
````txt
┌─────────────────────────────────────────────────┐
│ 1. wcet_init()          初始化 DWT + 清空分析器  │
└──────────────────────┬──────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────┐
│ 2. wcet_register("PID")   → 返回 idx_pid = 0    │
│    wcet_register("SPI")   → 返回 idx_spi = 1    │
│    wcet_register("ADC")   → 返回 idx_adc = 2    │
│    （测量点在此刻被"登记"，名字与槽位绑定）        │
└──────────────────────┬──────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────┐
│ 3. 运行阶段：                                    │
│    WCET_BEGIN(wcet, idx_pid);                   │
│    PID_Calc();                                  │
│    WCET_END(wcet, idx_pid);                     │
│                                                 │
│    或按名查找：                                  │
│    int i = wcet_find(&wcet, "PID");             │
│    WCET_BEGIN(wcet, i);                         │
│    ...                                          │
└──────────────────────┬──────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────┐
│ 4. 查询：                                        │
│    wcet_get_max_cycles(&wcet, idx_pid);         │
│    wcet_get_max_cycles(&wcet, wcet_find(&wcet,  │
│                          "PID"));               │
└─────────────────────────────────────────────────┘
````

### 测量

````c
void wcet_start(wcet_analyzer_t *analyzer, int index)
{
    if (analyzer == NULL || !analyzer->initialized) return;
    if (index < 0 || index >= (int)analyzer->counter_count) return;

    wcet_counter_t *c = &analyzer->counters[index];
    if (!c->enabled) return;

    c->start   = DWT_CYCCNT;
    c->running = true;
}

void wcet_stop(wcet_analyzer_t *analyzer, int index)
{
    if (analyzer == NULL || !analyzer->initialized) return;
    if (index < 0 || index >= (int)analyzer->counter_count) return;

    wcet_counter_t *c = &analyzer->counters[index];
    if (!c->enabled || !c->running) return;

    /* 无符号减法，自动处理 CYCCNT 回绕 */
    uint32_t delta = DWT_CYCCNT - c->start;

    c->cur_cycles = delta;
    c->running    = false;

    if (delta > c->max_cycles) c->max_cycles = delta;
    if (delta < c->min_cycles) c->min_cycles = delta;

    c->total_cycles += delta;
    c->call_count++;
}

void wcet_reset(wcet_analyzer_t *analyzer, int index)
{
    if (analyzer == NULL) return;
    if (index < 0 || index >= (int)analyzer->counter_count) return;

    wcet_counter_t *c = &analyzer->counters[index];
    c->start        = 0;
    c->cur_cycles   = 0;
    c->max_cycles   = 0;
    c->min_cycles   = 0xFFFFFFFFu;
    c->total_cycles = 0;
    c->call_count   = 0;
    c->running      = false;
}

void wcet_reset_all(wcet_analyzer_t *analyzer)
{
    if (analyzer == NULL) return;
    for (uint8_t i = 0; i < analyzer->counter_count; i++) {
        wcet_reset(analyzer, i);
    }
}
````