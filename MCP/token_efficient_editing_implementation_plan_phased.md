# Implementation Plan: Token-Efficient Symbol Editing (Phased for Agentic LLM)

## Objective
Implement the `ChunkDiffSymbolTool` for token-efficient, chunk-based symbol editing in Serena.

## Core Logic Files
- `tools/chunk_diff_symbol_tool.py` (main tool class and helper methods)
- `tests/tools/test_chunk_diff_symbol_tool.py` (unit and integration tests)

## Phased Implementation

### Stage 0: Foundational Utilities

**Goal:** Implement and test core string/line manipulation helpers.

1.  **Implement `_detect_line_ending(self, text)` in `chunk_diff_symbol_tool.py`:**
    *   Detects CRLF, LF, CR. Prioritize CRLF > LF > CR. Fallback to LF.
2.  **Implement `_lines_match(self, actual_lines, expected_lines)` in `chunk_diff_symbol_tool.py`:**
    *   Compares lists of strings, using `rstrip()` on each line before comparison.
3.  **Unit Test Stage 0:**
    *   In `test_chunk_diff_symbol_tool.py`, create `TestDetectLineEnding` and `TestLinesMatch` test classes.
    *   Cover various line ending combinations for `_detect_line_ending`.
    *   Cover matching, non-matching, and whitespace-only differences for `_lines_match`.

---

### Stage 1: Single Chunk - Basic Positioning & Application

**Goal:** Validate core concept of applying one chunk, focusing on positioning and basic replacement.

1.  **Implement simplified `_validate_and_position_chunk_stage1(self, original_lines, chunk)` in `chunk_diff_symbol_tool.py`:**
    *   Input: `original_lines` (list of strings), `chunk` (dict).
    *   Normalize `chunk["context_before"]` to a list if it's a string.
    *   Find candidate positions based *only* on `context_before` (use `_lines_match` for context lines).
    *   For the *first* candidate, perform basic validation:
        *   Do `chunk["old_lines"]` (using `_lines_match`) match the lines immediately following the found `context_before` end?
    *   Return `(position, None)` if valid, or `(None, error_message)`. `position` is the index of the last line of `context_before`.
2.  **Implement simplified `apply_single_chunk_stage1(self, symbol_body, chunk)` in `chunk_diff_symbol_tool.py`:**
    *   Input: `symbol_body` (string), `chunk` (dict).
    *   Call `_detect_line_ending(symbol_body)`.
    *   Split `symbol_body` into `original_lines`.
    *   Call `_validate_and_position_chunk_stage1(original_lines, chunk)`.
    *   If valid:
        *   Create `working_lines = list(original_lines)`.
        *   Replace `old_lines` with `new_lines` at `position + 1` in `working_lines`.
        *   Join `working_lines` using the detected original line ending. Handle trailing line ending.
        *   Return `new_symbol_body`.
    *   If invalid, return error message.
3.  **Implement basic `ChunkDiffSymbolTool.apply_stage1(self, relative_path, line, col, chunk_dict)`:**
    *   This method will be temporary for testing Stage 1.
    *   Retrieve symbol body using `SymbolManager`.
    *   Call `apply_single_chunk_stage1`.
    *   If successful, use `SymbolManager.replace_body`.
4.  **Test Stage 1:**
    *   In `test_chunk_diff_symbol_tool.py`:
        *   Test `apply_stage1` with simple replacements, insertions, deletions.
        *   Test correct positioning.
        *   Test with different file line endings.
        *   Test basic invalid scenarios (e.g., `context_before` not found, `old_lines` mismatch).

---

### Stage 2: Single Chunk - Full Validation Robustness

**Goal:** Implement all 7 validation checks for a single chunk.

1.  **Implement `_run_comprehensive_validation(self, original_lines, chunk, position, symbol_body_for_line_ending_detection)` in `chunk_diff_symbol_tool.py`:**
    *   `position` is the index of the last line of `context_before`.
    *   `symbol_body_for_line_ending_detection` is the original full symbol string.
    *   Normalize `chunk["context_before"]` and `chunk["context_after"]` to lists.
    *   Implement all 7 checks as per `token_efficient_editing_tech_spec.md`:
        1.  Context After Validation (using `_lines_match`).
        2.  Old Lines Matching (using `_lines_match`).
        3.  Insertion Logic (empty `old_lines`).
        4.  Deletion Logic (empty `new_lines`).
        5.  Symbol Boundary Validation.
        6.  End-of-Symbol Context Validation.
        7.  Line Ending Consistency (using `_detect_line_ending` on `symbol_body_for_line_ending_detection` and checking all lines in the chunk).
    *   Return `None` if all pass, or an `error_message` string.
2.  **Refactor `_validate_and_position_chunk` (rename from `_validate_and_position_chunk_stage1`) in `chunk_diff_symbol_tool.py`:**
    *   Input: `original_lines`, `symbol_body_for_line_ending_detection`, `chunk`.
    *   Find *all* candidate positions based on `context_before` (using `_lines_match`).
    *   For each candidate `position`:
        *   Call `_run_comprehensive_validation(original_lines, chunk, position, symbol_body_for_line_ending_detection)`.
        *   If validation returns `None` (success), return `(position, None)`.
    *   If no candidates pass, return `(None, "All candidate positions failed validation or context_before not found")`.
3.  **Refactor `apply_chunks_to_symbol` (rename from `apply_single_chunk_stage1` but still for single chunk in this stage) in `chunk_diff_symbol_tool.py`:**
    *   Input: `symbol_body` (string), `chunk` (dict).
    *   `original_lines = symbol_body.splitlines()`.
    *   Call `_validate_and_position_chunk(original_lines, symbol_body, chunk)`.
    *   If valid, apply the chunk and reconstruct body as before.
4.  **Test Stage 2:**
    *   In `test_chunk_diff_symbol_tool.py`, create focused tests for each of the 7 validation checks in `_run_comprehensive_validation`.
    *   Test scenarios where `context_before` is ambiguous but the full chunk validation disambiguates.
    *   Test error messages.

---

### Stage 3: Multiple Chunk Support (Orchestration & Atomicity)

**Goal:** Handle a list of chunks atomically and in correct order.

1.  **Modify `apply_chunks_to_symbol` in `chunk_diff_symbol_tool.py`:**
    *   Input: `symbol_body` (string), `chunks` (list of dicts).
    *   `original_lines = symbol_body.splitlines()`.
    *   `chunk_applications = []`.
    *   Loop `for i, chunk in enumerate(chunks)`:
        *   Call `_validate_and_position_chunk(original_lines, symbol_body, chunk)`.
        *   If `position is None` (validation failed):
            *   Log error.
            *   Return `f"Chunk {i+1} validation failed: {error_from_validation}"`. (Atomic failure).
        *   Else, `chunk_applications.append((position, chunk))`.
    *   If loop completes (all chunks validated):
        *   `working_lines = list(original_lines)`.
        *   Sort `chunk_applications` by `position` in `reverse=True`.
        *   Loop `for position, chunk in sorted_chunk_applications`:
            *   Apply chunk: `working_lines[position+1 : position+1+len(chunk["old_lines"])] = chunk["new_lines"]`.
        *   Reconstruct `new_symbol_body` using original line endings and handling trailing line ending.
        *   Return `new_symbol_body`.
2.  **Implement final `ChunkDiffSymbolTool.apply(self, relative_path, line, column, chunks)`:**
    *   Input validation for `chunks` (list, non-empty, keys exist).
    *   Normalize `context_before` and `context_after` in all chunks to lists.
    *   Retrieve `symbol_body` via `SymbolManager`.
    *   Call `apply_chunks_to_symbol(symbol_body, chunks)`.
    *   If successful (not an error string), use `SymbolManager.replace_body`.
    *   Return success/error message.
    *   Remove `apply_stage1`.
3.  **Test Stage 3:**
    *   Test with multiple non-overlapping chunks.
    *   Test atomicity: if one chunk in a list is invalid, no changes are applied.
    *   Test correct application order (reverse positional).
    *   Test with potentially overlapping chunk definitions.

---

### Stage 4: Integration, Final Touches & Broader Testing

**Goal:** Full integration, polish, documentation, and end-to-end testing.

1.  **Integrate `ChunkDiffSymbolTool` into Serena's tool registry.**
2.  **Ensure comprehensive logging throughout the tool's operations.**
3.  **Review and refine all error messages for clarity, especially for LLM consumption.**
4.  **Write developer documentation and LLM usage guidelines for the chunk format.**
5.  **Test Stage 4:**
    *   End-to-end tests invoking the tool as an MCP client would.
    *   Test with diverse code symbols (Python, Java, TypeScript, etc.).
    *   Performance tests (large symbols, many chunks).
    *   Test with manually crafted "challenging" chunks (e.g., minimal unique context).