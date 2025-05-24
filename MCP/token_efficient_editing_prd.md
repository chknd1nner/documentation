# Product Requirements Document: Token-Efficient Symbol Editing

## Overview

This document outlines requirements for a new feature in Serena that enables significantly more token-efficient code editing through chunk-based differential editing of code symbols.

## Problem Statement

When working with large code symbols, the current editing approach requires:
1. Reading the entire symbol content (consuming tokens)
2. Generating an entirely new version of the symbol (consuming more tokens)
3. Sending the complete new symbol body to the `replace_symbol_body` tool

This approach is highly inefficient when:
- Symbols are large (e.g., 100+ lines)
- Only small, scattered changes are needed
- Multiple iterations of edits are required

This inefficiency results in:
- Quickly exhausting token limits
- Higher API costs
- More frequent context refreshes/new conversations
- Inability to work with larger symbols

## Proposed Solution

Extend Serena with chunk-based differential editing capability by:
1. Creating a new tool that accepts only the changed portions of a symbol
2. Using a custom chunk-based format inspired by traditional diffs but designed specifically for LLM usage
3. Maintaining compatibility with existing symbol-based editing workflows

## User Experience

### Use Case Flow

1. User requests a code modification
2. LLM uses symbol-based search tools to investigate
3. LLM identifies the symbol to change and reads existing symbol code
4. LLM determines necessary edits
5. **New Step:** LLM prepares chunk-based diff instead of full replacement
6. LLM calls new `patch_symbol_with_chunks` tool with minimal diff data
7. Tool applies changes server-side and confirms success

### Benefits to User

- **Extended Session Length:** 80-95% reduction in tokens used for code edits
- **Larger Codebase Support:** Edit larger symbols without hitting token limits
- **Lower API Costs:** Significantly less data transferred through the API
- **Improved Flow:** Fewer interrupted sessions due to token exhaustion
- **Exact Same Editing Quality:** No reduction in editing precision or capability

## Requirements

### Functional Requirements

1. **Chunk-Based Symbol Editing Tool**
   - New tool that accepts minimal diff chunks rather than complete symbol bodies
   - Support for multiple edits in a single operation
   - Preservation of indentation and formatting
   - Accurate positioning through context lines

2. **Custom Chunk-Based Format**
   - Use a custom chunk-based format with context-based positioning
   - Support context-based chunk identification without relying on line numbers
   - Flexible context length - either single lines or multiple lines as needed
   - Process chunks server-side to minimize token usage

3. **Symbol Integration**
   - Maintain the symbol-based approach that differentiates Serena
   - Work with existing symbol location mechanisms
   - Support all symbol types (classes, functions, methods, etc.)

4. **Error Handling**
   - Robust handling of chunk application failures
   - Detailed error messages for debugging
   - Fallback mechanisms if chunks can't be applied

### Non-Functional Requirements

1. **Performance**
   - Tool execution time comparable to existing `replace_symbol_body`
   - No noticeable delay in applying changes

2. **Reliability**
   - 99.9% success rate for well-formed chunk requests
   - Proper validation of inputs to prevent corrupt edits

3. **Compatibility**
   - Work with all language servers supported by Serena
   - Compatible with existing Serena workflows

## Success Metrics

1. **Token Efficiency**
   - 80%+ reduction in tokens used for typical code edit operations
   - Measurable increase in edits possible within a single context window

2. **User Experience**
   - No increase in error rates compared to full symbol replacement
   - Seamless integration with existing editing workflows

3. **Usage Adoption**
   - New tool becomes preferred method for symbol edits within 1 month

4. **Robustness**
   - 99.9%+ success rate for well-formed chunks with comprehensive error reporting
   - Zero risk of partial edits corrupting files through atomic validation design

5. **Cross-platform Compatibility**
   - Seamless operation across Windows, macOS, and Linux with proper line ending handling
   - Automatic detection and preservation of original file line ending formats

## Out of Scope

- Changes to the MCP protocol itself
- Modifications to how language servers work
- Changes to how symbols are located and identified
- Non-symbol-based editing workflows

## Implementation Considerations

- The tool should be intuitive for LLMs to use without extensive instruction
- Documentation should clearly explain the token efficiency benefits
- Examples should demonstrate common editing patterns

## Timeline

- Design and specification: 1 week
- Implementation: 2 weeks
- Testing: 1 week
- Documentation: 1 week
- Release: End of month
