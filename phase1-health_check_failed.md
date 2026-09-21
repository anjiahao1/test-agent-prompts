## [Phase 1/5] 根因分析 — Health Check Failed / 重启健康检查异常

设备在重启/健康检查阶段诊断出异常。输入 JSON 的 `diagnose_report` 字段携带了各项诊断子报告，是本次分析的**首要线索**，必须先解析它再决定后续调查方向。

### 0. 硬规则：diagnose check 失败禁止降级（先于一切分析动作）

健康检查的每一项诊断子报告（`memleak` / `memcheck` / `net_route` / `iob_check` 等）是对设备健康状态的**检测手段**。某子报告 `result == "fail"/"failed"` 表示检测到了**真实的异常信号**，分析目标是**找出该信号背后的根因并修复**，而不是让信号消失。

- ❌ **禁止**把诊断子报告的 `result` 从 `fail`/`failed` 直接改为 `warn`/`info`/`pass`，并把该改动作为修复 patch 提交——这是绕过检测，不是修复（判定方只认 `fail`，降级即"灭火"）。
- ❌ **禁止**通过修改诊断工具源码（如 nxgdb 的 memleak/memcheck 实现）中该 check 的 `result` 判定来达成降级，再以"修复诊断工具"名义提交——同型违规。
- ❌ **禁止**在未对失败项做实际分析时，以「扫描器无法追踪 / 这是单例 / 这是活任务持有 / 该 check 误报」等理由跳过；这些**只能是分析结论，不能是分析起点**。dead-PID、单例、活任务都不构成"误报"的充分理由——块不可达正是要分析的信号。
- ✅ 每个失败项**必须**完成成因分析：逐项读取对应子报告的 `data`，定位到源码（分配点 / 越界点 / 持有链），并给出**修复被测代码**的方案或明确的证据不足说明。
- ✅ 产出必须包含逐失败项的分析结论（可审计）；结论缺失不得提交修复。

> **为什么 `memleak`/`memcheck` 的 fail 是真实信号（推翻"误报"假设的依据）**
> 这类检查是穷举可达性判定：从全局对象、堆节点、mempool 块等根集出发，扫描**每一个**按指针宽度对齐的内存槽；只有「无任何可达位置持有该块」才报告。任何运行时可用来引用内存的值只可能存在于：全局数据段（根集，已扫）、堆/mempool（已扫）、任务栈（本身也是堆分配，已被扫描）、寄存器（快照已扫）。因此报告 fail = 算法诚实结论「全系统无可达持有者」。要主张"误报"，必须**在快照中找出那个可达的持有槽**——找不出，块即泄漏（按定义）。扫描器可能**漏报**（根集未覆盖某些内存），但不会**发明**一个实际可达的块。据此，「无法追踪栈指针 / 是单例 / 属活任务」均不构成误报理由。

### 1. 解析 diagnose_report（必做，优先于 GDB）

`diagnose_report` 是一个数组，每个元素形如：

```json
{
  "title": "tlsdump report",
  "summary": "integrity check",
  "result": "failed",        // pass | failed
  "category": "system",       // sched | system | connectivity | ...
  "command": "tlsdump",       // 诊断命令
  "data": "..."               // 命令原始输出（字符串或数组，可能为空）
}
```

分析步骤：
1. 逐条读取，**筛出 `result == "failed"` 的子报告** —— 这些是根因的直接候选。
2. 对每个失败项，结合 `category` / `command` / `data` 判断故障子系统（如 `sched` 死锁、`system` 完整性、`connectivity` socket 泄漏）。
3. `result == "pass"` 的项作为排除项，帮助缩小范围（如死锁报告 pass 可排除调度死锁）。
4. `extra_context.state_reason` 通常给出触发原因（如 `Reboot diagnose found issues`）。

### 2. 结合 GDB / 日志深入定位

若 `diagnose_report` 不足以定位到源码，且提供了 `gdb_port` / `gdb_rpc_host` / `gdb_rpc_port`，这是在线调试：

- 强制使用 **skill: crash-analysis** 和 **skill: gdb-start** 连接设备做进一步分析。
- 非本机时用 `mcp__gdb-mcp__gdb_connect(host=<gdb_rpc_host>, port=<gdb_rpc_port>)`。
- 若 GDB 对端已失效（socket 在但设备已重启），按 base 里的「GDB 失效探测」切换到 `log_path` + `elf_path` 源码 fallback。

按子报告 `command` 路由到对应分析参考（详尽的 MUST/MUST NOT 见各参考，**均禁止以降级 result 作为修复**）：

| `command` | 参考分析协议 |
|---|---|
| `mm leak` / `memleak` | `gdb-start` skill 的 `references/diagnose-mm-leak.md` |
| `memcheck` / `mm check` | `gdb-start` skill 的 `references/diagnose-mm-check.md` |

分析结论必须能对应到 `diagnose_report` 中的某个失败项，并在 `diagnosis.exception_type` 填 `health_check_failed`。

**自动进入 Phase 2。**
