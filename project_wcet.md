# 基于 DWT 的插桩模块

## 背景

在实时嵌入式系统中，控制环、中断服务程序等关键代码段的执行时间直接影响系统的实时性与安全性。仅靠平均值或经验估算，无法保证系统在最坏情况下仍能满足截止时间要求。Cortex-M 内核自带 DWT 单元，其中的 CYCCNT 周期计数器可提供 1 个 CPU 周期精度 的计时能力，无需额外定时器或中断，是实现 WCET 测量最轻量的硬件基础。

## 目标

设计并实现一个 **WCET 插桩模块**，为混合 WCET 分析提供分段实测数据。

1. **高精度**：基于 DWT_CYCCNT，精度达单个 CPU 周期。
2. **低开销**：测量窗口内只有用户代码本身——插桩的记账动作全部落在窗口之外，
   窗口内不多执行一条指令。
3. **极简接口**：`WCET_BEGIN` / `WCET_END` 一对宏即可完成一次测量。
4. **统计能力**：记录调用次数与最大耗时。
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
测量：热路径为宏（展开在被测代码里，无函数调用），另提供统计复位与隔离测量
输出层：RTT or 串口

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
#define DWT_EXCCNT              (*(volatile uint32_t *)0xE000100C)     //异常开销计数器
#define CoreDebug_DEMCR         (*(volatile uint32_t *)0xE000EDFC)     //调试异常与监控控制寄存器

/* ===================== 控制位 ===================== */

/* CoreDebug->DEMCR：跟踪总开关 */
#define CoreDebug_DEMCR_TRCENA  (1u << 24)

/* DWT->CTRL：CYCCNT 使能位 */
#define DWT_CTRL_CYCCNTENA      (1u << 0)

/* DWT->CTRL：异常开销计数器（EXCCNT）使能位
#define DWT_CTRL_EXCEVTENA      (1u << 18)

/* ===================== 数据结构 ===================== */

/**
 * @brief 单个 WCET 计数器
 *
 * 基础统计面向"全部样本"；clean_* 一组只含"未被中断抢占"的样本，二者用途
 * 不同，不做覆盖——喂给静态工具的是 clean 组，all 组只作为中断干扰程度的
 * 旁证。字段按宽度排列消除填充，整体 64 字节。
 *
 * 关于 EXCCNT：它在这里不是 CPI/异常/睡眠那类"分析字段"，而是纯粹的
 * "样本有效性标记"——只用来回答"这次测量期间有没有被中断打断"。
 */
typedef struct {
    char     name[WCET_NAME_MAX_LEN]; // 计数器名称，即软件测量点的名称
    uint32_t addr;                    // 测量段的标签地址，供静态工具按地址对齐
    uint32_t start;                   // 进入时的 CYCCNT——测量窗口的起点
    uint32_t max_cycles;              // 全部样本的最大耗时（含中断干扰）

    /* --- clean 组：只统计窗口内未发生异常的样本，即纯代码耗时 --- */
    uint32_t clean_max_cycles;        // 纯代码观测上界——替换静态时间模型的输入
    uint32_t call_count;              // 样本总数
    uint32_t clean_count;             // 干净样本数（判断 clean 数据是否可信的依据）
    uint32_t preempted_count;         // 被中断抢占过的样本数

    uint8_t  start_exccnt;            // 进入时的 EXCCNT 快照（本就只有低 8 位有效）
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
 * @return 计数器索引，>=0 成功；-1 失败
 */
int wcet_register(wcet_analyzer_t *analyzer, const char *name);

/**
 * @brief 按名称查找计数器索引
 * @return 索引，>=0 成功；-1 未找到
 */
int wcet_find(wcet_analyzer_t *analyzer, const char *name);

/**
 * @brief 按索引取计数器指针
 * @return 指针；索引非法时返回 NULL
 *
 * 热路径的正确用法是**在初始化阶段解析一次、把指针存下来**，此后测量点
 * 就再也不用做 index 换算，见下方 WCET_BEGIN_P / WCET_END_P。
 */
wcet_counter_t *wcet_at(wcet_analyzer_t *analyzer, int index);

/**
 * @brief 按名称取计数器指针（= wcet_find + wcet_at）
 * @return 指针；未找到时返回 NULL
 */
wcet_counter_t *wcet_get(wcet_analyzer_t *analyzer, const char *name);

/* ===================== 混合分析接口 ===================== */

/**
 * @brief 复核插桩自身的固定开销
 *
 * 经典做法是"空跑 iterations 次，取最小值当开销扣掉"。改造之后窗口内已经
 * 不含任何记账代码，这个值理应只有几个周期，所以它的定位从"必须扣减的量"
 * 变成了"窗口干净与否的体检"：
 *
 *   结果接近 0   —— 正常，窗口里确实只有用户代码，无需扣减。
 *   结果明显偏大 —— 有编译器 / 链接器层面的东西插进了窗口，值得查下去。
 *
 * 取最小值而非均值：任何被中断污染的样本都只会抬高结果，取最小值最接近
 * 真实固定开销；被污染的样本直接丢弃。
 *
 * @param iterations 空跑次数，建议 >= 100
 * @return true 复核完成；false 表示参数非法，或全部样本都被抢占
 */
bool wcet_calibrate(wcet_analyzer_t *analyzer, uint32_t iterations);

/**
 * @brief 为某个测量点登记段地址
 *
 * 地址供静态分析工具把实测数据与它的基本块对齐。建议直接传标签地址，例如：
 *     wcet_set_addr(&wcet, idx, (uint32_t)&&lbl_pid);
 *
 * 注意：addr 属于配置而非统计量，wcet_reset 不会清除它。
 */
void wcet_set_addr(wcet_analyzer_t *analyzer, int index, uint32_t addr);

/**
 * @brief 离线隔离测量：用 BASEPRI 屏蔽优先级数值 >= threshold 的异常
 *
 * 屏蔽不了优先级 0 的异常（安全关键中断照常响应），也屏蔽不了 NMI 和
 * HardFault。比 CPSID i 温和，但仍在扰动系统时序，因此仅供离线路径
 * 特性化使用，不得作为运行期常态测量手段。
 *
 * 设计上不可重入（内部用文件级静态量保存现场），且必须成对调用。
 */
void wcet_isolated_begin(wcet_analyzer_t *analyzer, int index, uint32_t threshold);
void wcet_isolated_end(wcet_analyzer_t *analyzer, int index);

/* ===================== 复位接口 ===================== */

/*
 * 测量本身没有函数形式——热路径是头文件末尾的 WCET_BEGIN_P / WCET_END_P，
 * 直接展开在被测代码里，没有函数可调。本节只剩统计量的复位，它们不在热路径
 * 上，校验可以给足。
 */

/** 复位某个计数器的统计值 */
void wcet_reset(wcet_analyzer_t *analyzer, int index);

/** 复位所有计数器 */
void wcet_reset_all(wcet_analyzer_t *analyzer);

/* ===================== 结果查询 ===================== */

/** 获取全部样本中的最大耗时（周期，含中断干扰）——WCET 分析的主输出 */
uint32_t wcet_get_max_cycles(const wcet_analyzer_t *analyzer, int index);

/** 获取样本总数——判断上面那个最大值是否可信的依据 */
uint32_t wcet_get_call_count(const wcet_analyzer_t *analyzer, int index);

/** 周期 -> 纳秒 */
uint32_t wcet_cycles_to_ns(const wcet_analyzer_t *analyzer, uint32_t cycles);

/* ===================== clean 组查询 ===================== */

/**
 * 获取"纯代码"观测上界（周期，不含中断干扰）
 *
 * 这是喂给静态 WCET 工具做逐块替换的那个数。返回 0 有两种可能：该点从未
 * 被调用，或所有样本都被中断污染过——调用方应结合 wcet_get_clean_count()
 * 判断，样本太少时这个值不可信。
 */
uint32_t wcet_get_max_clean_cycles(const wcet_analyzer_t *analyzer, int index);

/** 获取干净样本数——判断 clean 数据可信度的依据 */
uint32_t wcet_get_clean_count(const wcet_analyzer_t *analyzer, int index);

/** 获取被中断抢占过的样本数 */
uint32_t wcet_get_preempted_count(const wcet_analyzer_t *analyzer, int index);

/** 获取复核出的插桩固定开销（周期） */
uint32_t wcet_get_offset_cycles(const wcet_analyzer_t *analyzer);

/* ===================== 便捷宏（热路径） ===================== */

#define WCET_COMPILER_BARRIER()   __asm volatile("" ::: "memory")

#define WCET_BEGIN_P(c)                                            \
    do {                                                           \
        (c)->start_exccnt = (uint8_t)DWT_EXCCNT;  /* 窗口外 */      \
        (c)->start        = DWT_CYCCNT;           /* ← 窗口起点 */  \
        WCET_COMPILER_BARRIER();                                   \
    } while (0)


#define WCET_END_P(c)                                              \
    do {                                                           \
        WCET_COMPILER_BARRIER();                                   \
        uint32_t _wcet_d = DWT_CYCCNT - (c)->start;  /* ← 窗口终点 */\
        /* 与 WCET_BEGIN_P 对称：CYCCNT 先读、EXCCNT 后读 */         \
        uint8_t  _wcet_x = (uint8_t)((uint8_t)DWT_EXCCNT            \
                                     - (c)->start_exccnt);          \
        if (_wcet_x != 0u) {                                        \
            (c)->preempted_count++;                                 \
        } else {                                                    \
            if (_wcet_d > (c)->clean_max_cycles) {                  \
                (c)->clean_max_cycles = _wcet_d;                    \
            }                                                       \
            (c)->clean_count++;                                     \
        }                                                           \
        if (_wcet_d > (c)->max_cycles) { (c)->max_cycles = _wcet_d; }\
        (c)->call_count++;                                          \
    } while (0)

#define WCET_BEGIN(an, idx)   WCET_BEGIN_P(&(an).counters[idx])
#define WCET_END(an, idx)     WCET_END_P(&(an).counters[idx])

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

/* ===================== BASEPRI 现场保存 ===================== */

/* 隔离测量仅用于离线特性化，设计上不可重入，故用文件级静态量保存现场。
   若将来需要在运行期并发使用，这里要改成由调用方显式传递的上下文。 */
static uint32_t g_wcet_saved_basepri = 0u;

static uint32_t wcet_get_basepri(void)
{
    uint32_t v;
    __asm volatile("mrs %0, basepri" : "=r"(v));
    return v;
}

static void wcet_set_basepri(uint32_t v)
{
    __asm volatile("msr basepri, %0" :: "r"(v) : "memory");
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

    /* 2b. 使能异常开销计数器并清零。
     *     它只累计异常处理的框架开销（入栈 / 出栈 / 尾链 / 向量取指），
     *     不含 ISR 主体执行时间。在这里它的用途是判定本次采样有没有被中断
     *     打断，而不是做异常分析。 */
    DWT_EXCCNT = 0;
    DWT_CTRL  |= DWT_CTRL_EXCEVTENA;

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

    c->start      = 0u;
    c->max_cycles = 0u;
    c->call_count = 0u;

    c->addr             = 0u;             /* 待 wcet_set_addr 填写 */
    c->start_exccnt     = 0u;
    c->clean_max_cycles = 0u;
    c->clean_count      = 0u;
    c->preempted_count  = 0u;

    analyzer->counter_count++;
    return (int)idx;
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

wcet_counter_t *wcet_at(wcet_analyzer_t *analyzer, int index)
{
    if (analyzer == NULL || index < 0 ||
        index >= (int)analyzer->counter_count) {
        return NULL;
    }
    return &analyzer->counters[index];
}

wcet_counter_t *wcet_get(wcet_analyzer_t *analyzer, const char *name)
{
    int index = wcet_find(analyzer, name);
    return (index < 0) ? NULL : &analyzer->counters[index];
}

/* ===================== 混合分析：开销复核 / 地址 / 隔离测量 ===================== */

bool wcet_calibrate(wcet_analyzer_t *analyzer, uint32_t iterations)
{
    if (analyzer == NULL || !analyzer->initialized) return false;
    if (iterations == 0u) return false;

    uint32_t best  = 0xFFFFFFFFu;
    uint32_t clean = 0u;

    for (uint32_t i = 0; i < iterations; i++) {
        uint8_t  e0 = (uint8_t)DWT_EXCCNT;
        uint32_t t0 = DWT_CYCCNT;
        uint32_t t1 = DWT_CYCCNT;       /* 两条 volatile 读不会被合并 */
        uint8_t  e1 = (uint8_t)DWT_EXCCNT;

        if ((uint8_t)(e1 - e0) != 0u) {
            continue;                   /* 窗口内进了中断，本次样本丢弃 */
        }

        uint32_t d = t1 - t0;
        if (d < best) { best = d; }
        clean++;
    }

    analyzer->calibrated = (clean > 0u);
    if (analyzer->calibrated) {
        analyzer->offset_cycles = best;
    }
    return analyzer->calibrated;
}

void wcet_set_addr(wcet_analyzer_t *analyzer, int index, uint32_t addr)
{
    if (analyzer == NULL) return;
    if (index < 0 || index >= (int)analyzer->counter_count) return;

    analyzer->counters[index].addr = addr;
}

void wcet_isolated_begin(wcet_analyzer_t *analyzer, int index, uint32_t threshold)
{
    if (analyzer == NULL) return;
    if (index < 0 || index >= (int)analyzer->counter_count) return;

    g_wcet_saved_basepri = wcet_get_basepri();
    wcet_set_basepri(threshold);                /* 写 BASEPRI 在测量窗口之外 */
    WCET_BEGIN_P(&analyzer->counters[index]);
}

void wcet_isolated_end(wcet_analyzer_t *analyzer, int index)
{
    if (analyzer == NULL) return;
    if (index < 0 || index >= (int)analyzer->counter_count) return;

    WCET_END_P(&analyzer->counters[index]);     /* 先闭合窗口 */
    wcet_set_basepri(g_wcet_saved_basepri);     /* 再恢复，恢复动作也在窗口之外 */
}
````

**注册为每个桩点分配id，查找负责按身份找回id，二者共同管理桩，提升代码的可读性可维护性。
热路径推荐在注册后就把指针取出来存好，此后测量点连索引换算都省掉。**
````txt
┌─────────────────────────────────────────────────┐
│ 1. wcet_init()          初始化 DWT + 清空分析器  │
│    可选的 wcet_calibrate()  复核插桩开销接近 0   │
└──────────────────────┬──────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────┐
│ 2. wcet_register("PID")   → 返回 idx_pid = 0    │
│    wcet_register("SPI")   → 返回 idx_spi = 1    │
│    wcet_register("ADC")   → 返回 idx_adc = 2    │
│    （测量点在此刻被"登记"，名字与槽位绑定）        │
│                                                 │
│    解析成指针，之后热路径只用指针：               │
│    wcet_counter_t *pid = wcet_get(&wcet, "PID");│
└──────────────────────┬──────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────┐
│ 3. 运行阶段（三种等价写法，开销依次递减）：        │
│                                                 │
│    按名查找（每周期都查，最慢，仅示意）：          │
│    WCET_BEGIN(wcet, wcet_find(&wcet,"PID"));    │
│                                                 │
│    用缓存的 id（保留老写法）：                    │
│    WCET_BEGIN(wcet, idx_pid);                   │
│    PID_Calc();                                  │
│    WCET_END(wcet, idx_pid);                     │
│                                                 │
│    用缓存的指针（推荐，热路径最快）：              │
│    WCET_BEGIN_P(pid);                           │
│    PID_Calc();                                  │
│    WCET_END_P(pid);                             │
└──────────────────────┬──────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────┐
│ 4. 查询：                                        │
│    wcet_get_max_cycles(&wcet, idx_pid);         │
│    wcet_get_call_count(&wcet, idx_pid);         │
│    （导给工具另需 max_clean / clean_count /      │
│      preempted_count / offset_cycles）          │
└─────────────────────────────────────────────────┘
````

### 复位

````c

void wcet_reset(wcet_analyzer_t *analyzer, int index)
{
    if (analyzer == NULL) return;
    if (index < 0 || index >= (int)analyzer->counter_count) return;

    wcet_counter_t *c = &analyzer->counters[index];
    c->start      = 0u;
    c->max_cycles = 0u;
    c->call_count = 0u;

    /* 统计量全部复位；addr 属于配置而非统计，保留不动 */
    c->start_exccnt     = 0u;
    c->clean_max_cycles = 0u;
    c->clean_count      = 0u;
    c->preempted_count  = 0u;
}

void wcet_reset_all(wcet_analyzer_t *analyzer)
{
    if (analyzer == NULL) return;
    for (uint8_t i = 0; i < analyzer->counter_count; i++) {
        wcet_reset(analyzer, i);
    }
}
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

uint32_t wcet_cycles_to_ns(const wcet_analyzer_t *analyzer, uint32_t cycles)
{
    if (analyzer == NULL || analyzer->cpu_freq_mhz == 0) return 0;
    /* ns = cycles / freq_mhz * 1000 */
    return (uint32_t)(((uint64_t)cycles * 1000u) / analyzer->cpu_freq_mhz);
}

uint32_t wcet_get_max_clean_cycles(const wcet_analyzer_t *analyzer, int index)
{
    if (analyzer == NULL || index < 0 ||
        index >= (int)analyzer->counter_count) return 0;
    const wcet_counter_t *c = &analyzer->counters[index];
    if (c->clean_count == 0u) return 0;     /* 没有干净样本，该值无意义 */
    return c->clean_max_cycles;
}

uint32_t wcet_get_clean_count(const wcet_analyzer_t *analyzer, int index)
{
    if (analyzer == NULL || index < 0 ||
        index >= (int)analyzer->counter_count) return 0;
    return analyzer->counters[index].clean_count;
}

uint32_t wcet_get_preempted_count(const wcet_analyzer_t *analyzer, int index)
{
    if (analyzer == NULL || index < 0 ||
        index >= (int)analyzer->counter_count) return 0;
    return analyzer->counters[index].preempted_count;
}

uint32_t wcet_get_offset_cycles(const wcet_analyzer_t *analyzer)
{
    if (analyzer == NULL) return 0;
    return analyzer->offset_cycles;
}
````

### 插桩开销

#### 每次测量的实际开销

| 项 | 量 |
|---|---|
| 窗口内执行 | 仅用户代码，插桩不占一条指令 |
| 每次测量多读寄存器 | 2（CYCCNT + EXCCNT） |
| 每次测量额外内存访问 | ≥ 5（start/start_exccnt 各写一次；max_cycles、clean_max_cycles 各读比写；call_count、clean_count 各读改写） |
| 单通道 RAM | 64 字节 |
| 32 通道 RAM | ~2.0 KB（其中名字数组占 1 KB） |

对比改造前：单通道 104 字节 → 64 字节（32 通道 3.3 KB → 2.0 KB），热路径从
"一次函数调用 + 两次边界检查 + 64 位累加器读改写"降到"零调用、零校验"。

#### 窗口之外与窗口之内

"低开销"这条目标拆开看是两件不同的事：

- **窗口之内**——从 BEGIN 的最后一次 CYCCNT 读到 END 的第一次读，这中间**只有
  用户代码**。统计更新、计数递增、中断判定全部排在窗口闭合之后，不影响被测量
  代码的时序特性。这是靠宏的写法保证的，不是靠"代码写得短"。
- **窗口之外**——每次测量仍要花十几个周期做记账。这部分拉长了测量点所在函数的
  总执行时间，但它是**常数**，且不落在被测量的区间里。

真正需要在意的是第一件事：插桩对被测代码的**时序扰动**（I-cache 占用、指令对齐、
预取状态）无法用"减去一个 offset"来补偿，只能在窗口内不放东西。窗口外的记账
则是可预测的常数开销，扣除即可。

#### 两条取舍原则

- **`offset_cycles` 抵消不了一切。** 它只能抵消插桩自身消耗的周期数，抵消不了
  插桩带来的对齐、cache、分支预测副作用。所以本模块的做法是把窗口内清空，而不是
  把开销算准了再减。
- **省一个周期也是省。** 热路径上的每一处都是"每次测量都付"。一个 64 位累加器
  读改写在高频控制环里就是每秒几十万次；一次函数调用同理。这也是平均值、最小值
  这类不服务于 WCET 工具的统计量被整个删掉的原因——它们不只是占 RAM，更是每次
  测量都要付出的周期。

### 与静态 WCET 工具对接（混合分析）

#### 为什么需要这种数据组织方式

静态分析工具穷举控制流图，保证的是**路径完备性**——它不会漏掉任何一条执行路径；
但它对每个基本块耗时的估计是模型化的（流水线、Flash 等待周期、cache、多周期
部件），只能偏保守。混合分析的分工是：**保留静态分析的路径枚举，用实测值替换
它的时间模型**，沿同一条最坏路径求和，得到一个更紧、但仍然安全的界。

这就决定了实测侧必须交出**按段组织**的数据：时间模型是逐块的，要替换它就得逐块
比对。只给一个总耗时，工具知道两边有差距，却不知道差距落在哪个块上——而差距往往
恰恰集中在少数几个块（Flash 取指等待、cache 未命中、除法器），定位不到就等于没测。

#### 数据契约

每个测量点向上层提供：

| 字段 | 获取方式 | 给工具做什么 |
|---|---|---|
| `addr` | `wcet_set_addr` 登记 | **与静态工具的基本块按地址对齐**，逐块比对的前提 |
| `clean_max` | `wcet_get_max_clean_cycles()` | 纯代码观测上界，**替换该块时间模型的输入** |
| `all_max` | `wcet_get_max_cycles()` | 系统视角观测上界（含中断干扰），供可调度性分析 |
| `clean_count` | `wcet_get_clean_count()` | 该值的样本量，**判断它是否可信** |
| `preempted_count` | `wcet_get_preempted_count()` | 中断干扰的严重程度 |
| `offset_cycles` | `wcet_get_offset_cycles()` | 插桩自身开销，逐周期比对时须扣除 |

导出建议每行一个测量点，便于工具直接解析：

````csv
segment,addr,clean_max,all_max,clean_count,preempted_count
PID,0x08001A2C,1842,2417,938,62
SPI,0x08001B04,410,655,1000,0
````

#### 地址怎么拿

用 GCC/Clang 的标签取址，它给出的是链接后的真实地址，与静态工具读 ELF 得到的
基本块地址处于同一坐标系：

````c
static wcet_analyzer_t wcet;
static wcet_counter_t *pid = NULL;

void control_loop(void)
{
    if (pid == NULL) {
        int idx = wcet_register(&wcet, "PID");
        pid = wcet_at(&wcet, idx);

        /* &&label 属于编译器扩展，且只能在定义该标签的函数内部求值，
           所以登记地址这一步必须和标签处在同一个函数里。 */
        wcet_set_addr(&wcet, idx, (uint32_t)&&lbl_pid);
    }

    WCET_BEGIN_P(pid);
lbl_pid:;                       /* 该地址即被测段起点，供静态工具对齐 */
    PID_Calc();
    WCET_END_P(pid);
}
````


#### 推荐的使用流程

1. `wcet_init` —— 打开 DWT，使能 CYCCNT 与 EXCCNT
2. `wcet_calibrate` —— 复核插桩开销；改造后应接近 0，不接近就说明窗口不干净
3. `wcet_register` + `wcet_at` + `wcet_set_addr` —— 登记测量点及其地址
4. **离线阶段**：对静态工具指认的最坏路径，用 `wcet_isolated_begin/end` 测出
   权威值，作为该路径的基准
5. **运行期**：用普通的 `WCET_BEGIN/END` 长期采集，靠 clean 样本交叉验证


