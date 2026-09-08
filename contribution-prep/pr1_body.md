## Purpose

Fixes #45167.

`Hermes2ProToolParser` located the end of a `<tool_call>` block with a non-greedy regex (`<tool_call>(.*?)</tool_call>`) in the non-streaming path and with a plain `str.find("</tool_call>")` in the streaming path. When a JSON string argument contains the literal text `</tool_call>` (for example a file-editing tool whose `content` argument quotes the tag), the block was cut at the first occurrence, `json.loads` failed with `Unterminated string`, and the whole tool call was silently dropped (`tools_called=False`, raw markup returned as content). In streaming the argument stream was truncated at the same point.

This PR scans the tool-call body with JSON string/escape awareness instead:

- `vllm/tool_parsers/utils.py`: new `find_tag_outside_json_strings(text, tag, start)` helper (next to `partial_tag_overlap`). It jumps between `"` characters with `str.find`, tracks string/escape state (reusing the existing `_is_escaped` backslash-parity check), and returns the first tag that is outside a string literal, plus whether the scan ended inside an unterminated string.
- `vllm/tool_parsers/hermes_tool_parser.py`: one region scanner `_iter_tool_call_bodies` shared by `extract_tool_calls` and `_extract_tool_call_jsons`. A region without an end tag still extends to end-of-text exactly as before; invalid JSON still falls through to the raw-content fallback. In streaming, a partial `</tool_call>` suffix is only withheld when the scan is not inside a string literal (inside a string those characters are argument data). The now-unused `tool_call_regex` attribute is removed.
- `vllm/tool_parsers/longcat_tool_parser.py`: drops its `tool_call_regex` override (it subclasses the Hermes parser and now works purely off `tool_call_start_token` / `tool_call_end_token`).

The start tag is deliberately still located with plain `str.find`: prose before a tool call may contain unbalanced quotes, and a start tag inside an already-terminated body is skipped because scanning resumes after the end tag.

This matches what the Rust frontend's Hermes parser already does (`take_json_object` in `rust/src/parser/src/utils.rs` tracks string/escape state), so Python and Rust now agree on this input.

Duplicate check: no open PR references #45167; the only related open PR is #51937 (migrating the Hermes parser to the streaming parser engine), which does not address literal end tags inside strings (the engine lexer also matches `</tool_call>` irrespective of JSON string state), so this fix is independent of it.

## Test Plan

New regression tests in `tests/tool_parsers/test_hermes_tool_parser.py` (all fail on `main` with `JSONDecodeError: Unterminated string`):

- non-streaming: the exact reproduction from the issue (literal `</tool_call>` inside a Korean string argument);
- non-streaming: a quoted `</tool_call>` in the first of two consecutive tool calls;
- non-streaming: escaped quotes (`\"hi\"`) right before the literal tag;
- streaming (token by token): the issue's text, asserting one tool call with the tag preserved in `content`;
- streaming: a quoted `<tool_call>` must not open a second tool call.

```bash
pytest tests/tool_parsers/test_hermes_tool_parser.py tests/tool_parsers/test_utils.py tests/tool_parsers/test_longcat_tool_parser.py -v
pre-commit run --files vllm/tool_parsers/utils.py vllm/tool_parsers/hermes_tool_parser.py vllm/tool_parsers/longcat_tool_parser.py tests/tool_parsers/test_hermes_tool_parser.py
pre-commit run mypy-3.12 --hook-stage manual --files vllm/tool_parsers/utils.py vllm/tool_parsers/hermes_tool_parser.py
```

## Test Result

- `tests/tool_parsers/test_hermes_tool_parser.py`: 35 passed, 1 skipped (30 passed, 1 skipped before; 5 new tests).
- `test_hermes_tool_parser.py` + `test_utils.py` + `test_longcat_tool_parser.py`: 225 passed, 1 skipped, 1 xfailed.
- Full `tests/tool_parsers` run before/after the source change: identical pass/fail sets (no behaviour change outside the fixed case).
- The scanner was cross-checked against a naive character-by-character reference on ~200k randomized inputs, and char-by-char streaming matches non-streaming parsing on the tricky bodies (escaped backslash tails, tag-only strings, two calls, content prefix).
- Because the streaming parser rescans the accumulated text on every token, the helper is `str.find`-based rather than a per-character Python loop: ~0.27 ms per scan on a 25 KB `edit_file`-style body versus ~1.65 ms for a character loop.
- `pre-commit` (ruff, ruff-format, typos, mypy 3.10, SPDX and the other repo hooks) passes; `mypy-3.12` manual hook passes.
- No model output/accuracy impact: the change only affects inputs that previously failed to parse.

AI assistance was used while developing this change (drafting and test scaffolding); I reviewed every line and ran the tests above myself.
