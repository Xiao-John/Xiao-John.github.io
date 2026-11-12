---
layout: default
---

> FAQ 3：构建与测试——工具链缺了怎么补、报错去哪看日志、`make run` 起不来怎么排查、`make smoke` 和客体测试怎么跑。每条都是"现象 → 怎么办"，深挖留给仓库文档。

## Q1：`make build` 说缺 nasm / x86_64-elf 工具链，装什么？

仓库根目录执行 `make build`（= `python3 scripts/build.py`）前，宿主要有这些东西：

| 缺什么 | 报错长什么样 | 怎么办（通用建议） |
|---|---|---|
| `nasm` | `NASM not found; install it or set NASM=/path/to/nasm` | macOS 上走 Homebrew 装 `nasm`；装到非标准路径就用 `make build NASM=/path/to/nasm` |
| `x86_64-elf-gcc/g++/as/ld/objcopy/nm/ar` | `Missing x86_64-elf-<name>; see docs/toolchain.md` | 交叉工具链同样可以走 Homebrew 的 `x86_64-elf-*` 系列 formula，或自己编一份；关键是**版本要对得上**，见下一条 |
| `qemu-system-x86_64` | 构建不查它，起虚拟机时才报 `QEMU not found; install it or set QEMU=/path/to/qemu-system-x86_64` | Homebrew 装 `qemu`；同样可用 `make run QEMU=...` 覆盖 |
| `python3` | 找不到 `python3`（shell `command not found`，make 收尾 `Error 127`） | 装 Python 3；`make` 变量 `PYTHON` 可覆盖 |

**版本是锁死的**，不是"随便一个 x86_64-elf-gcc 都行"：`config/toolchain.json` 写着 target `x86_64-elf`、GCC **16.2.0**、Binutils **2.47.20260726**、NASM **3.02**、QEMU 11.1.2。对不上会直接退出：

- `GCC version mismatch: ...; expected 16.2.0`
- `Cross target/Binutils mismatch: ...`
- `NASM version differs from config/toolchain.json`

所以装完先 `x86_64-elf-g++ --dumpmachine`、`nasm -v` 核一眼版本。改用别的前缀工具可用环境变量 `CROSS_PREFIX`（默认 `x86_64-elf-`）；`NASM`、`QEMU`、`PYTHON` 走 make 变量最省事（README 举例 `make smoke NASM=... QEMU=...`）。

> 普通构建**不会**下载或重编 GNU 工具链，工具链要你自己备好。

**详见**：[docs/toolchain.md](../../docs/toolchain.md)、[README「从源码构建」](../../README.md)、[docs/e08.md](../../docs/e08.md)

## Q2：`make build` 挂了，日志去哪看？

- **第一现场就是终端**：构建脚本用 `subprocess.run(..., check=True)` 逐条执行命令，失败会抛 traceback，**里面带完整命令行**（哪个编译器、哪些参数、哪个源文件），先照着它定位。
- **`build/` 下的固定日志**（构建成功才生成）：
  - `build/kernel_disassembly.log`（`objdump -d`）、`build/kernel_elf.log`（`readelf -h -l -S`）、`build/compiler_configuration.log`（`g++ -v` 完整配置）
  - `build/boot_sector.lst` 等 NASM `-l` 列表文件，汇编阶段出错看这里
  - `build/manifest.json`：本次构建的源码哈希、**完整命令列表**（`commands` 字段）、工具可执行文件哈希、产物清单——"上次到底跑了什么"查它
- **dash 构建（`make dash`）日志不在 build/**：写在 `.cache/xos_seed/dash/<key>/configure.log`、`make.log`；失败时脚本会把日志**末尾 7000 字符**直接打到终端。
- 看到 `Stale build: <文件>`：说明源码在上次 `make build` 之后被改过，`make build` 重来一次即可（构建 ID 会变，旧测试要按新 build 重跑）。

**详见**：[docs/toolchain.md](../../docs/toolchain.md)（manifest 记录什么）、[docs/usage/userland/getting_started.md「日志位置」](../../docs/usage/userland/getting_started.md)

## Q3：`make run` 起不来，怎么一步步排查？

`make run` = `install-iso`（含 `build` + `dash`）→ `python3 scripts/run_qemu.py --install-iso`，**它会自己先把 ISO 构建出来**，所以起不来基本是下面四种：

1. **缺 QEMU**：`QEMU not found; install it or set QEMU=/path/to/qemu-system-x86_64` → 装 QEMU 或用 `make run QEMU=...`。
2. **缺工作盘**：`Work disk missing; run 'make disk' once, or pass --work-disk` → 仓库根目录 `make disk` 一次（新盘 2 GiB FAT32，**已有盘不会被覆盖**）。
3. **缺镜像**：`Run make build first` → 说明依赖的镜像没生成；先 `make build` / `make install-iso`。
4. **ISO 没生成**：确认这两个文件在不在——
   ```sh
   # 仓库根目录
   ls -l build/xos-install.iso         # make build 产物（file-backed 安装 ISO）
   ls -l build/xos-install-dash.iso    # make install-iso 产物（带 dash payload），make run 启动的就是它
   ```
   `build_install_iso.py` 成功时会打印一段 JSON，含 `iso` 路径和 `iso_sha256`，可以直接核对。

**起来了但"没画面"是正常的**：默认 `-display none`，输出走串口，**直接打在你的终端上**（QEMU 参数 `-serial stdio`）。要图形窗口用 `make run_vga`（SDL 窗口）。退出按 `Ctrl-C`（这是关宿主 QEMU，不是客体命令）。

**详见**：[docs/usage/userland/getting_started.md](../../docs/usage/userland/getting_started.md)（含常见错误表）、[docs/usage/index.md](../../docs/usage/index.md)

## Q4：其他 `run_*` 目标起不来？

- `make run_file` 启动 `build/xos-install.iso`；`make run_install` 与 `make run` 同为 `--install-iso`（`build/xos-install-dash.iso`）；`make run_floppy` 是保留的目标名，实际经 `scripts/run_qemu.py` 引导 `build/xos-install.iso`（legacy 软盘镜像已移除，见 `docs/work_log/0224_drop_legacy_images.md`）。旧的 `run_iso` 目标已删除。
- legacy 的 `xos.img`/`xos.iso` 与 `legacy_images_available` 字段已在 main 上彻底废除（`docs/work_log/0224_drop_legacy_images.md`），`scripts/build.py` 的常规引导产物是 `build/xos-install.iso` 与 `build/real_mode.img`；日常用 `make run`（install ISO）即可。
- `make run_real_mode` 走 `build/real_mode.img`（始终生成），是 16 位实模式练习台，和主系统不互通。

## Q5：`make smoke` 是什么？跑不过怎么办？

**它是一道串行回归门**：`make smoke` = `make build` + 下面 10 个客体测试（都在 QEMU 里真跑）+ 最后一道宿主侧许可材料检查 `scripts/license_audit.py`：

```text
boot_smoke → kernel_smoke → task_smoke → process_smoke → file_smoke
→ persist_smoke → full_disk_smoke → libc_smoke → e08_storage_smoke
→ runtime_smoke → license_audit
```

- **先保证 build 是新的**：源码改动后必须重新 `make build`，否则 `kernel_smoke` 会先报 `Stale build: <路径>`（运行时提示，重建即可）。
- legacy 废除后（`docs/work_log/0224_drop_legacy_images.md`），`xos.img` 相关的 `KeyError` 挂账已过时：全部测试已迁到 `build/xos-install.iso`，`make smoke` 不再因缺 `xos.img` 失败。
- 想快速验证"能不能启动"，先单跑下面这条（见 Q6）。

**详见**：[docs/work_log/0224_drop_legacy_images.md](../../docs/work_log/0224_drop_legacy_images.md)、[docs/Discipline.md](../../docs/Discipline.md)（证据四态：pass/partial/fail/not_run）

## Q6：客体测试怎么单独跑？（三个真实例子）

都在**仓库根目录**执行，失败时看 `build/<名称>_qemu.log` 与 `build/<名称>_serial.log`。

**例 1：`tests/boot_smoke.py`——零参数，最快确认"系统能启动"**

```sh
python3 tests/boot_smoke.py
# 实测输出：PASS: 6 checks | build a209d0dc5ff71b9a
```

它没有 argparse（`--help` 会被忽略、直接开跑），用的是始终生成的 `build/real_mode.img`，验证 BIOS 启动、串口/键盘命令、损坏镜像拒绝，报告写 `build/smoke_report.json`。**当前这条是通的**。

**例 2：`tests/iso_file_kernel_smoke.py`——验 ISO 内核与文件服务**

```sh
python3 tests/iso_file_kernel_smoke.py            # 可选：--report build/xxx.json
```

用 `build/xos-install.iso` 起 QEMU，建临时 FAT 盘，断言到 `xos64> `、`run` 进 supervisor、`type hello` 有输出；实测 `status: passed`。

**例 3：`tests/dash_c1_smoke.py`——参数要给全（`--help` 实测）**

```text
usage: dash_c1_smoke.py [-h] [--isolated] --system SYSTEM --dash-manifest
                        DASH_MANIFEST [--report REPORT] [--artifacts ARTIFACTS]
```

```sh
python3 tests/dash_c1_smoke.py --system build \
  --dash-manifest .cache/xos_seed/dash/<key>/manifest.json
```

- `--system` 是含 `manifest.json` 的构建目录（如 `build`）；`--dash-manifest` 指向 `make dash` 写出的 `.cache/xos_seed/dash/<key>/manifest.json`（key 由输入哈希决定，`ls -t .cache/xos_seed/dash/*/manifest.json | head -1` 取最新一份；`make dash` 成功时也会打印该 manifest 的 `elf`/`sha256` 等字段）。
- **现状**：它按 `--system` 目录的 `manifest.json` 校验 `build/xos-install.iso` 的哈希并从该 ISO 启动（main 上已不引用 `xos.img`）；参数给全就能跑。

**详见**：[docs/usage/index.md「维护规则」](../../docs/usage/index.md)、[docs/work_log/0217_sl2_shell接入.md](../../docs/work_log/0217_sl2_shell接入.md)（dash 系列测试需 `--system/--dash-manifest`）、[docs/usage/userland/dash.md](../../docs/usage/userland/dash.md)

## Q7：测试要种子/资产（cpython、Qt）怎么办？

普通 `make build`/`make smoke` **不下载任何 GNU 源码**，需要额外输入的测试得自己准备：

| 测试 | 缺什么 | 怎么办 |
|---|---|---|
| `make binutils-smoke` | `SEED_MANIFEST` 环境变量指向种子清单 | `make binutils-smoke SEED_MANIFEST=<path>`；只跑已准备好的 Seed，普通构建不下载 |
| `tests/cpython_py0b_smoke.py` 等 | `.cache/xos_seed/cpython/*/py0a_report.json`；默认镜像 `build/xos-install.iso` | 先 `python3 scripts/build_cpython.py --compile-probe` 产出 PY0a 构建；缺了会明确提示 `run build_cpython.py --compile-probe` |
| `tests/qt6_gui_acceptance.py` 等 Qt 家族 | `--image` / `--payload` / `--output` / `--case` 四个必填参数（`--help` 实测）；payload 是用 **Qt 6.8.3 静态资产**交叉编译出的 ELF | Qt 库不在 `build/sdk`，要下发行资产 `xylophoneos_revival_<版本>_qt6_static.tar.gz` 或本地 Qt 构建树；`--output` 目录必须**尚不存在** |
| GCC/Seed 系列 | 各种 `--seed-manifest` / `--sysroot-manifest` | 先用 `python3 scripts/seed_preflight.py --seed-manifest ... --seed-root ... --sysroot-manifest ... --sysroot-root ...` 做只读校验 |

这些测试在 work_log 里常标为"未运行（环境）"——**缺资产跑不了就写 not_run，不要写成通过**。

**详见**：[docs/e08.md](../../docs/e08.md)（Seed/SDK 路径与命令）、[docs/usage/userland/qt_development.md](../../docs/usage/userland/qt_development.md)、[docs/usage/userland/python.md](../../docs/usage/userland/python.md)、[docs/plans/cpython.md](../../docs/plans/cpython.md)

## Q8：公共路径检查、证据报告是干嘛的？

一句话：**公共路径检查是"发出去之前先扫一遍别把 `/Users/xxx`、临时目录这类私人路径漏出去"；证据报告是"每个测试结果附一份带哈希的清单，别人能核对文件没被改过"。**

- 提交前：`python3 tests/check_public_evidence_paths.py --index --base origin/main`（扫暂存区新增/修改文件）。
- 报告目录：`tests/report_manifest_check.py` 按新旧分别报 `compliant` / `grandfathered`；发证据前用 `python3 scripts/verify_evidence_index.py <索引>` 核哈希。

**详见**：[docs/Discipline.md「证据完整性」](../../docs/Discipline.md)、[docs/contracts/evidence_integrity.md](../../docs/contracts/evidence_integrity.md)、[docs/work_log/README.md](../../docs/work_log/README.md)（work_log 编号规则）

## 卡住怎么办？

- 构建/启动报错先抄**完整报错行**，再对照 [docs/usage/userland/getting_started.md 的常见错误表](../../docs/usage/userland/getting_started.md)。
- 不确定某能力是否已验收 → 查 `docs/state.md` 能力表 + 对应 `docs/work_log/`，README「已知边界」列了当前明确不支持的面（UEFI、SMP、QML/Quick、完整 POSIX 等）。
- 测试报 `Stale build` → 先 `make build` 重建再跑；仍失败就对照 work_log 查已知问题，别自己改测试放水。
