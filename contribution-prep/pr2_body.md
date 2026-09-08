## Purpose

Part of the Rust frontend parser-parity work tracked in #44280.

The Python frontend ships `pythonic` and `llama4_pythonic` tool parsers for models whose tool calls are a Python list of keyword-argument calls (Llama 3.2, Llama 4, ToolACE, ...), but the Rust frontend had no parser for that format: `--tool-call-parser pythonic` / `llama4_pythonic` were not registered and `llama-4` could only be served through the JSON parser. This PR ports the format to the Rust frontend as a streaming parser:

- `rust/src/parser/src/tool/pythonic/mod.rs`: `PythonicToolParser`, a winnow/`Partial` buffered-event parser in the same shape as `Llama3JsonToolParser` (explicit mode enum, `parse_buffered_event` loop). The list must be the whole output; leading text permanently switches the parser to passthrough, exactly like the JSON parsers. Text that opens with `[` but is not a call list (`[1, 2, 3]`, `[]`, `[not a call`) is a `ParsingFailed` error; because the opening `[` is only consumed together with the first `name(`, `reset()` still returns the whole original text and the chat layer re-emits it as content, which reproduces the Python parser's "does not match the pattern, treat as text" behaviour. Trailing non-whitespace after `]` and an incomplete call at `finish()` are errors, following the same convention as `Llama3JsonToolParser`.
- `rust/src/parser/src/tool/pythonic/value.rs`: Python literal parsers producing `serde_json::Value` (single/double-quoted strings with full Python escape decoding incl. `\xhh`, octal, `\uXXXX` with surrogate pairing, `\UXXXXXXXX`, line continuation; ints, floats and signed numbers; `True`/`False`/`None` plus JSON-style `true`/`false`/`null`; nested lists and dicts with string keys, bounded by the existing `ParserRecursionGuard`), and an incremental Python-string decoder used for streaming. Tuples, sets, f-strings, non-string dict keys and `\N{...}` escapes are rejected and left as `TODO`s in the code.
- `rust/src/parser/src/tool/pythonic/llama4.rs`: `Llama4PythonicToolParser`, a thin wrapper over the same core that also strips the `<|python_start|>` / `<|python_end|>` markers Llama 4 sometimes emits.
- Streaming is fully incremental: the name delta is emitted at `name(`, then `{`, then one `"key":<json>` fragment per completed argument (with `,` separators) and `}` at `)`. String arguments are streamed as the Python literal arrives (opening quote, decoded runs, closing quote), so long string arguments such as file contents do not stall the stream; non-string values (numbers, keywords, nested lists/dicts) are emitted once complete. Arguments are compact `serde_json` text, matching the other Rust parsers that synthesize JSON (`qwen_coder`, `glm_xml`).
- Registration: `pythonic` and `llama4_pythonic` in `rust/src/chat/src/parser/tool/mod.rs`. No model-name routing changes: `llama-4` still resolves to `llama4_json`, and the pythonic parsers are selected explicitly with `--tool-call-parser`, the same as in Python. `structural_tag_builder()` stays `None` (xgrammar has no structural tag for this format; the Python parser also has `structural_tag_model = None`).
- `rust/src/parser/benches/pythonic.rs`: native-vs-external comparison bench in the same shape as the existing per-parser benches.

Duplicate check: no open PR adds a pythonic parser to the Rust frontend (`is:pr pythonic in:title` only lists Python-side fixes); the open `[Rust Frontend]` parser PRs cover ERNIE 4.5 (#52841), Hunyuan A13B (#52133) and MiMo (#54393).

## Test Plan

Unit tests live next to the code (`expect-test` snapshots, `rust/AGENTS.md` conventions): simple / parallel / parameterless calls, empty dict and list arguments, nested containers, escaped strings (the `ESCAPED_STRING_FUNCTION_OUTPUT` case from the Python tests plus `\n`/`\t`/unicode/octal/hex escapes), `True`/`False`/`None` and JSON literals, signed numbers, whitespace and newlines between calls and arguments, plain-text passthrough, non-call lists and malformed input (asserting the original text is handed back), the Llama 4 markers split across chunk boundaries, `finish()` on incomplete calls and trailing text, and a 4096-character string argument that must be committed before the call closes. Every parse test runs through a `parse_chunkings` helper that asserts the coalesced output is byte-identical for the whole text, 3-character chunks and 1-character chunks. Registry tests in `rust/src/chat/src/parser/tool/tests.rs` cover creation by name and that `llama-4` model routing is unchanged.

```bash
cd rust
cargo fmt --all -- --check
cargo clippy --workspace --all-targets --all-features --locked -- -D warnings
cargo test -p vllm-parser --all-features --locked
cargo test -p vllm-chat --all-features --locked --lib
cargo bench -p vllm-parser --features test-util --bench pythonic -- --test
pre-commit run --files <changed files>
```

## Test Result

- `cargo fmt --all -- --check`: clean.
- `cargo clippy --workspace --all-targets --all-features --locked -- -D warnings`: clean.
- `cargo test -p vllm-parser --all-features --locked`: 467 passed, 0 failed (26 new pythonic tests).
- `cargo test -p vllm-chat --all-features --locked --lib`: 293 passed, 0 failed.
- `cargo test --workspace --all-features --locked --lib --no-fail-fast`: everything passes except the pre-existing tests that download `Qwen/Qwen3-0.6B` from the Hugging Face Hub (`vllm-text` `lower_text_request_uses_real_qwen_generation_defaults` and the `vllm-chat` `roundtrip` integration tests), which cannot run in my sandbox and are unrelated to this change.
- `cargo bench -p vllm-parser --features test-util --bench pythonic -- --test`: all groups succeed.
- `pre-commit run --files <changed files>`: all applicable hooks pass (SPDX headers and the rust fmt hook included).
- No Python code is touched, so there is no model-output or accuracy impact for the Python frontend; the Rust frontend gains a new opt-in parser.

AI assistance was used while developing this change (drafting the parser and tests); I reviewed every line and ran the commands above myself.
