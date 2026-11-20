# 审计命令清单

配合 `SKILL.md` 使用。按顺序跑，别跳。

## 1. 编译期（先清零，再谈其他）

```bash
# 告警全开，警告当错误
gcc -c -std=c11 -Wall -Wextra -Wshadow -Wconversion -Wdouble-promotion \
    -Wformat=2 -Wundef -Wcast-align -Wstrict-prototypes \
    -Werror -O2 -o /dev/null your_file.c

# 每个函数的栈用量（生成 .su 文件，重点看 > 256B 的）
gcc -fstack-usage -c your_file.c
sort -t$'\t' -k2 -n your_file.su | tail -20

# 函数内栈超过 1KB 就报警
gcc -Wstack-usage=1024 -c your_file.c

# 栈保护（能开就开；裸机要配套 handler）
gcc -fstack-protector-strong -c your_file.c
```

交叉编译时把 `gcc` 换成 `arm-none-eabi-gcc`，参数相同。

## 2. 静态分析

```bash
# cppcheck：先看 error/warning，style 可后置
cppcheck --enable=warning,style,performance,portability \
         --inconclusive --std=c11 --template='{file}:{line}: {severity}: {message}' \
         src/ 2>&1 | tee cppcheck.txt

# clang 静态分析器（跨函数路径敏感，找空指针/资源泄漏最有效）
clang --analyze -Xanalyzer -analyzer-output=text src/*.c

# GCC 的 -fanalyzer（gcc >= 12，慢但能查泄漏和 UAF）
gcc -fanalyzer -c src/*.c 2>&1 | tee fanalyzer.txt

# 未定义行为（能上就上，尤其换编译器前）
clang -fsanitize=undefined,address -g -O1 test.c -o t && ./t
```

> `-fsanitize` 只能在主机侧（T0 单测）跑，目标板上跑不了。但它抓到的 UB，在目标板上就是随机死机。

## 3. grep 高危清单

| 目的 | 命令 |
|---|---|
| 动态申请 | `grep -rn "malloc\|calloc\|realloc\|rt_malloc\|pvPortMalloc\|k_malloc" src/` |
| 危险字符串函数 | `grep -rn "strcpy\|strcat\|sprintf\|gets\|alloca" src/` |
| 无长度 memcpy | `grep -rn "memcpy" src/ \| grep -v "sizeof"` |
| ISR 里用非 FromISR | `grep -rn "FromISR" src/ \| wc -l`，再 `grep -rn "xSemaphoreTake(\|xQueueSend(\|xSemaphoreGive(" src/` 对照 |
| 无超时阻塞 | `grep -rn "portMAX_DELAY\|RT_WAITING_FOREVER\|RT_WAITING_NO" src/` |
| 忙等 | `grep -rn "while *( *!\|for *( *;;* *)" src/` |
| 关中断 | `grep -rn "taskENTER_CRITICAL\|rt_hw_interrupt_disable\|__disable_irq\|local_irq_save" src/` |
| 遗留标记 | `grep -rn "TODO\|FIXME\|XXX\|HACK" src/` |
| 断言被跳过 | `grep -rn "assert\|configASSERT" src/` |
| 忽略返回值 | `grep -rn "( *void *)[a-z_]*(" src/` |

## 4. 运行时测量

| 指标 | FreeRTOS | RT-Thread | 裸机 / Linux |
|---|---|---|---|
| 栈高水位 | `uxTaskGetStackHighWaterMark(NULL)`（返回**剩余**字数，越小越危险） | `list_thread` → max used | 填 `0xA5` 后扫描；Linux 看 `/proc/<pid>/status` 的 VmStk |
| 堆剩余 | `xPortGetFreeHeapSize()`、`xPortGetMinimumEverFreeHeapSize()` | `list_mem` | `mallinfo2()` |
| CPU 占用 | 开 `configGENERATE_RUN_TIME_STATS` + `vTaskGetRunTimeStats()` | `cpuusage` | `perf stat` / `/proc/stat` |
| 复位原因 | 读 RCC 的复位标志寄存器 | 同 | `dmesg` / `/sys/class/watchdog` |
| 栈溢出检测 | `configCHECK_FOR_STACK_OVERFLOW=2` + 实现 hook | `RT_USING_OVERFLOW_CHECK` | MPU 保护区 / canary |

**判据提醒**：`uxTaskGetStackHighWaterMark` 返回的是**历史最小剩余量**，不是使用量。配置 512 word、返回 100，说明峰值用了 412 word（80%），已经超 70% 线。

## 5. HardFault 现场分析（Cortex-M）

```c
/* 在 HardFault_Handler 里先抓现场，再决定是否复位 */
typedef struct { uint32_t r0,r1,r2,r3,r12,lr,pc,psr; } fault_frame_t;

void HardFault_Handler(void) {
    __asm volatile (
        "tst lr, #4          \n"
        "ite eq              \n"
        "mrseq r0, msp       \n"
        "mrsne r0, psp       \n"
        "b hardfault_report  \n");
}
/* hardfault_report(fault_frame_t *f) 里打印 f->pc / f->lr 和 SCB 寄存器 */
```

要读的寄存器：

| 寄存器 | 看什么 |
|---|---|
| `SCB->CFSR` | bit[7:0] MMFSR（非法地址/访问违例）、bit[15:8] BFSR（总线错误）、bit[31:16] UFSR（未对齐/除零/非法状态） |
| `SCB->HFSR` | bit30 FORCED：真正的错在 CFSR 里 |
| `SCB->MMFAR` / `BFAR` | 触发错误的地址（配 `CFSR` 的 MMARVALID/BFARVALID 位才有效） |
| 栈帧 `PC` | 出错指令地址 → `arm-none-eabi-addr2line -e elf.axf <pc>` |

常见映射：

- MMFSR 的 DACCVIOL + MMFAR 非零 → 野指针写
- UFSR 的 UNALIGNED → 非对齐访问（`uint32_t*` 指向奇数地址）
- BFSR 的 PRECISERR → 访问了不存在的总线地址（常见于外设时钟没开）
- 复位后读不到原因 → 先补复位原因记录，否则永远在猜

## 6. 平台差异

| 平台 | 审计侧重 |
|---|---|
| 裸机 | 栈/中断嵌套、`volatile`、寄存器地址、无 OS 时的"锁"就是关中断 |
| FreeRTOS | 优先级唯一性、`FromISR` 版本、`configMAX_SYSCALL_INTERRUPT_PRIORITY`、heap 方案选择 |
| RT-Thread | 设备框架与中断上下文约束、`rt_mutex` 的继承、`RT_WAITING_FOREVER` 使用点 |
| Linux 用户态 | fd 泄漏、`malloc` 与 `fork`、线程栈默认 8MB 但 `pthread_attr_setstacksize` 会改、`errno` 检查 |
| Linux 内核/驱动 | 自旋锁与睡眠上下文、`copy_to_user` 校验、引用计数、`kfree` 后置 NULL |

## 7. 交付前最后一遍

```bash
grep -rn "TODO\|FIXME" src/ | wc -l          # 应为 0 或有明确 issue 号
cppcheck --error-exitcode=1 ...               # 应无 error
git grep -n "printf" -- '*.c' | grep -v log   # 确认没把调试输出留在中断里
```

三项都干净，再签"可合入"。
