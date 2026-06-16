# log-analysis-locate-guide

从代码仓提取指定特性的关键日志，生成《{特性}日志定位指南》Markdown 文档。

## 安装

本仓是 skills 仓库，使用者 clone 后，把需要的 skill 软链到本地 skills 目录即可：

```bash
# 1. clone 本仓
git clone <本仓地址> ~/projects/private-skills

# 2. 软链到本地 skills 目录（与已有 skill 同级）
ln -s ~/projects/private-skills/skills/log-analysis/log-analysis-locate-guide \
      ~/.claude/skills/log-analysis-locate-guide
```

也可以直接复制：

```bash
cp -r ~/projects/private-skills/skills/log-analysis/log-analysis-locate-guide \
      ~/.claude/skills/
```

软链/复制完成后重启 Claude Code，本 skill 就会出现在 skill 列表中。

## 使用

两种触发方式（Claude 识别后会向用户确认剩余输入）：

**方式 1：在目标代码仓里直接问**（最自然）

进到目标代码仓根目录，启动 Claude Code，直接说：

- 「整理 PD 建链 定位指南」
- 「整理 大 EP 启动 定位指南」
- 「从代码整理 X 定位指南」

Claude 会从当前目录推断代码根目录，只问仓库名、特性边界、输出目录。

**方式 2：显式指定代码仓和特性**

- 「整理 cmotor 的 PD 建链 定位指南」
- 「整理 vllm-ascend 的 池化 定位指南」

## 输入

| 输入 | 必需 | 说明 |
|------|------|------|
| 代码仓 | ✅ | `cmotor` / `pymotor` / `mindie-llm` / `vllm` / `vllm-ascend` 等 |
| 特性描述 | ✅ | 自然语言，如「PD 建链」「池化」 |
| 代码根目录 | ✅ | 仓库绝对路径（方式 1 下自动取 cwd） |
| 输出目录 | ✅ | `.md` 落盘目录（用户未指定时询问，勿硬编码） |

## 输出

| 产出 | 格式 | 文件名 |
|------|------|--------|
| 日志定位指南 | Markdown | `{输出目录}/{特性}日志定位指南.md` |

文档结构、固定章节、互链规则详见 [SKILL.md](SKILL.md)。
