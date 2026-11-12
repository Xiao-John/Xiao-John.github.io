---
layout: default
---

# XOS 中文入门（FAQ + 指南）

面向**零基础小白**与**用户态开发者**的独立写作项目——与仓库 `docs/`（规范文档）互不隶属，
只做"带你跑起来、玩明白"这件事。事实以 `origin/main` 为准（校对轮 2026-11-12）。

## 指南（ Guides，按顺序读 ）

| 篇 | 文件 | 你会得到 |
|---|---|---|
| 1 | [十分钟跑起 XOS](guides/01_quick_start.md) | 装依赖、`make run`、在串口里第一次敲命令 |
| 2 | [shell 导览](guides/02_shell_tour.md) | 提示符、supervisor 命令、dash 与管道小实验 |
| 3 | [写第一个用户态程序](guides/03_first_program.md) | 编译链接三要素、放盘、`exec-file`、pthread |
| 4 | [IPC 入门](guides/04_ipc_guide.md) | endpoint 收发三步、通知/等待集、特性位 0x0f |
| 5 | [文件、终端、显示与音频](guides/05_io_and_media.md) | 工作盘玩法、interactium demo、PC Speaker 现状 |

## FAQ（随便翻）

| 问题 | 文件 |
|---|---|
| XOS 是什么、不是什么？ | [faq/01](faq/01_what_is_xos.md) |
| **XOS 现在能做什么？** | [faq/02](faq/02_capabilities.md) |
| 构建/测试卡住了怎么办？ | [faq/03](faq/03_build_test_faq.md) |
| 从哪读起、怎么参与？ | [faq/04](faq/04_where_to_go.md) |

## 约定

- 命令均标注执行位置（仓库根 / `xos64>` / `xsh>` / `$`）；
- 能力状态如实标注（"已验收 / 待验收 / 尚不支持"）；
- 深入细节请回 `docs/`：契约 `docs/contracts/`、状态 `docs/state.md`、工作日志 `docs/work_log/`。
