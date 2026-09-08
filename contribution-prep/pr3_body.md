## Purpose

Part of the Rust frontend parser-parity work tracked in #44280. Stacked on the pythonic tool parser PR (#<pythonic-parser-PR-number>): only the last commit belongs to this PR.

The Python frontend has an `olmo3` tool parser (`Olmo3PythonicToolParser`) for OLMo 3 models, whose tool calls use the same keyword-argument call syntax as the pythonic format but with a different framing:

```text
<function_calls>
get_weather(city='San Francisco', metric='celsius')
get_weather(city='New York', metric='celsius')
</function_calls>
```

Parallel calls are newline-separated instead of items of a Python list, the calls are wrapped in `<function_calls>` / `</function_calls>`, and JSON `true` / `false` / `null` literals are accepted next to the Python ones. This PR adds that parser to the Rust frontend on top of the shared pythonic core:

- `rust/src/parser/src/tool/pythonic/mod.rs`: the shared `PythonicConfig` gains a `PythonicSequence { List, Lines }` field that selects how consecutive calls are delimited (a bracketed Python list, or whitespace-separated calls closed by the end marker or the end of the stream). It drives the commit-to-parsing check, the sequence opener, the separator/terminator, and whether the end of the stream closes the calls. Value, call and string parsing (including incremental string streaming) are untouched and shared.
- `rust/src/parser/src/tool/pythonic/olmo3.rs`: `Olmo3PythonicToolParser`, a thin wrapper over the core with `OLMO3_CONFIG` (start/end markers plus `Lines`), in the same shape as `Llama4PythonicToolParser`.
- Registration as `olmo3` in `rust/src/chat/src/parser/tool/mod.rs` (name only; no model-name pattern, matching Python where `olmo3` is selected explicitly with `--tool-call-parser`, and a factory test pins that `allenai/Olmo-3-*` resolves to no parser so a future routing change is deliberate). `structural_tag_builder()` stays `None`; the markers are ordinary text, so `preserve_special_tokens()` stays `false`.
- `rust/src/parser/benches/olmo3.rs`: native-only bench (the external `tool-parser` crate has no OLMo 3 parser to compare against), same shape as `granite4.rs`.

Behaviour decisions, all documented in the struct docs:

- Both markers are optional: text that opens with `<function_calls>` or directly with `name(` is parsed as calls (the non-streaming Python path also only strips the wrapper when present); any other leading text permanently switches to plain-text passthrough, like the other parsers in this frontend. A partial `<function_calls>` / `</function_calls>` prefix is withheld until it is decided, so marker fragments never leak into content.
- Newlines are ordinary whitespace in the shared grammar, so blank lines, indentation, trailing spaces and calls spanning several lines are accepted. This is strictly more tolerant than the Python parser, which joins non-empty stripped lines with `", "` and therefore breaks a call split across lines.
- A block cut short by the end of the stream keeps the calls parsed so far; an incomplete call and non-whitespace text after the calls are `ParsingFailed` errors that the streaming layer recovers as content, exactly like the pythonic parser.
- Arguments are compact `serde_json` text, as in the pythonic parser.

Duplicate check: no open PR adds an OLMo 3 parser to the Rust frontend; the open `[Rust Frontend]` parser PRs cover ERNIE 4.5 (#52841), Hunyuan A13B (#52133) and MiMo (#54393).

## Test Plan

17 `expect-test` snapshot tests in `olmo3.rs`, reusing the fixtures of `tests/tool_parsers/test_olmo3_tool_parser.py`: simple call, more types, JSON literals (asserted equal to the Python-literal parse), parameterless / empty dict / empty list, escaped strings, parallel calls on separate lines, blank lines + indentation + trailing whitespace, missing wrapper, block closed by end of stream, plain text, text before the block, near-miss blocks (`<function_calls></function_calls>`, `<functions>...`, bare `<function_calls>`), delta-level snapshots of streamed name/argument fragments, the two-chunk `test_streaming_tool_call_with_large_steps` fixture, both `finish()` error messages, and construction through `ToolParser::create`. Every parse test runs through `parse_chunkings` (whole text, 3-character and 1-character chunks must produce identical output). Two registry tests in `rust/src/chat/src/parser/tool/tests.rs` plus the updated registered-name snapshot in `rust/src/chat/src/lib.rs`.

```bash
cd rust
cargo fmt --all -- --check
cargo clippy -p vllm-parser -p vllm-chat --all-targets --all-features --locked -- -D warnings
cargo test -p vllm-parser --all-features --locked
cargo test -p vllm-chat --all-features --locked --lib
cargo bench -p vllm-parser --features test-util --bench olmo3 -- --test
cargo doc -p vllm-parser --no-deps
pre-commit run --files <changed files>
```

## Test Result

- `cargo fmt --all -- --check`: clean.
- `cargo clippy -p vllm-parser -p vllm-chat --all-targets --all-features --locked -- -D warnings`: no warnings.
- `cargo test -p vllm-parser --all-features --locked`: 484 passed, 0 failed (467 before, 17 new).
- `cargo test -p vllm-chat --all-features --locked --lib`: 295 passed, 0 failed (2 new factory tests).
- `cargo bench -p vllm-parser --features test-util --bench olmo3 -- --test`: all 6 cases succeed.
- `cargo doc -p vllm-parser --no-deps`: no warnings.
- `pre-commit run --files <changed files>`: all applicable hooks pass.
- The `vllm-chat` `roundtrip` integration tests need Hugging Face Hub access and could not run in my sandbox (pre-existing, unrelated).
- No Python code is touched; the Rust frontend gains a new opt-in parser, so there is no model-output or accuracy impact elsewhere.

AI assistance was used while developing this change (drafting the parser and tests); I reviewed every line and ran the commands above myself.
