---
name: log-quality-write
description: 从零为新功能编写符合 log-quality-standard 的日志语句。在不改变业务逻辑的前提下，给出每条日志的建议内容、级别、格式。触发词：写日志、编写日志、新增日志、日志设计。
---

# 日志编写器

从零为新功能编写符合 `log-quality-standard` 的日志语句。输入功能描述和代码，输出完整的日志设计方案。

---

## 触发词

写日志、编写日志、新增日志、日志设计

---

## 输入

| 输入 | 必需 | 说明 |
|------|------|------|
| 功能描述 | 必需 | 要打日志的功能，如"链路保活心跳"、"请求超时处理" |
| 代码片段 | 必需 | 要新增日志的代码（包含业务逻辑） |
| 涉及组件 | 必需 | 如 cmotor/router、vllm/scheduler |
| 已有日志 | 可选 | 该功能已有哪些日志（避免重复） |
| 目标仓库 | 可选 | cmotor / pymotor / mindie-llm / vllm / vllm-ascend（不填则从代码片段路径或 PROJECT-NAV.md 推断） |
| 代码路径 | 可选 | 仓库在机器上的路径，如 `/home/h00906152/projects/cmotor`，不填则按 PROJECT-NAV.md 推断 |

## 输出

| 输出 | 格式 | 说明 |
|------|------|------|
| 日志设计方案 | Markdown | 每条日志的：位置、内容、级别、理由 |
| 代码模板 | 伪代码/代码片段 | 可直接粘贴到代码中的日志语句 |

---

## 工作流程

### 第零步：确认目标代码仓的语言和格式（必须先执行）

> **重要：编写之前必须先了解目标代码的日志语言和格式风格，确保输出的日志与周围代码保持一致。**

**1. 确定目标仓库和代码路径**

- 如果用户指定了目标仓库和代码路径，直接使用
- 如果未指定，从以下线索推断：
  - 代码片段中的文件路径（如 `cmotor/src/router/...` → cmotor）
  - 代码片段中的 import / include 语句（如 `from motor import` → pymotor）
  - PROJECT-NAV.md 中的仓库路径映射

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

**3. 记录格式风格作为编写约束**

从扫描结果中提取该仓库的格式风格，作为编写时的格式约束：

| 格式维度 | 确认结果 |
|---------|---------|
| 编程语言 | C++ / Python |
| 自然语言 | 英文 / 中文 / 混用 |
| 宏/函数风格 | 如 `SLOG_ERROR(...)` vs `logger.error(...)` |
| 组件前缀格式 | 如 `[cmotor/router]` vs `[cmotor-router]` vs 无前缀 |
| trace_id 位置 | 如 `trace_id=xxx, ` 前缀固定位置 |
| 格式化占位符风格 | `%s` / `{name}` / `{name}=%s` 混用 |
| 日志方法调用形式 | `logger.error(...)` vs `self.logger.error(...)` vs `_logger.error(...)` |

> 如果同一仓库中混用多种风格，以被新增日志所在文件的风格为准。

### 第一步：分析代码逻辑

理解代码的：
1. **执行路径**：有哪些分支（成功/失败/异常/边界）
2. **上下文**：有哪些变量可用（error_code、peer_id、timeout 等）
3. **调用关系**：是入口/中间/出口

### 第二步：确定打日志的位置

根据 `log-quality-standard` 标准 1（何时打日志），确定日志位置：

| 场景 | 建议 | 级别 |
|------|------|------|
| 进入关键函数 | 可选，仅关键入口打 INFO | INFO |
| 参数校验失败 | 必须打 | WARNING 或 ERROR（取决于是否阻断） |
| 外部调用（RPC/网络）前 | 可选 | DEBUG 或 INFO |
| 外部调用成功后 | 必须打 | INFO |
| 外部调用失败 | 必须打 | ERROR |
| 重试成功 | 必须打 | WARNING |
| 循环内失败 | 最后统一打 ERROR | ERROR（防刷屏） |
| 退出函数（成功/失败） | 可选，仅关键路径打 INFO | INFO |

### 第三步：按 `log-quality-standard` 中所有标准编写每条日志

**必须遵循第零步确认的格式约束**（编程语言 + 自然语言 + 宏/函数风格 + 组件前缀 + trace_id 位置 + 占位符风格），示例中展示的风格仅作参考，实际输出以目标代码仓的风格为准。

#### 示例：为"链路建连"功能写日志

**代码片段：**
```cpp
int establish_link(const string& group_id, const string& peer) {
    if (group_id.empty() || peer.empty()) {
        return -1;  // 早返回
    }

    int ret = connect_to_peer(peer);
    if (ret != 0) {
        // 重试 3 次
        for (int i = 0; i < 3; i++) {
            ret = connect_to_peer(peer);
            if (ret == 0) break;
        }
        if (ret != 0) return -1;
    }

    return register_link(group_id, peer);
}
```

**日志设计方案：**

```markdown
## 日志设计方案：链路建连

| # | 代码位置 | 日志级别 | 日志内容 | 编写理由 |
|---|---------|---------|---------|----------|
| 1 | 参数校验后 | WARNING | `[cmotor/router] Link establish validation failed. group_id={group_id}, peer={peer}, reason=empty_id_or_peer` | 入参校验失败是异常，但不一定阻断，所以 WARNING |
| 2 | 首次连接失败 | DEBUG | `[cmotor/router] First connection attempt failed. peer={peer}, err={errno}` | 首次失败正常重试，DEBUG 足够 |
| 3 | 重试循环内 | （不打） | — | 防刷屏，不在循环内逐条打 |
| 4 | 重试全部失败 | ERROR | `[cmotor/router] trace_id={trace_id}, Link establish failed after 3 retries. group_id={group_id}, peer={peer}, err={errno}, 可检查：1. ping {peer_ip} 2. telnet {peer_ip} {port} 3. 对端进程是否存活` | 标准2+3+5，ERROR 级别，必须带 trace_id 和分析方案 |
| 5 | 建连成功 | INFO | `[cmotor/router] trace_id={trace_id}, Link established. group_id={group_id}, peer={peer}` | 成功路径打 INFO，带 trace_id 方便关联 |
```

### 第四步：输出代码模板

模板中使用的宏/函数名、格式风格**必须与第零步确认的结果一致**，不要出现 C++ 代码中用 Python 风格日志、或中文日志混入英文代码库的情况。

```cpp
// === 链路建连日志（按 log-quality-standard 编写） ===

// 位置1：参数校验后
if (group_id.empty() || peer.empty()) {
    logger_warning("[cmotor/router] Link establish validation failed. "
                   "group_id={}, peer={}, reason=empty_id_or_peer",
                   group_id.c_str(), peer.c_str());
    return -1;
}

// 位置2：首次连接失败（DEBUG）
int ret = connect_to_peer(peer);
if (ret != 0) {
    logger_debug("[cmotor/router] First connection attempt failed. "
                 "peer={}, err={}", peer.c_str(), ret);
}

// 位置3：重试（防刷屏，不打日志，计数器在循环外）
int retry_count = 0;
for (int i = 0; i < 3; i++) {
    ret = connect_to_peer(peer);
    if (ret == 0) break;
    retry_count++;
}

// 位置4：重试全部失败
if (ret != 0) {
    logger_error("[cmotor/router] trace_id=%s, Link establish failed after 3 retries. "
                 "group_id=%s, peer=%s, err=%d, "
                 "可检查：1. ping %s 2. telnet %s %s 3. 对端进程是否存活",
                 trace_id, group_id.c_str(), peer.c_str(), ret,
                 peer_ip.c_str(), peer_ip.c_str(), port.c_str());
    return -1;
}

// 位置5：建连成功
logger_info("[cmotor/router] trace_id=%s, Link established. group_id=%s, peer=%s",
             trace_id, group_id.c_str(), peer.c_str());
```

### 第五步：自检清单

写完后对照 `log-quality-standard` 自检：

```
[ ] 每条日志级别是否正确？（异常不是 INFO，成功不是 ERROR）
[ ] ERROR/WARNING 是否回答了"什么错+为什么+查哪里"？
[ ] 是否带了组件标识（[仓库/子模块]）？
[ ] 如果在循环中，是否用了防刷屏计数？
[ ] ERROR/WARNING 是否带了 trace_id？
[ ] 是否包含任何用户输入、prompt、PII？
[ ] 占位符是否完整（{id}、%s 等）？
[ ] 编程语言风格是否与目标代码一致？（宏 vs 方法调用）
[ ] 自然语言是否与目标代码一致？（英文 vs 中文）
```

---

## 注意事项

- 只提供日志方案，不改变业务逻辑
- 如果代码中已有日志，先评估是否合规，不合规的用 rewrite 方案替代
- 防刷屏计数器的维护方式需要在代码注释中说明
- trace_id 如果代码中没有，需要在函数签名中新增参数或从上下文获取
- **格式一致性优先**：输出的代码模板必须与目标代码仓的语言风格（编程语言 + 自然语言）和格式惯例保持一致，不要出现 C++ 代码中混用 Python 风格日志、或中文日志混入英文代码库的情况