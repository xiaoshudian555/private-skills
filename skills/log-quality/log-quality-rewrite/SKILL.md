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
| 目标仓库 | 可选 | cmotor / pymotor / mindie-llm / vllm / vllm-ascend（不填则从日志内容或 PROJECT-NAV.md 推断） |
| 代码路径 | 可选 | 仓库在机器上的路径，如 `/home/h00906152/projects/cmotor`，不填则按 PROJECT-NAV.md 推断 |

## 输出

| 输出 | 格式 | 说明 |
|------|------|------|
| 逐条修改对比 | Markdown 表格 | 原日志 → 修改后日志 + 修改原因 |
| 代码修改 diff | diff 格式 | 可直接 apply 的代码级 diff |

---

## 工作流程

### 第零步：确认目标代码仓的语言和格式（必须先执行）

> **重要：重写之前必须先了解目标代码的日志语言和格式风格，确保输出的日志与周围代码保持一致。**

**1. 确定目标仓库和代码路径**

- 如果用户指定了目标仓库和代码路径，直接使用
- 如果未指定，从以下线索推断：
  - 扫描报告中的文件路径（如 `cmotor/src/router/...` → cmotor）
  - PROJECT-NAV.md 中的仓库路径映射
  - 日志内容中的组件前缀格式（如 `[cmotor/...]` → cmotor）

**2. 扫描代码仓确认两个语言维度**

用 grep 提取目标仓库中同模块（或同文件）的日志语句，判断：

| 维度 | cmotor（C++） | pymotor（Python） | mindie-llm（C++） | vllm-ascend（Python） |
|------|--------------|-------------------|------------------|---------------------|
| 编程语言 | `LOG_I` / `LOG_W` / `LOG_E` / `SLOG_ERROR` 等宏 | `logger.info()` / `self.logger.warning()` 等方法 | 同 cmotor | 同 pymotor |
| 自然语言 | 英文 / 中文 / 混用（从日志内容判断） | 英文 / 中文 / 混用 | 同 cmotor | 同 pymotor |

**grep 示例（按仓库选择）：**

```bash
# cmotor / mindie-llm：扫描 C++ 文件中的日志宏
grep -rn "LOG_I\|LOG_W\|LOG_E\|LOG_D\|SLOG_\|MINDIE_LLM_LOG" \
  --include="*.cpp" --include="*.h" {代码路径}/src/{模块名}

# pymotor / vllm-ascend：扫描 Python 文件中的日志方法
grep -rn "logger\.\(info\|warning\|error\|debug\)" \
  --include="*.py" {代码路径}/motor  # pymotor 源码在 motor/ 子目录
```

**3. 记录格式风格作为重写约束**

从扫描结果中提取该仓库的格式风格，作为重写时的格式约束：

| 格式维度 | 确认结果 |
|---------|---------|
| 编程语言 | C++ / Python |
| 自然语言 | 英文 / 中文 / 混用 |
| 宏/函数风格 | 如 `SLOG_ERROR(...)` vs `logger.error(...)` |
| 组件前缀格式 | 如 `[cmotor/router]` vs `[cmotor-router]` vs 无前缀 |
| trace_id 位置 | 如 `trace_id=xxx, ` 前缀固定位置 |
| 格式化占位符风格 | `%s` / `{name}` / `{name}=%s` 混用 |

> 如果同一仓库中混用多种风格，以被改写日志所在文件的风格为准。

### 第一步：读取扫描报告

提取所有"不合格"和"警告"级别的日志条目，记录：
- 原日志内容
- 不合规原因
- 改进建议

### 第二步：按标准逐条重写

对每条不合规日志，读取 `log-quality-standard`，按其中所有标准逐项修正。重写时**必须遵循第零步确认的格式约束**，包括：
- 编程语言对应的宏/方法名（如 `SLOG_ERROR` 而非 `logger.error`）
- 自然语言风格（英文/中文，与周围代码一致）
- 组件前缀格式、trace_id 位置、占位符风格

> 注意：以下示例按当前 standard 中的标准名称展示，运行时按 standard 实际内容执行，不写死标准编号。

#### 按 standard 中"分级清晰"规则修正

```python
# 原：logger.info("Connection pool full")  # 异常打了 INFO
# 改：logger.warning("Connection pool nearly full, usage={}%".format(usage))

# 原：logger.error("Retry succeeded after N attempts")  # 重试成功打了 ERROR
# 改：logger.info("Retry succeeded, attempts={}".format(attempts))
```

#### 按 standard 中"描述充分"规则补充

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

#### 按 standard 中"组件归属"规则修正

```python
# 原：logger.error("Request failed")
# 改：logger.error("[vllm/scheduler] Request failed. ...")

# 检查：是否需要新增组件前缀
```

#### 按 standard 中"防刷屏"规则修正

```python
# 原（在循环内）：
# for i in range(10):
#     logger.error("Connection failed")  # 逐条打，刷屏
# 改：降为 DEBUG，保留信息但不刷屏
# for i in range(10):
#     logger.debug("Connection attempt %d failed: %s", i, str(e))  # 不影响线上可读性
# 循环外统一打一条 ERROR（防刷屏计数器逻辑省略）
# if error_count > 0:
#     logger.error("Connection failed after %d retries (last %ds saw %d occurrences)",
#                  error_count, WINDOW_SECONDS, error_count)
```

**原则：防刷屏整改不要直接删除日志，改为 DEBUG 级别。** 删除会导致关键信息丢失，降级才能既保留调试线索又不刷屏。

#### 按 standard 中"链路追踪"规则修正

```python
# 原：logger.error("[llm] Inference failed")
# 改：logger.error("[llm/model] trace_id={trace_id}, Inference failed. ...")

# 如果日志点没有 trace_id 上下文，需传入或生成：
# trace_id = context.get("trace_id", generate_trace_id())
```

#### 按 standard 中"隐私保护"规则修正

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

> 读取 `log-quality-standard`，按实际定义的标准数量和名称动态生成。

| 修改项 | 数量 |
|--------|------|
| （按 standard 动态填入） | X 条 |
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
- **防刷屏整改：不要直接删除重复日志，改为 DEBUG 级别**。删除会导致关键信息丢失，降级才能既保留调试线索又不刷屏
- trace_id 如果原代码没有上下文，需要说明如何传入或生成
- 隐私保护修改时，确保用脱敏后的值替代，不要凭空删除整条日志
- **格式一致性优先**：重写后的日志必须与目标代码仓的语言风格（编程语言 + 自然语言）和格式惯例保持一致，不要出现 C++ 代码中混用 Python 风格日志、或中文日志包围英文日志的情况