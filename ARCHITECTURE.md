# Kissat SAT Solver 架构设计说明书

## 1. 项目概述

### 1.1 项目背景与定位

Kissat ("kissat" 在芬兰语中意为"猫咪") 是一个用C语言编写的SAT（布尔可满足性）求解器，全称为 "Keep It Simple and Clean Bare Metal SAT Solver"。它是从CaDiCaL求解器移植回C语言的版本，在数据结构和算法实现上进行了优化改进。

**核心特性：**
- 完整的CDCL（Conflict-Driven Clause Learning）实现
- 丰富的预处理和inprocessing技术
- 支持DIMACS CNF格式输入
- DRAT proof trace输出
- 增量求解支持（IPASIR接口）
- 200+可配置参数

### 1.2 SAT求解器简介

SAT问题是布尔可满足性问题，是NP完全问题的典型代表。CDCL算法是目前最成功的SAT求解算法，核心思想包括：

1. **变量决策**：选择一个未赋值变量进行假设
2. **单元传播**：根据单子句 forced assignments
3. **冲突分析**：当出现冲突时分析原因，学习新子句
4. **回溯**：回溯到导致冲突的决策层级
5. **搜索重启**：定期重启搜索过程

---

## 2. 总体架构

### 2.1 架构分层图

```
┌─────────────────────────────────────────────────────────────────┐
│                     Application Layer                           │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │  main.c     │  │ application.c│  │  command-line parsing │  │
│  │  信号处理   │  │  文件I/O     │  │  DIMACS input/output  │  │
│  └─────────────┘  └──────────────┘  └───────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                     Public API (kissat.h)                       │
│  kissat_init / kissat_add / kissat_solve / kissat_value         │
│  kissat_release / kissat_set_terminate / kissat_set_option      │
├─────────────────────────────────────────────────────────────────┤
│                     Core Solver Engine                          │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    search.c (主搜索循环)                    │ │
│  │  ┌─────────┐  ┌──────────┐  ┌─────────┐  ┌──────────────┐  │ │
│  │  │ decide  │→ │propagate │→ │ analyze │→ │ backtrack    │  │ │
│  │  │ 决策    │  │ 传播     │  │ 冲突分析│  │ 回溯         │  │ │
│  │  └─────────┘  └──────────┘  └─────────┘  └──────────────┘  │ │
│  └────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│                   Preprocessing / Inprocessing                  │
│  ┌──────────┐ ┌──────────┐ ┌─────────┐ ┌────────┐ ┌─────────┐  │
│  │ probe    │ │ eliminate│ │ vivify  │ │sweep   │ │ backbone│  │
│  │ 探针     │ │ 变量消除 │ │ 子句活化│ │ 扫描   │ │ 骨干   │  │
│  └──────────┘ └──────────┘ └─────────┘ └────────┘ └─────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                      Data Structures                            │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐        │
│  │ clause │ │ arena  │ │ stack  │ │ heap   │ │ queue  │        │
│  │ 子句   │ │ 内存池 │ │ 动态数组│ │ 评分堆 │ │ 双向队列│        │
│  └────────┘ └────────┘ └────────┘ └────────┘ └────────┘        │
├─────────────────────────────────────────────────────────────────┤
│                    Memory Management                            │
│  allocate.c (内存分配) / arena.c (子句Arena) / collect.c (GC)   │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 求解流程图

```
                          ┌─────────────────┐
                          │  kissat_init()  │
                          │   初始化求解器   │
                          └────────┬────────┘
                                   │
                                   ▼
                          ┌─────────────────┐
                          │  kissat_add()   │
                          │  解析CNF文件    │
                          │  添加子句       │
                          └────────┬────────┘
                                   │
                                   ▼
                          ┌─────────────────┐
                          │ kissat_solve()  │
                          └────────┬────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
              ▼                    ▼                    ▼
      ┌───────────────┐   ┌───────────────┐   ┌─────────────────┐
      │ preprocessing │   │   main loop   │   │ postprocessing  │
      │   预处理      │   │   主搜索循环   │   │   后处理        │
      │               │   │               │   │                 │
      │ - probe       │   │ ┌───────────┐ │   │ - backbone      │
      │ - eliminate   │   │ │ 决策      │ │   │ - search        │
      │ - sweep       │   │ └─────┬─────┘ │   │   for model     │
      └───────┬───────┘   │       │       │   └─────────────────┘
              │           │       ▼       │
              │           │ ┌───────────┐ │
              │           │ │ propagate │ │
              │           │ │ (BCP)     │ │
              │           │ └─────┬─────┘ │
              │           │       │       │
              │           │       ▼       │
              │           │ ┌───────────┐ │
              │           │ │ 冲突?     │ │
              │           │ └─────┬─────┘ │
              │           │       │       │
              │     ┌─────┼───────┼───────┼─────┐
              │     │     │       │       │     │
              │     │     ▼       │       ▼     │
              │     │ ┌───────┐   │   ┌───────┐ │
              │     │ │ 分析   │   │   │ 返回  │ │
              │     │ │ 冲突   │   │   │ SAT  │ │
              │     │ └───────┘   │   └───────┘ │
              │     │       │       │           │
              │     │       ▼       │           │
              │     │ ┌───────────┐ │           │
              │     │ │ backtrack │ │           │
              │     │ │ + learn   │ │           │
              │     │ └───────────┘ │           │
              │     │       │       │           │
              │     └───────┼───────┘           │
              │             │                   │
              └─────────────┼───────────────────┘
                            │
                            ▼
                   ┌─────────────────┐
                   │   UNSAT / SAT   │
                   │   返回结果      │
                   └─────────────────┘
```

---

## 3. 核心模块详解

### 3.1 公共API接口

**文件：** `src/kissat.h`

```c
// 核心求解接口
kissat *kissat_init (void);                    // 初始化求解器
void kissat_add (kissat *solver, int lit);     // 添加文字 (正:lit, 负:-lit, 0:子句结束)
int kissat_solve (kissat *solver);             // 执行求解 (10:UNSAT, 20:SAT)
int kissat_value (kissat *solver, int lit);    // 获取变量赋值 (1:true, -1:false, 0:未定义)
void kissat_release (kissat *solver);          // 释放求解器

// 终止控制接口
void kissat_set_terminate (kissat *solver, void *state,
    int (*terminate)(void *state));            // 设置终止回调函数

// 选项控制接口
int kissat_get_option (kissat *solver, const char *name);  // 获取选项值
int kissat_set_option (kissat *solver, const char *name, int new_value); // 设置选项

// IPASIR接口 (部分实现)
const char *kissat_signature (void);           // 返回求解器签名
```

**使用示例：**

```c
#include "kissat.h"

int main() {
    kissat *solver = kissat_init();
    
    // 添加子句: (x1 OR NOT x2 OR x3)
    kissat_add(solver, 1);
    kissat_add(solver, -2);
    kissat_add(solver, 3);
    kissat_add(solver, 0);  // 子句结束
    
    // 添加子句: (NOT x1 OR x2)
    kissat_add(solver, -1);
    kissat_add(solver, 2);
    kissat_add(solver, 0);
    
    int result = kissat_solve(solver);
    
    if (result == 20) {  // SAT
        printf("SAT\n");
        // 获取解
        for (int i = 1; i <= 3; i++) {
            int val = kissat_value(solver, i);
            printf("x%d = %d\n", i, val);
        }
    } else {
        printf("UNSAT\n");
    }
    
    kissat_release(solver);
    return 0;
}
```

### 3.2 内部求解器结构

**文件：** `src/internal.h`

```c
struct kissat {
    // 状态标志
    bool inconsistent;      // 公式不可满足
    bool iterating;         // 迭代模式
    bool preprocessing;     // 预处理模式
    bool probing;           // 探针模式
    bool stable;            // 稳定搜索模式
    
    // 变量管理
    unsigned vars;          // 变量总数
    unsigned size;          // 内部数组大小
    unsigned active;        // 活跃变量数
    
    // 赋值结构
    assigned *assigned;     // 变量赋值信息
    value *values;          // 变量值 (0:未定义, 1:true, -1:false)
    phases phases;          // 变量的相位选择
    
    // 搜索状态
    unsigned level;         // 当前决策层级
    frames frames;          // 决策层级框架
    unsigned_array trail;   // 赋值 trail
    unsigned *propagate;    // 传播指针
    
    // 启发式数据结构
    heap scores;            // 变量评分堆 (VSIDS)
    queue queue;            // 变量活动队列 (VSID without heap)
    links *links;           // 队列链接
    
    // 子句存储
    arena arena;            // 子句内存池
    statches *watches;      // Watch列表
    
    // 统计信息
    statistics statistics;
    
    // 选项配置
    options options;
};
```

### 3.3 CDCL主搜索循环

**文件：** `src/search.c`

```c
// 简化版主循环
int kissat_search(kissat *solver) {
    // 初始化搜索
    solver->level = 0;
    solver->propagate = BEGIN_ARRAY(solver->trail);
    
    while (true) {
        // 执行约束传播 (Boolean Constraint Propagation)
        clause *conflict = kissat_search_propagate(solver);
        
        if (conflict) {
            // 发生冲突
            if (solver->level == 0) {
                // 根层级冲突 -> UNSAT
                solver->inconsistent = true;
                return 20;
            }
            
            // 冲突分析
            kissat_analyze(solver, conflict);
            
            // 回溯并学习新子句
            kissat_backtrack(solver);
        }
        
        // 检查终止条件
        if (TERMINATED(search_terminated))
            return 0;
        
        // 检查是否需要重启
        if (NEED_RESTART)
            kissat_restart(solver);
        
        // 检查是否需要inprocessing
        if (NEED_INPROCESSING)
            kissat_inprocessing(solver);
        
        // 做决策
        if (!kissat_decide(solver))
            return 20;  // 所有变量已赋值 -> SAT
    }
}
```

### 3.4 决策启发式

**文件：** `src/decide.c`

Kissat实现了多种决策启发式，核心是**VSIDS**（Variable State Independent Decaying Sum）：

```c
// VSIDS决策流程
int kissat_decide(kissat *solver) {
    // 获取未赋值变量
    if (!solver->unassigned)
        return false;  // 所有变量已赋值
    
    // 选择决策层级
    if (solver->stable)
        return kissat_stable_decide(solver);
    else
        return kissat_focused_decide(solver);
}

// Focused模式决策 (VSIDS with heap)
static int kissat_focused_decide(kissat *solver) {
    unsigned idx = kissat_maximum_variable(solver);
    
    // 检查是否达到限制
    if (!DECISION_LIMIT_OK())
        return false;
    
    // 选择相位 (基于phase saving)
    int phase = kissat_get_phase(solver, idx);
    unsigned lit = (phase > 0) ? 2 * idx : 2 * idx + 1;
    
    kissat_decide_literal(solver, lit);
    return true;
}

// 从堆中获取评分最高的变量
unsigned kissat_maximum_variable(kissat *solver) {
    heap *heap = &solver->scores;
    if (EMPTY_HEAP(heap))
        return 0;
    return heap->max;
}
```

### 3.5 约束传播 (BCP)

**文件：** `src/propsearch.c`

使用**Watched Literals**技术优化传播：

```c
// 传播主函数
clause *kissat_search_propagate(kissat *solver) {
    // 从propagate指针开始传播
    while (solver->propagate < END_ARRAY(solver->trail)) {
        unsigned lit = *solver->propagate++;
        
        // 更新被传播变量的信息
        assigned *a = &solver->assigned[lit >> 1];
        a->reason = reason;      // 传播原因
        a->level = solver->level; // 决策层级
        
        // 处理watched列表
        if (!kissat_propagate_watch(solver, lit))
            return CONFLICT;     // 返回冲突子句
    }
    return 0;  // 无冲突
}

// 处理单个watch
static bool kissat_propagate_watch(kissat *solver, unsigned lit) {
    statch *watches = &solver->watches[lit];
    watch *watch = BEGIN_WATCHES(*watches);
    watch *end = END_WATCHES(*watches);
    
    for (; watch != end; watch++) {
        if (watch->type == binary_watch) {
            // 二元子句传播
            unsigned other = watch->binary.lit;
            if (VALUE(other) == false) {
                // 冲突!
                return false;
            }
            if (UNASSIGNED(other)) {
                kissat_unit_propagate(solver, other, watch->binary.clause);
            }
        } else {
            // 大子句传播 (需要检查)
            clause *c = watch->clause;
            if (kissat_propagate_clause(solver, c, lit))
                continue;
            return false;  // 冲突
        }
    }
    return true;
}
```

### 3.6 冲突分析

**文件：** `src/analyze.c`

当发生冲突时，分析冲突原因并学习新子句：

```c
// 冲突分析流程
int kissat_analyze(kissat *solver, clause *conflict) {
    // 1. 收集冲突相关变量
    unsigned count = 0;
    unsigned *conflits = solver->conflits;
    
    // 从冲突子句开始分析
    for (all_literals_in_clause(lit, conflict)) {
        if (LEVEL(lit) == solver->level)
            conflits[count++] = lit;
    }
    
    // 2. 标记已分析
    kissat_mark_analyzed(solver);
    
    // 3. 遍历trail，收集导致冲突的变量
    while (count > 0) {
        unsigned lit = conflits[--count];
        assigned *a = &solver->assigned[lit >> 1];
        
        if (a->reason == 0) continue;  // 决策变量
        
        clause *c = a->reason;
        for (all_literals_in_clause(l, c)) {
            if (LEVEL(l) == solver->level && !ANALYZED(l)) {
                ANALYZE(l) = true;
                conflits[count++] = l;
            }
        }
    }
    
    // 4. 构建学习子句 (1st-UIP优化)
    build_learned_clause(solver);
    
    // 5. 子句最小化
    if (GET_OPTION(minimize))
        kissat_minimize_clause(solver);
    
    // 6. 添加学习子句
    reference ref = kissat_new_redundant_clause(solver, glue);
    solver->learned = ref;
    
    // 7. 更新VSIDS分数
    kissat_bump_variable_scores(solver, conflits, num_conflits);
    
    return solver->level;
}
```

### 3.7 回溯机制

**文件：** `src/backtrack.c`

```c
// 回溯到指定层级
void kissat_backtrack(kissat *solver, unsigned target_level) {
    // 1. 保存学习子句
    clause *learned = solver->learned;
    
    // 2. 清除target_level以上的所有赋值
    unsigned *trail = BEGIN_ARRAY(solver->trail);
    unsigned *p = END_ARRAY(solver->trail);
    
    while (p != trail) {
        unsigned lit = *--p;
        if (solver->assigned[lit >> 1].level > target_level) {
            UNASSIGN(lit);
            *p = INVALID_LIT;  // 暂存，稍后清理
        }
    }
    
    // 3. 调整trail指针
    solver->propagate = p;
    
    // 4. 重新赋值学习子句
    for (all_literals_in_clause(lit, learned)) {
        if (LEVEL(lit) <= target_level) {
            // 保留该赋值
            *p++ = lit;
        }
    }
    
    // 5. 更新trail
    solver->trail.end = p;
    
    // 6. 清除标记
    solver->level = target_level;
}
```

---

## 4. 数据结构

### 4.1 子句结构

**文件：** `src/clause.h`

```c
struct clause {
    // 元数据 (位域优化内存)
    unsigned glue : LD_MAX_GLUE;     // 学习子句粘连度 (用于生命周期管理)
    bool garbage : 1;                // 待回收
    bool redundant : 1;               // 学习子句标识
    bool reason : 1;                  // 是否作为传播原因
    bool shrunken : 1;               // 是否被收缩
    bool subsume : 1;                // 是否被蕴含
    
    unsigned used : LD_MAX_USED;     // 使用计数
    unsigned searched;               // 搜索次数
    unsigned size;                   // 子句长度
    
    // 文字数组 (至少3个，使用变长数组)
    unsigned lits[3];
};
```

### 4.2 动态数组 (Stack)

**文件：** `src/stack.h`

```c
// 栈结构定义
#define STACK(TYPE) \
    struct { \
        TYPE *begin;    // 数组起始
        TYPE *end;      // 当前结束位置
        TYPE *allocated;// 已分配内存末尾
    }

// 栈操作宏
#define EMPTY_STACK(S)    ((S).begin == (S).end)
#define SIZE_STACK(S)     ((size_t)((S).end - (S).begin))
#define CAPACITY_STACK(S) ((size_t)((S).allocated - (S).begin))

#define PUSH_STACK(S, E)  \
    do { \
        if (FULL_STACK(S)) ENLARGE_STACK(S); \
        *(S).end++ = (E); \
    } while (0)

#define POP_STACK(S)      (*--(S).end)
#define TOP_STACK(S)      ((S).end[-1])
```

### 4.3 变量评分堆 (Heap)

用于VSIDS启发式的二叉堆实现：

```c
// 堆结构
typedef struct {
    unsigned *data;      // 堆数组 (存储变量索引)
    unsigned *pos;       // 变量在堆中的位置 (反向索引)
    double *scores;      // 变量分数
    unsigned size;       // 当前大小
    unsigned capacity;   // 容量
} heap;

// 核心操作: 上浮/下沉维护堆性质
static void heapify_up(heap *h, unsigned idx) {
    while (idx > 0) {
        unsigned parent = (idx - 1) >> 1;
        if (h->scores[h->data[idx]] <= h->scores[h->data[parent]])
            break;
        heap_swap(h, idx, parent);
        idx = parent;
    }
}
```

### 4.4 Watched Literals

**文件：** `src/watch.h`

```c
// Watch结构 (一个字节优化)
typedef struct {
    unsigned lit : 31;   // 另一个文字
    bool binary : 1;     // 是否二元子句
} watch;

// 大子句watch
typedef struct {
    clause *clause;      // 指向子句的引用
} large_watch;

// Watch列表用于加速传播
typedef STACK(watch) watches;
```

---

## 5. 预处理/处理技术

### 5.1 预处理流程

**文件：** `src/preprocess.c`

```
┌─────────────────────────────────────────────┐
│           Preprocessing 循环                │
├─────────────────────────────────────────────┤
│  1. 初始传播 (initially propagate)          │
│     └─> 检测单元子句，强制赋值               │
│                                             │
│  2. 循环预处理 (preprocessrounds次)         │
│     ┌─ probe_initially (探针)               │
│     │   └─ 单变量探针、传递闭包             │
│     ├─ fast_variable_elimination            │
│     │   └─ 快速变量消除                     │
│     ├─ sweep (扫描)                         │
│     │   └─ 等价变量消除                     │
│     └─ collect (垃圾回收)                   │
│                                             │
│  3. 变量消除 (bounded variable elimination) │
│                                             │
│  4. 子句vivification                        │
└─────────────────────────────────────────────┘
```

### 5.2 变量消除 (BVE)

**文件：** `src/eliminate.c`

```c
// 变量消除主流程
bool kissat_eliminate(kissat *solver) {
    // 收集候选变量
    collect_candidate_variables(solver);
    
    for each variable v in candidates:
        // 计算消除成本
        int resolve_ops = estimate_resolve_count(v);
        
        if (resolve_ops > eliminatebound)
            continue;  // 跳过
        
        // 收集相关子句
        collect_occurrences(v, pos_clauses, neg_clauses);
        
        // 尝试消除
        if (can_eliminate(pos_clauses, neg_clauses)) {
            if (try_eliminate(solver, v, pos_clauses, neg_clauses))
                eliminated++;
        }
    }
    
    // 清理
    if (eliminated > 0)
        kissat_shrink_arena(solver);
}
```

### 5.3 Inprocessing

在主搜索循环中间执行的优化技术：

| 技术 | 作用 | 触发条件 |
|------|------|----------|
| probe | 单变量探针，发现隐含赋值 | 周期性 |
| eliminate | 变量消除 | 周期性 |
| subsume | 子句 subsumption | 周期性 |
| strengthen | 子句强化 | 周期性 |
| vivify | 子句活化 | 周期性 |
| backbone | 骨干变量计算 | 条件触发 |

---

## 6. 构建系统

### 6.1 编译流程

```bash
# 配置 (生成Makefile)
./configure [options]

# 选项:
#   -g    调试构建 (包含assert、符号)
#   -c    包含断言检查
#   -l    包含日志代码
#   -s    添加调试符号
#   -p    严格编译 (-Werror -std=c99 --pedantic)
#   --coverage  覆盖率检测
#   --profile   性能分析

# 构建
make

# 构建目标
make kissat       # 主求解器
make tissat       # 测试运行器
make libkissat.a  # 静态库
make test         # 运行测试
make format       # 代码格式化
```

### 6.2 目录结构

```
kissat/
├── src/              # 源代码
│   ├── *.c          # 实现文件
│   ├── *.h          # 头文件
│   └── internal.h   # 内部结构定义
├── test/             # 测试代码
│   ├── test*.c      # 单元测试
│   └── test.h       # 测试框架
├── build/            # 构建输出
├── configure        # 配置脚本
├── makefile.in      # Makefile模板
└── makefile         # 生成的Makefile
```

---

## 7. 关键配置选项

### 7.1 搜索策略选项

| 选项 | 默认值 | 说明 |
|------|--------|------|
| `stable` | 0 | 稳定搜索模式 (0:focused, 1:stable, 2:自动) |
| `phase` | 1 | 初始决策相位 (0:false, 1:true) |
| `phasesaving` | 1 | 启用相位保存 |
| `decay` | 50 | VSIDS衰减因子 (‰) |
| `chrono` | 1 | 允许时间回溯 |
| `minimize` | 1 | 学习子句最小化 |

### 7.2 预处理选项

| 选项 | 默认值 | 说明 |
|------|--------|------|
| `preprocess` | 1 | 启用预处理 |
| `probe` | 1 | 启用探针 |
| `eliminate` | 1 | 启用变量消除 |
| `preprocessrounds` | 1 | 预处理轮数 |

### 7.3 内存管理选项

| 选项 | 默认值 | 说明 |
|------|--------|------|
| `compact` | 1 | 启用压缩GC |
| `compactlim` | 10 | 压缩阈值 (%) |
| `reduce` | 1 | 启用子句缩减 |
| `reduceinit` | 100 | 初始缩减间隔 |

---

## 8. 总结

Kissat是一个设计精良的CDCL SAT求解器，具有以下特点：

1. **简洁的代码结构** - 模块化设计，每模块职责清晰
2. **高效的数据结构** - Watched Literals、Arena内存管理、VSIDS堆
3. **丰富的优化技术** - 预处理、inprocessing、子句学习、最小化
4. **高度可配置** - 200+参数可调
5. **符合C语言规范** - 使用C99标准，代码风格统一

该求解器在SAT竞赛中表现出色，是学习SAT求解器实现的优秀参考。