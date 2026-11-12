---
layout: default
---

# 篇2 · shell 导览

导语：进了系统该敲什么？这一篇带你认清三个提示符、用 supervisor 跑起一个程序和一个脚本，再做一组真的能跑通的管道小实验——带输出的都是本机实测，没跑过的会标明出处。

## 先分清：你现在在哪一层

XOS 里有三个提示符，一层套一层，认错了会以为“命令不存在”：

| 提示符 | 是什么 | 怎么进来 |
|---|---|---|
| `xos64> ` | 内核调试控制台（`src/kernel/core/console.cpp`） | `make run` 启动后就是它 |
| `xsh> ` | supervisor / 原生 Xsh，用户态入口 | 在 `xos64>` 敲 `run`（**单向，回不去**） |
| `$` | 可选的 dash（POSIX 子集 shell） | 在 `xsh>` 敲 `dash -i`，`exit` 回 `xsh>` |

完整入口表见 `docs/usage/index.md`；实模式练习镜像（`make run_real_mode`）是另一套 `xos>`，跟这里不互通。

## supervisor 常用命令（`xsh>`）

命令表来自 `src/user/programs/supervisor.cpp` 的内建名单，逐条说明在 `docs/usage/supervisor/`，测试在 `tests/process_smoke.py` 等。下面带输出的都是本机实测。

### 1. `exec-file`：跑一个程序

```text
xsh> exec-file hello.elf
hello from file
```

`hello.elf` 是 files 服务首次启动时写进工作盘的示例（启动时会打印 `[files] hello.elf present bytes=9128`），不用你编译。它装载静态 ELF、等待、打印结果；成功默认不刷日志。

### 2. `sync`：改完文件必须落盘

```text
xsh> sync
[files] sync=ok
```

只有全部成功才打印 `sync=ok`，失败会给真实错误（如 `[files] sync error=-3`）。**关 QEMU 之前先 `sync`**，否则重启后内容可能不在。

### 3. `stats`：看资源

```text
xsh> stats
[resources] free=130091 tasks=2 completions=0
```

三个数分别是物理空闲页、当前任务数、未回收的完成记录——exec 前后各看一眼最直观。

### 4. `trace`：把诊断打开

```text
xsh> trace on
xsh: trace on
xsh> exec-file hello.elf
[process] event=started pid=23 parent=7
[loader] started pid=23 file=/hello.elf
hello from file
[wait] pid=23 kind=exited code=0
xsh> trace off
xsh: trace off
```

`trace off` 恢复简洁输出。程序的正常输出、非零退出、故障和取消**任何时候都会显示**，不靠这个开关。

### 5. 其他常用

```text
xsh> help
xsh> ls /
xsh> cat /out/guide.sh
xsh> ps
pid=7 name=supervisor state=R ticks=241
pid=8 name=files state=r ticks=1355
xsh> type dash
dash -> /usr/bin/dash
xsh> demo quiet
[sup] demo tasks already stopped
```

`help` 分组列出全部内建命令，`type` 解释命令最终解析到哪，`kill <pid>` 终止任务，`demo quiet` 立即停掉演示任务（演示任务还在跑时打印的是 `[sup] demo tasks stopped`）。

> **`run` 不是 supervisor 命令。** 它属于内核控制台：`xos64> run` 启动 ring3 演示任务并进入调度，然后才轮到 `xsh>`。`help` 列表里没有 `run`，因为它已经被你用掉了。测试脚本里那些 `send_bytes(g, b'run\r')` 就是在按回车执行它（`tests/*.py`）。

## 跑 dash 与一个脚本

### 进入 dash

```text
xsh> dash -i
$ exit
xsh>
```

`exit` 回到 `xsh>`（正常退出不打印任何东西；非零退出会显示 `xsh: program exited with status N`）。

### 写一个脚本再跑

方式 A——宿主写好、导入工作盘（**先关掉用这块盘的 QEMU**），在仓库根执行：

```sh
python3 scripts/work_disk.py work_disk.img mkdir /out
python3 scripts/work_disk.py work_disk.img import /out/guide.sh guide.sh
```

方式 B——客体内 `xsh> edit /out/guide.sh`，`:wq` 保存，然后 `sync`（编辑器用法见 `docs/usage/supervisor/edit.md`，本机未实测）。

脚本内容（实测原文；只用内建的 `echo`/`printf`，不依赖管道等高级特性）：

```sh
echo SCRIPT_START
printf 'line %s\n' one
printf 'line %s\n' two
echo SCRIPT_END
```

跑它（本机实测用的是 `type dash` 找到的盘上新版；盘上是当前版本时直接 `xsh> dash /out/guide.sh` 即可，`tests/*.py` 的脚本用例也都是 `dash /path/script.sh` 这个形式）：

```text
xsh> /bin/dash /out/guide.sh
SCRIPT_START
line one
line two
SCRIPT_END
```

两个容易踩的点：

- **必须显式交给 dash**：`exec-file /out/guide.sh` 不会把它当脚本执行（xsh 不猜扩展名）。
- **`dash` 解析到哪个文件很重要**：`xsh> type dash` 看路径。本机这块老盘上 `/usr/bin/dash` 是旧 B2 版，`dash /out/guide.sh` 会报 `dash: 0: cannot open /out/guide.sh: No more processes`，换 `/bin/dash` 就正常——所以先 `type` 再怀疑脚本写错了。

## 管道小实验

### 第 0 步：确认 `$` 背后的 dash 支持管道

```text
xsh> type dash
dash -> /usr/bin/dash
xsh> dash -i
$ printf a | cat
dash: 0: pipeline unavailable in XOS dash B2
$ exit
xsh> /bin/dash -i
$ printf a | /bin/dash -c "read x; echo PIPED_OK"
PIPED_OK
```

看到 `B2` 说明盘上是**旧版 dash**（只到重定向那一档，没有管道），`exit` 回 `xsh>` 换用盘上带管道的那份再进（本机实测如此）；若只看到 `cat: not found`，那是这块盘没有 `cat` 命令，管道本身是好的。当前源码 `make dash` 出的是 C3 档（`scripts/build_dash.py` 的 manifest 写 `profile: dash_c3`），但 **install ISO 首启只在 `/usr/bin/dash` 不存在时才写入，已存在的旧文件不会被覆盖**（`src/user/services/file_ui.cpp` 的 `install_boot_payload`）。想换新的：在 `xsh>` 里 `rm /usr/bin/dash` → `sync` → 重启 QEMU 让 ISO 重装，或者把旧盘改名备份后 `make disk` 建一块新盘。此段为源码核实，未在本机实测重装流程。

### 实验 1、2：两段管道（生产者 → 消费者）

```text
$ printf "hello\n" | /bin/dash -c 'read line; echo got=[$line]'
got=[hello]
$ printf 'one\ntwo\n' | /bin/dash -c 'while read l; do echo "[$l]"; done'
[one]
[two]
```

### 实验 3：段数上限是 4

```text
$ printf a | /bin/dash -c "read x" | /bin/dash -c "read x; echo THREE_OK"
THREE_OK
$ printf a | /bin/dash -c "read x" | /bin/dash -c "read x" | /bin/dash -c "read x" | /bin/dash -c "read x"
/bin/dash: 0: XOS dash C1 supports at most 4 pipeline stages
$ echo status=$?
status=2
```

第 5 段在执行前就被拒绝（退出码 2），`xsh>` 侧会看到 `xsh: program exited with status 2`。实测 4 段时偶发 `Device or resource busy`（进程资源紧张），建议从 2~3 段玩起。

### 仓库里真实的管道测试长什么样

`tests/dash_c1_smoke.py` 会自己造一块私有 FAT32 盘，把 `dash`、测试程序 `/out/probe` 和一串脚本导进去，再在 QEMU 里逐条跑，例如：

```sh
/out/probe generate 127 | /out/probe expect 127     # early_producer
/out/probe exit 7 | /out/probe exit 23              # last_status：末段状态为准
/out/probe spin | /out/probe copy | /out/probe copy | /out/probe copy   # 4 段：exhausted
/out/probe spin | /out/probe copy | /out/probe copy | /out/probe copy | /out/probe copy  # too_long：第 5 段被拒
```

`/out/probe` 是测试 fixture（源码 `tests/fixtures/dash_c1/pipe_probe.c`），**客体上默认没有**，别照着敲；想复现就直接跑那个测试脚本。其余能力边界（重定向、命令替换、here-doc、后台 `&`/`trap` 明确不支持）见 `docs/usage/userland/dash.md` 的 C1–C3 表，验收状态看 `docs/reports/dash_c3_20261026/qualification.json` 与 `docs/state.md` 的 Shell 行。

## 清屏与退出

- **xsh 和 dash 都没有清屏命令**（内建名单里没有 `clear`；只有实模式练习控制台有，见 `docs/usage/real_mode/clear.md`）。嫌乱就用宿主终端自己的清屏快捷键。
- 层级退出：dash 里 `exit` → `xsh>`；`xsh>` **回不去** `xos64>`，只能重启 QEMU。
- 前台等待程序时，单独按 **Escape** 取消被等待的任务并回到 `xsh>`。
- 关掉整个模拟器用宿主 **Ctrl-C**；客体里的 `halt` 只停 CPU，不退出 QEMU。
- 退出前记得 `xsh> sync`。

## 卡住了？

去**篇3**（`xos_intro_zh/faq/03_build_test_faq.md`，FAQ 3《构建与测试》）：缺工具 Q1、构建日志 Q2、`make run` 排查 Q3、其他启动目标 Q4、`make smoke` Q5。命令语义细节则查 `docs/usage/index.md` 和 `docs/usage/supervisor/` 下的单命令页。
