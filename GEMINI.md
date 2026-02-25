# Gemini CLI Guidelines for iTerm2

As an AI agent working in the iTerm2 repository, I adhere to the following project-specific guidelines based on the existing `CLAUDE.md` and `AGENTS.md` instructions, as well as my core operational mandates.

## 1. Code Style and Best Practices
- **No Inline Web Languages in Native Code:** Do not write multi-line JavaScript, HTML, or CSS inline within Swift or Objective-C files. Always create a separate file and load it using `iTermBrowserTemplateLoader.swift`.
- **Assertions and Errors:** In Swift, exclusively use `it_fatalError` and `it_assert` instead of standard `fatalError` and `assert` to ensure useful crash logs are generated. In Objective-C, `ITAssertWithMessage` is preferred, though standard `assert` is acceptable.
- **Dependency Management:** strictly avoid creating dependency cycles. Rely on delegates or closures instead.
- **File Operations:**
  - After creating a new file, run `git add` immediately.
  - When renaming a tracked file, use `git mv` instead of standard `mv`.
- **String Formatting:** Do not replace curly quotes with straight quotes. In user-visible strings, strictly use curly quotes (e.g., “ and ”) unless representing inches.
- **OS Support:** The deployment target is macOS 12. Availability checks for older macOS versions are unnecessary.

## 2. Python API and Scrollback Rules
When interacting with or extending the iTerm2 Python API (especially for scrollback-aware features):
- **High-Level Reads:** Utilize `Session.async_get_contents(first_line, number_of_lines)` and `Session.async_get_line_info()` using absolute line numbers. 
- **Viewport Awareness:** The visible viewport is tracked via `SessionLineInfo.first_visible_line_number`. Note that `Session.async_get_screen_contents()` requests the mutable screen region at the bottom, *not* the currently scrolled viewport.
- **Transactions:** Always use `iterm2.Transaction(connection)` when sampling line info and content to ensure atomicity.
- **State Synchronization:** There is no dedicated Python callback for scroll-position changes. Polling `async_get_line_info()` is required to track viewport scroll state.

## 3. Testing and Building
- **Debug Builds:** Run `make Development` to create a debug build.
- **Unit Tests:** Execute tests within `ModernTests` using `tools/run_tests.expect <test_name>` (e.g., `tools/run_tests.expect ModernTests/iTermScriptFunctionCallTest/testSignature`).
- **Manual Tests:** Place small scripts or text files used for manual feature testing in the `tests/` directory.

## 4. Architectural Awareness
- iTerm2 is a hybrid Objective-C and Swift codebase heavily reliant on macOS AppKit (`sources/PTYTextView*`, `sources/iTermController*`).
- Proceed with caution and strictly adhere to established architectural boundaries between application coordination, the terminal model (`sources/VT100*`), and rendering.
