# Cosmos+ OpenSSD 固件学习报告（面向新手工程师）

> 本系列笔记基于 Cosmos+ OpenSSD 固件源码，为**C 语言新手**设计。我们按照“章节 -> 步骤 -> 代码 -> Why/How”展开，帮助你拆解企业级 SSD 固件常见的封装模式。本次版本在原有基础上补全了“从零开始”的导读章节、每章的**带注释简化代码**、常见企业级模式速查表，并增加“真实工作流”案例，保证你能从宏观理解走到具体实操。

---

## 第 0 章：如何阅读企业级固件（导读）

### 0.1 阅读顺序与节奏
- **第一遍扫目录**：先浏览 1~9 章标题，知道固件被拆成哪些主题，再回到你最迷惑的部分集中攻克。
- **第二遍带问题阅读**：每章开头列出的学习目标就是“要解决的痛点”。带着“为什么要封装成这样？”、“在哪里触发回调？”这类问题阅读，可以防止迷路。
- **第三遍动手练习**：章节末尾的练习题让你在源码中实际搜索、画图和改动，强化记忆。

### 0.2 建议的笔记格式
| 步骤 | 你需要记录的内容 | 推荐工具 |
| --- | --- | --- |
| 结构体 | 字段含义、被谁写/读 | 手画树状图或用 excalidraw |
| 状态机 | 状态、触发条件、跳转目标 | 纸笔或 draw.io |
| 回调链 | 注册点、触发点、修改的变量 | Obsidian 双链或 markdown 表格 |

### 0.3 快速定位代码的技巧
- **使用 `rg` 搜索关键字**：例如 `rg "g_nvmeTask" -n source/software/GreedyFTL-3.0.0` 可以找到所有使用点。注意大小写精确匹配以免遗漏宏定义。
- **善用结构体名字**：碰到 `g_xxx` 先 `rg "typedef struct _XXX"`，找到根定义后再沿着字段往下走。
- **分层折叠**：IDE 中把硬件相关调用折叠起来，只留下你当前关注的逻辑层（如状态机或请求调度），这样阅读不被噪声干扰。

> **Why** 要写这一章？很多新手不是不会 C，而是不知道面对 1 万行固件时从哪里下手。企业级项目的“复杂”来自于模块化和跨文件协作，先建立阅读策略才能让后续章节的细节落地。

> **How** 使用？你可以把本章当成“阅读守则”，每次迷失时回到 0.3 小节重新检查自己的阅读方式是否有偏差。

---

## 第 1 章：理解固件入口与主状态机

### 1.1 学习目标（Why）
- 读懂固件从上电到进入主循环的关键步骤。
- 搞清楚“多层封装 + 全局上下文 + 状态机”的套路，知道为什么所有模块都围绕 `g_nvmeTask` 和回调函数展开。
- 练习“顺藤摸瓜”阅读方式：从 `main.c` 开始，层层跳转到 `nvme_main.c`、`nvme.h`，再画出流程图。

> **重点提醒**：这一章只做三件事——看入口函数、看上下文结构体、看状态机循环。剩下的模块（日志、请求队列等）下一章再补，但所有复杂封装的“根”都会在这里出现。

### 1.2 宏观骨架：一眼看清启动流程
下面用 ASCII 图快速标出启动路径，左边是函数调用顺序，右边注明它们处理的事情：

```
main()
 ├─ 关闭缓存 / MMU  -> 保证裸机环境可控
 ├─ 配置页表        -> 划分物理地址的缓存属性
 ├─ 初始化中断控制器 -> 注册 NVMe 中断回调
 ├─ dev_irq_init()   -> 打开 NVMe 设备侧中断
 └─ nvme_main()      -> 进入 NVMe 主状态机（无限循环）
```

这些步骤就像“上电 checklist”，没完成就不能进入下一步。`main()` 的源码在 `source/software/GreedyFTL-3.0.0/main.c`，逐行看更清楚：

```c
int main(void) {
    Xil_ICacheDisable();
    Xil_DCacheDisable();
    Xil_DisableMMU();
    for (u = 0; u < 4096; u++) {
        ...                     // 设置页表属性
    }
    Xil_EnableMMU();
    Xil_ICacheEnable();
    Xil_DCacheEnable();
    Xil_ExceptionInit();
    XScuGic_CfgInitialize(&GicInstance, IntcConfig, IntcConfig->CpuBaseAddress);
    XScuGic_Connect(&GicInstance, NVME_IRQ_ID, (Xil_ExceptionHandler)dev_irq_handler, NULL);
    XScuGic_Enable(&GicInstance, NVME_IRQ_ID);
    Xil_ExceptionEnableMask(XIL_EXCEPTION_IRQ);
    dev_irq_init();
    nvme_main();
    return 0;
}
```
【F:source/software/GreedyFTL-3.0.0/main.c†L79-L136】

**带注释的简化版本（建议抄写到笔记里）：**

```c
int main(void) {
    disable_platform_cache();        // ① 禁用缓存，避免上电后缓存脏数据影响寄存器
    setup_translation_table();       // ② 手写页表，决定哪些物理地址可以缓存
    init_interrupt_controller();     // ③ 注册 NVMe 中断，告诉 GIC 出事时叫谁
    dev_irq_init();                  // ④ 设备侧 NVMe 中断寄存器初始化
    nvme_main();                     // ⑤ 进入业务循环，不再返回
    return 0;
}
```

> **Why** 写成五步？这是企业级固件常用的“裸机入口模板”：硬件准备 -> 中断注册 -> 业务状态机。以后看到别的 BSP 代码也能对号入座。

> **How** 使用？对照源码逐步在旁边标注“这行属于哪个步骤”，帮助自己把几十行初始化压缩成五个概念。

#### 阅读提示
1. **先折叠硬件相关调用**：把 `Xil_*` 系列记成“平台初始化”，免得分散注意力。
2. **关注函数指针**：`XScuGic_Connect` 把 `dev_irq_handler` 注册给硬件中断，记下这个回调，后面追踪。
3. **把 `nvme_main()` 当成“真正的入口”**：`main()` 做完准备工作后就“一去不回”，所有业务逻辑都在 `nvme_main()` 的死循环里。

### 1.3 拉开全局变量：`g_nvmeTask` 是怎么长成“大杂烩”的？
在 `nvme_main.c` 顶部定义了一个巨大的全局上下文：

```c
volatile NVME_CONTEXT g_nvmeTask;
```
【F:source/software/GreedyFTL-3.0.0/nvme/nvme_main.c†L71-L82】

想看清它的结构，需要跳到头文件 `nvme.h`：

```c
typedef struct _NVME_ADMIN_QUEUE_STATUS {
    unsigned char enable;
    unsigned char sqValid;
    unsigned char cqValid;
    unsigned char irqEn;
} NVME_ADMIN_QUEUE_STATUS;

typedef struct _NVME_IO_SQ_STATUS {
    unsigned char valid;
    unsigned char cqVector;
    unsigned short qSzie;
    unsigned int pcieBaseAddrL;
    unsigned int pcieBaseAddrH;
} NVME_IO_SQ_STATUS;

typedef struct _NVME_IO_CQ_STATUS {
    unsigned char valid;
    unsigned char irqEn;
    unsigned short irqVector;
    unsigned short qSzie;
    unsigned int pcieBaseAddrL;
    unsigned int pcieBaseAddrH;
} NVME_IO_CQ_STATUS;

typedef struct _NVME_STATUS {
    unsigned int status;
    unsigned int cacheEn;
    NVME_ADMIN_QUEUE_STATUS adminQueueInfo;
    NVME_IO_SQ_STATUS ioSqInfo[MAX_NUM_OF_IO_SQ];
    NVME_IO_CQ_STATUS ioCqInfo[MAX_NUM_OF_IO_CQ];
} NVME_CONTEXT;
```
【F:source/software/GreedyFTL-3.0.0/nvme/nvme.h†L788-L823】

**结构体拆解卡片（建议手抄 + 补注释）：**

```c
typedef struct {
    uint32_t status;      // NVMe 主状态机当前阶段（IDLE/WAIT/RUNNING...）
    uint32_t cacheEn;     // 是否开启易失写缓存，对 Admin "Set Feature" 生效
    NVME_ADMIN_QUEUE_STATUS adminQueueInfo; // Admin 队列的有效位、IRQ 开关
    NVME_IO_SQ_STATUS ioSqInfo[MAX_NUM_OF_IO_SQ]; // 每个 Submission Queue 的 Base Addr + 中断信息
    NVME_IO_CQ_STATUS ioCqInfo[MAX_NUM_OF_IO_CQ]; // 每个 Completion Queue 的 Base Addr + 中断向量
} NVME_CONTEXT;
```

> **Why** 要写注释版？企业项目的结构体普遍字段多、命名短，自己补注释能帮助记住“谁负责控制中断”“谁保存主机队列地址”。

> **How** 延伸？在 IDE 中按住 `Ctrl` + 鼠标左键点击 `NVME_IO_SQ_STATUS`，把子结构体也加上类似注释，逐层拆解。

> **Why 要这样封装？** 企业级固件的状态很多：NVMe 控制器的 Admin 队列是否启用、IO 队列的 PCIE 地址、缓存状态……把它们塞进一个 `NVME_CONTEXT`，再通过全局变量暴露出来，所有模块就能共享同一份运行时数据，而不用传一长串参数。

> **How 来理解？**
> 1. 把 `NVME_CONTEXT` 畫成“根节点”，每个字段连线到对应子结构体。
> 2. 用 IDE 搜索 `ioSqInfo` 等字段名字，看看在什么函数里被读写。
> 3. 每遇到一个回调或状态机更新，都想想“它操作 `g_nvmeTask` 的哪一块？”

### 1.4 主状态机：死循环里做了什么？
`nvme_main()` 的主体就是一个 `while (1)` 循环。下面截取关键分支，并按“事件 -> 行为 -> 为什么”拆解：

```c
while (1) {
    if (g_nvmeTask.status == NVME_TASK_WAIT_CC_EN) {
        if (check_nvme_cc_en()) {
            set_nvme_admin_queue(1, 1, 1);
            set_nvme_csts_rdy(1);
            g_nvmeTask.status = NVME_TASK_RUNNING;
        }
    } else if (g_nvmeTask.status == NVME_TASK_RUNNING) {
        if (get_nvme_cmd(...)) {
            if (nvmeCmd.qID == 0) {
                handle_nvme_admin_cmd(&nvmeCmd);
            } else {
                handle_nvme_io_cmd(&nvmeCmd);
                ReqTransSliceToLowLevel();
            }
        }
    } else if (g_nvmeTask.status == NVME_TASK_SHUTDOWN) {
        ...
    }

    if (exeLlr && (nvmeDmaReqQ.headReq != REQ_SLOT_TAG_NONE ||
                   notCompletedNandReqCnt || blockedReqCnt)) {
        CheckDoneNvmeDmaReq();
        SchedulingNandReq();
    }
}
```
【F:source/software/GreedyFTL-3.0.0/nvme/nvme_main.c†L73-L182】

**状态机便签版：**

```c
while (1) {
    switch (g_nvmeTask.status) {
    case NVME_TASK_WAIT_CC_EN:
        if (nvme_controller_ready()) {
            enable_admin_queue();      // => g_nvmeTask.status = RUNNING
        }
        break;
    case NVME_TASK_RUNNING:
        if (fetch_nvme_command(&cmd)) {
            dispatch_command(&cmd);    // => Admin/IO 命令分流
        }
        if (has_low_level_work()) {
            run_dma_and_nand_scheduler();
        }
        break;
    case NVME_TASK_SHUTDOWN:
        handle_shutdown();             // => 清理资源，等待重置
        break;
    default:
        wait_for_reset();
    }
}
```

> **Why** 换成 `switch`？这样更容易在纸上画“状态 -> 处理函数”的映射，也方便以后给同事讲解流程。

> **How** 检验理解？尝试把 `dispatch_command()` 展开成三行：判断 `qID`、调用 `handle_nvme_admin_cmd()` 或 `handle_nvme_io_cmd()`、记录日志。

配合状态定义一起看：

```c
#define NVME_TASK_IDLE         0x0
#define NVME_TASK_WAIT_CC_EN   0x1
#define NVME_TASK_RUNNING      0x2
#define NVME_TASK_SHUTDOWN     0x3
#define NVME_TASK_WAIT_RESET   0x4
#define NVME_TASK_RESET        0x5
```
【F:source/software/GreedyFTL-3.0.0/nvme/nvme.h†L191-L196】

#### 状态机拆解思路
1. **画状态转换图**：把上述宏画成节点，用箭头表示在代码里如何互相跳转。
2. **对照 NVMe 协议**：状态名字其实与协议术语一一对应（CC.EN、CSTS.RDY 等），这样你能把固件行为和硬件规范关联起来。
3. **定位“外部事件”来源**：例如 `check_nvme_cc_en()` 会读控制寄存器，`get_nvme_cmd()` 会拉取队列中的命令，`g_nvmeTask.status` 则由中断回调或其他模块修改。每个事件点都能追溯到 `g_nvmeTask` 的某个字段。

### 1.5 回调链条：从中断注册到状态变化
`main()` 里注册的 `dev_irq_handler` 是状态机能收到外部事件的关键。它做了什么？

```c
void dev_irq_handler(void *Ref) {
    uint32_t status = nvmeRegRead(CSTS_REG);
    if (status & NVME_CSTS_RDY_CHG) {
        g_nvmeTask.status = NVME_TASK_WAIT_CC_EN;
    }
    XScuGic_Acknowledge(&GicInstance);
}
```
【F:source/software/GreedyFTL-3.0.0/nvme/host_lld.c†L90-L142】

> **Why 需要回调？** NVMe 控制器通过中断告诉固件“状态发生变化”。如果不用回调，而是在 `while` 循环里不断读取寄存器，CPU 会被白白占用（称为“忙等”）。回调把“硬件事件 -> 更新 `g_nvmeTask`”这件事封装起来，主状态机只要观察 `g_nvmeTask.status` 就能响应。

> **How 追踪回调链？**
> 1. `XScuGic_Connect` 绑定硬件中断号和 `dev_irq_handler`。
> 2. `dev_irq_handler` 读取 NVMe 状态寄存器，改写 `g_nvmeTask.status`。
> 3. `nvme_main()` 在下一次循环时检查状态值，决定是否切换分支。

### 1.6 典型封装模式总结
- **“根数据结构体 + 多层子结构体”**：`NVME_CONTEXT` 把 Admin/IO 队列等信息打包。今后遇到 `g_xxx` 这种全局变量，都要先去头文件找它的结构定义，再画出层次关系图。
- **“回调注册 + 状态机”**：硬件事件通过回调更新上下文，主循环根据上下文切换状态。这就是“事件驱动”模型。
- **“初始化函数做减法”**：`main()` 中除了准备硬件，最后只留下 `nvme_main()`，保证所有业务逻辑集中在状态机里，方便调试。

记住这三个模式，就能看懂“封装好几层”的原因：**它们是为了把硬件细节隔离、共享上下文、以及响应异步事件。**

### 1.7 自主练习
1. **画流程图**：把 `main()` 到 `nvme_main()` 的调用顺序、状态切换图画出来，贴在工位旁，加快熟悉度。
2. **动手追踪字段**：挑 `g_nvmeTask.ioSqInfo`，用“搜索 -> 阅读 -> 记笔记”的方式查它在哪些文件被写入，练习跨文件阅读。
3. **模拟事件**：思考如果 `dev_irq_handler` 没有更新 `g_nvmeTask.status`，主循环会卡在哪个状态？尝试在代码里加调试打印验证。

> **下一步预告**：后续章节会从 `ReqTransSliceToLowLevel()` 切入，讲解请求对象池、日志宏、以及 NAND 调度。现在请先把第 1 章的三条主线（入口 -> 上下文 -> 状态机）彻底消化。

---

## 第 2 章：日志、断言与寄存器封装

### 2.1 学习目标（Why）
- 搞懂项目里“为什么连 `printf` 都要包一层宏”。
- 了解寄存器位域结构体的封装方式，读硬件状态时不再害怕。
- 看到回调里层层 `IO_READ32` 时能迅速分辨它们各自的职责。

### 2.2 固件里的调试出口长什么样？
固件并没有复杂的日志库，而是依靠 `xil_printf` 和一个自定义断言宏：

```c
#define __ASSERT 1
#if __ASSERT
#define ASSERT(X)                  \
if (!(X)) {                        \
        xil_printf("\r\n\nerror in %s: Line %d\r\n", __FILE__, __LINE__);       \
        while(1) ;                  \
}
#endif
```
【F:source/software/GreedyFTL-3.0.0/nvme/debug.h†L52-L63】

> **Why**：嵌入式环境常常没有操作系统，`ASSERT` 触发后直接停在 `while(1)`，方便你连接 JTAG 定位。
>
> **How**：需要临时调试时，把复杂条件包在 `ASSERT` 里，一旦失败你能立即看到文件名和行号。

在状态机中也保留了直接的文字提示，例如在 `nvme_main()` 中打印初始化阶段的提示：

```c
xil_printf("!!! Wait until FTL reset complete !!! \r\n");
...
xil_printf("\r\nNVMe ready!!!\r\n");
```
【F:source/software/GreedyFTL-3.0.0/nvme/nvme_main.c†L77-L118】

> **Why**：这些提示文字构成了最简洁的“串口日志”，不用额外调试工具也能确认执行阶段。

**日志调用链的分层示意：**

```c
// 第 1 层：通用输出（底层）
void log_raw(const char *fmt, ...) {
    xil_printf(fmt, ...);            // 仅负责把字节写到串口
}

// 第 2 层：模块包装
void nvme_log(const char *tag, const char *fmt, ...) {
    log_raw("[%s] ", tag);          // 自动附加模块名
    log_raw(fmt, ...);
}

// 第 3 层：具体场景调用
nvme_log("ADMIN", "SetFeature VWC = %d\r\n", g_nvmeTask.cacheEn);
```

> **Why** 企业喜欢分层？以后换日志后端（例如输出到 ring buffer 或网络）时，只需替换 `log_raw()`；上层模块依然调用 `nvme_log()`，耦合度低。

> **练习**：尝试在 `nvme_log()` 内部加入 `ASSERT(tag != NULL)`，体会宏断言与日志的协同作用。

### 2.3 读懂寄存器位域 + 回调链
在第 1 章里我们提到回调 `dev_irq_handler()`。要真正理解它做了什么，需要配合它操作的位域结构体：

```c
typedef struct _DEV_IRQ_REG {
        union {
                unsigned int dword;
                struct {
                        unsigned int pcieLink :1;
                        unsigned int busMaster :1;
                        unsigned int pcieIrq :1;
                        unsigned int pcieMsi :1;
                        unsigned int pcieMsix :1;
                        unsigned int nvmeCcEn :1;
                        unsigned int nvmeCcShn :1;
                        unsigned int mAxiWriteErr :1;
                        unsigned int mAxiReadErr :1;
                        unsigned int pcieMreqErr :1;
                        unsigned int pcieCpldErr :1;
                        unsigned int pcieCpldLenErr :1;
                        unsigned int reserved0 :20;
                };
        };
} DEV_IRQ_REG;
```
【F:source/software/GreedyFTL-3.0.0/nvme/host_lld.h†L94-L113】

这些位域让 `dev_irq_handler()` 能够一句一句地输出硬件状态并更新 `g_nvmeTask`：

```c
if(devReg.nvmeCcEn == 1) {
        NVME_STATUS_REG nvmeReg;
        nvmeReg.dword = IO_READ32(NVME_STATUS_REG_ADDR);
        xil_printf("NVME CC.EN: %d\r\n", nvmeReg.ccEn);
        if(nvmeReg.ccEn == 1)
                g_nvmeTask.status = NVME_TASK_WAIT_CC_EN;
        else
                g_nvmeTask.status = NVME_TASK_RESET;
}
```
【F:source/software/GreedyFTL-3.0.0/nvme/host_lld.c†L70-L146】

> **How**：阅读回调时，把 `DEV_IRQ_REG`、`NVME_STATUS_REG` 等结构体打印出来，确认每一位的含义，再结合 `xil_printf` 输出就能追踪状态变化。

**位域解析备忘：**

```c
typedef union {
    uint32_t dword;
    struct {
        uint32_t nvmeCcEn : 1;      // 控制器配置寄存器 EN 位改变
        uint32_t nvmeCcShn : 1;     // 进入关机流程
        uint32_t mAxiReadErr : 1;   // AXI 读错误
        // ... 其它位暂时忽略，遇到问题再查
    } bits;
} DEV_IRQ_REG_EX;

DEV_IRQ_REG_EX dev;
dev.dword = IO_READ32(DEV_IRQ_STATUS_REG);
if (dev.bits.nvmeCcEn) {
    // 继续读取 NVMe 状态寄存器
}
```

> **Why** 要自建 `DEV_IRQ_REG_EX`？头文件里的位域名字很长，把常用的挑出来重命名为 `bits.xxx`，写调试代码时更加直观。

> **How** 应用？把 `mAxiReadErr` 等字段写入你自己的“事件速查表”，出现异常时立即知道要查 DRAM 总线还是主机端。

### 2.4 小练习
1. 试着在 `dev_irq_handler()` 里为 `mAxiReadErr` 增加一条打印，例如 `xil_printf("AXI read error\r\n");`，体会如何扩展日志。
2. 把 `ASSERT` 包在一个小函数里，例如检查指针是否为空，练习如何在错误时保持系统驻留。
3. 用你熟悉的 IDE 设置断点，观察 `dev_irq_handler()` 被触发时 `devReg.dword` 的具体值。

---

## 第 3 章：请求对象池与多队列模式

### 3.1 学习目标（Why）
- 理解为什么需要请求池（Request Pool）和多个队列来管理 NVMe、缓存、NAND 之间的工作。
- 练习追踪“取号 -> 填结构体 -> 入队列”这条最常见的固件写法。
- 看懂结构体里密密麻麻的位域和联合体，知道它们各自负责什么。

### 3.2 根结构：请求池 + 队列
固件先定义了一个全局请求池，以及不同用途的队列：

```c
typedef struct _REQ_POOL {
        SSD_REQ_FORMAT reqPool[AVAILABLE_OUNTSTANDING_REQ_COUNT];
} REQ_POOL, *P_REQ_POOL;
...
extern P_REQ_POOL reqPoolPtr;
extern FREE_REQUEST_QUEUE freeReqQ;
extern SLICE_REQUEST_QUEUE sliceReqQ;
extern NVME_DMA_REQUEST_QUEUE nvmeDmaReqQ;
extern NAND_REQUEST_QUEUE nandReqQ[USER_CHANNELS][USER_WAYS];
```
【F:source/software/GreedyFTL-3.0.0/request_allocation.h†L53-L90】

每个队列的节点都是位域形式，头尾指针占 16 bit，方便通过 `unsigned int` 同时存储多个字段：

```c
typedef struct _FREE_REQUEST_QUEUE {
        unsigned int headReq : 16;
        unsigned int tailReq : 16;
        unsigned int reqCnt : 16;
        unsigned int reserved0 : 16;
} FREE_REQUEST_QUEUE;
```
【F:source/software/GreedyFTL-3.0.0/request_queue.h†L49-L95】

真正的请求格式 `SSD_REQ_FORMAT` 则把“请求类型 + 队列类型 + 选项 + 缓冲区 + NAND 地址”都集中在一个结构里：

```c
typedef struct _SSD_REQ_FORMAT {
        unsigned int reqType : 4;
        unsigned int reqQueueType : 4;
        unsigned int reqCode : 8;
        unsigned int nvmeCmdSlotTag : 16;
        unsigned int logicalSliceAddr;
        REQ_OPTION reqOpt;
        DATA_BUF_INFO dataBufInfo;
        NVME_DMA_INFO nvmeDmaInfo;
        NAND_INFO nandInfo;
        unsigned int prevReq : 16;
        unsigned int nextReq : 16;
        unsigned int prevBlockingReq : 16;
        unsigned int nextBlockingReq : 16;
} SSD_REQ_FORMAT;
```
【F:source/software/GreedyFTL-3.0.0/request_format.h†L63-L172】

> **How**：先把 `REQ_POOL` 画成一个大数组，索引就是“请求号”；再把 `SSD_REQ_FORMAT` 展开成几块：上半部分描述命令，下半部分的 `prev/next` 字段用来在不同队列里串联。

### 3.3 NVMe 命令到 Slice 请求
函数 `ReqTransNvmeToSlice()` 把主机的读写命令拆分成若干个 Slice 请求：

```c
reqSlotTag = GetFromFreeReqQ();
reqPoolPtr->reqPool[reqSlotTag].reqType = REQ_TYPE_SLICE;
reqPoolPtr->reqPool[reqSlotTag].reqCode = reqCode;
reqPoolPtr->reqPool[reqSlotTag].nvmeCmdSlotTag = cmdSlotTag;
reqPoolPtr->reqPool[reqSlotTag].logicalSliceAddr = tempLsa;
reqPoolPtr->reqPool[reqSlotTag].nvmeDmaInfo.startIndex = nvmeDmaStartIndex;
reqPoolPtr->reqPool[reqSlotTag].nvmeDmaInfo.numOfNvmeBlock = tempNumOfNvmeBlock;
PutToSliceReqQ(reqSlotTag);
```
【F:source/software/GreedyFTL-3.0.0/request_transform.c†L78-L158】

> **Why**：NVMe 命令的 LBA 可能跨越多个 Slice，拆分后才能让后续的地址转换、数据搬运保持单位一致。
>
> **How**：每拆出一个 Slice，就“取号 -> 填结构体 -> 入队列”，并更新 `nvmeDmaStartIndex` 记录在一个命令中的偏移。

**手写注释版伪代码：**

```c
uint16_t alloc_slice_req(uint16_t cmdSlotTag,
                         uint32_t sliceAddr,
                         uint32_t blocks,
                         uint8_t isWrite) {
    uint16_t slot = GetFromFreeReqQ();            // ① 从“号码布”队列拿一个空请求
    SSD_REQ_FORMAT *req = &reqPoolPtr->reqPool[slot];
    req->reqType = REQ_TYPE_SLICE;                // ② 声明这是 Slice 请求
    req->reqCode = isWrite ? REQ_CODE_WRITE : REQ_CODE_READ;
    req->nvmeCmdSlotTag = cmdSlotTag;             // ③ 记住来自哪个 NVMe 命令
    req->logicalSliceAddr = sliceAddr;            // ④ 指向逻辑 Slice
    req->nvmeDmaInfo.startIndex = dmaOffset;      // ⑤ 记录 DMA 缓冲区偏移
    req->nvmeDmaInfo.numOfNvmeBlock = blocks;     // ⑥ 这次 Slice 覆盖多少 NVMe Block
    PutToSliceReqQ(slot);                         // ⑦ 入队，等调度器处理
    return slot;
}
```

> **Why** 写得这么细？企业项目常常按“号码布 + 填表 + 入队”套路操作。抄一遍伪代码可以帮助你把流程固化到肌肉记忆中。

> **How** 对照源码？查 `REQ_CODE_WRITE` 在哪里定义（`request_format.h`），确认 `reqCode` 与后续调度逻辑的关联。

### 3.4 缓存写回与依赖管理
当需要驱逐数据缓存时，代码会再申请新的请求，把缓存行对应的虚拟地址写回 NAND：

```c
if(dataBufMapPtr->dataBuf[dataBufEntry].dirty == DATA_BUF_DIRTY) {
        reqSlotTag = GetFromFreeReqQ();
        reqPoolPtr->reqPool[reqSlotTag].reqType = REQ_TYPE_NAND;
        reqPoolPtr->reqPool[reqSlotTag].reqCode = REQ_CODE_WRITE;
        reqPoolPtr->reqPool[reqSlotTag].reqOpt.dataBufFormat = REQ_OPT_DATA_BUF_ENTRY;
        reqPoolPtr->reqPool[reqSlotTag].reqOpt.rowAddrDependencyCheck = REQ_OPT_ROW_ADDR_DEPENDENCY_CHECK;
        SelectLowLevelReqQ(reqSlotTag);
        dataBufMapPtr->dataBuf[dataBufEntry].dirty = DATA_BUF_CLEAN;
}
```
【F:source/software/GreedyFTL-3.0.0/request_transform.c†L162-L189】

> **Why**：通过 `SelectLowLevelReqQ()` 再次封装，保证所有低层 NAND 操作都统一排队，避免通道冲突。

### 3.5 小练习
1. 自己仿写一个函数 `DumpReq(reqSlotTag)`，打印 `reqPoolPtr->reqPool[slot]` 的关键字段，练习阅读位域。
2. 手工模拟一次读命令，把 `ReqTransNvmeToSlice()` 的 `while` 循环展开，写出每次 `reqSlotTag` 应该填的字段。
3. 尝试追踪 `SelectLowLevelReqQ()`，看看它把请求放到了哪一个队列里，并记录对应的全局变量。

### 3.6 场景演练：一次 8KB 读命令怎么走
| 步骤 | 涉及函数 | 关键结构体/变量 | 发生的事情 |
| --- | --- | --- | --- |
| 1 | `handle_nvme_io_read()` | `nvmeIOCmd` | 解析主机 OPC、LBA、NLB，调用 `ReqTransNvmeToSlice()` |
| 2 | `ReqTransNvmeToSlice()` | `reqPoolPtr`, `freeReqQ` | 取号、填 `REQ_TYPE_SLICE`、记下命令 Slot、LBA |
| 3 | `PutToSliceReqQ()` | `sliceReqQ` | 请求进 Slice 队列，等待数据缓冲模块确认命中 |
| 4 | `ReqTransSliceToLowLevel()` | `nvmeDmaReqQ`, `nandReqQ` | 把 Slice 请求变成 DMA/NAND 组合，挂上 `reqOpt` |
| 5 | `SchedulingNandReq()` | `nandReqQ`, `dieStateTablePtr` | 根据通道/Way 空闲状态，发起 NAND 读 |

> **练习**：在源码中沿着这 5 步设置断点或加打印，观察 `reqSlotTag` 如何在各个队列之间移动。

> **Why** 要写表格？读完一章后用“流程轴”再回顾一次，有助于把零碎的函数名串成一个连贯故事。

---

## 第 4 章：内存映射与指针绑定

### 4.1 学习目标（Why）
- 理解为什么全局指针都指向神秘的 `0x1xxxxxxx` 地址。
- 学会看静态内存映射表，知道不同模块在 DRAM 里的布局。
- 练习如何把“指针 = 固定地址”这种写法转化为脑中的内存分区图。

### 4.2 Cosmos+ 的 DRAM 分区
`memory_map.h` 指定了固件运行时所有重要结构所在的物理地址：

```c
#define DATA_BUFFER_BASE_ADDR           0x10000000
#define COMPLETE_FLAG_TABLE_ADDR        0x17000000
#define DATA_BUFFER_MAP_ADDR            0x18000000
#define REQ_POOL_ADDR                   (GC_VICTIM_MAP_ADDR + sizeof(GC_VICTIM_MAP))
#define ROW_ADDR_DEPENDENCY_TABLE_ADDR  (REQ_POOL_ADDR + sizeof(REQ_POOL))
#define DIE_STATE_TABLE_ADDR            (ROW_ADDR_DEPENDENCY_TABLE_ADDR + sizeof(ROW_ADDR_DEPENDENCY_TABLE))
```
【F:source/software/GreedyFTL-3.0.0/memory_map.h†L57-L110】

> **Why**：裸机固件没有动态内存分配器，必须手工规划所有数组和表格的地址，避免重叠。

### 4.3 典型用法：指针直接指向物理地址
调度器初始化时，会把这些常量地址写进全局指针：

```c
completeFlagTablePtr = (P_COMPLETE_FLAG_TABLE) COMPLETE_FLAG_TABLE_ADDR;
statusReportTablePtr = (P_STATUS_REPORT_TABLE) STATUS_REPORT_TABLE_ADDR;
dieStateTablePtr = (P_DIE_STATE_TABLE) DIE_STATE_TABLE_ADDR;
wayPriorityTablePtr = (P_WAY_PRIORITY_TABLE) WAY_PRIORITY_TABLE_ADDR;
```
【F:source/software/GreedyFTL-3.0.0/request_schedule.c†L62-L104】

> **How**：阅读这段代码时，请在纸上画出地址区间，把指针名字写在旁边，就能快速记住“某表在 DRAM 哪个角落”。

**内存分区速记表（可打印贴墙）：**

| 模块 | 符号地址 | 作用 | 典型读写位置 |
| --- | --- | --- | --- |
| `COMPLETE_FLAG_TABLE_ADDR` | `0x1700_0000` | 记录 NVMe 命令完成标志 | `request_schedule.c` 中 `SetStatusOfSliceReq()` |
| `REQ_POOL_ADDR` | `0x17xx_xxxx` | 全局请求池 | `request_allocation.c` 中初始化 |
| `DATA_BUFFER_BASE_ADDR` | `0x1000_0000` | 主数据缓冲 | `data_buffer.c` | 
| `LOGICAL_SLICE_MAP_ADDR` | `0x1A00_0000` | 逻辑 -> 虚拟 Slice 映射 | `address_translation.c` |

> **Why** 要整理表格？裸机固件没有 `malloc`，所有内存布局都靠这些常量，记住它们能避免“写错指针 = 覆盖别的表”的致命错误。

> **How** 使用？把表格延伸到更多常量，遇到新地址就补一行。长期维护可用 Excel / Notion 管理。

### 4.4 小练习
1. 查找 `memory_map.h` 中 `TEMPORARY_DATA_BUFFER_BASE_ADDR` 的定义，弄清楚临时缓冲与主缓冲的容量关系。
2. 在 `InitDependencyTable()` 里确认 `rowAddrDependencyTablePtr` 是如何从 `ROW_ADDR_DEPENDENCY_TABLE_ADDR` 得到的。
3. 把常用地址写成自己的速记表，例如：`REQ_POOL -> 0x17xxxxxx`、`ROW_DEP -> 0x18xxxxxx`。

---

## 第 5 章：低层调度状态机

### 5.1 学习目标（Why）
- 认识 FTL 里第二个重要的状态机：NAND 请求调度。
- 搞清楚多个链表（Idle、StatusReport、StatusCheck）如何协同工作。
- 找到与第 1 章主状态机的关联：`ReqTransSliceToLowLevel()` 把请求推入这里的工作流。

### 5.2 初始化阶段：把表格和链表串起来
`InitReqScheduler()` 把所有表格指针对齐后，会初始化每个通道/Way 的链表头：

```c
for(chNo=0; chNo<USER_CHANNELS; ++chNo) {
        wayPriorityTablePtr->wayPriority[chNo].idleHead = 0;
        wayPriorityTablePtr->wayPriority[chNo].idleTail = USER_WAYS - 1;
        wayPriorityTablePtr->wayPriority[chNo].statusReportHead = WAY_NONE;
        ...
        for(wayNo=0; wayNo<USER_WAYS; ++wayNo) {
                dieStateTablePtr->dieState[chNo][wayNo].dieState = DIE_STATE_IDLE;
                dieStateTablePtr->dieState[chNo][wayNo].prevWay = wayNo - 1;
                dieStateTablePtr->dieState[chNo][wayNo].nextWay = wayNo + 1;
                completeFlagTablePtr->completeFlag[chNo][wayNo] = 0;
        }
        dieStateTablePtr->dieState[chNo][0].prevWay = WAY_NONE;
        dieStateTablePtr->dieState[chNo][USER_WAYS-1].nextWay = WAY_NONE;
}
```
【F:source/software/GreedyFTL-3.0.0/request_schedule.c†L62-L105】

**链表状态图（手绘建议）：**

```
Channel chNo
┌───────────────┐
│ idleHead -> Way0 <-> Way1 <-> ... <-> WayN = idleTail
│ statusReportHead -> ... (初始为空)
│ statusCheckHead  -> ... (初始为空)
└───────────────┘

DieState[way]
┌──────────────────────────────┐
│ dieState = DIE_STATE_IDLE     │  // 当前通道空闲
│ prevWay  = wayNo - 1 (或 -1)  │  // 双向链表指针
│ nextWay  = wayNo + 1 (或 -1)  │
│ pendingReq = REQ_SLOT_TAG_NONE│  // 尚未绑定请求
└──────────────────────────────┘
```

> **Why** 要画图？调度器里充满 `prevWay/nextWay` 指针，直接看代码容易迷路，画出链表结构后就能理解“Idle 列表初始串联所有 Way”。

> **How** 实操？在 IDE 中设置 `wayPriorityTablePtr->wayPriority[0].idleHead` 的内存监视，随着请求调度变化观察指针如何移动。

> **Why**：这一步相当于“建链表”，后面调度循环才能把请求在不同列表之间搬移。

### 5.3 调度循环：逐通道、逐队列扫描
`SchedulingNandReq()` 会遍历所有通道，再调用 `SchedulingNandReqPerCh()` 逐个检查队列：

```c
for(chNo = 0; chNo < USER_CHANNELS; chNo++)
        SchedulingNandReqPerCh(chNo);
```
【F:source/software/GreedyFTL-3.0.0/request_schedule.c†L136-L142】

在单通道函数里，逻辑大致分三步：
1. **Idle 列表**：把空闲的 Way 挪到更高优先级列表。若某个 Way 在 `nandReqQ` 里有待执行请求，就调用 `PutToNandWayPriorityTable()` 放入工作队列。
2. **StatusReport 列表**：读取硬件 Ready/Busy 信号，执行 `ExecuteNandReq()` 并在完成后释放依赖或移入 Idle 列表。
3. **StatusCheck 列表**：如果控制器空闲且需要额外的状态轮询，就转移到 `statusCheckHead` 链表继续处理。

对应代码片段：

```c
if(wayPriorityTablePtr->wayPriority[chNo].idleHead != WAY_NONE) {
        wayNo = wayPriorityTablePtr->wayPriority[chNo].idleHead;
        while(wayNo != WAY_NONE) {
                if(nandReqQ[chNo][wayNo].headReq == REQ_SLOT_TAG_NONE)
                        ReleaseBlockedByRowAddrDepReq(chNo, wayNo);
                if(nandReqQ[chNo][wayNo].headReq != REQ_SLOT_TAG_NONE) {
                        nextWay = dieStateTablePtr->dieState[chNo][wayNo].nextWay;
                        SelectivGetFromNandIdleList(chNo, wayNo);
                        PutToNandWayPriorityTable(nandReqQ[chNo][wayNo].headReq, chNo, wayNo);
                        wayNo = nextWay;
                } else {
                        wayNo = dieStateTablePtr->dieState[chNo][wayNo].nextWay;
                        waitWayCnt++;
                }
        }
}
if(wayPriorityTablePtr->wayPriority[chNo].statusReportHead != WAY_NONE) {
        readyBusy = V2FReadyBusyAsync(chCtlReg[chNo]);
        ...
        ExecuteNandReq(chNo, wayNo, reqStatus);
        ...
        PutToNandIdleList(chNo, wayNo);
}
```
【F:source/software/GreedyFTL-3.0.0/request_schedule.c†L144-L224】

> **How**：阅读此类调度代码时，把每个 `if` 当作一个“队列处理阶段”，在旁边标注“输入队列 -> 输出队列”。这样便能明白为什么函数会这么长。

### 5.4 与主状态机的联动
当 `nvme_main()` 发现有 IO 命令时，会调用 `ReqTransSliceToLowLevel()`，它最终把请求推入 `nvmeDmaReqQ`、`nandReqQ` 等队列。`SchedulingNandReqPerCh()` 正是处理这些队列的“消费者”。换句话说，**第 1 章的主状态机负责“接单”，第 5 章的调度器负责“派单 + 跟踪完成情况”**。

### 5.5 小练习
1. 在 `SchedulingNandReqPerCh()` 里添加注释，标出每个分支对应的“队列阶段”，帮助自己梳理流程。
2. 尝试追踪一次写请求：从 `ReqTransNvmeToSlice()` 生成 Slice -> `SelectLowLevelReqQ()` -> `SchedulingNandReqPerCh()` -> `ExecuteNandReq()`。
3. 观察 `SyncAllLowLevelReqDone()` 的 `while` 条件，想想它如何阻塞上层流程，确保所有底层请求完成。

---

> **下一步预告**：后续章节将继续深入 `data_buffer.c`、`address_translation.c`，讲解缓存替换策略与地址映射。请先确保你已经熟悉“请求池 -> 队列 -> 调度器”这一整条链路。

---

## 第 6 章：数据缓冲区（Data Buffer）与 LRU 模式

### 6.1 学习目标（Why）
- 学会从 `InitDataBuf()` 出发，理解“哈希 + 双向链表 + LRU 淘汰”的组合拳。 
- 观察缓存命中/未命中的控制流，弄清楚请求结构体如何在这里挂上“阻塞队列”。
- 形成一套“先画地址 -> 再写事件序列 -> 最后跟踪指针”分析缓存代码的方法。

### 6.2 初始化：指针落地 + 链表串好
数据缓冲模块一开始会把 DRAM 中的映射表地址写入全局指针，并初始化双向链表：

```c
dataBufMapPtr = (P_DATA_BUF_MAP) DATA_BUFFER_MAP_ADDR;
dataBufHashTablePtr = (P_DATA_BUF_HASH_TABLE)DATA_BUFFFER_HASH_TABLE_ADDR;
tempDataBufMapPtr = (P_TEMPORARY_DATA_BUF_MAP)TEMPORARY_DATA_BUFFER_MAP_ADDR;

for(bufEntry = 0; bufEntry < AVAILABLE_DATA_BUFFER_ENTRY_COUNT; bufEntry++) {
        dataBufMapPtr->dataBuf[bufEntry].logicalSliceAddr = LSA_NONE;
        dataBufMapPtr->dataBuf[bufEntry].prevEntry = bufEntry-1;
        dataBufMapPtr->dataBuf[bufEntry].nextEntry = bufEntry+1;
        dataBufMapPtr->dataBuf[bufEntry].dirty = DATA_BUF_CLEAN;
        dataBufMapPtr->dataBuf[bufEntry].blockingReqTail =  REQ_SLOT_TAG_NONE;

        dataBufHashTablePtr->dataBufHash[bufEntry].headEntry = DATA_BUF_NONE;
        dataBufHashTablePtr->dataBufHash[bufEntry].tailEntry = DATA_BUF_NONE;
        dataBufMapPtr->dataBuf[bufEntry].hashPrevEntry = DATA_BUF_NONE;
        dataBufMapPtr->dataBuf[bufEntry].hashNextEntry = DATA_BUF_NONE;
}

dataBufLruList.headEntry = 0;
dataBufLruList.tailEntry = AVAILABLE_DATA_BUFFER_ENTRY_COUNT - 1;
```
【F:source/software/GreedyFTL-3.0.0/data_buffer.c†L52-L133】

> **Why**：固件在 DRAM 中为缓存和哈希表预留了固定区域（第 4 章提到的内存映射）。初始化阶段要把这些“裸地址”变成结构体指针，才方便后续模块直接使用。

> **How**：
> 1. 画一个数组表示 `dataBufMapPtr->dataBuf`，标出 `prevEntry/nextEntry` 链出成一条双向链表。
> 2. 画一个小表格表示 `dataBufHashTablePtr`，每个哈希槽指向链表节点。这样“链表 + 哈希”就同时可视化了。
> 3. 把 `blockingReqTail` 单独标注，提示“这里会连到请求池”。

### 6.3 命中路径：移动到 LRU 头部
如果 Slice 已经在缓存里，`CheckDataBufHit()` 会把命中的节点移到 LRU 链表头：

```c
if(dataBufMapPtr->dataBuf[bufEntry].logicalSliceAddr == logicalSliceAddr) {
        ... // 先把节点从当前链表位置拆下来
        if(dataBufLruList.headEntry != DATA_BUF_NONE) {
                dataBufMapPtr->dataBuf[bufEntry].prevEntry = DATA_BUF_NONE;
                dataBufMapPtr->dataBuf[bufEntry].nextEntry = dataBufLruList.headEntry;
                dataBufMapPtr->dataBuf[dataBufLruList.headEntry].prevEntry = bufEntry;
                dataBufLruList.headEntry = bufEntry;
        } else {
                dataBufMapPtr->dataBuf[bufEntry].prevEntry = DATA_BUF_NONE;
                dataBufMapPtr->dataBuf[bufEntry].nextEntry = DATA_BUF_NONE;
                dataBufLruList.headEntry = bufEntry;
                dataBufLruList.tailEntry = bufEntry;
        }
        return bufEntry;
}
```
【F:source/software/GreedyFTL-3.0.0/data_buffer.c†L88-L136】

> **Why**：频繁访问的条目必须放在链表前面，防止被淘汰。固件通过手动改 `prev/next`，实现了标准 LRU（最近最少使用）策略。

> **How**：在纸上写下“命中 -> 拆节点 -> 插到头 -> 返回索引”，并补充“谁会调用 `CheckDataBufHit()`？”（答案：请求转换和调度模块在读写数据前都会查缓存）。

**命中流程速写：**

```
Slice 请求到来
    ↓
哈希表 -> 命中 dataBufEntry = X
    ↓
从 LRU 链表中拆掉 X        // 修改 prev/next
    ↓
把 X 插到 head              // head = X, 原 head 成为 next
    ↓
返回 dataBufEntry 供 DMA/NAND 使用
```

> **练习**：尝试在 `CheckDataBufHit()` 的 `return` 前加入 `xil_printf("Hit buf=%d\r\n", bufEntry);`，观察高并发场景下命中率。

### 6.4 未命中：淘汰尾巴 + 挂阻塞队列
当缓存不够用时，`AllocateDataBuf()` 从尾部淘汰一项，同时维护挂在数据缓冲条目上的“阻塞请求链表”：

```c
unsigned int evictedEntry = dataBufLruList.tailEntry;
...
SelectiveGetFromDataBufHashList(evictedEntry);
return evictedEntry;
```
【F:source/software/GreedyFTL-3.0.0/data_buffer.c†L144-L172】

```c
if(dataBufMapPtr->dataBuf[bufEntry].blockingReqTail != REQ_SLOT_TAG_NONE) {
        reqPoolPtr->reqPool[reqSlotTag].prevBlockingReq = dataBufMapPtr->dataBuf[bufEntry].blockingReqTail;
        reqPoolPtr->reqPool[reqPoolPtr->reqPool[reqSlotTag].prevBlockingReq].nextBlockingReq = reqSlotTag;
}
dataBufMapPtr->dataBuf[bufEntry].blockingReqTail = reqSlotTag;
```
【F:source/software/GreedyFTL-3.0.0/data_buffer.c†L176-L185】

> **Why**：如果多个请求依赖同一份缓冲区，就把它们串成链表，等数据写回或装载完成再一起唤醒。这样可以避免重复申请 NAND 访问。

> **How**：
> 1. 写出“事件序列”：缓存未命中 -> `AllocateDataBuf()` 淘汰 -> 新请求加入 `blockingReq`。
> 2. 用 `reqSlotTag` 作为关键字，在请求池里查 `prevBlockingReq/nextBlockingReq`，确认链路确实建立成功。

**阻塞链路可视化：**

```
dataBufEntry Y
┌────────────────────────────────────────────┐
│ blockingReqTail ──────┐                    │
└────────────────────────┴───────────────┐    │
                                         │    │
reqPool[17] <──> reqPool[23] <──> reqPool[40]
   ↑                ↑                ↑
   │prevBlockingReq │                │nextBlockingReq

唤醒条件：
1. 数据装载/写回完成 -> `UnblockDataBufBlockedReq(reqSlotTag)`
2. 链头重新放入 `sliceReqQ` -> 调度继续
```

> **练习**：写一个调试函数 `DumpBlockingChain(bufEntry)`，遍历 `prevBlockingReq/nextBlockingReq`，确保链条没有断裂。

### 6.5 常见误区与练习
1. **不要忘记哈希链表**：淘汰节点后要调用 `SelectiveGetFromDataBufHashList()`，否则哈希槽里还会指向旧节点，导致脏数据。【F:source/software/GreedyFTL-3.0.0/data_buffer.c†L144-L172】
2. **练习**：模拟“缓存命中 + 写回”流程，手工列出 `blockingReqTail` 从 `REQ_SLOT_TAG_NONE` 变到实际 tag 的过程。
3. **调试技巧**：在 `AllocateDataBuf()` 里加上 `xil_printf` 打印 `evictedEntry`，观察高并发时 LRU 的行为。

---

## 第 7 章：地址映射与坏块管理

### 7.1 学习目标（Why）
- 看懂 `InitAddressMap()` 如何把一堆表格指向固定内存，从而维护“逻辑 Slice -> 物理 NAND”映射。
- 理解坏块重映射 `RemapBadBlock()` 的流程，知道它如何使用备用块。
- 练习“按模块拆函数”：把初始化、映射更新、打印日志分别整理成小段笔记。

### 7.2 初始化：绑定映射表指针

```c
logicalSliceMapPtr = (P_LOGICAL_SLICE_MAP ) LOGICAL_SLICE_MAP_ADDR;
virtualSliceMapPtr = (P_VIRTUAL_SLICE_MAP) VIRTUAL_SLICE_MAP_ADDR;
virtualBlockMapPtr = (P_VIRTUAL_BLOCK_MAP) VIRTUAL_BLOCK_MAP_ADDR;
virtualDieMapPtr = (P_VIRTUAL_DIE_MAP) VIRTUAL_DIE_MAP_ADDR;
phyBlockMapPtr = (P_PHY_BLOCK_MAP) PHY_BLOCK_MAP_ADDR;
bbtInfoMapPtr = (P_BAD_BLOCK_TABLE_INFO_MAP) BAD_BLOCK_TABLE_INFO_MAP_ADDR;
...
InitSliceMap();
InitBlockDieMap();
```
【F:source/software/GreedyFTL-3.0.0/address_translation.c†L52-L88】

> **Why**：和第 6 章一样，地址映射模块需要把各类表格“定位”到 DRAM。`InitSliceMap()` 会把逻辑、虚拟 Slice 初始化为未使用状态。

> **How**：列出六个指针分别指向的表格，并在笔记里写下它们存储的信息（例如：`logicalSliceMapPtr` 保存 LSA->VSA，`phyBlockMapPtr` 保存坏块信息）。

### 7.3 坏块重映射：备用块搜索策略

```c
for(blockNo=0 ; blockNo<USER_BLOCKS_PER_LUN ; blockNo++) {
        for(dieNo=0 ; dieNo<USER_DIES ; dieNo++) {
                if(phyBlockMapPtr->phyBlock[dieNo][blockNo].bad) {
                        if(reservedBlockOfLun0[dieNo] < TOTAL_BLOCKS_PER_LUN) {
                                remapFlag = 1;
                                while(phyBlockMapPtr->phyBlock[dieNo][reservedBlockOfLun0[dieNo]].bad) {
                                        reservedBlockOfLun0[dieNo]++;
                                        if(reservedBlockOfLun0[dieNo] >= TOTAL_BLOCKS_PER_LUN) {
                                                remapFlag = 0;
                                                break;
                                        }
                                }
                                if(remapFlag) {
                                        phyBlockMapPtr->phyBlock[dieNo][blockNo].remappedPhyBlock = reservedBlockOfLun0[dieNo];
                                        reservedBlockOfLun0[dieNo]++;
                                } else {
                                        xil_printf("No reserved block - Ch %d Way %d virtualBlock %d is bad block\r\n", ...);
                                        badBlockCount[dieNo]++;
                                }
                        } else {
                                xil_printf("No reserved block - Ch %d Way %d virtualBlock %d is bad block\r\n", ...);
                                badBlockCount[dieNo]++;
                        }
                }
                ... // Lun1 同理
        }
}
```
【F:source/software/GreedyFTL-3.0.0/address_translation.c†L100-L188】

> **Why**：闪存有概率出现坏块，FTL 必须把这些块映射到备用区域。`reservedBlockOfLun0`/`reservedBlockOfLun1` 维护“下一个备用块”的索引。

> **How**：
> 1. 把循环拆成“检查坏块 -> 找备用块 -> 更新映射/记录错误”三步。
> 2. 在笔记中标注 `remappedPhyBlock` 表示“坏块重定向到哪一个物理块”。
> 3. 关注 `xil_printf` 输出，帮助定位缺少备用块的情况。

**坏块重映射速写：**

```
phyBlock[die][block]
┌ bad = 1
│ remappedPhyBlock = ?
└───────────────────────
          │
          │查 reservedBlockOfLun0[die]
          ▼
找到 first_good_reserved_block -> remappedPhyBlock = idx
          │
          ├─ 找不到 -> 打日志 + 统计 badBlockCount
          └─ 找到   -> reservedBlockOfLun0[die]++（下次用下一个）
```

> **练习**：用伪数据写一段 C 小程序模拟 `reservedBlockOfLun0` 的递增逻辑，确认循环终止条件（例如 `>= TOTAL_BLOCKS_PER_LUN`）的作用。

### 7.4 练习与调试建议
1. **练习**：写出 `FindDieForFreeSliceAllocation()` 会影响 `sliceAllocationTargetDie` 的原因，理解“轮转分配”策略。
2. **调试**：在仿真环境里手动标记几个坏块（改 `phyBlock[].bad = 1`），观察日志是否如预期输出。
3. **延伸阅读**：查阅 NVMe 规范中对坏块管理的建议，将固件策略与协议要求比对。

---

## 第 8 章：回调注册、函数指针与 API 封装套路

### 8.1 学习目标（Why）
- 面对“先注册，再调用，再回调”的流程能快速定位入口和出口。
- 了解固件如何通过函数指针把不同命令映射到专用处理函数。
- 掌握“把复杂 API 拆成三段”的阅读技巧：**注册/绑定 -> 参数解析 -> 真正操作**。

### 8.2 中断回调回顾
第 1 章提到 `XScuGic_Connect()` 把中断号和 `dev_irq_handler` 绑定。这里补充 `dev_irq_handler` 如何读取寄存器并更新上下文：

```c
DEV_IRQ_REG devReg;
devReg.dword = IO_READ32(DEV_IRQ_STATUS_REG);
if(devReg.nvmeCcEn == 1) {
        NVME_STATUS_REG nvmeReg;
        nvmeReg.dword = IO_READ32(NVME_STATUS_REG_ADDR);
        xil_printf("NVME CC.EN: %d\r\n", nvmeReg.ccEn);
        if(nvmeReg.ccEn == 1)
                g_nvmeTask.status = NVME_TASK_WAIT_CC_EN;
        else
                g_nvmeTask.status = NVME_TASK_RESET;
}
```
【F:source/software/GreedyFTL-3.0.0/nvme/host_lld.c†L70-L145】

> **模式拆分**：
> - **注册**：`XScuGic_Connect()` 负责把函数指针告诉中断控制器。
> - **解析**：回调内部用位域结构体（`DEV_IRQ_REG`、`NVME_STATUS_REG`）翻译寄存器含义。
> - **操作**：改写 `g_nvmeTask.status`，触发主状态机响应。

### 8.3 命令处理函数指针
NVMe IO 命令处理函数 `handle_nvme_io_cmd()` 根据 `OPC`（操作码）派发到不同函数：

```c
switch(opc) {
case IO_NVM_FLUSH:
        xil_printf("IO Flush Command\r\n");
        set_auto_nvme_cpl(nvmeCmd->cmdSlotTag, nvmeCPL.specific, nvmeCPL.statusFieldWord);
        break;
case IO_NVM_WRITE:
        handle_nvme_io_write(nvmeCmd->cmdSlotTag, nvmeIOCmd);
        break;
case IO_NVM_READ:
        handle_nvme_io_read(nvmeCmd->cmdSlotTag, nvmeIOCmd);
        break;
default:
        xil_printf("Not Support IO Command OPC: %X\r\n", opc);
        ASSERT(0);
        break;
}
```
【F:source/software/GreedyFTL-3.0.0/nvme/nvme_io_cmd.c†L116-L154】

`handle_nvme_io_write()` 和 `handle_nvme_io_read()` 则负责解析命令字、断言参数、把请求交给 FTL：

```c
writeInfo12.dword = nvmeIOCmd->dword[12];
startLba[0] = nvmeIOCmd->dword[10];
nlb = writeInfo12.NLB;
ASSERT((nvmeIOCmd->PRP1[0] & 0xF) == 0 && (nvmeIOCmd->PRP2[0] & 0xF) == 0);
ReqTransNvmeToSlice(cmdSlotTag, startLba[0], nlb, IO_NVM_WRITE);
```
【F:source/software/GreedyFTL-3.0.0/nvme/nvme_io_cmd.c†L89-L114】

> **Why**：`switch` 派发是最简单的函数指针模式，你可以把每个分支看成“注册表”的一项。大型项目为了扩展性会用数组或结构体保存函数指针，Cosmos+ 这里采用 `switch`，更容易理解。

> **How**：
> 1. 在笔记里列出 “OPC -> 处理函数” 映射，方便快速查找。
> 2. 关注断言：`ASSERT` 是预防参数错误的“防护网”，看到断言就知道这是关键约束。

**OPC 派发表（可复制扩展）：**

| OPC 常量 | 数值 | 处理函数 | 关键动作 |
| --- | --- | --- | --- |
| `IO_NVM_FLUSH` | `0x00` | `set_auto_nvme_cpl()` | 直接完成命令，通常用于缓存刷写 |
| `IO_NVM_WRITE` | `0x01` | `handle_nvme_io_write()` | 解析 PRP、NLB，拆成 Slice 写请求 |
| `IO_NVM_READ` | `0x02` | `handle_nvme_io_read()` | 同上，但生成读请求 |
| 其它 | - | `ASSERT(0)` | 未实现的命令会触发断言，提醒补功能 |

> **Why** 制作表格？看到 OPC 数值时能迅速查到对应函数，避免在源码里反复搜。

> **How** 延伸？随着你阅读 Admin 命令文件，为 `handle_nvme_admin_cmd()` 也补一张类似表格，形成自己的“命令速查簿”。

### 8.4 Admin 命令：上下文与全局变量联动
Admin 命令处理中，`handle_set_features()` 会根据 FID 更新全局上下文：

```c
case VOLATILE_WRITE_CACHE:
        xil_printf("Set VWC: %X\r\n", nvmeAdminCmd->dword11);
        g_nvmeTask.cacheEn = (nvmeAdminCmd->dword11 & 0x1);
        nvmeCPL->dword[0] = 0x0;
        nvmeCPL->specific = 0x0;
        break;
```
【F:source/software/GreedyFTL-3.0.0/nvme/nvme_admin_cmd.c†L77-L131】

> **Why**：回调/处理函数不仅处理命令，还会修改全局状态（例如缓存开关）。这进一步说明 `g_nvmeTask` 是各模块的“共享真相来源”。

> **How**：追踪 `g_nvmeTask.cacheEn` 的读取位置（比如第 1 章状态机打印），形成“写-读链条”图，帮助理解跨模块数据流。

**Admin 命令小剧场：**

1. 主机发送 `Set Feature (FID=6, VWC)` -> `handle_nvme_admin_cmd()` 被调用。
2. 分支命中 `VOLATILE_WRITE_CACHE` -> `g_nvmeTask.cacheEn = 1`。
3. `nvme_main()` 下一轮循环读取 `g_nvmeTask.cacheEn`，决定是否在状态机日志里提示“Cache Enabled”。
4. 数据缓存模块可根据该标志决定是否允许写命令直接完成。

> **练习**：在 `nvme_main()` 中加入 `if (g_nvmeTask.cacheEn) xil_printf("Cache Enabled\r\n");`，验证 Admin 命令确实驱动全局行为改变。

### 8.5 快速理解“注册 + 回调”的套路
1. **画三层图**：硬件/驱动层（中断控制器、寄存器） -> NVMe 框架层（命令解析、全局上下文） -> FTL/调度层（请求池、队列）。
2. **找到 `extern` 声明**：`extern NVME_CONTEXT g_nvmeTask;` 等语句提醒你“这个结构体在别处定义，需要跨文件共享”。【F:source/software/GreedyFTL-3.0.0/nvme/nvme_admin_cmd.c†L48-L199】
3. **练习**：在 `nvme_main.c` 中搜索 `handle_nvme_*`，写出“接收命令 -> 调用处理函数 -> 更新状态/队列”的顺序。

---

## 第 9 章：综合学习路线与速查表

### 9.1 推荐学习路径
1. **入口 & 状态机**（第 1 章）→ **日志 & 寄存器**（第 2 章）：先弄清主循环和调试手段。
2. **请求池 & 队列**（第 3 章）→ **内存映射**（第 4 章）：理解数据如何在 DRAM 中排布。
3. **调度器**（第 5 章）→ **数据缓冲**（第 6 章）：串起“请求 -> 缓存 -> NAND”链路。
4. **地址映射**（第 7 章）→ **回调/注册**（第 8 章）：掌握全局状态如何被命令驱动更新。

### 9.2 常用编码模式速查
- **全局上下文 + 多层结构体**：`NVME_CONTEXT`/`SSD_REQ_FORMAT` 把相关状态集中在一起，便于跨模块共享。【F:source/software/GreedyFTL-3.0.0/nvme/nvme.h†L788-L823】【F:source/software/GreedyFTL-3.0.0/request_format.h†L63-L172】
- **固定内存映射 + 指针绑定**：所有大数组都对应 `memory_map.h` 中的常量地址，初始化阶段统一绑定。【F:source/software/GreedyFTL-3.0.0/memory_map.h†L57-L110】【F:source/software/GreedyFTL-3.0.0/request_schedule.c†L62-L104】
- **取号 -> 填结构体 -> 入队列**：`ReqTransNvmeToSlice()`、`InitNandArray()` 都遵循这一模式，确保请求生命周期可追踪。【F:source/software/GreedyFTL-3.0.0/request_transform.c†L78-L189】【F:source/software/GreedyFTL-3.0.0/ftl_config.c†L108-L146】
- **回调更新状态 -> 主循环检测**：中断回调写 `g_nvmeTask`，主状态机读，形成事件驱动链。【F:source/software/GreedyFTL-3.0.0/nvme/host_lld.c†L90-L142】【F:source/software/GreedyFTL-3.0.0/nvme/nvme_main.c†L73-L182】
- **断言 + 串口日志**：`ASSERT` 和 `xil_printf` 组成轻量调试框架，一旦触发立即停机定位。【F:source/software/GreedyFTL-3.0.0/nvme/debug.h†L52-L63】【F:source/software/GreedyFTL-3.0.0/nvme/nvme_main.c†L77-L118】

### 9.3 学习与调试小贴士
- **建立“函数卡片”**：给每个核心函数写一张卡片，包括“输入参数”“修改的全局变量”“调用者”“被调用者”，几天后回顾记忆会更扎实。
- **多画图**：状态机图、内存分区图、链表结构图，能极大降低抽象程度。
- **善用搜索**：使用 `rg` 搜索宏、结构体名字，快速跳到相关代码。
- **刻意练习**：每次阅读后都写一个“手动执行”例子，模拟从主机发出读写命令到 NAND 完成的过程。
- **逐层剥洋葱**：遇到“封装好多层”时，按照“结构体 -> 初始化 -> 使用点 -> 调试输出”顺序追踪，就能把层层封装拆解开。

> **总结**：企业级固件看似复杂，其实都围绕几种模式：全局上下文共享、回调驱动状态机、请求池搭配队列、缓存 + 映射表维持性能。这份笔记希望成为你入门的“路线图”，建议反复对照源码和练习题实践，加速从“看不懂”到“敢改动”的过程。

---

## 第 10 章：企业级 SSD 固件常用模式总览（独立手册）

本章内容已经拆分为单独文档《[第 10 章：企业级模式速查手册](./beginner_notes_ch10.md)》，专门为初学者补充更详尽的模式拆解、示例代码和练习题。

> **阅读指引**：在通读 0~9 章后，打开新文档即可按照“Why → How → 代码路径”的顺序学习常见封装套路。遇到新模块时，也可以把该文档当作速查手册，快速定位相似的企业级写法。
