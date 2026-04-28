---
name: log-quality-rewrite
description: 将不合规的日志改写为符合 log-quality-standard 的版本。输入扫描报告，输出每条日志的修改前后对比（diff）和具体修改建议。触发词：重写日志、改写日志、日志整改、修复日志。
---

# 日志质量重写器

将不合规的日志改写为符合标准的版本，输入是 `log-quality-scan` 的扫描报告，输出修改前后对比。

---

## 触发词

重写日志、改写日志、日志整改、修复日志

---

## 输入

| 输入 | 必需 | 说明 |
|------|------|------|
| 扫描报告 | 必需 | log-quality-scan 输出的检查结果 |
| 日志原文 | 必需 | 原始日志内容（日志文件或代码） |

---

## 输出

| 输出 | 格式 | 说明 |
|------|------|------|
| 逐条修改对比 | Markdown 表格 | 原日志 → 修改后日志 + 修改原因 |
| 代码修改 diff | diff 格式 | 可直接 apply 的代码级 diff |

---

## 工作流程

### 第一步：读取扫描报告

提取所有"不合格"和"警告"级别的日志条目，记录：
- 原日志内容
- 不合规原因
- 改进建议

### 第二步：按标准逐条重写

对每条不合规日志，按 `log-quality-standard` 逐项修正：

#### 标准1：分级修正

```python
# 原：logger.info("Connection pool full")  # 异常打了 INFO
# 改：logger.warning("Connection pool nearly full, usage={}%".format(usage))

# 原：logger.error("Retry succeeded after N attempts")  # 重试成功打了 ERROR
# 改：logger.info("Retry succeeded, attempts={}".format(attempts))
```

#### 标准2：描述补充

```python
# 原：logger.error("Link failed")
# 改：
# logger.error(
#     "[cmotor/router] Link failed. "
#     "group_id={group_id}, peer={peer}, error={err}, "
#     "可检查：1. ping {peer_ip} 2. telnet {peer_ip} {port} 3. 对端进程是否存活"
# )

# 原：logger.error("Param error: expected > 0, got {}".format(val))
# 改：
# logger.error(
#     "[pymotor/coordinator] Parameter validation failed. "
#     "param=timeout, expected: timeout > 0, actual: timeout = {val}"
# )
```

#### 标准3：组件标识

```python
# 原：logger.error("Request failed")
# 改：logger.error("[vllm/scheduler] Request failed. ...")

# 检查：是否需要新增组件前缀
```

#### 标准4：防刷屏

```python
# 原（在循环内）：
# for i in range(10):
#     logger.error("Connection failed")  # 逐条打
# 改：
# # 循环外维护计数器
# error_count = 0
# for i in range(10):
#     if not success:
#         error_count += 1
# if error_count > 0:
#     logger.error("Connection failed after {} retries (last {}s saw {} occurrences)".format(
#         error_count, WINDOW_SECONDS, error_count))
```

#### 标准5：链路追踪

```python
# 原：logger.error("[llm] Inference failed")
# 改：logger.error("[llm/model] trace_id={trace_id}, Inference failed. ...")

# 如果日志点没有 trace_id 上下文，需传入或生成：
# trace_id = context.get("trace_id", generate_trace_id())
```

#### 标准6：隐私保护

```python
# 原：logger.error("User query failed: {}".format(user_query))
# 改：logger.error(
#     "[llm/model] User request failed. model={model_id}, input_len={input_len}, "
#     "user_id={user_id_hash}  (原始输入不记录)"
# )

# 原：logger.error("API key used: {}".format(api_key))
# 改：logger.error("API key prefix: {}".format(api_key[:8] + "****"))
```

### 第三步：生成修改对比表

```markdown
## 修改对比

| # | 原日志 | 修改后 | 涉及标准 | 修改原因 |
|---|--------|--------|---------|----------|
| 1 | `[ERROR] link failed` | `[ERROR] [cmotor/router] trace_id=xxx, Link failed, error=..., 可检查...` | 标准2/3/5 | 缺少组件标识、根因、trace_id、分析方案 |
```

### 第四步：生成代码 Diff

```diff
- logger.error("link failed")
+ logger.error(
+     "[cmotor/router] Link failed. "
+     "group_id={group_id}, peer={peer}, error={err}, "
+     "可检查：1. ping {peer_ip} 2. telnet {peer_ip} {port} 3. 对端进程是否存活. "
+     "trace_id={trace_id}"
+ )
```

### 第五步：汇总统计

```markdown
## 修改汇总

- **总修改数**：X 条
- **标准1（分级）修复**：X 条
- **标准2（描述）修复**：X 条
- **标准3（组件）修复**：X 条
- **标准4（防刷屏）修复**：X 条
- **标准5（链路）修复**：X 条
- **标准6（隐私）修复**：X 条
```

---

## 输出文件

| 输出 | 文件名 |
|------|--------|
| 修改对比 Markdown | `{功能名}_日志修改对比.md` |
| 代码 Diff | `{功能名}_日志修改.diff` |

---

## 注意事项

- 优先保证逻辑等价，只改日志格式和内容，不改业务逻辑
- 如果原日志描述已经足够好，只补充缺失部分，不过度改写
- 防刷屏修改涉及代码结构变化，需要额外说明上下文依赖
- trace_id 如果原代码没有上下文，需要说明如何传入或生成
- 隐私保护修改时，确保用脱敏后的值替代，不要凭空删除整条日志
