# vLLM 贡献调研笔记（2026-09-07/08）

## 1. 项目贡献规则（来自 AGENTS.md / docs/contributing/README.md）

- 每个 commit 必须带 DCO 签名：`git commit -s` → `Signed-off-by: Name <email>`；邮箱必须与 commit author 一致。
- PR 标题必须以方括号标签开头：`[Bugfix]`, `[Feature]`, `[Perf]`, `[Refactor]`, `[CI/Build]`, `[Test]`, `[Doc]`, `[Misc]`
  + 范围标签 `[Frontend]`, `[Rust Frontend]`, `[Core]`, `[Kernel]`, `[Model]`, `[Tool Parser]` 等，可叠加：`[Bugfix][Tool Parser] ...`。
- PR 描述模板（.github/PULL_REQUEST_TEMPLATE.md）：`## Purpose` / `## Test Plan` / `## Test Result`；模板底部说明性文字会被 GitHub Actions 自动删除。
- 硬性要求（AGENTS.md，违反可能被自动封禁）：
  - 禁止“纯 agent PR”，提交人必须逐行 review 并能解释每一行改动；
  - 禁止琐碎的 busywork PR（单个 typo / 单处风格修改）；
  - 提 PR 前做重复检查：issue 评论、已开 PR 搜索；
  - **AI 辅助的 PR 必须在描述里明确声明使用了 AI 辅助**，并说明为何不与现有 PR 重复、跑过哪些测试及结果；
  - 建议在 commit trailer 里加 `Co-authored-by:`（这是“建议”，DCO 签名才是硬性要求）。
- Lint：`pre-commit run --all-files`（ruff check/format、typos、mypy 在 CI 手动 stage）；Python 行宽 88；Google 风格 docstring。
- CI：Buildkite，PR 需要 maintainer 加 `ready` 标签才会跑完整 CI；reviewer 用 `/ci run` 触发。
- Rust 前端（rust/）有自己的 AGENTS.md：winnow 声明式解析器、expect-test 快照测试、`cargo nextest`、workspace 依赖。

## 2. 现状观察

- 仓库极其活跃：近期 issue 几乎在 24h 内就有人（大量是 agent 驱动的贡献者）开 PR；`good first issue` 列表里的任务基本已被认领（assignee + linked PR）。
- 长期无人认领且根因清晰的 issue 主要集中在 tool-calling / parser 领域（Python 侧）以及 Rust 前端的“parser parity”缺口（#44280 roadmap：36 个 tool parser、15 个 reasoning parser 待补齐）。
- Rust 前端 PR 由 BugenZhao 维护，外部贡献者的小 PR 合并较快（reidliu41 的多个 PR 数日内合并）；Python parser 类 PR 排队较久（多数 8 月的 PR 仍在等 review）。

## 3. 排除掉的候选（已核实）

| issue | 结论 |
|---|---|
| #55633 legacy qwen3_xml 空白 content | 针对 vllm-ascend 固定的旧版本；main 上该 parser 已被 engine 版替代（文件不存在） |
| #55733 encoder cache 计数 | 已有 PR #55734 |
| #55556 Phi-4 content format | 已有 PR #55587 / #55567 |
| #32588 Whisper 时间戳 | 已 assign（dr75/aadeshupadhyay）+ PR #48225 |
| #55195 reasoning 多 token delta | 已有 PR #55210 |
| #55395 Muse Glimmer ATEM | 已有 PR #55419 |
| #55592 / #55530 Responses call_id / JSON retry | 已有 PR #55596 / #55540 |
| #55495 qwen3_xml `</parameter` 泄漏 | 已有 PR #55497 |
| #50901 reasoning 前缀丢失 | 已有 PR #50918（含 follow-up #51164） |
| #49412 流式/非流式空白不一致 | 作者有草稿 PR #49426（7 月起停滞） |
| #47734 Mistral required 流式 tool id | 作者有 PR #47739 |
| #49316 kimi_k2 流式类型强制 | 需要设计讨论（流式前缀稳定性），作者提到 draft |
| #48753 Qwen3 参数空白被 strip | main 上已修复（`_trim_wrapping_newlines`） |
| #38488 输入侧 reasoning_content | main 上 ChatCompletionRequest 已归一化；PR #52513 处理 tokenize/pooling |
| #54273 / #53246 / #50604 Kimi K3 | 已 assign chaunceyjiang / PR #54314 / #50627 |
| #54528 MuseGlimmer 迁移 | corona10 已认领 |
| #39479 / #32268 / #31249 / #41230 good first issue | 均已 assign 且有 PR |
| #50989 Qwen3 strict 无参数工具死循环 | 根因在 xgrammar 内置 `qwen_3_coder` structural tag：无参数时语法要求 `<function=NAME>\n\n</function>`（两个换行），而模板/模型只输出一个换行 → 被约束成只能吐空白。属于 xgrammar 侧缺陷（vLLM 可做 workaround，但更适合向 mlc-ai/xgrammar 提 issue/PR）。本地用 `xgrammar.Grammar.from_structural_tag` 验证过。 |

## 4. 选定的三个 PR

1. **[Bugfix][Tool Parser] Hermes parser 忽略 JSON 字符串内的 `</tool_call>`**（issue #45167，2026-06-10 开，无 PR）。Python。
2. **[Rust Frontend] 新增 pythonic tool parser**（Llama 3.2 / Llama 4 官方 pythonic 格式，Rust 前端缺失；roadmap #44280）。Rust。
3. **[Rust Frontend] 新增 OLMo 3 tool parser**（pythonic 变体：换行分隔 + `<function_calls>` 包裹 + JSON 字面量），叠加在 PR 2 之上。Rust。
