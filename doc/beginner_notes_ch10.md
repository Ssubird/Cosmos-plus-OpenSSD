# 第 10 章：企业级模式速查手册（面向 C 语言新手）

> **使用方式**：先依次阅读 `doc/beginner_notes.md` 的第 0~9 章；当你在 Cosmos+ OpenSSD 或自己公司的 BSP/SSD 项目里遇到“看不懂的封装”时，再回到本章查阅对应模式。每个模式都按照 *Why → 结构拆图 → 简化代码 → 阅读动作 → 实战练习* 展开，并辅以项目源码行号，方便你边看边对照。

---

## 10.1 模式总览速记卡

| 模式名称 | 解决的问题 | Cosmos+ 代码位置 | 新手快速动作 |
| --- | --- | --- | --- |
| **全局上下文（Context Object）** | 把几十个共享状态集中管理，避免散落全局变量 | `NVME_CONTEXT g_nvmeTask`【F:source/software/GreedyFTL-3.0.0/nvme/nvme_main.c†L71-L139】【F:source/software/GreedyFTL-3.0.0/nvme/nvme.h†L788-L823】 | 画结构体树，标注“谁写/谁读/何时更新” |
| **号码布式请求池 + 多队列流转** | 大量请求生命周期管理、避免频繁 `malloc` | `InitReqPool`/`GetFromFreeReqQ`/`PutTo*Q`【F:source/software/GreedyFTL-3.0.0/request_allocation.c†L41-L143】【F:source/software/GreedyFTL-3.0.0/request_allocation.h†L53-L93】 | 列出队列顺序，追踪每个 tag 的去向 |
| **多级结构体嵌套** | 把“控制器→队列→条目”写进类型，降低脑内记忆负担 | `NVME_STATUS`、`SSD_REQ_FORMAT`【F:source/software/GreedyFTL-3.0.0/nvme/nvme.h†L788-L823】【F:source/software/GreedyFTL-3.0.0/request_format.h†L51-L172】 | 画树状图，注明嵌套字段与枚举取值 |
| **函数指针与回调注册** | 把“谁触发”与“怎么处理”分离，降低耦合 | GIC 注册 NVMe 中断 + IO 命令分发【F:source/software/GreedyFTL-3.0.0/main.c†L112-L136】【F:source/software/GreedyFTL-3.0.0/nvme/nvme_io_cmd.c†L116-L154】 | 记录注册点，写“事件→处理函数→状态修改”表 |
| **事件驱动状态机** | 处理异步硬件事件时让主循环可读且可控 | `nvme_main()` 状态切换 + 中断更新状态【F:source/software/GreedyFTL-3.0.0/nvme/nvme_main.c†L73-L157】【F:source/software/GreedyFTL-3.0.0/nvme/host_lld.c†L86-L174】 | 列出状态、触发条件、出口条件 |
| **日志与断言多层宏** | 保证输出统一格式、附带上下文信息 | `ASSERT` 展开【F:source/software/GreedyFTL-3.0.0/nvme/debug.h†L47-L65】 | 把宏展开成普通代码，理解副作用 |
| **内存地图（Memory Map）** | 裸机环境手动划分 DRAM，确保地址不冲突 | `memory_map.h` + 初始化指针【F:source/software/GreedyFTL-3.0.0/memory_map.h†L57-L107】【F:source/software/GreedyFTL-3.0.0/data_buffer.c†L57-L85】 | 画地址尺子，标注每段用途与 cache 属性 |
| **LRU + 阻塞链表组合** | 缓存替换与“谁在等同一块数据”同步管理 | 数据缓冲 LRU 更新 + 阻塞链追踪【F:source/software/GreedyFTL-3.0.0/data_buffer.c†L52-L186】 | 画双向链表、阻塞链，打印命中/淘汰日志 |
| **切片流水线（RootData 风格封装）** | 从 NVMe 命令到 NAND 访问的层层拆解 | `ReqTransNvmeToSlice` 等函数链路【F:source/software/GreedyFTL-3.0.0/request_transform.c†L59-L190】【F:source/software/GreedyFTL-3.0.0/request_schedule.c†L1-L160】 | 写出“入口 API → 中间结构 → 出口队列”的顺序图 |
| **表驱动依赖检查** | 避免写放大、顺序冲突 | 行地址依赖表初始化/更新【F:source/software/GreedyFTL-3.0.0/request_transform.c†L57-L188】【F:source/software/GreedyFTL-3.0.0/address_translation.c†L63-L203】 | 标出表格字段含义，确认何时清零 |

> **小技巧**：真正阅读源码时，先在速记卡里找到模式名字，再跳到对应小节；别急着全读完，优先解决你当前手里的疑惑。

---

## 10.2 全局上下文：`g_nvmeTask` 为什么要这么大？

**Why（动机）**：NVMe 控制器有多个中断、命令与后台任务。若把状态拆成零散的全局变量，工程师会忘记谁在何时修改过。`NVME_CONTEXT g_nvmeTask` 把“当前状态、缓存使能、各队列配置”集中到一个对象里，中断和主循环统一读写它，让状态机逻辑不再分散。【F:source/software/GreedyFTL-3.0.0/nvme/nvme_main.c†L71-L157】【F:source/software/GreedyFTL-3.0.0/nvme/host_lld.c†L86-L152】

**结构拆图**：

```
NVME_CONTEXT (nvme.h)
├─ status, cacheEn                               // 当前 NVMe 任务状态、缓存开关
├─ adminQueueInfo (NVME_ADMIN_QUEUE_STATUS)      // 管理命令队列配置
├─ ioSqInfo[MAX_NUM_OF_IO_SQ]                    // 每个 IO 提交队列的寄存器参数
└─ ioCqInfo[MAX_NUM_OF_IO_CQ]                    // 每个 IO 完成队列的寄存器参数
```
【F:source/software/GreedyFTL-3.0.0/nvme/nvme.h†L788-L823】

**简化代码（结合主循环）**：

```c
volatile NVME_CONTEXT g_nvmeTask;

void nvme_main(void) {
    while (1) {
        if (g_nvmeTask.status == NVME_TASK_WAIT_CC_EN) {
            if (check_nvme_cc_en()) {
                set_nvme_admin_queue(1, 1, 1);
                set_nvme_csts_rdy(1);
                g_nvmeTask.status = NVME_TASK_RUNNING;
            }
        } else if (g_nvmeTask.status == NVME_TASK_RUNNING) {
            NVME_COMMAND nvmeCmd;
            if (get_nvme_cmd(..., nvmeCmd.cmdDword) == 1) {
                if (nvmeCmd.qID == 0) {
                    handle_nvme_admin_cmd(&nvmeCmd);
                } else {
                    handle_nvme_io_cmd(&nvmeCmd);
                    ReqTransSliceToLowLevel();
                }
            }
        }
        // ... shutdown & reset 省略
    }
}
```
【F:source/software/GreedyFTL-3.0.0/nvme/nvme_main.c†L73-L157】

**阅读动作**：
1. 打开 `nvme.h`，把 `NVME_CONTEXT` 画成树状图；右侧写上“谁会修改”“谁会读取”。
2. 在 `nvme_main.c` 标出每个 `status` 的进入和离开位置，理解状态机节奏。
3. 阅读 `dev_irq_handler()` 时，标记它在不同事件下如何写 `g_nvmeTask.status`，把“中断→状态”连线补到图里。【F:source/software/GreedyFTL-3.0.0/nvme/host_lld.c†L86-L152】

**练习**：
- 写一个小脚本/纸笔表格，记录“某状态由哪些函数写入”，帮助自己熟悉“所有状态都集中在一个结构体里”的模式。
- 增加一个调试打印函数，当 `status` 变化时输出旧值和新值，体验企业固件常见的“状态观察点”。

---

## 10.3 多级结构体：把层级关系写进类型

**Why**：企业级固件喜欢用嵌套结构体把硬件层级直接编码在类型里。Cosmos+ 中的 `NVME_STATUS` 与 `SSD_REQ_FORMAT` 就是例子：通过数组和嵌套字段，代码能在编译期限制访问路径，避免“数组越界/忘记字段”的低级错误。【F:source/software/GreedyFTL-3.0.0/nvme/nvme.h†L788-L823】【F:source/software/GreedyFTL-3.0.0/request_format.h†L51-L172】

**拆图示例**：

```
SSD_REQ_FORMAT (request_format.h)
├─ reqType/reqQueueType/reqCode/nvmeCmdSlotTag      // 基础信息
├─ logicalSliceAddr                                 // 逻辑切片地址
├─ reqOpt (bit-field)                               // NAND / 缓冲区选项
├─ dataBufInfo / nvmeDmaInfo / nandInfo             // 三段子结构，记录不同阶段需要的数据
└─ prevReq/nextReq + prevBlockingReq/nextBlockingReq// 队列和阻塞链指针
```

**简化代码段**：

```c
// NVMe 队列状态嵌套
typedef struct {
    unsigned char valid;
    unsigned char cqVector;
    unsigned short qSize;
    unsigned int pcieBaseAddrL;
    unsigned int pcieBaseAddrH;
} NVME_IO_SQ_STATUS;

typedef struct {
    unsigned int status;
    unsigned int cacheEn;
    NVME_ADMIN_QUEUE_STATUS adminQueueInfo;
    NVME_IO_SQ_STATUS ioSqInfo[MAX_NUM_OF_IO_SQ];
    NVME_IO_CQ_STATUS ioCqInfo[MAX_NUM_OF_IO_CQ];
} NVME_CONTEXT;
```
【F:source/software/GreedyFTL-3.0.0/nvme/nvme.h†L788-L823】

**阅读动作**：
1. 复制结构体到笔记里，把字段右侧改成“注释 + 修改者函数”，如 `ioSqInfo[i].valid // set_io_sq()`。
2. 把枚举/宏（`REQ_TYPE_*`、`REQ_QUEUE_TYPE_*`、`REQ_OPT_*`）写成一张对照表，阅读时随时查。【F:source/software/GreedyFTL-3.0.0/request_format.h†L51-L150】
3. 对照 `handle_nvme_admin_cmd()`/`handle_nvme_io_cmd()`，标出它们如何更新嵌套字段，帮助自己理解“多层结构体 + API”这种写法。

**练习**：
- 用纸笔画出 `SSD_REQ_FORMAT` 的嵌套关系，并标注“进入 NVMe 队列前哪些字段必须被填好”。
- 在 IDE 中对结构体使用“展开宏/跳转定义”功能，熟悉如何快速定位嵌套类型。

---

## 10.4 号码布式请求池：请求对象怎么来怎么回

**Why**：SSD 每个命令可能拆出多个“切片请求”“NAND 请求”。频繁 `malloc` 既慢又不稳定，于是项目把所有请求放进大数组 `REQ_POOL`，像发号码布一样借出/归还，同时用多个队列控制它们在不同阶段的排队情况。【F:source/software/GreedyFTL-3.0.0/request_allocation.c†L57-L143】【F:source/software/GreedyFTL-3.0.0/request_allocation.h†L53-L93】

**结构图**：

```
REQ_POOL (数组)
├─ freeReqQ → 刚初始化、空闲的号码
├─ sliceReqQ → 等待 FTL 切片调度
├─ nvmeDmaReqQ → 准备进行主机 DMA
├─ nandReqQ[ch][way] → 分发到每个通道/way 的 NAND 队列
└─ blocked* 队列 → 某些依赖条件未满足（缓存/行地址）
```

**核心代码节选**：

```c
void InitReqPool(void) {
    reqPoolPtr = (P_REQ_POOL)REQ_POOL_ADDR;
    freeReqQ.headReq = 0;
    freeReqQ.tailReq = AVAILABLE_OUNTSTANDING_REQ_COUNT - 1;
    // 逐个初始化队列、链表指针
    for (reqSlotTag = 0; reqSlotTag < AVAILABLE_OUNTSTANDING_REQ_COUNT; reqSlotTag++) {
        reqPoolPtr->reqPool[reqSlotTag].reqQueueType = REQ_QUEUE_TYPE_FREE;
        reqPoolPtr->reqPool[reqSlotTag].prevReq = reqSlotTag - 1;
        reqPoolPtr->reqPool[reqSlotTag].nextReq = reqSlotTag + 1;
        reqPoolPtr->reqPool[reqSlotTag].prevBlockingReq = REQ_SLOT_TAG_NONE;
        reqPoolPtr->reqPool[reqSlotTag].nextBlockingReq = REQ_SLOT_TAG_NONE;
    }
}

unsigned int GetFromFreeReqQ(void) {
    unsigned int reqSlotTag = freeReqQ.headReq;
    if (reqSlotTag == REQ_SLOT_TAG_NONE) {
        SyncAvailFreeReq();
        reqSlotTag = freeReqQ.headReq;
    }
    // 把 head 移除队列
    if (reqPoolPtr->reqPool[reqSlotTag].nextReq != REQ_SLOT_TAG_NONE) {
        freeReqQ.headReq = reqPoolPtr->reqPool[reqSlotTag].nextReq;
        reqPoolPtr->reqPool[freeReqQ.headReq].prevReq = REQ_SLOT_TAG_NONE;
    } else {
        freeReqQ.headReq = freeReqQ.tailReq = REQ_SLOT_TAG_NONE;
    }
    reqPoolPtr->reqPool[reqSlotTag].reqQueueType = REQ_QUEUE_TYPE_NONE;
    freeReqQ.reqCnt--;
    return reqSlotTag;
}
```
【F:source/software/GreedyFTL-3.0.0/request_allocation.c†L57-L143】

**阅读动作**：
1. 在纸上画出“命令进入 → 借号 → 填字段 → 入队 → 完成归还”的流程图。
2. 搜索 `PutToFreeReqQ` / `PutToSliceReqQ` / `SelectLowLevelReqQ` 等函数，写表格记录“状态转换前需要满足的条件”。
3. 跟踪 `SSD_REQ_FORMAT` 的 `reqQueueType` 字段，确认它在每次队列迁移时被正确更新。

**练习**：
- 跟着 `ReqTransNvmeToSlice()` 的代码走一遍，记录一个写请求会借多少个 tag、分别进入哪些队列。【F:source/software/GreedyFTL-3.0.0/request_transform.c†L78-L157】
- 在调试日志里打印 `reqSlotTag`、`reqQueueType`，观察同一个请求在生命周期中如何流动。

---

## 10.5 日志与断言：宏背后的“多层封装”

**Why**：项目通过宏把日志/断言统一格式化。例如 `ASSERT(X)` 失败时自动打印文件、行号，并停在死循环等待调试器连接。了解宏展开方式可以帮助你在阅读时快速理解“这行代码到底干了什么”。【F:source/software/GreedyFTL-3.0.0/nvme/debug.h†L47-L65】

**宏展开示例**：

```c
#define ASSERT(X)                                   \
if (!(X)) {                                         \
    xil_printf("\r\n\nerror in %s: Line %d\r\n", __FILE__, __LINE__); \
    while(1);                                       \
}
```

**阅读动作**：
1. 遇到 `ASSERT()` 时，把它脑内还原成上面的等价代码，理解“失败后程序会卡住等待调试”。
2. 注意宏内部是否访问了多次参数（避免副作用），例如 `LOG_DEBUG(expr)` 可能执行多遍参数。
3. 查看调用位置：`handle_nvme_io_write()` 中的断言确保 LBA 对齐、PRP 指针合法，是检查输入的第一道关。【F:source/software/GreedyFTL-3.0.0/nvme/nvme_io_cmd.c†L104-L144】

**练习**：
- 模仿 `ASSERT` 写一个 `LOG_ERROR(tag, fmt, ...)`，输出时附带 `reqSlotTag`，帮助自己理解宏中可变参数的写法。
- 把常用日志封装列入速查表：例如 `xil_printf`、`xil_printf_debug`（若有）分别用于什么级别。

---

## 10.6 回调注册与函数指针：事件如何“派发”

**Why**：硬件事件往往通过中断进入系统，软件功能则通过函数指针/回调解耦。Cosmos+ 在 `main.c` 里注册 NVMe 中断处理函数，真正的 NVMe 逻辑则在 `host_lld.c` 中处理；命令执行同样借助 `switch` 把 opcode 分发给具体函数。【F:source/software/GreedyFTL-3.0.0/main.c†L112-L136】【F:source/software/GreedyFTL-3.0.0/nvme/host_lld.c†L86-L174】

**注册流程**：

```c
Xil_ExceptionInit();
XScuGic_CfgInitialize(&GicInstance, IntcConfig, IntcConfig->CpuBaseAddress);
Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
                             (Xil_ExceptionHandler)XScuGic_InterruptHandler,
                             &GicInstance);
XScuGic_Connect(&GicInstance, 61,
                (Xil_ExceptionHandler)dev_irq_handler,
                (void *)0);
XScuGic_Enable(&GicInstance, 61);
dev_irq_init();
```
【F:source/software/GreedyFTL-3.0.0/main.c†L112-L133】

**事件响应**：

```c
void dev_irq_handler(void) {
    DEV_IRQ_REG devReg;
    devReg.dword = IO_READ32(DEV_IRQ_STATUS_REG_ADDR);
    IO_WRITE32(DEV_IRQ_CLEAR_REG_ADDR, devReg.dword);

    if (devReg.pcieLink) { ... g_nvmeTask.status = NVME_TASK_RESET; }
    if (devReg.nvmeCcEn) {
        NVME_STATUS_REG nvmeReg = ...;
        g_nvmeTask.status = (nvmeReg.ccEn) ? NVME_TASK_WAIT_CC_EN
                                           : NVME_TASK_RESET;
    }
    if (devReg.nvmeCcShn) { g_nvmeTask.status = NVME_TASK_SHUTDOWN; }
}
```
【F:source/software/GreedyFTL-3.0.0/nvme/host_lld.c†L86-L152】

**命令分发**：

```c
void handle_nvme_io_cmd(NVME_COMMAND *nvmeCmd) {
    NVME_IO_COMMAND *nvmeIOCmd = (NVME_IO_COMMAND*)nvmeCmd->cmdDword;
    switch (nvmeIOCmd->OPC) {
        case IO_NVM_WRITE: handle_nvme_io_write(nvmeCmd->cmdSlotTag, nvmeIOCmd); break;
        case IO_NVM_READ:  handle_nvme_io_read(nvmeCmd->cmdSlotTag, nvmeIOCmd);  break;
        case IO_NVM_FLUSH: ...; break;
        default: xil_printf("Not Support IO Command OPC: %X\r\n", opc); ASSERT(0);
    }
}
```
【F:source/software/GreedyFTL-3.0.0/nvme/nvme_io_cmd.c†L116-L154】

**阅读动作**：
1. 写一张“中断号 → 注册函数 → 最终处理逻辑”的表。NVMe 中断号 `61` → `dev_irq_handler` → 修改 `g_nvmeTask.status`。
2. 在 `handle_nvme_io_write()` 中找到 `ReqTransNvmeToSlice()` 的调用，记录命令在回调中被拆分成哪些步骤。【F:source/software/GreedyFTL-3.0.0/nvme/nvme_io_cmd.c†L104-L144】
3. 对于函数指针数组（如果后续模块使用），记得查初始化位置与触发条件。

**练习**：
- 自己写一个简单的 `register_handler(eventId, handler)` 函数数组，并在小 demo 中模拟触发，体会“注册→触发”的节奏。
- 把 `dev_irq_handler()` 的打印语句开启，观察真实硬件事件发生时的顺序。

---

## 10.7 切片流水线：从 NVMe 命令到 NAND 操作

**Why**：你在问题中提到“先封装基础结构体，再包几层，再设计 API，然后状态机使用”——Cosmos+ 的请求流水线正是这样：
1. NVMe IO 命令解析阶段（`handle_nvme_io_write/read`）。
2. 请求切片阶段（`ReqTransNvmeToSlice`）。
3. 调度/依赖检查阶段（`ReqTransSliceToLowLevel`、`SelectLowLevelReqQ` 等）。
4. 最终 NAND 访问或 DMA 执行。

**代码桥接示例**：

```c
void ReqTransNvmeToSlice(unsigned int cmdSlotTag,
                         unsigned int startLba,
                         unsigned int nlb,
                         unsigned int cmdCode) {
    reqSlotTag = GetFromFreeReqQ();             // 1. 借号
    reqPoolPtr->reqPool[reqSlotTag].reqType = REQ_TYPE_SLICE;
    reqPoolPtr->reqPool[reqSlotTag].reqCode = (cmdCode == IO_NVM_WRITE)
                                              ? REQ_CODE_WRITE : REQ_CODE_READ;
    reqPoolPtr->reqPool[reqSlotTag].logicalSliceAddr = tempLsa;
    reqPoolPtr->reqPool[reqSlotTag].nvmeDmaInfo.startIndex = nvmeDmaStartIndex;
    PutToSliceReqQ(reqSlotTag);                 // 2. 入切片队列
    // ... 根据 LBA 跨越情况继续拆分多个 reqSlotTag
}
```
【F:source/software/GreedyFTL-3.0.0/request_transform.c†L78-L157】

> **观察点**：函数并没有直接访问 NAND；它只关心“把 NVMe 命令切成均匀的片段，并记录需要的 DMA 信息”，这就让后续模块只需关注“如何安排这些片段”。

**下一站：依赖处理与调度**：

```c
void EvictDataBufEntry(unsigned int originReqSlotTag) {
    unsigned int reqSlotTag = GetFromFreeReqQ();    // 再借一个 tag 做 NAND 写回
    reqPoolPtr->reqPool[reqSlotTag].reqType = REQ_TYPE_NAND;
    reqPoolPtr->reqPool[reqSlotTag].reqOpt.rowAddrDependencyCheck = REQ_OPT_ROW_ADDR_DEPENDENCY_CHECK;
    SelectLowLevelReqQ(reqSlotTag);                // 根据通道/way 排队
}
```
【F:source/software/GreedyFTL-3.0.0/request_transform.c†L162-L188】

`SelectLowLevelReqQ()`、`ReqTransSliceToLowLevel()` 会进一步把请求送入不同的 NAND 队列，并触发行地址依赖表检查，保证不会出现对同一个物理块的冲突。【F:source/software/GreedyFTL-3.0.0/request_schedule.c†L1-L160】【F:source/software/GreedyFTL-3.0.0/request_transform.c†L162-L188】

**阅读动作**：
1. 在纸上画一个四阶段流程（NVMe → Slice → DMA/NAND → 完成）。每个阶段写上入口函数、关键结构体字段。
2. 跟踪同一个 `reqSlotTag` 在 `ReqTransNvmeToSlice` → `SelectLowLevelReqQ` → `PutToNandReqQ` 之间的字段变化，理解“层层封装”的真正含义。
3. 练习把自己的业务需求套进这个框架：例如“如何在切片阶段追加加密标记？”。

**练习**：
- 写出 `ReqTransNvmeToSlice()` 中 `tempNumOfNvmeBlock`、`nvmeDmaStartIndex` 的计算公式，并解释为什么要在跨切片时拆成多条请求。
- 结合 `SSD_REQ_FORMAT`，给出一个示例表：当 `reqType=REQ_TYPE_NAND` 时，`nandInfo` 中的字段分别记录了什么含义。

---

## 10.8 事件驱动状态机：主循环不再“乱跳”

**Why**：NVMe 固件同时面对主机命令、中断、后台清理。项目把 `nvme_main()` 写成状态机，只有状态满足条件时才执行对应逻辑。中断负责修改状态，从而驱动主循环进入下一个阶段。【F:source/software/GreedyFTL-3.0.0/nvme/nvme_main.c†L73-L157】【F:source/software/GreedyFTL-3.0.0/nvme/host_lld.c†L86-L152】

**状态表（节选）**：

| 状态 | 进入条件 | 主循环动作 | 退出条件 |
| --- | --- | --- | --- |
| `NVME_TASK_WAIT_CC_EN` | 中断检测到 `CC.EN=1` | 初始化 Admin Queue、设置 `RDY` | `check_nvme_cc_en()` 返回 1 → 切换到 `RUNNING` |
| `NVME_TASK_RUNNING` | 队列就绪 | 获取命令、分发给 Admin/IO 处理 | 没有命令时空转；若 `CC.SHN=1`，中断将状态改为 `SHUTDOWN` |
| `NVME_TASK_SHUTDOWN` | 主机发起关机 | 关闭队列、刷新坏块表 | 完成后切换到 `WAIT_RESET` |
| `NVME_TASK_RESET` | PCIe 链路掉线或主机复位 | 等待主机重新使能 | `CC.EN` 清零后转到 `IDLE` |

**阅读动作**：
1. 用颜色标注 `nvme_main()` 中的 `if/else` 分支，帮助自己快速区分状态块。
2. 在 `dev_irq_handler()` 里写注释，说明每个 `devReg` 位对应哪个状态转换。
3. 当需要加新功能时，先问自己：它属于现有状态还是应该新增状态？

**练习**：
- 实现一个迷你状态机（例如缓存淘汰流程 `IDLE→SCAN→EVICT→DONE`），并打印状态转换日志。
- 把 `nvme_main()` 中 `exeLlr` 等辅助变量的作用写在笔记上，理解状态机如何与后台任务协同。

---

## 10.9 内存地图：地址硬编码也有套路

**Why**：裸机固件必须手动分配 DRAM 区域，避免重叠。`memory_map.h` 用宏定义写出每段起止地址，再由各模块把地址转换成指针（例如 `InitDataBuf()`）。理解内存地图后，你就能快速知道“这个指针指向哪一块物理内存”。【F:source/software/GreedyFTL-3.0.0/memory_map.h†L57-L107】【F:source/software/GreedyFTL-3.0.0/data_buffer.c†L57-L86】

**地址尺子**：

```
0x1000_0000 ── DATA_BUFFER_BASE_ADDR
              ├─ Data buffer (主缓存)
              ├─ Temporary buffer
              ├─ Spare buffer
              └─ Reserved buffer
0x1700_0000 ── COMPLETE_FLAG_TABLE_ADDR ...
0x1800_0000 ── DATA_BUFFER_MAP_ADDR
              ├─ 哈希表/映射表
              └─ FTL 管理结构体
...
REQ_POOL_ADDR ── 请求池所在区域
ROW_ADDR_DEPENDENCY_TABLE_ADDR ── 行地址依赖表
```
【F:source/software/GreedyFTL-3.0.0/memory_map.h†L57-L104】

**初始化指针**：

```c
void InitDataBuf(void) {
    dataBufMapPtr = (P_DATA_BUF_MAP)DATA_BUFFER_MAP_ADDR;
    dataBufHashTablePtr = (P_DATA_BUF_HASH_TABLE)DATA_BUFFFER_HASH_TABLE_ADDR;
    tempDataBufMapPtr = (P_TEMPORARY_DATA_BUF_MAP)TEMPORARY_DATA_BUFFER_MAP_ADDR;
    // 逐个清零、建立 LRU 链
}
```
【F:source/software/GreedyFTL-3.0.0/data_buffer.c†L57-L86】

**阅读动作**：
1. 把所有地址常量抄下来，画成一条水平线，标注用途以及“缓存/非缓存”属性。
2. 在调试时，如果指针异常，先核对它是否落在正确范围；若需要新增结构体，检查是否有地址空间可用。
3. 与硬件同事确认“哪些区域需要缓存、哪些必须非缓存”，避免引入一致性问题。

**练习**：
- 使用 Excel/画图工具制作“内存段列表”，列出每段大小、用途、对应结构体，作为随身备忘录。
- 阅读 `main.c` 中 TLB 属性设置的循环，理解为什么 0x0020_0000~0x017F_FFFF 配置为“uncached & nonbuffered”。【F:source/software/GreedyFTL-3.0.0/main.c†L90-L136】

---

## 10.10 LRU + 阻塞链表：缓存命中与排队策略

**Why**：数据缓冲既要做 LRU（最近最少使用）替换，又要追踪哪些请求在等待同一片数据。项目通过两个数据结构组合解决：
- `dataBufLruList` 维护双向链表，越靠尾越冷。
- 每个条目有 `blockingReqTail`，把等待它的请求串成链，解除阻塞时一次唤醒。
【F:source/software/GreedyFTL-3.0.0/data_buffer.c†L52-L186】

**命中更新逻辑**：

```c
unsigned int CheckDataBufHit(unsigned int reqSlotTag) {
    bufEntry = dataBufHashTablePtr->dataBufHash[hash].headEntry;
    while (bufEntry != DATA_BUF_NONE) {
        if (dataBufMapPtr->dataBuf[bufEntry].logicalSliceAddr == logicalSliceAddr) {
            // 把命中节点移到 LRU 头
            ...
            return bufEntry;
        }
        bufEntry = dataBufMapPtr->dataBuf[bufEntry].hashNextEntry;
    }
    return DATA_BUF_FAIL;
}
```
【F:source/software/GreedyFTL-3.0.0/data_buffer.c†L88-L142】

**阻塞链维护**：

```c
void UpdateDataBufEntryInfoBlockingReq(unsigned int bufEntry, unsigned int reqSlotTag) {
    if (dataBufMapPtr->dataBuf[bufEntry].blockingReqTail != REQ_SLOT_TAG_NONE) {
        unsigned int tail = dataBufMapPtr->dataBuf[bufEntry].blockingReqTail;
        reqPoolPtr->reqPool[reqSlotTag].prevBlockingReq = tail;
        reqPoolPtr->reqPool[tail].nextBlockingReq = reqSlotTag;
    }
    dataBufMapPtr->dataBuf[bufEntry].blockingReqTail = reqSlotTag;
}
```
【F:source/software/GreedyFTL-3.0.0/data_buffer.c†L176-L185】

**阅读动作**：
1. 画出 LRU 头尾、hash 链、阻塞链之间的关系，理解为什么需要三个指针集。
2. 跟踪一个读请求：命中时如何移动到头；未命中时如何挂到阻塞链，等待 NAND 完成。
3. 调试时打印 LRU 头/尾索引以及阻塞链长度，快速判断缓存压力。

**练习**：
- 写一个 Python/纸面模拟：给 5 个缓存条目，随机访问并更新 LRU，记录链表变化。
- 尝试给阻塞链添加“计数器”，当超过阈值时打印警告，理解工程中如何监控瓶颈。

---

## 10.11 表驱动依赖：避免冲突写入

**Why**：写入 NAND 时需要考虑同一物理块的顺序，避免出现页序错误或擦除冲突。`rowAddrDependencyTable` 通过表格记录每个 block 当前允许编程的页、被阻塞的请求数量；请求在入队前检查该表，必要时挂到 `blockedByRowAddrDepReqQ`。【F:source/software/GreedyFTL-3.0.0/request_transform.c†L57-L188】【F:source/software/GreedyFTL-3.0.0/request_allocation.h†L71-L76】

**初始化**：

```c
void InitDependencyTable(void) {
    rowAddrDependencyTablePtr = (P_ROW_ADDR_DEPENDENCY_TABLE)ROW_ADDR_DEPENDENCY_TABLE_ADDR;
    for (blockNo = 0; blockNo < MAIN_BLOCKS_PER_DIE; blockNo++)
        for (wayNo = 0; wayNo < USER_WAYS; wayNo++)
            for (chNo = 0; chNo < USER_CHANNELS; chNo++) {
                rowAddrDependencyTablePtr->block[chNo][wayNo][blockNo].permittedProgPage = 0;
                rowAddrDependencyTablePtr->block[chNo][wayNo][blockNo].blockedReadReqCnt = 0;
                rowAddrDependencyTablePtr->block[chNo][wayNo][blockNo].blockedEraseReqFlag = 0;
            }
}
```
【F:source/software/GreedyFTL-3.0.0/request_transform.c†L57-L76】

**与地址映射协作**：地址映射模块初始化坏块表、逻辑到物理的映射，提供 `AddrTransRead/Write` 等 API，供请求流水线查询或更新虚拟地址。【F:source/software/GreedyFTL-3.0.0/address_translation.c†L63-L203】

**阅读动作**：
1. 把表结构的三维索引写清楚（Channel → Way → Block），理解每维含义。
2. 查 `SelectLowLevelReqQ()` 或相关函数，看看在将请求投入 NAND 队列前，如何根据依赖表决定是否阻塞。
3. 当你怀疑出现“写冲突”时，优先打印依赖表条目，而不是盲目怀疑硬件。

**练习**：
- 根据 `InitDependencyTable`，写出一个伪代码：当写请求完成时，如何更新 `permittedProgPage` 和解除阻塞。
- 在笔记里写下“坏块重映射流程”：`InitAddressMap()` → `RemapBadBlock()`，帮助自己理解映射表与依赖表的协作。【F:source/software/GreedyFTL-3.0.0/address_translation.c†L100-L203】

---

## 10.12 实战路线：从“看不懂”到“能定位”

1. **先查模式**：遇到陌生写法，先回到 10.1 速记卡找到名字，再跳到对应小节，看它的 Why/How/练习。
2. **边读边画**：每读一个结构体或状态机，就画图记录；别指望一次性记住所有字段。
3. **维护资源账本**：为请求池、缓冲区、回调注册分别写一页“借出 → 使用 → 归还/解除”的表格，保证自己能回答“这个资源什么时候释放？”。
4. **动手练习**：每节的练习题都与企业固件日常工作高度相关。把练习做完，再回代码时自然就知道在哪里加日志、如何定位问题。
5. **复盘三个核心问题**：
   - 为什么所有 NVMe 状态要塞进 `g_nvmeTask`？
   - 一个请求的 tag 从借出到归还要经过哪些函数？
   - 哪个事件会把状态机从 `RESET` 拉回 `RUNNING`？

当你能清楚回答这三点，就已经掌握了 Cosmos+ 乃至大多数企业级 SSD 固件的主流封装模式，可以自信地继续深入 BSP、FTL 以及自家项目的特定模块。
