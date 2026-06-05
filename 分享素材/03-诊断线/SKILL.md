---
name: log-diagnosis-large-ep-startup
description: 大 EP 场景启动失败/拉不起来的日志诊断。当 Controller 或 Coordinator 无法到达 MindIE-MS coordinator is ready，或长期 not ready 时，按 cmotor 启动五阶段定位；PD 分离卡在就绪前则下钻 mindie-llm 建链。触发词：大EP拉不起来、大EP启动失败、大EP启动卡住、coordinator not ready、coordinator is ready、MindIE-MS coordinator is not ready、大EP起不来。
parent: log-diagnosis-framework
---

# 大 EP 启动失败 — 日志诊断

## 权威参考文档（必读）

诊断前加载以下定位指南作为日志基线；本 skill 是**可执行的诊断流程**，细节以指南为准：

| 文档 | 路径 | 覆盖范围 |
|------|------|----------|
| 大 EP 场景启动 | `~/projects/doc/业务用/日志专项/日志整理/cmotor/大EP场景启动日志定位指南.md` | Controller + Coordinator 进程启动 → `ready!!!`（五阶段） |
| PD 实例建链 | `~/projects/doc/业务用/日志专项/日志整理/cmotor/PD实例建链日志定位指南.md` | mindie-llm 侧 KV 建链四阶段（步骤 4→5 下钻） |
| 日志收集 | `~/projects/doc/业务用/日志专项/日志整理/日志收集方法.md` | K8s 批量捞 Controller/Coordinator/节点日志 |

---

## 问题描述

**大 EP 启动**指 cmotor 大 EP（`pd_separate`）场景下，Controller 完成选主与集群初始化，向各 Coordinator EP 下发角色，Coordinator 完成实例刷新与（PD 场景）建链后打印 **`MindIE-MS coordinator is ready!!!`** 的全过程。

**涉及仓库/组件**：

| 侧 | 仓库 | 组件 |
|----|------|------|
| 控制面 | cmotor | Controller、NodeScheduler、RankTableLoader、ClusterClient、ServerRequestHandler |
| 协调面 | cmotor | Coordinator、ControllerListener、RequestRepeater、RequestListener |
| 建链（PD） | mindie-llm + cmotor | ControllerListener 层 add link；底层 SeparateDeploymentWorker / LLMDataDist |

**典型异常表现**：

- Controller 进程反复 `failed to initialize server cluster` 每 5s 重试
- Coordinator 长期只有 `MindIE-MS coordinator is not ready...`，无 `ready!!!`
- 有 `coordinator start successful` 但实例刷新/建链未完成
- 推理请求返回 `Coordinator is not ready`
- PD 场景：`Failed to add link with decode node` 或 mindie-llm 建链 ERROR

**关联问题**：

- 卡在 **步骤 4→5 且涉及 KV 建链** → 加载 `log-diagnosis-pd-link-establishment` 查 P/D 节点 mindie-llm 日志
- 缩 P 保 D 触发的恢复 → 先 `log-diagnosis-shrink-p-reserve-d`，再本 skill 或建链 skill

---

## 诊断入口日志

用户说「大 EP 拉不起来」时，先确认日志来源，再 grep 入口：

| 入口日志 | 侧 | 含义 | grep 命令 |
|----------|-----|------|-----------|
| `[Controller] Init controller failed` | Controller | 配置/Init 失败 | `grep "Init controller failed\|Run controller failed" $C_LOG` |
| `failed to initialize server cluster` | Controller | 集群初始化失败循环 | `grep "failed to initialize server cluster" $C_LOG` |
| `is not leader or ready, just wait` | Controller | 非 Leader 备节点 | `grep "is not leader or ready" $C_LOG` |
| `MindIE-MS coordinator is not ready` | Coordinator | 未就绪（最常见） | `grep "coordinator is not ready" $CO_LOG` |
| `[Start] MindIE-MS coordinator start successful` | Coordinator | 本地服务已起但未 ready | `grep "coordinator start successful" $CO_LOG` |
| `Failed to add link with decode node` | Coordinator | cmotor 层 PD 建连失败 | `grep "Failed to add link with decode node" $CO_LOG` |
| `instance update success` 无 `ready!!!` | Coordinator | 刷新成功但未满足就绪条件 | 见阶段 4→5 |
| `Link failed, error code is` | mindie-llm（P/D） | 底层 KV 建链失败 | `grep "Link failed\|Link exception" $LLM_LOG` |

变量约定（按 `日志收集方法.md` 收集后设置）：

```bash
C_LOG=/path/to/controller*.log      # Controller Pod/进程
CO_LOG=/path/to/coordinator*.log    # 目标 EP 的 Coordinator
LLM_LOG=/path/to/mindie-llm*.log    # 对应 P 或 D 节点引擎日志
```

---

## 总诊断决策树

```
[入口] 大 EP 未 ready / 拉不起来
│
├── Step 0：区分 Controller vs Coordinator
│   ├── Controller 无 leader Campaign / 反复 init cluster failed → 阶段 1–2（Controller）
│   ├── Coordinator 无 start successful → 阶段 1（Coordinator 参数/配置）
│   └── Coordinator 有 start successful + not ready → 阶段 3–5
│
├── 阶段 1：进程启动 + 选主
│   ├── Controller `is leader: 0` + wait → 【非故障】备节点，查 Leader 节点
│   ├── Init/Run failed → 【故障 1A】配置或权限
│   └── Coordinator `Invalid scheduler type` / `predict_ip invalid` → 【故障 1B】启动参数
│
├── 阶段 2：集群初始化
│   ├── `Parse distributed instance failed` → 【故障 2A】rank_table
│   ├── `RankTable Register failed` 重试 → 【故障 2B】ClusterD/gRPC
│   ├── 无 `Finished to initialize server cluster` → 【故障 2C】节点/DIGS
│   └── Coordinator 无 `Heartbeat producer started` → 【故障 2D】CO 本地 Init
│
├── 阶段 3：角色下发 + CO 本地启动
│   ├── 无 `Start sending role` → 回查阶段 2 或非 Leader
│   ├── 有 PostRole 无 CO `instance update` → 【故障 3A】URL/网络/CO 未监听
│   ├── `Some nodes' role are not ready` → 【故障 3B】部分节点 PostRole 失败
│   └── 无 `coordinator start successful` → 【故障 3C】CO 服务启动失败
│
├── 阶段 4：等待实例刷新
│   ├── 长期 not ready 无 `instance update` → 【故障 4A】Controller 未 PostRole/刷新未触发
│   ├── `Add instance failed` / label 错误 → 【故障 4B】刷新 body 或 pd_separate 标签
│   └── `Failed to add link with decode node` → 【故障 4C】D 节点/端口/网络（cmotor 层）
│
└── 阶段 4→5：有 instance update 无 ready!!!
    ├── pd_separate + 无 `Successfully add link with decode node` → 故障 4C 或下钻建链
    ├── 有 CO 层 link success，仍无 ready!!! → 【下钻 PD 建链】加载 log-diagnosis-pd-link-establishment
    └── 单机/无 PD 建链 → 【故障 5A】IsAvailable && dataReady 未满足（查 CO #CO43 条件）
```

**一步判断口诀**（与定位指南 §一 一致）：

1. Controller 只有 `is not leader or ready, just wait` → **非 Leader**，不是故障
2. 有 `Finished to initialize server cluster`，无 `Start sending role` → **阶段 2→3**
3. CO 有 `start successful`，长期 `not ready` 无 `instance update` → **阶段 4**（Controller PostRole）
4. 有 `instance update success`，无 `ready!!!` → **阶段 4→5**（PD 建链或 dataReady）
5. 出现 **`MindIE-MS coordinator is ready!!!`** → 该 EP 启动成功

---

## 诊断执行流程

### Step 0：定界 — 看哪份日志

| 检查项 | 命令 | 结论 |
|--------|------|------|
| 最后一条 Controller 标志 | `grep -E "leader Campaign finish\|Finished to initialize server cluster\|Start sending role\|Send role for all\|BatchLinkNodes" $C_LOG \| tail -5` | 对应大 EP 步骤 1–4（C 侧） |
| 最后一条 Coordinator 标志 | `grep -E "scheduler type\|Heartbeat producer started\|start successful\|not ready\|instance update\|add link with decode\|coordinator is ready!!!" $CO_LOG \| tail -10` | 对应大 EP 步骤 1–5（CO 侧） |
| deploy_mode | `grep -E "deploy mode\|pd_separate\|single_node" $C_LOG $CO_LOG` | 确认是否 PD 分离 |

记录：**最后出现的步骤编号 = 当前卡点**。

---

### 阶段 1：进程启动 + 选主

| 步骤 | 检查日志 | 预期 | 异常 → 根因 | 优先级 |
|------|----------|------|-------------|--------|
| 1.1 | `grep "Initializing controller, using deploy mode" $C_LOG` | 有 | 无 → Init 前即失败 | P0 |
| 1.2 | `grep "leader Campaign finish, is leader" $C_LOG` | `is leader: 1` 为 Leader | `is leader: 0` → **备节点等待**，查 etcd/Leader Pod | — |
| 1.3 | `grep "Init controller failed\|Run controller failed" $C_LOG` | 无 | 有 → 配置路径/JSON/权限 | P0 |
| 1.4 | `grep -E "scheduler type|Current deploy mode|predict_ip invalid|Incorrect usage" $CO_LOG` | scheduler + deploy OK | Invalid/invalid ip → 启动参数 | P0 |

---

### 阶段 2：集群初始化

| 步骤 | 检查日志 | 预期 | 异常 → 根因 | 优先级 |
|------|----------|------|-------------|--------|
| 2.1 | `grep "Parse distributed instance" $C_LOG` | success | failed → rank_table.json | P0 |
| 2.2 | `grep "number of available nodes is" $C_LOG` | >0 | 0 → 节点未注册/拓扑错 | P0 |
| 2.3 | `grep "DIGS role manager initialized" $C_LOG` | 有 prefill/decode rate | 无 → InitServerCluster 失败 | P0 |
| 2.4 | `grep "Finished to initialize server cluster" $C_LOG` | 有 | 无 + 5s 重试 → 查 #C27 前后 ERROR、节点存活 | P0 |
| 2.5 | `grep "RankTable Register failed\|register success" $C_LOG` | success | failed 重试 → ClusterD 地址/gRPC | P1 |
| 2.6 | `grep "Heartbeat producer started successfully" $CO_LOG` | 有 | failed → CO 本地组件 Init | P0 |

---

### 阶段 3：角色下发 + Coordinator 本地启动

| 步骤 | 检查日志 | 预期 | 异常 → 根因 | 优先级 |
|------|----------|------|-------------|--------|
| 3.1 | `grep "Role decisions start size\|Start sending role" $C_LOG` | size>0，有 Start sending | size=0 或无 Start → 阶段 2 未完成 | P0 |
| 3.2 | `grep "Posting single role" $C_LOG` | 有，且 CO 侧有对应请求 | CO 无日志 → manage_ip/端口/防火墙 | P0 |
| 3.3 | `grep "Send role for all prefill and decode nodes success" $C_LOG` | 有 | `Some nodes' role are not ready` → 逐节点 PostRole | P0 |
| 3.4 | `grep "BatchLinkNodes: CheckStatus" $C_LOG` | done, ready 计数合理 | not ready → PD 链路预检失败 | P1 |
| 3.5 | `grep "start manager server\|start successful" $CO_LOG` | start successful | Start * failed → CO 服务端口/权限 | P0 |

---

### 阶段 4：等待实例刷新

| 步骤 | 检查日志 | 预期 | 异常 → 根因 | 优先级 |
|------|----------|------|-------------|--------|
| 4.1 | `grep "coordinator is not ready" $CO_LOG \| wc -l` | 初期有，后续应减少 | 长期刷屏无 update → Controller #C32/#C33 未触发 | P0 |
| 4.2 | `grep "instance update success\|Add instance failed" $CO_LOG` | success | failed / JSON parse → 刷新 body、label | P0 |
| 4.3 | `grep "Invalid instance label in 'pd_separate' mode" $CO_LOG` | 无 | 有 → 实例 label 配置 | P0 |
| 4.4 | `grep "Successfully add link with decode node\|Failed to add link with decode node" $CO_LOG` | success（PD） | failed → D 进程/端口/网络 | P0 |

---

### 阶段 4→5：就绪前 PD 建链下钻

**触发条件**（满足任一）：

- CO 有 `instance update success`，无 `coordinator is ready!!!`
- CO 有 `Failed to add link with decode node`
- pd_separate 且 CO 层 link 已成功，仍 not ready

**动作**：

1. 在 **P/D 节点 mindie-llm 日志**（`$LLM_LOG`）按 PD 建链指南 §一 四步定位：
   - 步骤 1：`system init pd role success` / `Engine started and ready`
   - 步骤 2：`start to set PD link/unlink info`
   - 步骤 3：`Create all clusters kvcache links start` → `Link succeeded`
   - 步骤 4：`Setting role status to READY`
2. 加载并执行 **`log-diagnosis-pd-link-establishment`**，将其结论合并进本诊断输出。
3. 对照 CO 就绪条件（指南 D.4）：`IsAvailable() && dataReady`；mindie-llm READY 但 CO 仍 not ready → 查 ControllerListener 状态位与实例列表。

| mindie-llm 最后步骤 | 含义 | 下一步 |
|---------------------|------|--------|
| 无步骤 1 | P/D 引擎/角色未就绪 | 查 PostRole 是否到达该节点 Server |
| 步骤 2，无 Create clusters | 配置判定不需 link 或 Config 未触发 | 查 PD switch、link num、rank 拓扑 |
| 步骤 3 卡住 | 底层 link / 内存注册 | pd-link skill 阶段 C/D |
| 步骤 4 无 READY | 链路状态轮询 | pd-link skill 阶段 E/F |

---

### 阶段 5：确认成功

| 步骤 | 检查日志 | 预期 |
|------|----------|------|
| 5.1 | `grep "MindIE-MS coordinator is ready!!!" $CO_LOG` | **大 EP 该 EP 启动成功** |
| 5.2 | `grep "All nodes are available" $C_LOG` | Controller 侧可选确认 |

---

## 诊断输出格式

输出必须包含：

```markdown
## 诊断结论

**判断方向**：大 EP 启动失败 — 卡在阶段 N（Controller / Coordinator / mindie-llm 建链）
**当前卡点**：最后一条标志日志「…」（文件:行号或时间戳）
**可信度**：高/中/低（依据：是否有多条日志互证）

## 证据链

1. [时间] Controller: `...`
2. [时间] Coordinator: `...`
3. [时间] mindie-llm（若已下钻）: `...`

## 根因定位

（一句话根因 + 对应故障编号，如 故障 2A / 4C / pd-link C1）

## 修复建议

| 优先级 | 动作 |
|--------|------|
| P0 | ... |
| P1 | ... |

## 关联诊断

- 若已下钻建链：见 log-diagnosis-pd-link-establishment 输出摘要
- 参考文档：`大EP场景启动日志定位指南.md` §三 卡点速查；`PD实例建链日志定位指南.md` §三
```

---

## 快速诊断表

| 日志现象 | 大阶段 | 根因 | 修复方法 | 优先级 |
|----------|--------|------|----------|--------|
| `Init controller failed` | 1 | Controller 配置 | 查 config 路径、JSON、权限 | P0 |
| `is leader: 0` + `just wait` | 1 | 非 Leader 备节点 | 查 Leader Pod/etcd，**非故障** | — |
| `Parse distributed instance failed` | 2 | rank_table 错误 | 校验 JSON、IP、节点数 | P0 |
| `init server cluster` 5s 重试 | 2 | 节点/DIGS 未就绪 | 查节点上线、GRT、#C16/#C27 ERROR | P0 |
| `RankTable Register failed` | 2 | ClusterD 不可达 | ClusterD 地址、gRPC、网络 | P1 |
| 无 `Start sending role` | 3 | 阶段 2 未完成或非 Leader | 回查 2.4、1.2 | P0 |
| PostRole 有、CO 无 update | 3→4 | manage URL/防火墙 | 核对 predict_ip/manage_ip、端口 | P0 |
| 仅 `start successful` + `not ready` | 4 | 实例刷新未触发 | Controller PostRole #C32/#C33 | P0 |
| `Add instance failed` / label 无效 | 4 | 刷新请求/标签 | body JSON、pd_separate label | P0 |
| `Failed to add link with decode node` | 4（PD） | D 未就绪或网络 | D 进程、端口、防火墙 | P0 |
| `instance update success` 无 `ready!!!` | 4→5 | PD 建链或 dataReady | 下钻 mindie-llm + pd-link skill | P0 |
| `Link failed, error code`（llm 日志） | 4→5 | KV 底层建链 | pd-link 故障 C1/C2 | P0 |
| **`coordinator is ready!!!`** | 5 | 成功 | — | — |

**排查顺序**：`Controller #C4→#C5→#C24→#C32→#C36` → `Coordinator #CO34→#CO35→#CO38→#CO43` → 若 PD：`mindie-llm 建链步骤 1→4`

---

## 常用检查命令汇总

```bash
# 大 EP 五阶段 — Controller 侧一条线
grep -E "leader Campaign finish|Finished to initialize server cluster|Start sending role|Send role for all|BatchLinkNodes" "$C_LOG"

# 大 EP 五阶段 — Coordinator 侧一条线
grep -E "scheduler type|Heartbeat producer started|start successful|not ready|instance update|add link with decode|coordinator is ready!!!" "$CO_LOG"

# PD 建链快速入口（P/D 节点）
grep -E "start to set PD link|Create all clusters|Link succeeded|Link failed|Setting role status to READY" "$LLM_LOG"

# 对照 deploy_mode
grep -E "deploy mode|pd_separate" "$C_LOG" "$CO_LOG"
```

---

## 关联问题

| 关联问题 | 关联方向 | 对应 Skill |
|----------|----------|------------|
| PD 实例 KV 建链失败 | 下游（阶段 4→5） | `log-diagnosis-pd-link-establishment` |
| 缩 P 保 D 后 EP 起不来 | 上游 | `log-diagnosis-shrink-p-reserve-d` |
| 日志从哪捞 | 准备 | 见 `日志整理/日志收集方法.md` |

---

## 执行要点

1. **先读** `大EP场景启动日志定位指南.md` §一 快速定位表，确定步骤 1–5 卡在哪。
2. **Controller 与 Coordinator 分开看**；备节点 `not leader wait` 不要当故障。
3. **pd_separate** 且卡在 4→5 时，**必须**同时看 CO 日志 + P/D 的 mindie-llm 日志，并调度 `log-diagnosis-pd-link-establishment`。
4. 细节日志序号与 Mermaid 图见两份定位指南附录 C/D，本 skill 不重复展开。
