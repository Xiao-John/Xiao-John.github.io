---
layout: default
---

# 篇 5 · 文件、终端、显示与音频

导语：这一篇带你把 XOS 的"日常使用"跑一遍——往工作盘放文件、用终端和编辑器、打开图形窗口看双窗口演示、试探单音服务。每节都写清"做什么、看到什么、卡住怎么办"，命令全部来自仓库里的 `Makefile`、`scripts/`、`tests/`，深挖留给仓库文档。

## 0. 先把系统跑起来（仓库根）

```sh
make disk     # 新建工作盘：默认 2 GiB FAT32；已存在则不改动
make run      # = make install-iso + python3 scripts/run_qemu.py --install-iso
```

看到什么：串口输出直接打在当前终端，出现 `xos64> `（内核控制台，COM1 轮询输入）。接着：

```text
xos64> run
xsh>
```

- `run` 单向进入 supervisor，回不到 `xos64>`；想重来就 Ctrl-C 关 QEMU 再 `make run`。
- 缺盘报 `Work disk missing; run 'make disk' once`；缺镜像报 `Run make build first`。
- 进 `xsh>` 后约 1 秒无输入，旧演示任务会自动停；想留着用 `demo keep`，想立刻停用 `demo quiet`。

## 1. 工作盘与 FAT 文件

### 宿主侧：造盘与搬文件（仓库根，先关掉用这张盘的 QEMU）

```sh
python3 scripts/make_work_disk.py                 # 默认 2 GiB FAT32，等价于 make disk
python3 scripts/make_work_disk.py --size-mib 16   # 旧 16 MiB FAT16 兼容路径
python3 scripts/work_disk.py work_disk.img mkdir /out
python3 scripts/work_disk.py work_disk.img import /out/hello.sh hello.sh
python3 scripts/work_disk.py work_disk.img export /out/hello.sh backup.sh
```

`work_disk.py` 的真实用法：`image {mkdir,import,export} path [source]`。`path` 必须是客体绝对路径、不含 `.`/`..`、不能用反斜杠；**`import` 拒绝覆盖**，重复导入报 `FatFs error=8`（换个文件名或新盘副本）。首次执行会用 `clang++`/`c++` 把 `scripts/work_disk_host.cpp` 编到 `build/work_disk_host`。

### 客体内：读写与落盘（`xsh>`）

```text
xsh> ls /out
xsh> cat /out/hello.sh
xsh> mkdir /src
xsh> mv /out/hello.sh /src/hello.sh
xsh> edit /src/hello.sh
xsh> sync
```

看到什么：改完必须 `sync`，看到 `[files] sync=ok` 才能关 QEMU，否则这次修改不算保存。`edit` 是行式编辑器：正文行直接输入，`:p` 打印、`:i N` 插入、`:d` 删除、`:wq` 保存退出、`:q!` 放弃退出；载入上限 8192 字节、单行 159 字节、路径限 ASCII。

事实与边界（`docs/usage/index.md`、`docs/work_log/0068_utf8_filenames.md`）：支持目录与长文件名，中文文件名有专项客体验证；路径最多 255 字节，不做 Unicode 规范化，大小写只折叠 ASCII；每会话 16 个打开、全局 32 个对象锁；FatFs 8.3 短名显示成大写（如 `HELLO.ELF`）。没有权限模型、时间戳和 fsck。

### 测试里怎么用 `create_fat32` / `transfer`（Python，仓库根）

```python
from make_work_disk import create_fat32   # scripts/make_work_disk.py
from work_disk import transfer            # scripts/work_disk.py

create_fat32(disk, 512)                        # 稀疏 FAT32 测试盘，支持 512..2048 MiB
transfer(disk, "mkdir", "/bin")
transfer(disk, "import", "/bin/probe.elf", probe)   # 导入宿主文件，拒绝覆盖
transfer(disk, "export", "/out/report.json", host)  # 导出到宿主
```

这是 `tests/file_posix_guest.py`、`tests/adventure_smoke.py`、`tests/interactium_guest.py` 等几十个客体测试的通用起手式：造临时盘 → 铺目录和 ELF → 让 QEMU 从这张盘启动。卡住时先确认盘没被 QEMU 占着（`transfer` 是离线操作）。

## 2. 终端与编辑器现状

- **两个提示符**：`xos64> ` 是内核控制台，`xsh>` 是 supervisor（原生 Xsh）。命令逐条见 `docs/usage/index.md` 的 Xsh 命令索引。
- **xsh 是按空格分词的行命令**：不支持引号、管道、重定向；脚本要显式交给 dash。
- **dash 是可选 shell**（install ISO 首启自动装到 `/usr/bin/dash`，`make disk` 的新盘不预装）：

```text
xsh> dash -i
$ printf 'one\ntwo\n' | cat          # 前台流水线最多 4 段，第 5 段拒绝（退出 2）
$ exit                               # 回到 xsh>
xsh> dash /out/hello.sh              # 跑脚本；exec-file 不会把文件当脚本执行
```

- **全屏/宽字符程序走终端服务**：`tty-run <程序>` 首次会启动 `/bin/terminal.elf`，为声明 `.xos.tty` 的程序接上固定 xterm/80×24 会话，跑完回到提示符。ncursesw 宽字符底座已通过 W1 最终验收（`docs/work_log/0187_w1_final_acceptance.md`，含 GNU Screen 24×80 附着与 CJK 断言）。
- **编辑器**：目前只有 `edit` 这个行编辑器；全屏编辑器（nano 优先评估）仍在选型阶段，尚未移植（`docs/plans/terminal_editor.md`，状态 P）。别去找 vim/emacs。
- **卡住怎么办**：前台等待时单独按 Escape 取消被等待任务；Ctrl-C 关的是宿主 QEMU；`trace on` 打开进程启动/等待/回收诊断，`trace off` 恢复简洁；装不上程序先 `type <命令>` 看 PATH 解析到了哪。

## 3. 显示：QPA + interactium 演示

### 打开一个能看见的窗口（仓库根）

`make run` 默认 `-display none`（只看串口）。要 GUI 窗口：

```sh
python3 scripts/run_qemu.py --install-iso --display sdl
# 或 make run_vga：build/xos-install.iso + SDL 窗口（Makefile 里是 --iso-file --vga）
```

`scripts/run_qemu.py` 的 `--display` 接受 `sdl`/`gtk`/`none` 等 QEMU 后端，与 `--install-iso` 互不影响。

### 玩法 A：完整窗口管理器（`xsh>`）

```text
xsh> interactium-start
```

supervisor 会先起 Display 服务并授予服务角色，再起用户态 `interactiumd`。它占满 Display v1 唯一的客户端槽，之后宿主交叉编译的 Qt 程序（QPA）都经它开窗、合成、收输入。

### 玩法 B：双窗口模型演示（不带 Qt）

```text
xsh> demo quiet
xsh> display-start
xsh> exec interactium
```

看到什么：QEMU 窗口里出现两个演示窗口；点击窗口提升到顶层、拖标题栏移动；按 **Q 或 Escape** 回到 `xsh>`。`display-start`、`gfxdemo`、`interactiumd` 不能同时跑——它们抢同一个 Display 客户端槽，第二个会失败。

### 玩法 C：Qt Widgets 样例与自动验证

- Qt 应用**在宿主编译**（客体内没有 Qt），用 `src/ports/qt6/build_widgets_fixture.py` 按当前 profile 编出 ELF，导入工作盘为 `/AURORA.ELF`，`interactium-start` 后 `exec /AURORA.ELF`，Escape 退出（`docs/contracts/interactium.md`）。
- 仓库根一条命令的自动验证：`python3 tests/interactium_guest.py`（默认吃 `build/xos-install.iso`，自己造临时 FAT 盘、用 QMP 注入拖动输入，截图写进 `build/`）。

边界：每会话最多 8 个窗口、服务端 8 个像素 surface、CPU source-over 合成、800×600、无 vblank/撕裂保证；QML/Quick 未实现。细节见 `docs/contracts/interactium.md` 与 `docs/implementation/display.md`。

## 4. 音频：PC Speaker 现状（含诚实说明）

Shell 入口已经写进 supervisor（`src/user/programs/supervisor.cpp` 的 help 里能看到 Audio 一组）：

```text
xsh> tone 440 250                 # 异步提交单音，返回 job ID
xsh> melody /out/tune.txt         # 整文件预校验后播放
xsh> audio status <job>
xsh> audio cancel <job>
xsh> audio close
xsh> audio start                  # 启动用户态音频服务
xsh> audio restart                # 杀掉重启，旧 session/job 全失效
```

`melody` 文件格式（`docs/contracts/audio_service.md`）：纯文本，每行两个十进制整数 `frequency_hz duration_ms`，`0 duration_ms` 是休止；1–64 个音、40–5000 Hz、每项 10–2000 ms、总长 ≤10 秒、文件 ≤2048 字节，空行/注释/多余字段一律拒绝。

**卡住与现状，务必如实预期**：

- AU0 已用 QEMU 探针证明 PC Speaker→WAV 的路由（约 5.01 kHz 信号被捕获，`docs/reports/audio_output_au0_20261109/`）。
- AU1（2 秒 PIO 租约、强制静音）与 AU2（用户态单音服务、`tone`/`melody`、服务恢复）**源码已合入**，但当时的正式客体构建因环境缺少登记的 x86_64-elf 工具链没能跑起来：**客体 build、WAV 捕获、人工试听都是"未运行"，AU1/AU2 不能算客体验收通过**（`docs/work_log/0221`、`0222`，`docs/plans/audio_output.md`）。所以"`tone` 敲进去一定出声"目前不作承诺。
- `make run` 走的 `run_qemu.py` 没有配 QEMU 音频后端；计划里候选的捕获片段是 `-audiodev wav,id=xos_audio,path=<capture.wav>` 加 `-machine pc-i440fx-10.0,pcspk-audiodev=xos_audio`，AU0 探针验证过，但还不是正式启动选项。
- 能力边界：只有方波单音；无音量、混音、WAV 播放、PCM（AU3–AU5 未开始）。

## 5. 深挖去哪

| 想看什么 | 文档 |
|---|---|
| 命令总索引、Xsh/内核控制台逐条说明 | `docs/usage/index.md` |
| 工作盘、FAT、保存与恢复机制 | `docs/e07_persistent_work_disk.md` |
| dash 能力边界（C1–C3 已验收/明确拒绝） | `docs/usage/userland/dash.md` |
| 从零写 hello、编译、导入、运行 | `docs/usage/userland/getting_started.md` |
| 显示、QPA、权限与生命周期 | `docs/implementation/display.md`、`docs/contracts/interactium.md` |
| 音频分期、验收门槛与限制 | `docs/plans/audio_output.md`、`docs/work_log/0220`–`0222` |
| 全局能力与边界快照 | `docs/state.md`、`README.md` 的"已知边界" |
