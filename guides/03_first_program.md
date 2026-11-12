---
layout: default
---

# 篇 3 · 写第一个用户态程序

> 导语：从跑通现成的 `hello` 开始，把"编译 → 放进系统 → `exec-file` 跑 → 起线程"这条最短路径走一遍。所有构建命令在仓库根执行，`xos64>` / `xsh>` 开头的命令在 QEMU 串口终端里敲。

## FAQ：动笔前先对齐几个问题

**Q：什么是"用户态程序"？**
`supervisor`（xsh）、`files`、`dash`、`hello` 都是用户态 ELF：内核只提供调度、隔离和系统调用，其余全在用户态。你的程序和它们同级，放进系统后用 `exec-file` 起。

**Q：我需要装什么？**
macOS 上要 `qemu-system-x86_64`、`nasm`、`x86_64-elf-gcc/binutils`、`python3`。版本锁在 `config/toolchain.json`（GCC 16.2.0、Binutils 2.47.20260726、NASM 3.02、QEMU 11.1.2），`scripts/toolchain.py` 构建时逐项核对，版本不符直接报错，见 `docs/toolchain.md`。

**Q：写 `main` 还是 `xos_user_main`？**
看链接时用哪个 CRT（见下文三要素）。仓库里的 `src/user/programs/hello.cpp` 用 `xos_user_main` + `xos::sdk::print`，是入门最省事的一档；发行 SDK 的 `compile.py` 路线则写普通 `main`，见 `docs/usage/userland/getting_started.md`。

## 第一步：不写代码，先跑通

```sh
# 仓库根
make disk     # 首次创建 work_disk.img；已存在则不动
make run      # = build + dash + install-iso + scripts/run_qemu.py --install-iso
```

串口里依次输入：

```text
xos64> run
xsh> exec-file hello.elf
```

看到 `hello from file` 并回到 `xsh> ` 就成了。`hello.elf` 是 files 服务首次启动时从内置引导模块写进工作盘的，**不需要你编译**。再玩一个内置镜像：`xsh> exec echo`（回显输入）。

## 第二步：编译链接三要素

`make build` 除了出镜像，还会导出中间 SDK `build/sdk/`（由 `scripts/user_build.py` 打包），三要素都在里面：

| 要素 | 在哪 | 来自哪里 |
|---|---|---|
| 1. 头文件 | `build/sdk/include/`（newlib C 头 + `xos/*.hpp`） | `src/shared/include/xos`、`src/user/include/xos` |
| 2. CRT 与链接脚本 | `build/sdk/crt0.o`、`crt0_libc.o`、`crt0_posix.o`、`user.ld` | `src/user/arch/x86_64/start.asm`（nasm 编）、`src/user/arch/x86_64/user.ld` |
| 3. 库 | `build/sdk/lib/libxos.a`、`libxos_loader.a`、`libc.a`、`libm.a` | `libxos.a` 是 XOS 自己的 SDK 实现；`libc.a`/`libm.a` 是 newlib |

CRT 的区别只在入口（`start.asm` 里的分支）：`crt0.o` 直接调 `xos_user_main`；定义 `XOS_LIBC` 的 `crt0_libc.o` 先 `xos_libc_init` 再调 `xos_user_main`（带 newlib 初始化与收尾）；定义 `XOS_POSIX_MAIN` 的 `crt0_posix.o` 走 `xos_libc_main_entry`，最后进你写的 `main`。SDK 目录里还有 `manifest.json`，记录产物与配置，是验收比对的回执。

> 发行包另有一份带 `compile.py` 的 SDK（`scripts/package_release.py` 打包），本篇只用仓库里的 `build/sdk`。

### 最小编译命令（照抄 `tests/shell_launch_sl1.py`）

```sh
# 仓库根。先把 src/user/programs/hello.cpp 复制成 my_hello.cpp
x86_64-elf-g++ -std=c++20 -ffreestanding -nostdinc++ -fno-exceptions -fno-rtti \
  -fno-stack-protector -fno-threadsafe-statics -fno-use-cxa-atexit \
  -fno-unwind-tables -fno-asynchronous-unwind-tables -fno-omit-frame-pointer \
  -fno-pic -fno-pie -g3 -gdwarf-4 -O2 -Wall -Wextra -Werror -Wconversion -Wshadow \
  -m64 -march=x86-64 -mtune=generic -mcmodel=small -mno-red-zone -mgeneral-regs-only \
  -I build/sdk/include -c my_hello.cpp -o build/my_hello.o

x86_64-elf-g++ -m64 -march=x86-64 -mtune=generic -mcmodel=small -mno-red-zone -mgeneral-regs-only \
  -nostdlib -static -no-pie \
  -Wl,--build-id=none,--no-undefined,--fatal-warnings,-z,noexecstack,-z,max-page-size=4096 \
  -T build/sdk/user.ld build/sdk/crt0.o build/my_hello.o \
  -Wl,--start-group build/sdk/lib/libxos.a build/sdk/lib/libxos_loader.a -Wl,--end-group \
  -o build/my_hello.elf
```

两段的旗标组合就是 `scripts/toolchain.py` 里的 `COMMON_FLAGS` + `X86_FLAGS`，测试里原样使用。`hello.cpp` 只用 `xos::sdk` 接口，所以只链 `libxos.a`；要 newlib 的 `printf` 就再加 `-lc -lm -lgcc`（参考 `scripts/user_build.py` 里 `libc_hello` 的链法）。

## 第三步：把 ELF 放进系统

**方式 A：离线导入（推荐）**。先关掉占用工作盘的 QEMU：

```sh
# 仓库根
python3 scripts/work_disk.py work_disk.img mkdir /out          # 新盘才需要
python3 scripts/work_disk.py work_disk.img import /out/my_hello.elf build/my_hello.elf
make run
```

`import` 拒绝覆盖已存在的路径（报 `FatFs error=8`），重测换文件名或换盘副本。

**方式 B：在系统里现编**。发行盘带 `/usr/bin/x86_64-elf-gcc` 与 `/sysroot`，在 `xsh>` 里：

```text
edit /src/my_hello.cpp
exec-driver /usr/bin/x86_64-elf-g++ -static --sysroot=/sysroot /src/my_hello.cpp -o /out/my_hello.elf
exec-file /out/my_hello.elf
sync
```

编译器要启动 `cc1`/`as`/`ld`（构造子进程），所以必须用 `exec-driver`；普通程序用 `exec-file` 即可。细节见 `docs/usage/userland/in_guest_build.md`。

## 第四步：跑起来并看结果

```text
xos64> run
xsh> ls /out
xsh> exec-file /out/my_hello.elf
xsh> trace on
xsh> exec-file /out/my_hello.elf
xsh> sync
```

- `exec-file`：从工作盘装载静态 ELF 并等待结果；成功默认只打印程序输出。
- `trace on`：打开诊断，能看到 `[wait] pid=… kind=exited code=0` 这类行。
- `exec-driver`（别名 `exd`）：同 `exec-file`，额外授予构造直接子任务的权限；`spawn-file` 则是后台起、不等。
- 改完文件执行 `sync`，看到 `[files] sync=ok` 再关 QEMU，否则内容可能没落到盘上。

## 第五步：用 pthread 起线程

普通 `build/sdk` **不导出线程创建库**（`docs/implementation/threads.md`），线程程序要用独立的线程 SDK `build/threading_sdk`：

```sh
# 仓库根。需要显式准备 newlib/gcc 源包种子（SHA256 见 docs/implementation/threads.md）
make threading-newlib NEWLIB_ARCHIVE=/path/newlib-4.6.0.20260123.tar.gz
make threading-sdk        # 默认 GCC_SOURCE=.cache/xos_seed/threading_sources/gcc-16.2.0
python3 build/threading_sdk/compile.py my_thread.cpp -o build/my_thread.elf
```

产出的 ELF 再按第三、四步导入工作盘，`exec-file` 跑。线程 SDK 的 `compile.py` 与发行 SDK 的同名文件参数不同，别混用。

现成的验收样例：`make threading-probes` 生成 `build/threading_pthread_probe.elf`，`python3 tests/threading_pthread_smoke.py` 会在真实 guest 里反复 `exec-file /bin/pthread.elf <mode>`，检查 mutex/cond/once/TSS/TLS/stdio 与资源回收。

**边界**：不承诺完整 POSIX pthread；SMP 未验；没有异步取消、robust 锁、进程共享锁；C 的 realtime timed wait 返回 `ENOTSUP`。详见 `docs/implementation/threads.md` 的"边界与尚未运行"。

## 卡住怎么办

| 现象 | 原因与处理 |
|---|---|
| `Work disk missing; run 'make disk' once...` | 还没有工作盘，`make disk` 一次即可，已有盘不会被覆盖 |
| `Run make build first` | 缺镜像，先 `make build` |
| `Missing x86_64-elf-g++` / 版本不符 | 装交叉工具链或对齐 `config/toolchain.json`，见 `docs/toolchain.md` |
| 编译报 warning | 旗标里有 `-Werror`，告警即错误，这是仓库纪律；先改代码别删旗标 |
| `import` 报 `FatFs error=8` | 目标路径已存在，`import` 不覆盖 |
| `xsh: cannot … (error=N)` | 装载失败；`trace on` 看阶段与格式细节 |
| 用 `exec-file` 跑编译器失败 | 编译器需要构造子进程权限，改用 `exec-driver` |
| 程序跑不起来、只有静态 ELF | 只接受静态 ELF64/x86-64，动态链接与 `PT_INTERP` 会被拒绝 |
| 提示符搞混了 | `xos64>` 是内核控制台，`xsh>` 是 supervisor；`run` 后不回内核控制台，想回去只能重启 QEMU |
| 刚导入的文件找不到 | 先关 QEMU 再导入，`ls` 核对；构建成功不等于已安装 |

## 相关

- 最短路径与发行 SDK：`docs/usage/userland/getting_started.md`
- 机制与 ABI：`docs/e04_loader_sdk.md`、`docs/e08.md`、`docs/toolchain.md`
- 命令：`docs/usage/supervisor/exec-file.md`、`exec-driver.md`、`sync.md`、`trace.md`
- 线程：`docs/implementation/threads.md`
