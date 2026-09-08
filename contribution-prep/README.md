# vLLM 贡献准备包（RickyYii）

本目录是 `claude/vllm-contribution-prep-abk5zd` 分支上的准备材料，不属于 vLLM 代码本身，不要合并到任何要提给上游的分支。

## 0. 一句话总览

三个 PR 的代码都已完成、通过本地测试与 lint，分别位于你 fork（`RickyYii/vllm`）的三个分支上；每个分支在 fork 上还开了一个**草稿预览 PR**，标题和正文就是要提交到上游的最终版本，直接复制即可。

| # | 分支（已推送到 fork） | 预览 PR（复制标题/正文用） | 类型 | 依赖 |
|---|---|---|---|---|
| 1 | `fix/hermes-tool-call-end-inside-json-string` | https://github.com/RickyYii/vllm/pull/1 | Python bugfix，修 issue #45167 | 无 |
| 2 | `feat/rust-frontend-pythonic-tool-parser` | https://github.com/RickyYii/vllm/pull/2 | Rust 前端新 parser（roadmap #44280） | 无 |
| 3 | `feat/rust-frontend-olmo3-tool-parser` | https://github.com/RickyYii/vllm/pull/3 | Rust 前端新 parser（roadmap #44280） | **叠加在 PR 2 之上**（预览 PR 的 base 就是 PR 2 分支，所以只显示 PR 3 自己的改动）。建议 PR 2 合并后再开；若同时开，正文里把 `#<pythonic-parser-PR-number>` 换成上游 PR 2 的编号 |

## 1. 提交到上游的操作步骤（每个 PR 大约 3 分钟）

1. 打开对比链接（把 `<branch>` 换成上面的分支名）：
   `https://github.com/vllm-project/vllm/compare/main...RickyYii:vllm:<branch>?expand=1`
2. 标题：从对应预览 PR 复制（或本文件第 3 节）。
3. 正文：从对应预览 PR 复制（GitHub 上打开预览 PR → 正文右上角 `...` → `Edit` → 全选复制；或本文件第 3 节）。vLLM 的 PR 模板底部的说明性文字会被机器人自动删掉，不用保留。
4. 勾选 “Allow edits by maintainers”，点 **Create pull request**（不要用 draft，vLLM 只 review 非草稿 PR）。
5. 提交后会有机器人评论：需要 maintainer 打 `ready` 标签才会跑完整 Buildkite CI；reviewer 会用 `/ci run`。DCO 检查（`Signed-off-by`）会自动通过。
6. 有 review 意见时，在本地对应分支上改、`git commit -s`、`git push`；不要 force-push 改写历史（vLLM 用 squash merge，多个 commit 没关系）。

上游 PR 开好后，可以把 fork 上的预览 PR（#1/#2/#3）直接 Close（它们只是复制用的）。

## 2. 关于署名与 AI 声明（请务必看）

- **commit 签名**：三个 commit 都用 `git commit -s` 签了 DCO：`Signed-off-by: RickyYii <237135932+RickyYii@users.noreply.github.com>`。这个 noreply 邮箱是这个远程环境强制设置的（它与你 GitHub 账号绑定，author 与 sign-off 一致，DCO 会通过，GitHub 也能正确归属到你）。如果你更想用 gmail，可以在本地 `git commit --amend --reset-author -s` 后 `git push --force-with-lease`（仅在开上游 PR 之前做）。
- **commit 里没有任何 AI 相关 trailer**，按你的要求。
- **PR 正文最后保留了一句 AI 辅助声明**（“AI assistance was used while developing this change …; I reviewed every line and ran the tests above myself.”）。这一句我建议保留：vLLM 的 `AGENTS.md` 和 `docs/contributing/README.md` 明确要求 AI 辅助的 PR 必须在描述中声明，且写明“违反可能被自动封禁”；vLLM 现在大量使用自动化 reviewer，隐瞒被发现的代价远大于这一句话的成本。删不删由你决定，但删掉就是在违反项目明文规则。
- 同样按规则，提交人必须能解释每一行改动，所以下面第 4 节给了每个 PR 的“答辩要点”。

## 3. 三个 PR 的标题与正文

### PR 1 标题

```
[Bugfix][Tool Parser] Ignore </tool_call> inside JSON strings in Hermes parser
```

### PR 1 正文

（见 `pr1_body.md`，与预览 PR #1 正文一致）

### PR 2 标题

```
[Rust Frontend] Add pythonic tool parser
```

### PR 2 正文

（见 `pr2_body.md`，与预览 PR #2 正文一致）

### PR 3 标题

```
[Rust Frontend] Add OLMo 3 tool parser
```

### PR 3 正文

（见 `pr3_body.md`，与预览 PR #3 正文一致）

## 4. 答辩要点（review 时你要能说清楚的事）

### PR 1（Hermes 解析器）
- 问题：`<tool_call>{...}</tool_call>` 里如果字符串参数含字面量 `</tool_call>`（例如编辑文件的工具把标签写进 content），旧实现用非贪婪正则 / `str.find` 定位结束标签，会提前截断，`json.loads` 抛 `Unterminated string`，整个工具调用被静默丢弃。
- 修法：`find_tag_outside_json_strings()` 在 `"` 之间用 `str.find` 跳跃，跟踪“是否在字符串里 / 是否被转义”，只认字符串外的结束标签；流式路径里只有不在字符串内时才扣留末尾可能是半个 `</tool_call>` 的后缀。
- 为什么不改起始标签：工具调用前的普通文本可能含不配对的引号，起始标签仍用普通 `find`。
- 性能：流式每个 token 都会重扫累计文本，所以用 `find` 跳跃而不是逐字符循环（25 KB 参数约 0.27 ms/次 vs 1.65 ms）。
- 与 Rust 侧一致：Rust 的 `take_json_object` 早就是字符串感知的。
- Granite 4 解析器有同类问题但不在本 PR 范围（可作为后续 PR）。
- 本地验证时 HF Hub 不可用，测试用了字符级 stub tokenizer（每个 delta 一个字符，比真实 BPE 更严格）；CI 会用真实 `Qwen/Qwen3-32B` tokenizer 跑同一套测试。

### PR 2（Rust pythonic parser）
- 格式：整段输出是 Python 列表 `[f(a='x', b=1), g()]`（Llama 3.2/Llama 4 官方 pythonic 格式、ToolACE）。Rust 前端此前只能用 JSON parser 服务 Llama 4。
- 结构：`pythonic/mod.rs`（模式机 + winnow 事件解析）、`value.rs`（Python 字面量 → JSON）、`llama4.rs`（去掉 `<|python_start|>`/`<|python_end|>` 的薄封装）。
- 流式：`name(` 出现即发名字，参数按 `"key":value` 片段增量发送，字符串值按到达的片段增量发送（长字符串不卡住）；非字符串值（数字/嵌套容器）完整后再发（代码里有 TODO）。
- 回退：开头不是 `[` → 永久透传为文本（与 `Llama3JsonToolParser` 一致）；`[` 开头但不是调用列表 → 解析错误，但 `reset()` 会把原文完整返还给 chat 层重新作为文本输出（等价于 Python 的“不匹配就当文本”）。
- 未改任何模型名路由：`llama-4` 仍走 `llama4_json`，pythonic 只能通过 `--tool-call-parser` 显式选择（和 Python 一致）。
- 输出 JSON 是紧凑格式（`serde_json`），与 Rust 侧其他合成 JSON 的 parser 一致；Python 侧是带空格的 JSON，值相同。

### PR 3（Rust OLMo 3 parser）
- 格式：与 pythonic 相同的 `name(k=v)` 调用，但多个调用按换行分隔（不是 Python 列表），整体包在 `<function_calls>`…`</function_calls>` 里，并接受 JSON 的 `true/false/null`。
- 实现：不复制语法，而是给共享的 `PythonicConfig` 加了 `PythonicSequence { List, Lines }`，控制“如何判断进入工具解析、序列开头、分隔/结束符、流结束是否关闭调用”；`olmo3.rs` 只是带 `OLMO3_CONFIG` 的薄封装（与 `llama4.rs` 同构）。
- 行为：两个标记都可选（裸的 `name(` 开头也当调用解析，对应 Python 非流式只在有包裹时去掉包裹）；换行按普通空白处理，比 Python（把非空行用 `", "` 拼起来）更宽容；流结束即关闭块，保留已解析的调用；不完整的调用 / 调用后有多余文本 → 解析错误，由 chat 层回退成文本。
- 未注册模型名路由（Python 侧也只能 `--tool-call-parser olmo3` 显式选择），并用测试钉住 `allenai/Olmo-3-*` 不会被自动路由。
- 叠加关系：分支包含 PR 2 的 commit + 本 PR 的一个 commit；上游 PR 2 合并后再开本 PR（或开的时候在正文里把 `#<pythonic-parser-PR-number>` 换成 PR 2 的编号）。

## 5. 本地复现测试的方法

```bash
# Python（PR 1）
uv venv --python 3.12 && source .venv/bin/activate
VLLM_USE_PRECOMPILED=1 uv pip install -e . --torch-backend=auto
uv pip install pytest pytest-asyncio pre-commit
pytest tests/tool_parsers/test_hermes_tool_parser.py tests/tool_parsers/test_utils.py tests/tool_parsers/test_longcat_tool_parser.py -v
pre-commit run --files vllm/tool_parsers/utils.py vllm/tool_parsers/hermes_tool_parser.py vllm/tool_parsers/longcat_tool_parser.py tests/tool_parsers/test_hermes_tool_parser.py

# Rust（PR 2 / PR 3）
cd rust
cargo fmt --all -- --check
cargo clippy --workspace --all-targets --all-features --locked -- -D warnings
cargo test -p vllm-parser --all-features --locked
cargo test -p vllm-chat --all-features --locked --lib
```

## 6. 调研摘要与后续线索

详见 `research_notes.md`。要点：

- 上游 issue 竞争极激烈（多数 bug 在 24 小时内就有 PR），所以本次选的是：一个 6 月起无人处理、根因明确的 Python bug，以及 Rust 前端 roadmap 明确列出的 parser parity 缺口（Rust 侧维护者对外部小 PR 合并很快）。
- 可继续做的方向（都已核实目前无人认领）：
  1. Granite 4 解析器的同类 `</tool_call>` 字面量 bug（复用 PR 1 的 helper）。
  2. #50989：Qwen3 strict 模式无参数工具死循环——根因在 xgrammar 内置 `qwen_3_coder` structural tag（无参数时语法要求 `<function=NAME>\n\n</function>` 两个换行）。适合去 mlc-ai/xgrammar 提 issue/PR，或在 vLLM 里做 workaround（先在 issue 里和 maintainer 讨论）。
  3. Rust 前端继续补 parser：LFM2（pythonic 变体，但 Python 侧有很多容错逻辑）、OLMo 3 reasoning parser（`<think>` 是普通文本而非特殊 token，需要文本级扫描）、`nemotron_v3` tool 别名等。
