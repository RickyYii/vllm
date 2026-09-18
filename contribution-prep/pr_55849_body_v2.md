## Purpose

Part of the Rust frontend parser-parity work tracked in #44280.

The Python frontend ships `pythonic` and `llama4_pythonic` tool parsers for models whose tool calls are a Python list of keyword-argument calls (Llama 3.2, Llama 4, ToolACE, ...), but the Rust frontend has no parser for that format: `--tool-call-parser pythonic` / `llama4_pythonic` are not registered and `llama-4` can only be served through the JSON parser. This PR ports the format to the Rust frontend as a streaming parser:

- `rust/src/parser/src/tool/pythonic/mod.rs`: `PythonicToolParser`, a winnow/`Partial` buffered-event parser in the same shape as `Llama3JsonToolParser` (explicit mode enum, `parse_buffered_event` loop). The list must be the whole output; leading text permanently switches the parser to passthrough, exactly like the JSON parsers. Text that opens with `[` but is not a call list (`[1, 2, 3]`, `[]`, `[not a call`) is a `ParsingFailed` error; because the opening `[` is only consumed together with the first `name(`, `reset()` still returns the whole original text and the chat layer re-emits it as content, which reproduces the Python parser's "does not match the pattern, treat as text" behaviour. Trailing non-whitespace after `]` and an incomplete call at `finish()` are errors, following the same convention as `Llama3JsonToolParser`.
- `rust/src/parser/src/tool/pythonic/value.rs`: Python literal parsers producing compact JSON text — single/double-quoted strings with full Python escape decoding (`\xhh`, octal, `\uXXXX` with surrogate pairing, `\UXXXXXXXX`, line continuation, unknown escapes kept verbatim as Python does); decimal, binary, octal and hexadecimal integers with `_` separators; floats; signed numbers; `True`/`False`/`None` plus JSON-style `true`/`false`/`null`; lists, tuples and dicts with string keys, bounded by the existing `ParserRecursionGuard`; and an incremental Python-string decoder used for streaming.
- `rust/src/parser/src/tool/pythonic/llama4.rs`: `Llama4PythonicToolParser`, a thin wrapper over the same core that also strips the `<|python_start|>` / `<|python_end|>` markers Llama 4 sometimes emits.
- Streaming is fully incremental: the name delta is emitted at `name(`, then `{`, then one `"key":<json>` fragment per completed argument (with `,` separators) and `}` at `)`. String arguments are streamed as the Python literal arrives (opening quote, decoded runs, closing quote), so long string arguments such as file contents do not stall the stream; non-string values (numbers, keywords, nested containers) are emitted once complete. Arguments are compact JSON, matching the other Rust parsers that synthesize JSON (`qwen_coder`, `glm_xml`).
- Registration: `pythonic` and `llama4_pythonic` in `rust/src/chat/src/parser/tool/mod.rs`. No model-name routing changes: `llama-4` still resolves to `llama4_json`, and the pythonic parsers are selected explicitly with `--tool-call-parser`, the same as in Python. `structural_tag_builder()` stays `None` (xgrammar has no structural tag for this format; the Python parser also has `structural_tag_model = None`).
- `rust/src/parser/benches/pythonic.rs`: native-vs-external comparison bench in the same shape as the existing per-parser benches.

### Agreement with the Python parser

Argument values were compared case by case against vLLM's own `PythonicToolParser`, which drove several behaviour decisions:

- **Integers are carried through as text rather than as a `serde_json::Value`.** `serde_json::Number` stops at `u64`, so a wider integer would have to be rounded into an `f64` and the tool would be called with a different amount than the model wrote — `wei=123456789012345678901234567890` became `1.2345678901234568e+29`. The Python parser keeps such values exact, and so does this now.
- **Tuples parse as JSON arrays**, as the Python parser does. Parentheses without a trailing comma group a value rather than building a tuple, matching Python: `(1)` is `1`, `(1,)` is `[1]`.
- **Binary, octal and hexadecimal integers and `_` digit separators are accepted**, which the Python parser resolves through `ast`.
- **Whitespace is allowed between a function name and its `(`**, which Python accepts.
- **`\N{NAME}` escapes are rejected.** Decoding them needs a Unicode name table; passing the escape through verbatim would hand the tool a different string than the model wrote, so the output falls back to content instead. This is the one place where a value the Python parser decodes is deliberately refused, and it is marked as a `TODO` in the code.
- **An invalid escape after already-decoded text no longer discards that text.** It used to be committed when a chunk boundary happened to fall before the escape and dropped otherwise, so the same output parsed differently depending on chunking.

Sets, placeholder-free f-strings, triple-quoted strings and non-string dict keys are still rejected and left as `TODO`s; models emitting them fall back to plain text.

### Duplicate check

Upstream `main` has no pythonic parser under `rust/src/parser/src/tool/`, and no open PR adds one. The related open `[Rust Frontend]` parser PRs are #52579 (OLMo 3, which uses the same keyword-argument call syntax but brings its own literal parser), #52841 (ERNIE 4.5) and #54393 (MiMo). If #52579 lands first I am happy to rebase on top of it and have the OLMo 3 parser reuse this shared core instead of carrying a second Python-literal grammar.

## Test Plan

Unit tests live next to the code (`expect-test` snapshots, `rust/AGENTS.md` conventions): simple / parallel / parameterless calls, empty dict and list arguments, nested containers, escaped strings (the `ESCAPED_STRING_FUNCTION_OUTPUT` case from the Python tests plus `\n`/`\t`/unicode/octal/hex escapes), `True`/`False`/`None` and JSON literals, signed numbers, integers wider than `u64`, tuples and grouped parentheses, radix and underscored integers, whitespace between calls, arguments and before the argument list, plain-text passthrough, non-call lists and malformed input (asserting the original text is handed back), invalid escapes recovering as text, `\N{...}` rejection, the Llama 4 markers split across chunk boundaries, `finish()` on incomplete calls and trailing text, and a 4096-character string argument that must be committed before the call closes. Every parse test runs through a `parse_chunkings` helper that asserts the coalesced output is byte-identical for the whole text, 3-character chunks and 1-character chunks. Registry tests in `rust/src/chat/src/parser/tool/tests.rs` cover creation by name and that `llama-4` model routing is unchanged.

Beyond the checked-in tests, the parser was compared against the Python `PythonicToolParser` on a differential corpus: 21 literal cases that the two parsers previously disagreed on, and 133 inputs run through 200 random chunk splits each, asserting that every chunking commits the same output and that no already-emitted tool name or argument fragment is ever revised.

```bash
cd rust
cargo fmt --all -- --check
cargo clippy -p vllm-parser -p vllm-chat --all-targets --all-features --locked -- -D warnings
cargo test -p vllm-parser --all-features --locked
cargo test -p vllm-chat --all-features --locked --lib
cargo bench -p vllm-parser --features test-util --bench pythonic -- --test
cargo doc -p vllm-parser --no-deps
pre-commit run --files <changed files>
```

## Test Result

- `cargo fmt --all -- --check`: clean.
- `cargo clippy -p vllm-parser -p vllm-chat --all-targets --all-features --locked -- -D warnings`: no warnings.
- `cargo test -p vllm-parser --all-features --locked`: 495 passed, 0 failed.
- `cargo test -p vllm-chat --all-features --locked --lib`: 349 passed, 0 failed.
- `cargo bench -p vllm-parser --features test-util --bench pythonic -- --test`: all groups succeed.
- `cargo doc -p vllm-parser --no-deps`: no warnings.
- `pre-commit run --files <changed files>`: all applicable hooks pass.
- Differential corpus against the Python parser: 21 of 21 literal cases now produce identical arguments; 133 inputs x 200 random chunk splits produced 0 chunking disagreements and 0 streaming prefix violations.
- The `vllm-chat` `roundtrip` integration tests need Hugging Face Hub access and could not run in my sandbox (pre-existing, unrelated).
- No Python code is touched, so there is no model-output or accuracy impact for the Python frontend; the Rust frontend gains a new opt-in parser.

AI assistance was used while developing this change (drafting the parser and tests); I reviewed every line and ran the commands above myself.
