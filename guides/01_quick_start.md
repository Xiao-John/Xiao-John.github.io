---
layout: default
---

# 篇1 · 十分钟跑起 XOS

导语：这一篇只做一件事——在仓库根敲一条 `make run`，看懂屏幕上滚过的字，然后学会怎么退出。命令链来自 `Makefile`/`scripts/` 核对，屏幕输出是本机实测（用仓库现成的 ISO 与工作盘**副本**启动，没有重新构建），深挖留给 `docs/`。

## 你需要先准备什么

依赖清单（只列仓库实际用到的；装好后要能在 `PATH` 里找到）：

| 工具 | 本机实测版本 | 仓库里的依据 |
|---|---|---|
| `python3` | 3.12.15 | 所有 `make` 目标都是 `python3 scripts/*.py` |
| `nasm` | NASM 3.02 | `config/toolchain.json` 钉的 `nasm: 3.02` |
| `x86_64-elf-gcc` / `g++` / `ld` 等 | GCC 16.2.0，Binutils 2.47.20260726 | `config/toolchain.json` 钉的 `gcc`/`binutils` |
| `qemu-system-x86_64` | QEMU 11.1.2 | `config/toolchain.json` 钉的 `qemu`；`scripts/run_qemu.py` 启动它 |
| `make` | 系统自带即可 | 仓库根 `Makefile` |

- 版本不是建议值：`scripts/toolchain.py` 会拿实测版本和 `config/toolchain.json` 对账，不一致直接退出，报错形如 `GCC version mismatch: ...`。
- 本机这几个工具都装在 `/opt/homebrew/bin`（Homebrew），在 `PATH` 里就能被找到；仓库不负责安装它们。
- 缺工具时的原始报错（原文）：缺交叉工具链是 `Missing x86_64-elf-g++; see docs/toolchain.md`；缺 QEMU 是 `QEMU not found; install it or set QEMU=/path/to/qemu-system-x86_64`。

## 三步启动

以下都在**仓库根目录**（`/Users/xylo/Documents/XylophoneOS`）执行。

### 第 0 步：需要工作盘吗？

```sh
ls work_disk.img     # 仓库根；本机已有一块 16 MiB 的老盘
```

没有就先建一次（默认 2 GiB FAT32，**已存在则不会动它**）：

```sh
make disk
```

### 第 1 步：启动

```sh
make run
```

这条目标在 `Makefile` 里做的事（已核对）：

1. `install-iso` = `make build`（`scripts/build.py`）+ `make dash`（`scripts/build_dash.py`）+ `scripts/build_install_iso.py`；
2. 然后 `python3 scripts/run_qemu.py --install-iso`，启动 `build/xos-install-dash.iso`，挂上 `work_disk.img`。

QEMU 参数（`scripts/run_qemu.py` 实测，`.iso` 走 `-cdrom`）：`-machine pc-i440fx-10.0 -accel tcg -m 512M -smp 1 -nic none -vga std -serial stdio -display none -no-reboot`。

> **注意：`make run` 默认没有图形窗口。** 串口接到当前终端，所有输出直接打在你敲 `make run` 的那个终端里；输入也在这里敲。想看画面用 `make run_vga`（`Makefile` 里是 `--iso-file --vga`，同一块 `build/xos-install.iso`，额外开一个 SDL 窗口）；无论开不开窗口，**输入都在终端的串口**，不在窗口里。

### 第 2 步：你会看到什么

启动后终端按顺序出现（本机实测原文节选）：

```text
XOS: loading stage_two
[loader] firmware=bios target=x86_64
[boot] event=kernel_enter arch=x86_64 build=a209d0dc5ff71b9a
[mem] event=initialized usable_kib=523775 ... free_pages=130412
[task] event=created pid=1 name=heartbeat
...
[task] event=ready count=8
[boot] result=ready stage=E04
   XOS 0.0.2 | 64-bit | more checks than excuses
xos64>
```

停在 `xos64> ` 就算成功——这是**内核调试控制台**（提示符来自 `src/kernel/core/console.cpp`）。此时还没有任何 `A`/`ping`。

接着敲（就在终端里，一行一行来）：

```text
xos64> run
```

会先看到 `[sched] event=starting` / `[sched] event=start tasks=8`，然后是一段演示任务刷屏，这是**正常现象**，不是启动失败：

- `A`：heartbeat 每个时间片打一个，说明抢占在工作；
- `ping` / `tx`：receiver/sender 在端点 1 收发；
- `bad length` / `bad pointer` / `bad handle rejected` / `no send right`：故意的边界测试；
- `[task] event=fault pid=5 ... result=killed`：probe 故意读内核地址触发故障，只杀它自己；
- `bye`：quitter 主动退出。

约一两秒后出现：

```text
[files] hello.elf present bytes=9128

Welcome to Xshell | XylophoneOS
help: commands   dash -i: POSIX shell   trace on: diagnostics

xsh>
```

`xsh>` 就是 supervisor（原生 Xsh）。到这里，前十分钟的目标完成了。

> **关于刷屏**：进 `xsh>` 后演示任务还会再刷一会儿。本机实测：**敲下第一个键之后约 1 秒它们会自动停**（`docs/usage/index.md` 写的是“约 1 秒无输入自动停”，实测计时要等你先有输入才开始）。等不及就敲 `demo quiet` 立即停；想留着观察抢占就 `demo keep`。

### 第 3 步：验一下系统是活的

```text
xsh> stats
[resources] free=130091 tasks=2 completions=0
xsh> help
Files     ls [path] | cat/stat/rm/edit <file> | mkdir <path> | mv <old> <new> | sync
Programs  <command> [args] | exd <command> [args] | type <command>
...
```

`stats` 的三个数字是真实的空闲页/任务数/未回收完成记录；`help` 会按 Files / Programs / Debug 分组列出命令（`src/user/programs/supervisor.cpp`）。

## 怎么退出

按“想退出哪一层”从里到外：

| 想干什么 | 怎么做 |
|---|---|
| 取消正在前台等待的程序 | 单独按一次 **Escape**（终止被等待的任务，回到 `xsh>`） |
| 从 dash 回到 `xsh>` | 在 `$` 下敲 `exit` |
| 从 `xsh>` 回 `xos64>` | **做不到**：`run` 是单向的，回到内核控制台只能重启 QEMU |
| 停掉客体 CPU | `xsh>` 或 `xos64>` 敲 `halt`（只 HLT，**不关闭 QEMU**） |
| 关掉整个模拟器 | 在宿主终端按 **Ctrl-C**（`scripts/run_qemu.py` 捕获 `KeyboardInterrupt` 后干净退出） |

关掉之前如果改过文件，先 `xsh> sync`，看到 `[files] sync=ok` 再关，否则重启后内容可能不在。

## 卡住了？

这一篇不展开排查，直接去**篇3**（`xos_intro_zh/faq/03_build_test_faq.md`，FAQ 3《构建与测试》）：

- 缺 nasm / x86_64-elf 工具链、版本对不上 → Q1；
- `make build` 挂了、日志去哪看 → Q2；
- `make run` 起不来（缺镜像、缺盘、QEMU 找不到）→ Q3；
- 其他启动目标（`run_file`/`run_install`/`run_floppy`/`run_vga`/`run_real_mode`/`debug`）→ Q4；旧的 `run_iso` 目标已移除（2026-11-10 起 legacy `xos.img`/`xos.iso` 删除，统一走 `xos-install.iso`）；
- `make smoke` 是什么、跑不过怎么办 → Q5。

另外两个本机踩过的坑：停在 `xos64>` 时**不该**有任何 `A`/`ping`，一开机就刷说明你还在上一个 QEMU 窗口/终端里，或者串口残留了上次的输入（先关掉旧的再 `make run`）；`Work disk missing; run 'make disk' once, or pass --work-disk` 就是没建工作盘。

## 想继续

- 命令逐条说明：`docs/usage/index.md`（内核控制台、Xsh、实模式三套提示符的入口表）。
- 当前状态与验收边界：`docs/state.md`（能力表）、`docs/work_log/README.md`（编号与索引规则）。
- 下一篇：篇2《shell 导览》——`xsh>` 和 dash 里能敲什么、管道怎么玩。
