# Technical Specification: Chunk-Based Symbol Editing

## Overview

This document provides technical specifications for implementing the chunk-based differential editing capability in Serena. The feature aims to dramatically reduce token usage during code editing operations by transmitting only the changed portions of code symbols rather than entire symbol bodies.

## Why Not Traditional Diffs?

The chunk-based format was inspired by traditional unified diff patches but specifically designed to address LLM limitations:

1. **LLMs struggle with line counting**: Traditional diffs rely on accurate line numbers, which LLMs cannot reliably generate
2. **Token efficiency contradiction**: Diff libraries require both the original and modified content to generate patches, defeating our token-saving purpose
3. **Context-based positioning**: Our format uses surrounding context lines to locate changes, similar to how developers visually identify code locations

The resulting format maintains the conceptual clarity of diffs while being practical for LLM-generated edits.

## Architecture

### New Component: `ChunkDiffSymbolTool`

```
SerenaAgent
└── Tools
    └── ChunkDiffSymbolTool (new)
        ├── apply()
        └── apply_chunks_to_symbol()
```

### Relationship to Existing Components

The new tool will:
- Utilize `SymbolManager` for locating and manipulating symbols
- Maintain the same security and validation patterns as other editing tools
- Follow the existing pattern of marking files as modified after edits

## Technical Design

### 1. API Definition

```python
class ChunkDiffSymbolTool(Tool, ToolMarkerCanEdit):
    """Apply diff chunks to a symbol without requiring the entire symbol body."""

    def apply(
        self,
        relative_path: str,
        line: int,
        column: int,
        chunks: list[dict],
    ) -> str:
        """
        Apply changes to a symbol using diff chunks.
        
        :param relative_path: Path to the file containing the symbol
        :param line: Symbol start line number
        :param column: Symbol start column
        :param chunks: List of diff chunks, each containing:
                      {
                        "context_before": str | list[str],  # Line(s) before change
                        "old_lines": list[str], # Lines to be replaced/deleted
                        "new_lines": list[str], # Lines to insert/replace with
                        "context_after": str | list[str],   # Line(s) after change
                      }
        :return: Success message or error description
        """
```

### 2. Implementation Details
#### 2.1 Core Algorithm - Enhanced Robust Design

```python
def apply_chunks_to_symbol(self, symbol_location, chunks):
    """Apply a series of diff chunks to a symbol with comprehensive validation."""
    # Phase 1: Get original state
    symbol = self.symbol_manager.find_by_location(symbol_location)
    if not symbol or not symbol.body:
        return "Error: Symbol not found or has no body"
    
    original_lines = symbol.body.splitlines()
    original_line_ending = self._detect_line_ending(symbol.body)
    
    # Phase 2: Validate ALL chunks before any modifications
    chunk_applications = []
    for i, chunk in enumerate(chunks):
        position, error = self._validate_and_position_chunk(original_lines, symbol, chunk)
        if position is None:
            log.error(f"Chunk {i+1} validation failed: {error}")
            return f"Chunk {i+1} validation failed: {error}"
        chunk_applications.append((position, chunk))
        log.debug(f"Chunk {i+1} validated successfully at position {position}")
    
    # Phase 3: Apply all validated chunks (guaranteed to succeed)
    working_lines = list(original_lines)  # Fresh copy for modifications
    for position, chunk in sorted(chunk_applications, key=lambda x: x[0], reverse=True):
        working_lines[position+1:position+1+len(chunk["old_lines"])] = chunk["new_lines"]
    
    # Phase 4: Single atomic write with preserved line endings
    new_body = original_line_ending.join(working_lines)
    if not new_body.endswith(original_line_ending) and symbol.body.endswith(original_line_ending):
        new_body += original_line_ending
    
    self.symbol_manager.replace_body(symbol_location, new_body)
    return SUCCESS_RESULT

def _validate_and_position_chunk(self, original_lines, symbol, chunk):
    """Comprehensive chunk validation with detailed error reporting."""
    context_before_lines = chunk["context_before"]  # Now always a list
    log.debug(f"Validating chunk with {len(context_before_lines)} context_before lines")
    
    # Find candidate positions
    candidates = []
    context_len = len(context_before_lines)
    for i in range(len(original_lines) - context_len + 1):
        if all(original_lines[i + j].rstrip() == context_before_lines[j].rstrip() 
               for j in range(context_len)):
            candidates.append(i + context_len - 1)  # Position of last context line
    
    if not candidates:
        error = f"No sequence matches context_before: {context_before_lines}"
        log.debug(f"Validation failed: {error}")
        return None, error
    
    log.debug(f"Found {len(candidates)} candidate positions: {candidates}")
    
    # Validate each candidate position comprehensively
    for position in candidates:
        validation_error = self._run_comprehensive_validation(original_lines, chunk, position, symbol)
        if validation_error is None:
            log.debug(f"Chunk validation successful at position {position}")
            return position, None
        log.debug(f"Position {position} failed: {validation_error}")
    
    return None, f"All {len(candidates)} candidate positions failed validation"

def _run_comprehensive_validation(self, original_lines, chunk, position, symbol):
    """Run all validation checks. Returns None if valid, error message if invalid."""
    context_after_lines = chunk["context_after"]  # Now always a list
    old_lines = chunk["old_lines"]
    new_lines = chunk["new_lines"]
    
    # Check 1: Context After Validation
    expected_after_pos = position + 1 + len(old_lines)
    if expected_after_pos + len(context_after_lines) > len(original_lines):
        return f"Check 1 failed: context_after sequence extends beyond symbol"
    
    for j, context_line in enumerate(context_after_lines):
        if original_lines[expected_after_pos + j].rstrip() != context_line.rstrip():
            return f"Check 1 failed: context_after mismatch at line {expected_after_pos + j}"
    
    # Check 2: Old Lines Matching
    actual_old_lines = original_lines[position+1:position+1+len(old_lines)]
    if not self._lines_match(actual_old_lines, old_lines):
        return f"Check 2 failed: old_lines don't match actual content"
    
    # Check 3: Insertion Logic (empty old_lines)
    if len(old_lines) == 0 and expected_after_pos != position + 1:
        return f"Check 3 failed: Insertion requires adjacent context lines"
    
    # Check 4: Deletion Logic (empty new_lines)
    if len(new_lines) == 0 and actual_old_lines != old_lines:
        return f"Check 4 failed: Deletion requires exact line matching"
    
    # Check 5: Symbol Boundary Validation
    chunk_start, chunk_end = position + 1, position + 1 + len(old_lines)
    if chunk_start < 0 or chunk_end > len(original_lines):
        return f"Check 5 failed: Chunk extends outside symbol boundaries"
    
    # Check 6: End-of-Symbol Context Validation
    if len(context_after_lines) == 0 and expected_after_pos < len(original_lines):
        return f"Check 6 failed: Empty context_after specified but content exists after"
    
    # Check 7: Line Ending Consistency
    original_line_ending = self._detect_line_ending(symbol.body)
    all_chunk_lines = chunk["context_before"] + chunk["context_after"] + old_lines + new_lines
    for line_content in all_chunk_lines:
        if '\n' in line_content or '\r' in line_content:
            line_ending = self._detect_line_ending(line_content)
            if line_ending != original_line_ending:
                return f"Check 7 failed: Line ending mismatch - file uses {repr(original_line_ending)}, chunk uses {repr(line_ending)}"
    
    return None  # All checks passed

def _lines_match(self, actual_lines, expected_lines):
    """Check if lines match, allowing for some flexibility in whitespace."""
    if len(actual_lines) != len(expected_lines):
        return False
    
    for actual, expected in zip(actual_lines, expected_lines):
        if actual.rstrip() != expected.rstrip():
            return False
    
    return True

def _detect_line_ending(self, text):
    """Detect the dominant line ending in text."""
    cr_count = text.count('\r\n')
    lf_count = text.count('\n') - cr_count
    cr_only_count = text.count('\r') - cr_count
    
    if cr_count > max(lf_count, cr_only_count):
        return '\r\n'
    elif lf_count > cr_only_count:
        return '\n'
    elif cr_only_count > 0:
        return '\r'
    else:
        return '\n'
```

#### 2.2 Robustness Features

The chunk-based editing system implements several layers of protection against partial failures:

**Atomic Validation Design:**
- All chunks are validated completely before any modifications begin
- Invalid chunks cause the entire operation to fail with no changes applied
- Single point of failure eliminates inconsistent intermediate states

**Comprehensive Validation Checks:**
1. **Position Validation**: Ensures context lines can be located accurately
2. **Content Matching**: Verifies old_lines match exactly what exists in the file
3. **Insertion Logic**: Validates adjacent context for pure insertions (empty old_lines)
4. **Deletion Logic**: Ensures exact matching for deletions (empty new_lines)
5. **Boundary Protection**: Prevents chunks from extending outside symbol boundaries
6. **End-of-Symbol Handling**: Validates empty context_after scenarios
7. **Line Ending Consistency**: Prevents cross-platform line ending corruption

**Error Reporting:**
- Each validation failure provides specific diagnostic information
- Comprehensive logging enables debugging of complex chunk scenarios
- Clear error messages indicate which specific check failed and why

**Cross-Platform Safety:**
- Automatic detection and preservation of original file line endings
- Validation prevents mixed line ending corruption
- Support for CRLF, LF, and legacy CR line ending formats

#### 2.3 Error Handling

```python
def apply(self, relative_path, line, column, chunks):
    """Main tool entry point with error handling."""
    try:
        symbol_location = SymbolLocation(relative_path, line, column)
        
        # Input validation
        if not chunks or not isinstance(chunks, list):
            return "Error: chunks must be a non-empty list"
        
        for chunk in chunks:
            required_keys = ["context_before", "old_lines", "new_lines", "context_after"]
            if not all(key in chunk for key in required_keys):
                return f"Error: chunk missing required keys: {required_keys}"
            
            # Normalize context to lists for consistent processing
            if isinstance(chunk["context_before"], str):
                chunk["context_before"] = [chunk["context_before"]]
            if isinstance(chunk["context_after"], str):
                chunk["context_after"] = [chunk["context_after"]]
        
        # Apply the chunks to the symbol
        return self.apply_chunks_to_symbol(symbol_location, chunks)
        
    except Exception as e:
        log.error(f"Error in ChunkDiffSymbolTool: {str(e)}", exc_info=e)
        return f"Error applying chunks: {str(e)}"
```

### 3. Integration with Symbol Manager

The tool will use the existing `SymbolManager` class for:
1. Finding symbols by location
2. Retrieving symbol bodies
3. Applying changes to symbols
4. Marking files as modified

### 4. Dependencies

- Existing Serena components (`Tool`, `SymbolManager`, etc.)
- No external diff libraries required - the chunk format is self-contained

## Performance Considerations

### Token Efficiency

- Average token reduction: 80-95% compared to full symbol replacement
- Example: For a 200-line class with 5 scattered single-line changes:
  - Full replacement: ~6000 tokens
  - Chunk-based diff: ~500 tokens (92% reduction)

### Execution Performance

- Time complexity: O(n) where n is the number of lines in the symbol
- Space complexity: O(n) additional memory for line-by-line processing
- Expected execution time: comparable to `replace_symbol_body`

## Testing Strategy

### Unit Tests - Comprehensive Validation System

1. **Individual Validation Check Testing**
   - Test each of the 7 validation checks independently
   - Verify Check 1 (Context After) with mismatched context lines
   - Verify Check 2 (Old Lines Matching) with content that doesn't match
   - Verify Check 3 (Insertion Logic) with non-adjacent context for empty old_lines
   - Verify Check 4 (Deletion Logic) with inexact matching for empty new_lines
   - Verify Check 5 (Boundary Validation) with chunks extending outside symbol boundaries
   - Verify Check 6 (End-of-Symbol) with empty context_after scenarios
   - Verify Check 7 (Line Ending Consistency) with mixed line ending formats

2. **Atomic Operation Testing**
   - Verify all-or-nothing behavior: if any chunk fails validation, no changes are applied
   - Test scenarios where chunk 1 succeeds but chunk 3 fails - ensure original file is unchanged
   - Verify working copy isolation: original_lines remains unmodified during validation

3. **Edge Case Validation**
   - **Pure Insertions** (empty old_lines): Test adjacent context validation
   - **Pure Deletions** (empty new_lines): Test exact line matching requirements
   - **Boundary Chunks**: Test start-of-symbol and end-of-symbol positioning
   - **Multiple Candidate Positions**: Test when context_before appears multiple times

4. **Cross-Platform Line Ending Tests**
   - Test files with CRLF line endings (\r\n) with chunks containing LF (\n)
   - Test files with LF line endings with chunks containing CRLF
   - Test legacy CR-only files (\r) with modern chunk content
   - Test line ending detection accuracy with mixed content
   - Verify line ending preservation in output

5. **Error Reporting Tests**
   - Verify specific error messages for each validation check failure
   - Test error message clarity and diagnostic information
   - Verify comprehensive logging output for debugging

6. **Flexible Context Tests**
   - Test single-line context (string format)
   - Test multi-line context (list format)
   - Test mixed context formats in the same operation
   - Test context normalization logic

### Integration Tests - Real-World Scenarios

1. **Language Server Compatibility**
   - Test with Python, TypeScript, Java language servers
   - Verify symbol location accuracy across different languages
   - Test symbol boundary detection consistency

2. **Complex Multi-Chunk Operations**
   - Test 5+ chunks applied to the same symbol in reverse order
   - Test chunks with overlapping context (should be rare but handled)
   - Test chunks scattered throughout large symbols (200+ lines)

3. **Real-World Code Patterns**
   - Test with actual code symbols from the Serena codebase
   - Test refactoring operations (method extraction, parameter changes)
   - Test adding imports, exception handling, logging statements

4. **Failure Recovery**
   - Test language server restart scenarios
   - Test file permission issues during symbol replacement
   - Test disk space failures during write operations

### Performance Tests - Token Efficiency Validation

1. **Token Usage Measurements**
   - Measure actual token reduction for various symbol sizes (50, 100, 200+ lines)
   - Compare chunk-based vs full replacement for scattered edits
   - Verify 80-95% token reduction claims with real examples

2. **Execution Performance**
   - Benchmark validation time for complex multi-chunk operations
   - Compare execution time with original replace_symbol_body
   - Test with large symbols (500+ lines) to verify O(n) complexity

3. **Memory Usage**
   - Verify working copy memory usage stays within expected bounds
   - Test memory cleanup after failed operations

### Robustness Tests - Atomic Behavior Verification

1. **Simulated Failure Scenarios**
   - Inject failures at different points in the validation pipeline
   - Verify file remains unchanged when validation fails
   - Test recovery from unexpected exceptions during processing

2. **Concurrent Modification Protection**
   - Test behavior when file is modified by external process during operation
   - Verify detection of stale symbol content

3. **Stress Testing**
   - Test with extreme chunk counts (20+ chunks per symbol)
   - Test with very large context strings
   - Test with symbols at the limits of language server capabilities

## Security Considerations

- Input validation to prevent injection attacks
- Sanitization of chunk content before application
- Maintain existing permissions model of Serena

## Documentation Requirements

1. User-facing documentation with examples
2. Developer documentation for maintaining the tool
3. Examples showing token efficiency benefits

## Implementation Plan

### Phase 1: Core Implementation

1. Create the `ChunkDiffSymbolTool` class
2. Implement chunk application algorithm
3. Add basic error handling and validation

### Phase 2: Testing & Refinement

1. Develop comprehensive test suite
2. Refine algorithm based on test results
3. Optimize for edge cases

### Phase 3: Integration & Documentation

1. Integrate with Serena's tool registry
2. Document the tool usage for both developers and users
3. Create examples showing token efficiency benefits

## Sample Usage

```python
# Example of how an LLM would use the tool with flexible context

# Single-line context when unique
chunks = [
    {
        "context_before": "    def process_data(self, input_data):",
        "old_lines": ["        result = transform(input_data)", "        return result"],
        "new_lines": ["        try:", "            result = transform(input_data)", "            return result", "        except ValueError as e:", "            self.logger.error(f\"Error processing data: {e}\")", "            return None"],
        "context_after": ""
    }
]

# Multi-line context when needed for uniqueness
chunks = [
    {
        "context_before": [
            "    # Process each item in the batch",
            "    for item in batch:",
            "        if item.needs_processing:"
        ],
        "old_lines": ["            result = process(item)"],
        "new_lines": ["            result = process(item)", "            self.log_processing(item, result)"],
        "context_after": "            processed_count += 1"
    }
]

result = chunk_diff_symbol("src/processor.py", 45, 8, chunks)
```

## Migration Strategy

- No migration needed as this is an additive feature
- Existing tools will continue to function as before
- Documentation will encourage LLMs to prefer the chunk-based approach for large symbols
