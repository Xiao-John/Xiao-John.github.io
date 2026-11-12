---
layout: default
---

# 篇 4 · IPC 入门

> 导语：XOS 的服务都在用户态，进程之间怎么说话？这一篇讲清"为什么需要 IPC"、endpoint 的 call/receive/reply 最小套路、notification 与 wait_set 各管一件事，以及怎么查"这个系统到底支持什么"。

## FAQ：为什么需要 IPC？

**Q：进程各跑各的，为什么要通信？**
因为文件、显示、终端、音频这些服务都是独立进程。你要读文件，就得跨进程问 `files`；服务崩了不能把内核带下水，所以隔离是前提，通信就必须有明确机制。

**Q：为什么不能直接用老的 send/recv？**
老的 `sys4/sys5` 单槽异步通道只保证"消息入槽"，不保证对端收到、更不保证业务完成。契约 `docs/contracts/ipc_v1.md` 明确要求：不能把单槽队列伪装成同步握手。IPC v1 提供的是**同步 RPC 语义**——`call` 拿到的 reply 才算数。

**Q：内核管多少？**
按契约"目标与约束"：服务发现、协议、重试、故障恢复全在用户态；内核只做认证、隔离、受界消息、等待与资源生命周期。

## 最小例子的思路：endpoint call / receive / reply

结构照 `tests/fixtures/ipc_v1/ipc_v1_probe.cpp`（361 行，含并发与竞态检查），思路三步：

1. **建端点**：`ipc_v1::endpoint_create(rights, handle)` 拿到 handle。权利位是 `call=0x1`、`receive=0x2`、`manage=0x4`、`delegate=0x8`，创建者拿非零子集，转授只能减少。
2. **服务端**：线程里循环 `ipc_v1::receive(...)` —— 收到消息、认证 badge，以及一个**一次性 reply_token**；回包用 `ipc_v1::reply(token, ...)`，token 用一次就作废（`stale_token`）。
3. **客户端**：`ipc_v1::call(...)` 同步等 reply，返回后看 `phase_out == replied` 才算成功；超时时 phase 可能停在 `delivered`，**业务结果未知，不能自动重试**。

配套约束（都在契约里）：payload 0–256 字节（label 不计入）；每进程 8 个 handle；系统 32 个 endpoint、32 个排队 call、每个 endpoint 8 个排队 call；`deadline_ns` 是单调绝对纳秒，`0` 只做立即配对（没配对就是 `would_block`），`UINT64_MAX` 表示无限等。

### 跑一遍现成探针

```sh
# 仓库根
make build                    # 产出 build/xos-install.iso 与 build/manifest.json
make threading-sdk            # 探针用线程 SDK；需 newlib/gcc 源包种子，见 docs/implementation/threads.md
python3 tests/ipc_v1_guest.py --system build --output build/ipc_v1_out --runs 1
```

预期看到 `[ipc-v1] race-resource … PASS` 和 `kind=exited code=0`，报告写在 `build/ipc_v1_out/report.json`。`--output` 目录必须还不存在（脚本自己建）；探针会自建临时 FAT 工作盘、把 ELF 放进 `/bin` 再启动 guest，覆盖并发 RPC、deadline/reply 竞态、close 唤醒、handle 配额恢复、notification 合并与 wait-set 就绪，且要求资源计数回到基线。

> **前置提醒**：IPC v1 的契约、头文件与探针（`docs/contracts/ipc_v1.md`、`src/user/include/xos/ipc_v1.hpp`、`tests/ipc_v1_guest.py`）随 PR #103/#107/#109/#111/#114 合入 main。

## notification 与 wait_set：各一句话

- **notification**：无 payload 的"信号灯"——多个 `signal` 按 badge 位 OR 合并，一次 `wait` 原子取走当前位集合。适合"发生了什么"（数据就绪、该醒了），**不计数、不保证每次事件都能观察到**。
- **wait_set**：把多个 endpoint / notification 挂进一个集合，一次等"任一就绪"。适合多路复用的事件循环；首版每个 wait set 最多 8 个 source，且只支持 endpoint 与 notification（IRQ、进程退出、旧 `wait_many` 不能加进来）。

对象上限：系统各 32 个 notification 与 wait set；handle 仍占每进程 8 个总额度。

## 特性位 0x0f 与"如何查询系统支持什么"

feature 位是分组开关，互不捆绑：

| 位 | 分组 | 当前状态 |
|---|---|---|
| `0x1` | `endpoint_call_v1` | supported |
| `0x2` | `deadline_v1` | supported |
| `0x4` | `notification_v1` | supported |
| `0x8` | `wait_set_v1` | supported |
| `0x10` | `bulk_grant_v1` | **关闭**，待 IP4 客体验收 |

known mask 固定 `0x1f`，当前 supported mask 是 `0x0f`。短消息直切快路径已实现（PR #114，限非 poll、请求与回复都不超过 64 字节的 call），但契约注明其性能/语义探针仍须由维护者另行运行——别把快路径当成已验收的性能结论。

**三层查法，从活的到死的：**

1. **运行时**：`ipc_v1::query_features(query)`（sys88 的 action 0）返回 `known_features` / `supported_features` 和固定资源上限；`ipc_v1::supports(feature, query)` 直接判一位。探针 `main()` 第一件事就是查它，查不到就 `feature-gate` 失败。
2. **SDK manifest**：`build/sdk/manifest.json` 与 `build/threading_sdk/manifest.json` 里的 `ipc_v1` 条目（main 上由 `scripts/ipc_v1_manifest.py` 统一生成，两份内容一致），只列**已过客体验收**的 feature。同文件里其他扩展也带 `status`（如 `implemented_unqualified`、`claim_unsupported`），照实读。
3. **文档登记**：`docs/contracts/ipc_v1.md`（能力边界与限额）、`docs/contracts/abi_registry.md`（syscall 88 的 allocated 登记）、`docs/state.md`（各线验收状态）。`allocated` 不等于"已验收"。

## 卡住怎么办

| 现象 | 原因与处理 |
|---|---|
| 返回 `-104 unsupported` | 操作 / feature / ABI 版本没实现；先 `query_features`，别猜 |
| `-105 would_block` | deadline=0 的轮询没配到对端；改成有限期限或无限等 |
| `-109 timeout` 且 phase 是 `delivered` | 已交付后超时，结果未知，**不要自动重试**，按协议做幂等键 |
| `-110 buffer_too_small` | 缓冲不足：会报出所需长度且**不消费消息**，扩容后重试 |
| `-111 stale_token` | reply token 已用过 / 已失效；token 不可转授、不可伪造 |
| `-107 resources` | handle、endpoint、排队 call 或 reply token 配额用尽；先 close 再建 |
| 想用 `bulk_grant` 共享大块内存 | 暂不支持，等 IP4 验收；小消息也不承诺自动零拷贝 |
| 旧程序行为变了 | 没变：`sys4/5`、`sys81`、`sys84–86` 与旧通道语义保持，按契约"兼容与迁移"并行迁移 |

## 相关

- 契约（必读）：`docs/contracts/ipc_v1.md`；路线：`docs/plans/ipc_sel4.md`
- 探针：`tests/ipc_v1_guest.py`、`tests/ipc_v1_fastpath_guest.py`、`tests/ipc_v1_reference_model.py`
- 共享 ABI 头：`src/shared/include/xos/ipc_v1_abi.hpp`；SDK 薄封装：`src/user/include/xos/ipc_v1.hpp`
- 登记与状态：`docs/contracts/abi_registry.md`、`docs/state.md`；线程背景见 `docs/implementation/threads.md`
