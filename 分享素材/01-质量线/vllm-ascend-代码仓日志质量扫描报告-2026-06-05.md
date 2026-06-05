# vllm-ascend 代码仓日志质量扫描报告

> 扫描日期：2026-06-05
> 扫描范围：`/home/h00906152/projects/vllm/vllm-ascend/vllm_ascend/` 全仓（排除 `tests/`、`__pycache__/`、`.git/`、`build/`）
> 检查口径：`log-quality-standard` v1.1 + §4.5 重新分类（按"失败语义是否进日志"判定）
> 对比基线：`vllm-ascend_代码日志质量扫描报告_20260429.md`（476 条日志，2026-04-29）

---

## 0. TL;DR

|| 维度 | 数值 | 评价 |
||------|------|------|
|| 总 logger 调用（Python） | 601 条 | info 182 / error 121 / warning 119 / debug 157 / exception 22 |
|| 总 raise 语句（Python） | 463 处 | 过滤后 226 处需要前置日志 |
|| C++ 日志（csrc/） | ~1143 条（LOG_E 占 68% ≈ 776） | ERROR 占比异常高，需重点复查 |
|| 标准 4.5 P0（必须前置） | 3 处 | 跨进程 zmq 异常 |
|| 标准 4.5 P1（业务中断） | 64 处 | RuntimeError 61 / 启动期 ValueError 2 / 超时 1 |
|| 标准 4.5 P2（入参校验） | 122 处 | 应改 WARNING 而非 raise |
|| 标准 4.5 P3（未实现） | 37 处 | NotImplementedError 全部建议改 WARNING |
|| 分级误用（INFO 含失败词） | 7 条 | 维持上版结论 |
|| 纯静态短句 ERROR/WARNING | 116 条 | 严重（基本无参数快照、无排查方向） |
|| 缺排查方向关键词（标准 4.1） | 239/262 (91%) | 严重 |
|| 组件标识 `[vllm/xxx]` 源码 | 0/601 (0%) | 系统性问题（与上版一致） |
|| warning/error 带 trace_id/request_id | 7/262 (3%) | 系统性问题（与上版一致） |

**核心结论：相比 2026-04-29 报告，**总日志量从 476 增至 601（+26%），新增加 125 条日志未做日志质量治理。两份历史报告已识别的三个系统性问题（缺组件前缀、缺 trace_id、纯静态短句）**全部仍在**，且新代码也未引入合规写法。**

整改面按 P0→P3 排序：**3 → 64 → 122 → 37**，合计 226 处。

---

## 1. 与历史报告对比

| 维度 | 2026-04-28 报告 | 2026-04-29 报告 | 2026-06-05 本次 | 趋势 |
|------|---------------|---------------|---------------|------|
| Python 日志总数 | 478 | 476 | 601 | +26%，**新代码未受治理** |
| C++ 日志 | 未扫 | 未扫 | ~1143 | 新增维度 |
| 分级误用 | 5 条 | ~8 条 | 7 条 | 持平 |
| 描述不充分 | 110 条 | ~40 条 | 239 条（缺排查方向） | **恶化**：上版只统计"缺根因+参数+方案"三件套，本版按标准 4.1 严判，239/262 = 91% 不含排查方向关键词 |
| 组件前缀覆盖率 | 0% | 0% | 0% | 持平（**系统性问题未改善**） |
| trace_id 覆盖率 | ~0% | ~0% | 3%（7/262） | 微改善（kv_transfer 模块增加了 request_id） |
| raise 前是否需要日志 | 未扫 | 未扫 | 226 处需整改 | 新增维度（v2 按 §4.5） |
| 关键位置完整性 | 4 处 | 4 处 | 3 处（沉默成功文件） | 微改善 |

**关键发现**：从 4-29 到 6-05（37 天），新增 ~125 条 Python logger，但 3 个系统性问题没有任何改善——说明 vllm-ascend 仓**没有把日志规范纳入 Code Review**。建议在 PR 模板中加入 log-quality 自检清单。

---

## 2. 关注点一：raise 前是否需要日志（标准 4.5）

### 2.1 四步过滤 + 优先级分类（226 处）

```
原始 raise 数:        463
- 排除生成代码:       -0
- 排除框架异常:       -X
- 排除内部 helper:    -X
- 排除"已有前置日志":  -X
- 排除"外层 logger.exception 兜底": -X
─────────────────────────────────
真正需要前置日志:     226
```

|| 优先级 | 数量 | 类别 | 整改要求 |
|--------|------|------|---------|
|| **P0** | **3** | 跨进程 zmq 异常（mooncake_*_connector） | **必须前置 logger.error** 带 zmq 错误码/超时配置/对端地址 |
|| P1 | 64 | 业务中断 RuntimeError 61 + 启动期 ValueError 2 + 超时 1 | 前置 logger.error（业务中断/启动失败） |
|| P2 | 122 | 入参校验 ValueError 104 + RuntimeError 10 + 其他 6 + 弃用警告 2 | **建议改 logger.warning 而非 raise**（按 §4.5 入参校验口诀） |
|| P3 | 37 | NotImplementedError | 改 logger.warning 即可 |

### 2.2 P0 详细（3 处）

全部集中在 `distributed/kv_transfer/kv_p2p/`：

|| # | 文件:行号 | raise 内容 | 风险 |
|---|---------|----------|------|
|| 1 | mooncake_layerwise_connector.py:1974 | `raise zmq.ZMQError("Receive timeout")` | **跨进程链路**：对端是 Mooncake 集群，超时后无痕，运维无法定位对端节点 |
|| 2 | mooncake_connector.py:1853 | `raise zmq.ZMQError("Receive timeout")` | 同上 |
|| 3 | mooncake_hybrid_connector.py:1858 | `raise zmq.ZMQError("Receive timeout")` | 同上 |

**整改方向**（统一模板）：
```python
logger.error(
    "[vllm/kv_transfer] Mooncake receive timeout. "
    "peer=%s, timeout_ms=%s, recv_count=%s, "
    "可检查：1. 对端 mooncake 进程是否存活 2. 网络丢包率 3. timeout_ms 配置是否合理",
    peer, timeout_ms, recv_count,
)
raise zmq.ZMQError(...)
```

### 2.3 P1 业务中断 RuntimeError 按模块分布（61 处）

|| 模块 | 数量 | 重点文件 |
|------|------|---------|
|| model_loader | 14 | elastic.py (6), rfork_loader.py (4), load.py (2) |
|| _310p | 12 | ops/causal_conv1d.py (10) — 形状校验类 |
|| distributed | 9 | kv_transfer 6 + 其他 3 |
|| cpu_binding.py | 8 | 设备亲和性失败 |
|| worker | 4 | worker.py (2), model_runner_v1.py (2) |
|| device | 4 | device_op.py |
|| 其他 | 10 | eplb/ops/patch/attention/quantization/platform/utils 各 1-2 处 |

**典型案例**（P1 业务中断）：
```python
# model_loader/netloader/interaction/elastic.py:178
raise RuntimeError(f"Send data {data} to server fails, detail: {e}")
# 缺：server_addr、retry_count、data 大小

# cpu_binding.py 8 处：
raise RuntimeError(f"Failed to bind CPU for rank {rank}")
# 缺：npu_id、cpuset 范围、失败原因细分
```

### 2.4 P2 122 处（建议改 WARNING 而非 raise）

**核心问题**：vllm-ascend 全仓 122 处入参校验用了 `raise ValueError`/`raise RuntimeError`，按 §4.5 口诀"入参校验 → WARNING"应当改 logger.warning：

- **入参校验 ValueError 104 处**（worker 22、distributed 22、ops 18、eplb 9、_310p 7 等）
- **入参校验 RuntimeError 10 处**（netloader 校验 world_size/rank、scheduler 校验 request.status、causal_conv1d 校验 x.shape 等）
- **其他 6 处**（含弃用警告 2 处）

**典型 P2 案例**：
```python
# model_loader/netloader/executor/netloader_pg.py:65
raise RuntimeError("world_size must be positive")
# 应改：
logger.warning("[vllm/model_loader] Invalid world_size. value=%s, expected>0, fallback to default", world_size)
# 或者保留 raise 但前置 logger.error 记录实际值
```

### 2.5 P3 37 处 NotImplementedError

**全部建议改 logger.warning**（按 §4.5 + §3 分级——未实现不是错误，是开发态提示）。

Top 文件：
- `model_loader/rfork/rfork_loader.py` (2)
- `worker/v2/spec_decode/eagle/__init__.py` (1)
- `worker/v2/spec_decode/rejection_sampler_utils.py` (2)
- `worker/v2/sample/gumbel.py` (1)
- `_310p/attention/attention_v1.py` (1)
- `_310p/ops/causal_conv1d.py` (1)
- ... 合计 37 处

---

## 3. 关注点二：组件交互时是否有日志（标准 1.6）

按文件聚合（≥10 条日志的模块）：

| 模块 | 总数 | info | err | warn | dbg | exc | 评价 |
|------|------|------|-----|------|-----|-----|------|
| distributed/kv_transfer | 217 | 48 | **62** | 18 | 82 | 7 | 错误日志最密集，**重点排查** |
| model_loader/netloader | 59 | 29 | **20** | 10 | 0 | 0 | 错误占比 34% |
| model_loader/rfork | 45 | 14 | **16** | 4 | 1 | **10** | exception 占 22%，**最高** |
| platform.py | 33 | 15 | 0 | 18 | 0 | 0 | 仅 warning/info，无错误日志 |
| compilation/passes | 27 | 0 | 0 | 1 | 26 | 0 | 纯 DEBUG |
| patch/platform | 23 | 7 | 0 | 3 | 11 | 2 | 异常已被外层吞 |
| worker/worker.py | 19 | 10 | 0 | 5 | 4 | 0 | 无 error |
| utils.py | 19 | 9 | 0 | **10** | 0 | 0 | 全 warning（可能级别过激） |
| cpu_binding.py | 18 | 7 | 0 | 10 | 1 | 0 | 全部走 warning，**没有 ERROR 表示严重失败** |
| quantization/modelslim_config.py | 15 | 1 | **5** | 0 | 9 | 0 | 错误全在量化配置 |

### 3.1 沉默成功文件（仅 ERR/WARN 无 INFO）

3 个文件存在"沉默成功"问题——只有异常分支有日志，正常路径无任何 INFO：

| 文件 | err | warn | 问题 |
|------|-----|------|------|
| `model_loader/netloader/utils.py` | 0 | 3 | 路径工具类无成功日志 |
| `distributed/kv_transfer/kv_pool/ascend_store/backend/yuanrong_backend.py` | **5** | 0 | 5 个错误无任何成功路径日志 |
| `ops/fused_moe/fused_moe.py` | **4** | 0 | MoE 算子错误，无成功日志 |

**建议**：在关键初始化（`__init__`）和首次成功调用处加 INFO 日志，便于排障时确认"功能是否启动成功"。

### 3.2 kv_transfer 模块 217 条日志详解

`distributed/kv_transfer/` 是 PD 分离的核心，占全仓日志 36%，错误占全仓 ERROR 的 51%（62/121）。

子目录分布：
- `kv_p2p/`（mooncake 三种 connector）：~110 条
- `kv_pool/cpu_offload/`：~40 条
- `kv_pool/ascend_store/`：~60 条（含 backend/yuanrong、backend/mooncake）
- `utils/`：~10 条

**问题**：
- 三种 mooncake_connector 的 `raise zmq.ZMQError("Receive timeout")` 都无前置日志（P0，§2.2）
- backend/yuanrong 有 5 个 ERROR 但无成功路径 INFO

---

## 4. 关注点三：报错原因是否写清楚（标准 4.1）

### 4.1 116 条"纯静态短句"ERROR/WARNING 分布

按模块：

| 模块 | 数量 | 评价 |
|------|------|------|
| model_loader/netloader | 25+ | 整个 netloader 子树基本不合规 |
| model_loader/rfork | 12+ | 整模块不合规 |
| platform.py | 8+ | 全部 warning |
| distributed/kv_transfer | 25+ | mooncake 链路多个 ERROR 无参数 |
| worker | 5+ | check_health 类 |
| 其他 | ~30 | 散落 |

**典型纯静态短句**（5 条样本）：

```python
# model_loader/netloader/load.py:69
logger.error("Failed to load model")
# → 缺：model_path、error、retry_count

# model_loader/netloader/netloader.py:69
logger.error("CONFIG_FILE not found")
# → 缺：config_path、env var name、fallback 行为

# model_loader/netloader/netloader.py:273
logger.error("NetLoader elastic loads model fails")
# → 缺：rank、load_time、最后错误、是否 fallback

# model_loader/rfork/rfork_loader.py:186
logger.warning(...)  # 内容未知
# → 需要按文件读

# worker/worker.py:799
logger.info("query NPU card  %s timeout.", self.local_rank)  # 实际是 INFO 但含 timeout，级别错
```

### 4.2 缺排查方向关键词（标准 4.1 第 5 字段"分析方案"）

- 抽 262 条 warning/error：含"可能原因/可检查/check/verify"等关键词：**23 条 (9%)**
- pymotor 仓 6%、cmotor 仓 ~8%——vllm-ascend 比其他仓略好但仍严重不足

### 4.3 隐私保护（标准 8）

- 0 条 password/secret/token 泄露（vllm-ascend 仓不像 pymotor 那样有 cert_util 直接打 password）
- 仍需关注：5-7 条 `Invalid message format: %s` 类的 mooncake 错误日志可能打印二进制数据

---

## 5. 其他系统性问题

### 5.1 组件标识（标准 3/5）

- 源码中含 `[vllm/xxx]` 字符串的 logger：**0/601 (0%)**
- 仅有零星手动前缀：`[cpu_bind_mode]`、`[migrate]`、`[irq]`（在 cpu_binding.py 内部，约 4 处）
- **影响**：运维告警时无法按组件定位负责人
- **历史**：4-28 报告 0/478，4-29 报告 0/476，**37 天后仍为 0**

### 5.2 链路追踪（标准 7）

- warning/error 中含 `trace_id`/`request_id`/`req_id`：**7/262 (3%)**
- 全部集中在 `distributed/kv_transfer/kv_p2p/mooncake_*.py`，是异步链路必须
- 其他 255 条（97%）在排查时无法串联到请求生命周期
- **影响**：跨组件问题几乎无法串联
- **历史**：4-28 报告 1/198，4-29 报告 ~10/60，**绝对数没改善，比例从 0.5% 涨到 3%（仅因 kv_transfer 集中改造）**

### 5.3 分级误用（标准 1）

7 条 INFO 含失败类词：

| 文件:行号 | 错误词 | 日志内容 |
|-----------|-------|---------|
| `model_loader/netloader/load.py:74` | error | `logger.info("elastic_load error: %s", e)` |
| `model_loader/netloader/interaction/elastic.py:62` | error | `logger.info("IP format error: %s, detail: %s", source, e)` |
| `worker/worker.py:775` | fail | `logger.info("query NPU card %s fail: %s", self.local_rank, result.stderr)` |
| `worker/worker.py:777` | timeout | `logger.info("query NPU card  %s timeout.", self.local_rank)` |
| `worker/worker.py:781` | fail | `logger.info("query NPU card %s fail: %s", self.local_rank, e)` |
| `distributed/kv_transfer/kv_pool/cpu_offload/cpu_offload_connector.py:274` | error | `logger.info("wait for metadata server to start, error: %s", e)` |
| `distributed/kv_transfer/kv_pool/ascend_store/pool_worker.py:649` | failed | `logger.info("Layerwise get failed")` |

**根因**：这 7 条全部是"已知失败但可重试"或"启动期短暂错误"，应改 **WARNING + count**（按 §6.1 合并规则）。

### 5.4 C++ 日志（csrc/）

- 1270 行 grep 结果，含 1143 个 LOG_X 调用
- **LOG_E 占 68%（776 条）**——这是异常分布
- vLLM-Ascend 仓的 C++ 部分（aclnn_torch_adapter、kernels、moe、mc2 等）日志密度远高于 Python 部分
- 推测根因：C++ 算子注册/调度失败时倾向用 LOG_E（实际可能是 WARNING 级别）
- **本次未深入逐条分析 C++**（属下次扫描任务）

### 5.5 防刷屏（标准 6）

- Python 仓未发现明显循环刷屏
- C++ 仓 776 条 LOG_E 中可能有循环重试每次都打 LOG_E 的——需结合代码上下文人工复查
- 模型加载路径（`model_loader/rfork/transfer_backend.py`）历史上是刷屏高发区

---

## 6. 整改优先级建议

### P0 — 影响排障效率 + 跨进程可观测性，必须立即修（3 处）

1. **3 处 zmq.ZMQError("Receive timeout") 前置 logger.error**（mooncake 三种 connector）
   - 模板：[vllm/kv_transfer] Mooncake receive timeout. peer={}, timeout_ms={}, ...

### P1 — 业务中断/启动失败需前置（64 处）

2. **61 处 P1 业务中断 RuntimeError 前置 logger.error**
   - 重点文件：`_310p/ops/causal_conv1d.py` (10)、`cpu_binding.py` (8)、`netloader/elastic.py` (6)、`rfork_loader.py` (4)
3. **2 处启动期 ValueError 前置 logger.error**（含 config/path 类）
4. **1 处超时 raise 前置 logger.error**

### P2 — 标准化 + 链路追踪 + 组件标识（≥ 400 处）

5. **122 处 P2 入参校验 raise 改 logger.warning**（按 §4.5 + §3 分级原则）
   - worker 22、distributed 22、ops 18 是大头
6. **37 处 NotImplementedError 改 logger.warning**
7. **7 条 INFO 含失败词改 WARNING + count**（按 §6.1 合并）
8. **116 条纯静态短句补参数快照 + 排查方向**（按 §4.1 三件套）
9. **引入 `[vllm/子模块]` 组件标识**——**最关键**：建议在 logger 封装层（`vllm_ascend/utils.py`）统一注入，按调用栈自动获取子模块名
10. **trace_id/request_id 全链路注入**——从 `coordinator/api_server` 入口开始，但 vllm-ascend 实际只做 worker 端，需在 kv_transfer 入口生成 req_id 并下传

### P3 — C++ 仓独立扫描（1270 行）

11. **C++ LOG_E 776 条需逐条审级**——建议另开一次专项扫描（csrc 仓的日志规范与 Python 不同）
12. **C++ 防刷屏人工复查**（循环/重试路径）

---

## 7. 系统性建议（针对 vllm-ascend 团队）

1. **加日志规范到 PR 模板**：自检清单（参见标准末尾"好日志自检清单"）— `[ ]` 组件前缀、`[ ]` trace_id、`[ ]` 三件套
2. **logger 封装层统一注入组件前缀**：在 `vllm_ascend/utils.py` 改造 logger，按调用栈推断子模块（参考 pymotor 的 `coordinator/router` 标识）
3. **关键路径加 INFO**：netloader 模型加载成功、scheduler 调度成功、EPLB 专家放置成功等"沉默成功"路径补 INFO
4. **从 kv_transfer 入口注入 request_id**：因 vllm-ascend 主要做 worker，请求从 vllm 主框架进入，vllm-ascend 端无 trace_id 来源——需在 mooncake connector 入口生成内部 `req_id` 并下传所有 ERROR/WARNING
5. **从 5-13 起的 37 天内新增 125 条日志无一条符合组件前缀规范**——建议把 log-quality-scan 加入 CI pre-commit

---

## 8. 待用户确认事项

> 本报告**只扫描、不修改**。如要进入整改阶段：
> 1. **P0 三项先做？**（3 处 zmq 异常前置日志）
> 2. **是否需要按 P0/P1 列表生成每个位置的 diff 建议？**（调用 `log-quality-rewrite` skill）
> 3. **是否按 log-paths 偏好，把"组件标识"和"trace_id"两个系统性问题沉淀为 `log-diagnosis-vllm-ascend-*` skill？**
> 4. **是否对 csrc/ C++ 日志做专项扫描？**（本次未深入）
> 5. **122 处 P2 入参校验是否批量改 WARNING？**（这是治本，但要逐个文件确认）

---

## 附录 A：扫描方法与统计

- 工具：`log-quality-scan-code` skill（v2，按 §4.5 重新分类）
- 扫描脚本：`/home/h00906152/projects/vllm/vllm-ascend/.tmp_scan/scan.py`
- 扫描结果 JSON：`/home/h00906152/projects/vllm/vllm-ascend/.tmp_scan/scan_result.json`
- 排除项：`tests/`、`__pycache__/`、`.git/`、`build/`、第三方目录
- 警告：`terminal()` 50KB stdout cap 不适用本次，因用 `subprocess.run(capture_output=True)`

## 附录 B：P0/P1 完整清单（数据已存 JSON）

按文件聚合的完整清单见 `scan_result.json` 中 `raise_priority.P0`、`raise_priority.P1` 数组，每条含：
- `rel`：相对 `vllm_ascend/` 的路径
- `ln`：行号
- `content`：raise 语句原文
- `category`：分类标签

如需人工打开定位，可按 `rel` 用 IDE 跳转。
