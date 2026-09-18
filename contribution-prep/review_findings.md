# 对抗性自查结果（2026-09-18）

三个上游 PR 当前状态：#55848 open、#55849 open（`needs-rebase`）、#55850 open。
fork 上的预览 PR #1/#2/#3/#4 已全部关闭。

---

## 一、重复检查（原 PR 正文的说法是错的）

| 我的 PR | 撞上的已开 PR | 结论 |
|---|---|---|
| #55848 Hermes `</tool_call>` | #48353（非流式，Incheonkirin，由已关闭的 #45168 重开）+ #45310（流式，Sunt-ing，正文写明 "follow-up to non-streaming fix #45168"） | 完全重复，建议关闭 |
| #55850 Rust OLMo 3 | #52579（982945902，`[Rust Frontend] Add Olmo3 tool parser`，open，`needs-rebase`） | 完全重复，建议关闭 |
| #55849 Rust pythonic | 无。upstream main 至今没有 pythonic parser（`rust/src/parser/src/tool/` 下只有 json/deepseek/glm/hy/kimi/minimax/qwen_coder/seed_oss） | 不重复，但见第三节的缺陷 |

原因：我当时用了 `is:pr 45167 OR "literal </tool_call>" OR raw_decode hermes` 这类查询，
GitHub 的 OR 语义把词条静默丢弃了，返回的列表不完整。

---

## 二、验证干净的部分

- **Hermes 扫描器（#55848 的核心）**：`find_tag_outside_json_strings` 与逐字符参考实现对拍，
  穷举 20,176,794 例 + 随机 400,000 例，0 处不一致。代码本身没问题，只是重复。
- **分块等价 + 流式前缀稳定性（#55849/#55850）**：124 条输入 × 200 种随机切分 = 24,800 次分块，
  覆盖 pythonic / llama4 / olmo3 三种配置。
  - 分块结果与整段解析不一致：**0**
  - 前缀稳定性违例（已发出的 name 被改写、已发出的 arguments 不是最终值的前缀）：**0**
- **转义解码**：80 个 Python 字符串字面量与 CPython `ast.literal_eval` 对比，**69 个完全一致**，
  包括八进制（含 `\400`/`\777` 这种 CPython 视为未知转义、原样保留的情况）、`\xhh`、
  未知转义原样保留、反斜杠换行续行。
- **OLMo 3 框架行为**：12 个夹具全部符合文档描述（两个标记都可选、空行/缩进/跨行调用可接受、
  `<functions>` 近似标记走文本、`<function_calls></function_calls>` 走文本）。

---

## 三、#55849 已证实的偏差（对照 vLLM 自己的 Python `PythonicToolParser`）

### A. 静默改值（两边都解析成功，但值不同）—— 最严重

| 输入 | Python | Rust（本 PR） |
|---|---|---|
| `[pay(to='0xab', wei=123456789012345678901234567890)]` | `{"wei": 123456789012345678901234567890}` 精确 | `{"wei":1.2345678901234568e+29}` |
| `[f(x=-9223372036854775809)]` | 精确整数 | `-9.223372036854776e+18` |
| `[f(x='\N{BULLET}')]` | `"•"` | `"\\N{BULLET}"` 原样文本 |

u64/i64 范围内（含 `18446744073709551615`、`9007199254740993`）都是精确的，
窗口是超出该范围的整数。#52579 的自动 reviewer 已经就 integer precision 提过同样的问题并且改掉了。

### B. Python 能解析成工具调用、Rust 却拒绝（功能回退）

| 输入 | Python | Rust |
|---|---|---|
| `[f (x=1)]`（名字与括号间有空格） | 流式得到 `{"x": 1}` | 拒绝 → 回退文本 |
| `[f(x=(1, 2))]` 元组 | `{"x": [1, 2]}` | 拒绝 |
| `[f(x={'a', 'b'})]` 集合 | `{"x": ["a", "b"]}` | 拒绝 |
| `[f(x=f'lit')]` f-string | `{"x": "lit"}` | 拒绝 |
| `[f(x='''triple''')]` 三引号 | `{"x": "triple"}` | 拒绝 |
| `[f(x=0x1f)]` | `{"x": 31}` | 拒绝 |
| `[f(x=1_000)]` | `{"x": 1000}` | 拒绝 |
| `[f(x='\ud83d')]` 孤立代理项 | 接受（产生孤立代理项） | 拒绝 |
| `[f(x=1e400)]` | `{"x": Infinity}`（非法 JSON） | 拒绝（Rust 这里更好） |

`[f(x='C:\Users\me')]`（Windows 路径）两边都失败，**不是**本 PR 的回退。

### C. 需要修正我先前的说法

我之前说“中途拒绝时 Python 会整段干净地变成普通内容、不产生工具调用”——
那只对**非流式**路径成立。实测 Python 的**流式**路径同样会留下半开的调用：

```
[edit(path='/tmp/a.p          Python 流式 -> {"0": ["edit", "{\"path\": \"/tmp/a.p"]}
[f(s='unterminated            Python 流式 -> {"0": ["f", "{\"s\": \"unterminated"]}
```

Rust 在同样的输入上行为相当。所以“截断导致参数 JSON 不闭合”**不是** Rust 侧独有的缺陷，
不应作为本 PR 的问题提出。真正属于本 PR 的问题是 **B 表**：Rust 的拒绝面比 Python 大，
每多拒绝一种合法字面量，就把一次本可成功的工具调用变成半开的调用 + 残缺文本。

---

## 四、#55849 的合并冲突

mergify 打了 `needs-rebase`。冲突文件只有注册表/快照这四个，且都是机械冲突
（upstream 期间新增了 `deepseek_v41` 和 `hy` 两个 parser）：

- `rust/src/chat/src/lib.rs`（已注册名字的快照）
- `rust/src/chat/src/parser/tool/mod.rs`
- `rust/src/chat/src/parser/tool/tests.rs`
- `rust/src/parser/src/tool/mod.rs`

分支落后 upstream main 585 个 commit。

---

## 五、可直接复制的文本

### 5.1 关闭 #55848 的评论

```
Closing this as a duplicate. I missed two open PRs during my duplicate check:

- #48353 fixes the non-streaming path
- #45310 fixes the streaming path

Together they cover exactly the same issue (#45167) and the same scope as this PR, and
their authors coordinated the split, so there is nothing left here that is not already
being reviewed there. Sorry for the noise.
```

### 5.2 关闭 #55850 的评论

```
Closing this as a duplicate of #52579, which already adds an OLMo 3 tool parser to the
Rust frontend. I missed it during my duplicate check. Sorry for the noise.
```

### 5.3 （可选）在 #45310 上的评论

```
Not a review request, just a data point in case it is useful: I wrote the same
string-aware scan independently and cross-checked it against a character-by-character
reference implementation over 20,176,794 exhaustive inputs plus 400,000 randomized ones,
with no mismatches. Two cases that are easy to get wrong and are worth a regression test
either way: an end tag that appears after an escaped backslash (`"...\\\\</tool_call>"`,
where the tag really is outside the string), and a partial `</tool_call>` suffix arriving
while the scan is still inside an unterminated string — withholding it there truncates
the argument stream.
```

### 5.4 （可选）在 #52579 上的评论

```
Not a review request. I had independently written a pythonic-family parser for the Rust
frontend (#55849) and closed my OLMo 3 variant in favour of this PR. One thing that fell
out of testing my version against the Python `PythonicToolParser`: the Python side accepts
tuples, sets, f-strings, triple-quoted strings, hex literals and underscored integers as
argument values, and converts tuples/sets to JSON arrays. If your literal parser rejects
any of those, the call falls back to plain text, which in streaming leaves a tool call with
unclosed JSON arguments already emitted. Happy to share the differential test corpus if it
is useful.
```

### 5.5 #55849 正文里 duplicate check 段落的更正版

替换原来的那一段：

```
Duplicate check: upstream `main` still has no pythonic parser under
`rust/src/parser/src/tool/`, and no open PR adds one. The related open `[Rust Frontend]`
parser PRs are #52579 (OLMo 3, which uses the same call syntax but its own literal
parser), #52841 (ERNIE 4.5) and #54393 (MiMo). If #52579 lands first I am happy to rebase
this on top of it and have the OLMo 3 parser reuse this shared core instead of duplicating
the literal grammar.
```

---

## 六、如果要修 #55849，改动清单

1. **大整数**（必须修，静默改值）：整数字面量超出 i64/u64 时不要落到 f64。
   `value.rs` 目前产出 `serde_json::Value`；最小改法是让整数保留字面量原文并直接写进
   参数 JSON 文本（参数本来就是按文本片段拼的），或者启用 serde_json 的 `arbitrary_precision`。
2. **`\N{...}`**（静默改值）：没有 Unicode 名字表就别原样透传，改成解析错误，
   让 chat 层回退成文本，至少不会把错误的值当成功的工具调用发出去。
3. **`f (x=1)`**：名字与 `(` 之间允许空白，一行改动。
4. **元组 / 集合 / 十六进制 / 下划线数字 / 三引号 / f-string**：按 Python 侧的语义补进语法
   （元组和集合转 JSON 数组，与 Python 一致）。这几项各自都是局部改动。
5. 合并 upstream main，解掉第四节那四个注册表冲突。
6. 每项都要补 `expect-test` 快照；现有 `parse_chunkings` 已经覆盖分块等价，不需要新框架。
