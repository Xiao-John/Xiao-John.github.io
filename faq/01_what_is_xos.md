---
layout: default
---

> FAQ 1：XOS 是什么、不是什么——先弄清定位，再决定往哪挖。

## Q1：XOS 到底是什么？

一句话：**自己和朋友玩的独立 x86-64 微内核操作系统**——能真启动、真编程序、真跑移植件，
但离"生产系统"还远。

- **独立 OS，不是组装货**。README 开头写明：x86-64、BIOS 启动、NASM + 受限 C++ 微内核，
  目标是"能在系统内编辑、保存、编译和运行程序"。内核、supervisor、文件服务、xsh 都是自研的。
- **微内核方向（C01）**：内核只提供隔离、线程/地址空间、受保护 IPC、等待/中断和资源生命周期
  这些机制；文件、终端、装载、应用策略原则上都在用户态；驱动尽量做成用户态驱动服务（C28）。
- **自研栈 + 真实移植件**：`src/ports/` 下有 dash、make、sbase、binutils、ncurses、cpython、
  gcc、newlib、qt6、adventure、aurora 浏览器/桌面等移植件。验收状态各不相同（有的受限通过、
  有的未验收），一律以 `docs/state.md` 和对应 work_log 为准——**不看目录存在就当"能用"**。
- **定位是"可玩可用"**：C08——为自己和朋友建设可玩、可用的系统；研究失败、Linux 更快、
  没有论文都不构成停止条件（C04 也把 AI、论文、商业化挡在版本关键路径外）。

## Q2：它实际长什么样？

三层控制台，别走错门（见 `docs/usage/index.md`）：

| 控制台 | 怎么进 | 提示符 |
|---|---|---|
| 内核调试控制台 | `make run` 启动 | `xos64> ` |
| 用户态 supervisor / Xsh | 在 `xos64>` 输入 `run` | `xsh>` |
| 实模式练习控制台 | `make run_real_mode`（另一个镜像） | `xos>` |

典型一屏：

```text
xos64> run
xsh> exec-file hello.elf
hello from file
xsh> dash -i
$ python3
```

`run` 是单向的：进了 supervisor 就回不到 `xos64>`，想回来只能重启 QEMU。

## Q3：它和 Linux 是什么关系？能跑 Linux 程序吗？

- **不是 Linux 发行版**。XOS 没有用 Linux 内核，也不是"内核 + Debian 用户态"那种拼法；
  包管理是自研的 xypm（C17），不是 apt/dnf。
- **不兼容 Linux ABI**。宪法 P01 明确"不承诺 Linux ABI"；POSIX 路线也把 Linux ABI 列为
  当前不推进项（`docs/contracts/posix_subset.md`）。程序按自己的静态 ELF64 / SysV 约定开发。
- **有 Linux 经验会眼熟，但不能照搬**。xsh/dash、make、GCC 的用法、部分 shell 语义都很熟悉；
  dash 已过 C1–C3 受限 gate。但"Linux 上能跑"不等于"XOS 上能跑"，移植件要走自己的验收。
- **对照 Linux 不是目标**：C04 明确 Linux 研究对照不进入版本关键路径。

## Q4：宪法、纪律、C/P/O/D 是什么？

三个文件分工不同，别混着读：

| 文件 | 角色 |
|---|---|
| `docs/xos_plan.md` | **宪法**：第一部分设计原则（裁决设计），第二部分事实登记（描述现状） |
| `docs/Discipline.md` | **流程纪律**：怎么开任务、怎么交接、证据怎么交、PR 怎么走 |
| `docs/state.md` | **当前状态快照**：基线、各线一句话状态、能力表、下一步 |

**C/P/O/D 怎么读**：

- **C 类 = 已确认原则**：C01 微内核、C02 x86-64 单目标、C03 NASM + 受限 C++、
  C08 为自己和朋友建可玩可用系统、C31 建设基础而非制造完成假象……
  实现违反原则时**修实现**，不拿登记表豁免；改原则要用户明确确认并记录修订。
- **P = 建议**：如 P01"不承诺 Linux ABI"。
- **O = 开放选择**：如当前已采用的 BIOS/QEMU、GCC 16.2.0、Binutils 2.47、NASM 3.02。
- **D = 暂缓**：如全 POSIX、GPU、SMP、UEFI（D04）。
- 关键：**P/O/D 都不是"已实现"**，也不能单凭它们判定实现违规（宪法效力表）。

**证据纪律**（通俗版）：验收结果只分 `pass / partial / fail / not_run` 四态；
没跑的就写"未运行"，不以跳过冒充通过；报告要带哈希清单（manifest）；
公开提交前要扫掉个人主目录与机器专属命令路径。C31 的原话是：不以表面可用冒充完成。

## Q5：XOS 不是什么？（边界清单）

按 README「已知边界」、`docs/acceptance.md` 与 `docs/state.md` 核实：

- **不是生产系统**。定位是"自己和朋友可玩可用"的系统（C08）；镜像只读、网络关闭、
  不安装到宿主磁盘，也不以商业化为目标（C04）。
- **不是完整桌面环境**。Qt Widgets / Aurora 有受限验收（G04/G05 已过），但
  **QML/Quick 未实现**（G06/G07 `not_run`），完整整合与朋友试用（G08–G10）未完成。
- **不是 Linux 兼容层**（见 Q3）。
- **不是完整 POSIX**。UEFI、SMP、动态链接、32 位兼容按需另行规划（D04）；
  dash 不是完整 POSIX shell——后台 `&`、`trap`、`jobs`、信号与作业控制明确拒绝，
  前台流水线最多 4 段（`docs/usage/userland/dash.md` C1–C3 边界表）。
- **不是网络就绪的系统**。VirtIO-Net 驱动保持 RC 未合并，用户态网络线 BLOCKED；
  aurora 浏览器目前是本地 HTML/CSS 子集（B0/B1 已合，B2 网络 BLOCKED）。
- **不承诺系统重建/自举**。系统内重建 XOS（E10）、GCC 自举比较（E11）都是 `not_run`
  （acceptance O21/O22）。
- **"跑起来了"不等于"完成了"**。Discipline 写得很直白：欢迎串口文字不等于微内核完成；
  独立启动、用户隔离、原生编译、系统重建、编译器自举要分别证明（C05），谁也不能冒充谁。
- CPython 没有安全熵、墙钟、socket/TLS、fork/subprocess 等完整能力；
  xypm P3（完整事务/签名仓库）未完成，签名不等于沙箱。

## Q6：这项目严肃吗？能有多好玩？

Discipline 的「轻松愉快准则」允许无害彩蛋、里程碑外号和朋友间的玩笑，但要求
彩蛋小而可关、不能吞掉错误或篡改数据。比如内核控制台的 `guo` / `joke` / `about`
由 `config/x86_64_bios.json` 的 `easter_eggs` 控制，关闭后连帮助项都不存在
（`docs/usage/index.md`）。诊断先把事情说清楚，笑话随后再讲——这是仓库的原话风格。

许可证方面：新自研代码 Apache-2.0，历史 MIT 与第三方原条款保留（C10），
`make license-audit` 检查材料完整性；材料检查通过不冒称逐成员法律审计。

## 卡住怎么办？

- 想确认某个功能"到底行不行"：查 `docs/state.md` 能力表 → `docs/acceptance.md` 判据状态 →
  对应 work_log / `docs/reports/` 里的冻结证据。三处不一致时，按 Discipline 核源码与证据
  再更新登记，**不用改状态来掩盖问题**。
- 想继续深挖：从 `docs/README.md`（文档索引）按任务进 `docs/plans/`、`docs/implementation/`；
  下一件事从哪读起、第一个任务怎么找，见 [FAQ 4](04_where_to_go.md)。
