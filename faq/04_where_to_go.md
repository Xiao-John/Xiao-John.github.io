---
layout: default
---

> FAQ 4：从哪读起、怎么参与——新人的阅读路线、第一个任务、测试证据与协作约定。

## Q1：第一次来，按什么顺序读？

按仓库入口文档（`AGENTS.md`、`docs/README.md`「开始工作」）的顺序，五步走：

1. **`README.md`**——项目是什么、怎么构建、已知边界。
2. **`docs/project_overview.md`**——一段话总览；能力与证据统一指向 state，它不维护第二份通过清单。
3. **`docs/state.md`**——当前状态：基线、各线一句话状态、能力表、下一步方向。
4. **契约与宪法**——`docs/xos_plan.md` 第一部分（设计原则）、`docs/Discipline.md`（流程）、`docs/contracts/`（ABI 登记 `abi_registry.md` + 你改动涉及的专项契约）。
5. **`docs/work_log/README.md`**——编号索引；行级细节、证据、过程都在 work_log 里，state.md 只是快照。

两个补充：

- 动内核或装载器前，还要读 `docs/architecture_and_boot.md`、`docs/cpp_subset.md`、`docs/toolchain.md`；动用户态任务/SDK 前读 `docs/e04_loader_sdk.md` 和对应 supervisor 命令页。
- `docs/state.md` 只由合并者在整体变动时更新，**别随手改**；自己的活记忆写工作区根的 `STATE_L.md`（不入 git、不作证据）。

## Q2：先把系统跑起来（最快 10 分钟）

环境前提（见 `docs/usage/userland/getting_started.md`「环境前提」）：`python3`、`nasm`、`qemu-system-x86_64`，宿主编译还要 `x86_64-elf-gcc` / `x86_64-elf-g++` 在 `PATH`（或用 `XOS_CC` / `XOS_CXX` 指定）。工具链固定版本见 `docs/toolchain.md`。

```sh
# 以下均在仓库根目录执行
make disk    # 新建 2 GiB FAT32 工作盘（已有 work_disk.img 则不动）
make run     # 构建 install ISO（build + dash）并启动 QEMU
```

启动后在串口依次输入：

```text
xos64> run
xsh> exec-file hello.elf
```

看到 `hello from file` 就通了——`hello.elf` 是文件服务首次启动时自动写入工作盘的，不需要你编译。

**卡住怎么办**：

- 报 `Work disk missing; run 'make disk' once...` → 先 `make disk`。
- 缺 QEMU → `scripts/run_qemu.py` 会直接报错退出，装好 `qemu-system-x86_64` 再来。
- 找不到 `dash` → `make run` 的 install ISO 首次启动会自动把 dash 装到工作盘；新工作盘本身不预装（也可 `python3 scripts/install_dash.py --disk <盘>`）。
- `run` 是**单向**进入 supervisor 的，回不到 `xos64>`，想回来只能重启 QEMU。

回归门（改完代码至少过一遍相关的）：

```sh
make smoke   # 仓库根执行：串行跑既有回归 + 许可材料检查
```

## Q3：怎么找一个合适的入门任务？

按可信度从高到低翻这几处：

1. **`docs/state.md`「下一步」**——维护者当前排的活（设备授权段 1b、显示迁移、ps2/ata 等）。
2. **`docs/tech_debt.md`**——技术债总账，每条带**退出条件**（架构债/证据债/功能债），范围天然收敛。
3. **`docs/acceptance.md` 的 `not_run` 项**——明确写着"还没验"的判据，适合做小验证任务。
4. **`docs/README.md`「按任务阅读」里的 `docs/plans/`**——标着"尚未实现/未验"的计划。
5. **你关心那条线的最新 work_log 末尾**「下一步 / 接续」。

**选任务的标准**（Discipline「开始一组任务」）：

- 范围小、预期证据写得清。开工前先读 `state.md`、宪法、ABI 登记与专项契约，确认"哪些已完成、哪些只是建议、哪些尚未验证"，然后**声明本组范围和预期证据**；能按已确认路线做出的局部可逆选择直接推进并记录。
- 多人协作时避免同时改同一个文件；`build/` 是共享目录，不并发构建或清理。
- 拿不准就先问，"先不写"时只讨论、不擅自实施。

## Q4：测试和证据怎么交？

- **跑测试**：仓库根执行 `python3 tests/<名字>.py`；`make smoke` 就是把一批串起来跑。客体测试在 QEMU 里跑，依赖 `make build` 的产物。按改动跑相关探针即可，不因文档改动重跑旧全量（`docs/toolchain.md`）。
- **结果只有四态**：`pass / partial / fail / not_run`。没执行的检查标"未运行"，不以跳过冒充通过（Discipline）。
- **报告要素**（`docs/acceptance.md`）：源码/SDK/镜像/夹具摘要、工具与环境、步骤、结果、资源基线/终值、未运行项。
- **新报告目录**按[证据完整性契约](https://github.com/Xiao-John/XylophoneOS/blob/main/docs/contracts/evidence_integrity.md)用平铺根清单 `manifest.json`（`{"version":1,"files":{路径:sha256}}`）；`python3 tests/report_manifest_check.py` 会如实报 compliant / grandfathered。
- **提交前扫描**（Discipline 明文命令）：

```sh
# 仓库根执行；扫暂存的新增/修改文件里的个人路径与机器专属命令路径
python3 tests/check_public_evidence_paths.py --index --base origin/main

# 发布证据前核对文件哈希
python3 scripts/verify_evidence_index.py "${XOS_EVIDENCE_INDEX}"
```

- **结论落盘**：关键结论、哈希、失败原因写进持久文档（work_log），别只留在 `build/` 日志里——清理产物后不能"交接失忆"。

## Q5：分支、PR、工作日志有哪些约定？

- **分支 + PR**（C19 / Discipline「Git交付与贡献归属」）：2026-10-27 起新开发一律走独立分支和 PR，**不直接提交/推送到 main**；仓库当前约定新开发分支默认用 `codex/` 前缀。保留真实作者，不把他人实现重新署为自己；合并方式与共同作者归属由维护者 review 时确定。
- **工作日志**：每组完成时新增 `docs/work_log/NNNN_主题.md`，至少写清五件事（Discipline）：
  1. 任务范围、已完成行为和修改文件；
  2. 本组选择与原因（原则修订进宪法、接口进契约、事实进登记）；
  3. 实际执行的命令、工具版本、配置、产物标识与证据位置；
  4. 通过 / 失败 / 未运行的检查，已知限制和未完成内容；
  5. 下一位可直接执行的下一步，以及**不能误认成已实现**的功能。
- **编号规则**：以 `docs/work_log/README.md` 索引为准。历史上有重复编号；现行规则是自 0188 起全局唯一，新记录**自 0223 起取号、不得复用**，取号后在索引里登记一行。
- **别改冻结件**：`docs/reports/`、`docs/releases/` 冻结后不移动、不改写；`state.md` 只由合并者更新，行级细节进 work_log。

## 卡住怎么办？

- 命令不知道用法 → `docs/usage/index.md` 命令手册（内核控制台 / supervisor / 实模式逐条有说明）。
- 想写用户态程序 → `docs/friend_user_program_quickstart.md` 与 `docs/usage/userland/getting_started.md`（从零 hello 的最短路径）。
- 不确定某功能是否已验收 → 别猜，查 `docs/state.md` 能力表 + `docs/acceptance.md` + 对应 work_log；文档与实现冲突时按 Discipline 核源码与证据，修实现或更新登记，互相不替代。
