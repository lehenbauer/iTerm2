# iTerm2 Agent Guide

This guide is for AI/code agents working in this repository. It focuses on how to work safely in iTerm2 and how to use the Python API correctly, especially for scrollback-aware integrations like `../ai-whisperer`.

## Critical Rules

Read `CLAUDE.md` first.

1. Never write multi-line JavaScript, HTML, or CSS inline in Swift/ObjC. Use template files loaded via `iTermBrowserTemplateLoader.swift`.
2. In Swift, use `it_fatalError` and `it_assert` (not `fatalError` / `assert`).
3. Do not create dependency cycles. Use delegates or closures.
4. If you create a new file, `git add` it immediately.

## Architecture Snapshot

iTerm2 is hybrid Objective-C/Swift.

- Application coordinator: `sources/iTermController*`
- Window/tab/session: `sources/PseudoTerminal*`, `sources/PTYTab*`, `sources/PTYSession*`
- Terminal model: `sources/VT100*`
- Rendering: `sources/PTYTextView*`
- Python API package: `api/library/python/iterm2/iterm2/`
- WebSocket proto contract: `proto/api.proto`

## Python API Surface (What To Use)

- High-level session reads:
  - `Session.async_get_line_info()`
  - `Session.async_get_contents(first_line, number_of_lines)`
  - `Session.async_get_screen_contents()` (mutable screen only)
  - `Session.get_screen_streamer()` (screen update notifications; mutable screen only)
- Low-level RPC wrapper:
  - `iterm2.rpc.async_get_screen_contents(...)`
- Data types:
  - `iterm2.screen.ScreenContents`, `LineContents`
  - `iterm2.util.Point`, `CoordRange`, `WindowedCoordRange`

## Scrollback And Viewport Support (Verified)

### Yes: Read Scrollback Buffer

Use `Session.async_get_contents(first_line, number_of_lines)` with absolute line numbers.

- First available line is `line_info.overflow`.
- Absolute line numbering is stable even when old history overflows.

Relevant code:
- `api/library/python/iterm2/iterm2/session.py` (`async_get_contents`, `async_get_line_info`)
- `sources/PTYSession.m` (`handleGetBufferRequest:`)
- `proto/api.proto` (`Coord.y`, `LineRange`, `GetBufferRequest/Response`)

### Yes: Read The Screen At The Current Scrollback Position

The API gives the top currently visible line via `SessionLineInfo.first_visible_line_number`.

Read the currently visible viewport like this:

```python
async with iterm2.Transaction(connection):
    li = await session.async_get_line_info()
    top = li.first_visible_line_number
    height = li.mutable_area_height
    visible_lines = await session.async_get_contents(top, height)
```

Use a transaction so line info and content are sampled atomically.

### Yes: Determine "How Scrolled" The User Is

You can compute scroll distance from bottom using line info:

```python
bottom_top = li.overflow + li.scrollback_buffer_height
lines_scrolled_up = max(0, bottom_top - li.first_visible_line_number)
```

- `lines_scrolled_up == 0` means viewport is at bottom.
- Positive values mean the user is scrolled up into history.

### Important Limitation: `async_get_screen_contents()` Is Not Viewport-Aware

`Session.async_get_screen_contents()` and `ScreenStreamer.async_get()` request `screen_contents_only = true`, which maps to the mutable screen region at the bottom, not the user’s currently scrolled viewport.

Relevant code:
- `api/library/python/iterm2/iterm2/rpc.py` (`async_get_screen_contents`)
- `sources/PTYSession.m` (`absoluteWindowedCoordRangeFromLineRange:`)

### No Direct Scroll-Position Event API

There is no dedicated Python callback that fires only on viewport scroll-position changes. Screen-update notifications are content-driven, not a clean scroll-state signal. For scroll-aware features, poll `async_get_line_info()` (or sample on your own trigger cadence).

## Practical Guidance For `../ai-whisperer`

1. Maintain per-session state keyed by `session.session_id`.
2. On each sampling tick:
   - Fetch `line_info`.
   - If `first_visible_line_number` changed, fetch `async_get_contents(first_visible, mutable_area_height)` for the viewport text.
3. For long-context retrieval, fetch additional history above viewport using `async_get_contents`.
4. Use `iterm2.Transaction` when combining multiple reads that must be consistent.
5. Treat line numbers as absolute; do not renormalize by subtracting overflow unless you have a specific reason.

## Source Of Truth Files

- Python wrappers:
  - `api/library/python/iterm2/iterm2/session.py`
  - `api/library/python/iterm2/iterm2/screen.py`
  - `api/library/python/iterm2/iterm2/rpc.py`
- Protocol:
  - `proto/api.proto`
- Backend behavior:
  - `sources/PTYSession.m`
  - `sources/iTermAPIHelper.m`
  - `sources/PTYTextView.m`

## Notes

- Some comments/examples are stale (for example, older references that imply adding overflow to `first_visible`). Prefer current source behavior over old prose.
- Browser sessions intentionally reject buffer-range reads in `handleGetBufferRequest:`.
