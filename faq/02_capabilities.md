---
layout: default
---

# FAQ · XOS 现在能做什么

导语：这是一份"能力清单式"问答——每项只回答两件事：**能玩什么**、**验收到什么程度**。验收状态以 `docs/work_log/README.md`（编号索引，自 0188 起全局唯一）、`docs/state.md` 为准，功能边界以 `README.md` 的"已知边界"、`docs/xos_plan.md` D 规则与各专项计划为准（`docs/Discipline.md` 没有单独的 Limits 段）。

## 启动与测试：怎么进系统、怎么算"通过"

- **能玩**：仓库根 `make build`、`make disk`、`make run`（= `make install-iso` + `scripts/run_qemu.py --install-iso`）；另有 `run_file` / `run_install` / `run_floppy` / `run_vga` / `run_real_mode` / `debug` 各条启动路径。进 `xos64> ` 后输入 `run` 到 `xsh>`，install ISO 首启会把 dash 装到 `/usr/bin/dash`。
- **验收**：`make smoke` 是回归门，串行跑 10 个冒烟脚本（boot/kernel/task/process/file/persist/full_disk/libc/e08_storage/runtime）加 `license_audit`。发行 0.0.2-1103 的 `release_smoke` **11/11**（`docs/work_log/0203`）。单条能力的资格报告在 `docs/reports/` 各 qualification.json。

## Shell 与管道

- **能玩**：`xsh>` 是默认入口（`ls` `cat` `edit` `sync` `exec-file` `exec-driver` `ps` `kill` 等内建）；`dash -i` 切到可选 POSIX shell，管道、重定向 `< > >> 2>`、命令替换 `$()`/反引号、here-document、函数、`case`、`set -e`、点脚本都能用；裸命令按 `/usr/bin:/bin:/` 搜索，所以 `dash`、`make`、`xypm` 直接敲。
- **验收**：dash **C1–C3 受限通过**（`docs/reports/dash_c3_20261026/qualification.json`，scope 写明 "declared bounded foreground XOS profile; not full POSIX"）：前台流水线最多 **4 段**、fork 深度 3、全局 8 进程、命令替换 stdout 上限 1 MiB。
- **明确没有**：后台 `&`、`trap`/`jobs`/`wait` 内建、异步 POSIX 信号、Ctrl-C/Ctrl-Z 作业控制（取消 = 前台 Escape）。

## 文件与工作盘

- **能玩**：`make disk` 造默认 2 GiB FAT32 工作盘（旧 16 MiB FAT16 仍兼容）；客体内 `ls/cat/stat/mkdir/mv/rm/edit/sync`，支持目录与长文件名，中文文件名有专项客体验证（`docs/work_log/0068`）；宿主用 `python3 scripts/work_disk.py <img> {mkdir,import,export} <path> [host]` 离线搬文件（`import` 拒绝覆盖）。写完必须 `sync`，看到 `[files] sync=ok`。
- **验收**：E07"正常保存→重启仍在"、`file_posix_guest` 的路径/偏移/rename/spawn 检查、`make smoke` 里的 file/persist/full_disk 三门。
- **边界**：路径 ≤255 字节、每会话 16 个打开/全局 32 个对象锁；无权限模型、无时间戳、无 fsck；突然掉电、坏扇区不承诺（`docs/e07_persistent_work_disk.md`）。

## 线程与进程

- **能玩**：threading SDK（`scripts/build_threading_sdk.py`，`make threading-smoke`），pthread 可用；1:1 内核线程、单核 PIT 抢占；限制是 8 个进程、每进程 4 个线程（含主线程）、全局 32 槽。
- **验收**：T0–T2、T3-PY 与 PY2 专项已有资格记录（`docs/reports/threading_20261025/qualification.json`）；Qt QThread 受限实测。
- **没有**：SMP（QEMU 固定 `-smp 1`，state.md 写明 SMP 未验）、完整 pthread 取消。

## IPC（进程间通信）

- **能玩**：sys81 受控通道（create/enumerate/close/wait/wait_many/fork_pair，容量 1–4096 字节），终端服务的读写线程和音频客户端握手都用它（`docs/work_log/0124`、`0223`）；sys84/85/86 的 endpoint/wait/name 命名 IPC 是 `interactiumd` 和 `xos-audio` 服务发现的通道；另有 sys82 PIO 授权、sys83 IRQ 事件、sys87 进程权威。
- **验收**：通道契约 `docs/contracts/channel.md`，加上 work_log 0159/0161/0162/0178 等真实双进程客体探针（等待、对端关闭、fork 派生与容量/权限边界）。
- **已合入**：IPC v1（`sys88`）在 `docs/contracts/abi_registry.md` 里已 **allocated** 为 `sys_ipc_v1`，契约 `docs/contracts/ipc_v1.md`、头文件 `src/user/include/xos/ipc_v1.hpp` 与探针（`tests/ipc_v1_guest.py`、`tests/ipc_v1_fastpath_guest.py`）均在 main；feature 位 supported=**0x0f**（endpoint_call/deadline/notification/wait_set），`bulk_grant` 待 IP4 验收。

## 显示与 GUI

- **能玩**：QEMU 窗口里跑窗口管理器——`xsh> interactium-start` 起 Display 服务 + `interactiumd`，或 `demo quiet` → `display-start` → `exec interactium` 看双窗口模型演示（点击提升、拖标题栏、Q/Escape 退出）；Qt Widgets 样例由宿主交叉编译成 ELF 后 `exec` 运行。
- **验收**：sys65/66 显示路径 G02 **19 组通过**（`docs/reports/qt_e16_qualified_20261028/g02/report.json`）；Qt 6.8.3 静态资产随 0.0.2-1103 发行，E16 的 QtCore/QtGui+QPA 受限客体功能通过、Widgets G05 受限验证；ncursesw 宽字符终端 W1 最终验收全 gate 通过（`docs/work_log/0187`）。
- **边界**：单全屏 raster、每会话 8 窗/8 个像素 surface、CPU 合成、无 vblank；**QML/Quick 未实现**；客体内没有 Qt（应用在宿主编译）；`gfxdemo`/`display-start`/`interactiumd` 抢同一个客户端槽，不能并存。

## 音频

- **能玩**：`xsh> tone 440 250`、`melody <file>`、`audio status|cancel|close|start|restart`（命令已在 supervisor 里）；旋律文件每行 `频率Hz 时长ms`，0 时长是休止，1–64 音、40–5000 Hz、总长 ≤10 秒。
- **验收到哪**：AU0 勘查通过（QEMU PC Speaker→WAV 探针捕获到约 5.01 kHz 信号）；**AU1/AU2 源码已合入但客体验收"待工具链"**——当时环境缺登记的 x86_64-elf 工具链，客体 build、WAV 捕获、人工试听全部"未运行"（`docs/work_log/0220`–`0222`、`docs/plans/audio_output.md`）。所以"敲 `tone` 一定出声"目前不作承诺。
- **没有**：音量、混音、WAV/PCM 播放、录音（AU3–AU5 未开始）；`make run` 默认不配 QEMU 音频后端。

## 包管理（xypm / xpk）

- **能玩**：宿主 `python3 scripts/xypm.py install|upgrade|remove|list|info|verify-installed --image <img> [--output-image ...] --offline` 在副本盘上做事务；客体内 `xypm list` / `xypm info <id>` 只读查询（两个 Shell 都行），写事务回 `xsh>` 用 `maint-run /usr/bin/xypm install /out/pkg.xpk`（也支持 `upgrade`/`remove`，dash 里不能跑维护入口）。打包用 `scripts/xos_app.py init/build/check/pack` 出 `.xpk`。
- **验收**：P1/P2 在限定范围内验收通过（`docs/work_log/0066`、`docs/plans/packages.md`："P2 客体事务已实现，最终验收与限制见工作记录"）；`release_smoke` 里有 `xypm info python_idna` 与"发行盘 local profile 被拒绝"两门。
- **没有**：P3 仓库分发、签名与密钥轮换未完成；包校验是完整性不是签名授权，也不是沙箱。

## 移植（端口）现状

| 端口 | 能玩什么 | 验收到什么程度 |
|---|---|---|
| dash | `dash -i`、`dash /out/x.sh`、管道脚本 | C1–C3 受限通过（资格报告见上） |
| make | `make -B -f /out/project.mk all`（sbase Make 子集，强制 `-B` 串行重建） | `release_smoke` 客体门通过（0.0.2-1103）；端口 README 仍注明其构建 manifest 标 `built_not_run` |
| sbase / 日常工具 | `cat cp mv rm mkdir ls rmdir head tail wc tee basename dirname` | 13 个工具经 `tests/daily_utilities_smoke.py` 定向验证（`docs/work_log/0069`），非全失败面 gate |
| binutils | `/usr/bin/x86_64-elf-as`、`ld`、`objdump` 等 | E08-1/2/3 Seed 受限 gate 通过 |
| ncurses | `tty-run` 跑宽字符/全屏程序 | W1 最终验收 12 项子集 + Screen 附着通过（`0187`） |
| cpython | 客体内 `python3` 交互、脚本、线程、JSON/zlib/离线 wheel | PY0–PY2、PY3 A1–A3/B1–B4、T3-PY 专项证据；无 socket/TLS、无 pip |
| Qt6 | 宿主编译静态 Widgets 应用并在 QPA 上跑 | E16 G01/G03/G04 受限、Widgets G05 受限；QML/Quick 未实现 |
| adventure | 客体内生成 `dungeon.c` 并编译，中文叙述、英文命令 | `tests/adventure_smoke.py` ADV0 门（需 `.cache/xos_seed/adventure/...` 种子） |
| aurora 浏览器/桌面 | 打开本地 HTML/CSS 子集页面 | B0/B1 已合入；**B2 网络 BLOCKED** |

## 玩法点子（挑一个开玩）

1. **跑一个 dash 脚本**：照抄 `docs/usage/userland/dash.md` 里的最小脚本（`set -e` + heredoc + 管道），宿主 `python3 scripts/work_disk.py work_disk.img import /out/hello.sh hello.sh`，客体里 `dash /out/hello.sh`，看到它打出 `RELEASE_DASH_C3_OK` 那一行。
2. **在 XOS 里编译并运行 C**：`xsh> exec-driver /usr/bin/x86_64-elf-gcc --sysroot=/sysroot -I/src -c /src/main.c -o /out/main.o`，或 `make -B -f /out/project.mk all` 打出 `Hello from XOS C!`。
3. **玩一把 adventure**：`python3 tests/adventure_smoke.py`（仓库根，需 adventure/GCC/dash/cpython 种子），看客体内生成 dungeon 数据、编译、进入"欢迎来到冒险"。
4. **看双窗口演示并留证据**：`python3 tests/interactium_guest.py`，截图落在 `build/`；或手动开 SDL 窗口 `python3 scripts/run_qemu.py --install-iso --display sdl` → `run` → `interactium-start`。
5. **跑 Python**：发行盘上 `python3` 进 REPL，`python3 /out/py3.py` 打出 threads/json/zlib/idna 全通过那行（`release_smoke` 的 python_py3 门）。

## 还不能做（别按这个预期来）

- **网络**：QEMU 启动固定 `-nic none`；VirtIO-Net N1 保持 RC 未合并、N2 用户态网络 BLOCKED，浏览器 B2 网络 BLOCKED（`docs/state.md`）。
- **真机与 UEFI**：只在 QEMU x86-64 BIOS 上验过；真机/UEFI/任意掉电未验（`docs/history/boot_display.md`），UEFI loader 尚不存在，**不提前引入 Runtime Services、Secure Boot 或固件变量写入**（`docs/architecture_and_boot.md`）。
- **多核 SMP**：`-smp 1`，SMP 未验（D04：全 POSIX、GPU、SMP、UEFI、动态链接分别规划，不是当前前置）。
- **完整 POSIX / 作业控制**：后台作业、异步信号、termios 全套、动态链接、32 位兼容都没有。
- **QML/Quick**：未实现；Qt 只有 Widgets/raster 这条受限路径。
- **系统内重建与编译器自举**：D6 尚未开始，"在 XOS 里重建整个系统"未验收。
- **在线能力**：没有 pip、没有网络包仓库（xypm P3 未完成），CPython 无 socket/TLS/fork。
- **音频的"已发声"**：AU1/AU2 未客体验收（见上）；PCM/WAV/录音不存在。
- **掉电与坏介质**：保存流程有 journal/backup 恢复，但不承诺突然掉电原子性、坏扇区、fsck。
- **IPC v1（sys88）**：已合入 main（allocated 为 `sys_ipc_v1`，supported=0x0f），bulk_grant 待 IP4。
