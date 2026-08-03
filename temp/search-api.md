# Ghostty Search API — Deep-Dive

This document walks through how **search** works across the entire Ghostty
codebase, from the GUI apps (macOS AppKit/SwiftUI and Linux GTK4) down through
the shared Zig core (`Surface`), the dedicated search thread
(`terminal/search/`), the renderer, and the public C API (`include/ghostty.h`).

It is written as a "learning path": start with the big picture, then read each
layer in order. All line numbers refer to the codebase at the time of writing
and will drift over time — use `git log -L` or `rg` to find moved code.

---

## 1. Architecture at a Glance

```
   macOS GUI (Swift)                GTK/Linux GUI (Zig)
 +-------------------+            +----------------------+
 | SwiftUI overlay   |            | search-overlay.blp   |
 | SurfaceView/      |            | SearchOverlay GObject|
 |   SearchState     |            +----------------------+
 +---------+---------+                 +----------------+
           |                           |
   down: binding action strings  down: performBindingAction()
   via ghostty_surface_          (template callbacks on
   binding_action()               SurfaceOverlay)
           |                           |
           +-------------+-------------+
                         |
              input.Binding.Action
              (parse "search:x", "start_search", ...)
                         |
                         v
              +---------------------+         +------------------+
              |   core Surface.zig  | ------> | apprt.action     |
              |  (performBindingAction)        | .Action (upward) |
              +----------+----------+         +--------+---------+
                         |                            |
   spawns on first needle|                            v
                         v                    GUI handlers
              +---------------------+     (macOS Ghostty.App.swift,
              | terminal/search/    |      GTK application.zig)
              |   Thread.zig        |
              |  (dedicated thread) |
              +----------+----------+
                         |  events: viewport_matches,
                         |  total_matches, selected_match
              +----------+----------+
              |    renderer thread  |
              |  (search highlights)|
              +---------------------+
```

There are **two communication directions** and they use different mechanisms:

### 1.1 Downward: GUI → core (the user asks the core to do something)

The GUI sends **binding-action strings**. These are the exact same strings
that appear in a user's `keybind = ...` config, parsed by
`input.Binding.Action.parse` into the `input.Binding.Action` tagged union.

| String | `input.Binding.Action` | Meaning |
|---|---|---|
| `search:<text>` | `.search = text` | Set/change the needle. Empty text cancels the search (but does *not* hide the GUI). |
| `start_search` | `.start_search` | Open the search UI with an empty needle. |
| `search_selection` | `.search_selection` | Search for the current selection text. |
| `end_search` | `.end_search` | Stop the search and hide the GUI. |
| `navigate_search:next` | `.navigate_search = .next` | Move to the next match. |
| `navigate_search:previous` | `.navigate_search = .previous` | Move to the previous match. |

* macOS sends these via the exported C function
  `ghostty_surface_binding_action(surface, "search:...", len)`
  (`src/apprt/embedded.zig:1963`), which parses the string and calls
  `core_surface.performBindingAction`.
* GTK calls `surface.performBindingAction(.{ .search = ... })` directly from
  its overlay's template callbacks.

### 1.2 Upward: core → GUI (the core tells the GUI what happened)

The core sends **`apprt.action.Action` events** through
`rt_app.performAction(target, action, value)`. The target is
`apprt.Target` (`.app` or `.surface`). The four search-related events:

| `apprt.action.Action` | Payload | Meaning |
|---|---|---|
| `.start_search` | `StartSearch { needle }` | "Show the search UI" (sent when a keybinding triggers search). |
| `.end_search` | `{}` | "Hide the search UI". |
| `.search_total` | `SearchTotal { total: ?usize }` | Match-count changed (`null` = reset/none). |
| `.search_selected` | `SearchSelected { selected: ?usize }` | Selected match index changed (0-based; `null` = none). |

The same enum drives both GUIs **and** the public C ABI: every case has a
matching `ghostty_action_*_s` C struct and a `GHOSTTY_ACTION_*` tag, kept in
sync by `// Sync with:` comments (see §4).

---

## 2. The Shared Core Contract (`src/apprt/`)

Start here. This directory is the API surface between the terminal core and
the app runtime ("apprt").

### 2.1 `src/apprt/action.zig` — the event union

* `pub const Action = union(enum) { ... }` — all cross-boundary events. Search
  cases are at `action.zig:335-344`:

  ```zig
  start_search: StartSearch,   // open UI, optional initial needle
  end_search,                  // close UI, clear state
  search_total: SearchTotal,   // match count changed
  search_selected: SearchSelected, // selected index changed
  ```

* `pub const Key = enum(c_int) { ... }` (`action.zig:354-427`) — a C-compatible
  tag for the union, generated to mirror `ghostty_action_tag_e`. There's a
  compile-time test `"ghostty.h Action.Key"` that validates it against the C
  header (`action.zig:424`).

* `pub const CValue` (`action.zig:430`) — the C union (`ghostty_action_u`),
  auto-generated from the `Action` union.

* Payload structs and their C mirrors:

  ```zig
  pub const StartSearch = struct {
      needle: [:0]const u8,
      pub const C = extern struct { needle: [*:0]const u8 }; // ghostty_action_start_search_s
      pub fn cval(self: StartSearch) C { ... }
  };

  pub const SearchTotal = struct {
      total: ?usize,           // null => unknown / no matches yet
      pub const C = extern struct { total: isize }; // -1 encodes null
      pub fn cval(self: SearchTotal) C { ... }
  };

  pub const SearchSelected = struct {
      selected: ?usize,        // null => no selection
      pub const C = extern struct { selected: isize }; // -1 encodes null
      pub fn cval(self: SearchSelected) C { ... }
  };
  ```
  (`action.zig:997-1040`). Note `?usize` maps to `isize` in C with `-1` meaning
  "null" — the GUI layer must decode that.

### 2.2 `src/apprt/surface.zig` — the surface mailbox

* `apprt.surface.Message` (`surface.zig:105-109`) is the message type sent
  from core threads to the surface. Two search messages:

  ```zig
  /// Search progress update
  search_total: ?usize,
  /// Selected search index change
  search_selected: ?usize,
  ```

  These are **intra-process** (core-internal), distinct from the
  `apprt.action.Action` events that go to the GUI.

* `apprt.surface.Mailbox.push` (`surface.zig:135`) re-wraps a surface message
  into `App.Mailbox` with a `.surface_message` payload so the app thread can
  dispatch it to the right surface. It is intentionally "sent to the app thread,
  not the surface."

### 2.3 `src/input/Binding.zig` — downward action parsing

* `input.Binding.Action` is a *different* union from `apprt.action.Action`. It
  is the full set of user-performable actions. Search members
  (`Binding.zig:409-433`):

  ```zig
  search: []const u8,        // "search:<text>"
  search_selection,
  navigate_search: NavigateSearch, // .next | .previous
  start_search,
  end_search,
  ```

  Note the doc comment on `search`: *"If the text is empty, then the search is
  canceled. A canceled search will not disable any GUI elements showing
  search. For that, the explicit end_search binding exists."* This is why the
  macOS debouncer sends `end_search` when the overlay closes, not an empty
  needle.

* `Action.parse` (`Binding.zig:1253-1317`) splits on the first `:` — the
  part before is the action name, the part after is the parameter. `[]const u8`
  fields (like `search`) take everything after the colon verbatim.

* `Action.scope` (`Binding.zig:1327`) classifies actions as `.app` or
  `.surface`. All search actions are `.surface` (`Binding.zig:1356-1360`), so
  they are handled per-surface by `Surface.performBindingAction`.

* `src/input/command.zig:186-211` registers the search actions in the command
  palette (with descriptions like "Navigate to the next search result, if
  any.").

---

## 3. Core Behavior (`src/Surface.zig`)

`Surface` owns the search lifecycle. Read `performBindingAction` first
(`Surface.zig:4798`), then the search handler block (`Surface.zig:4915-5015`).

### 3.1 Search state on the surface

```zig
// Surface.zig:178
search: ?Search = null,

// Surface.zig:202
const Search = struct {
    state: terminal.search.Thread,
    thread: std.Thread,

    pub fn deinit(self: *Search) void {
        // Notify the thread to stop
        self.state.stop.notify() ...;
        // Wait for the OS thread to quit
        self.thread.join();
        // Now it is safe to deinit the state
        self.state.deinit();
    }
};
```

The search thread is **lazily created**: it only exists once the first
non-empty needle arrives. `deinit` stops the thread via the xev `stop` async
handle and joins it (important: never `deinit` the search state while the
thread is running). `end_search` calls this.

### 3.2 The search action handlers (`Surface.zig:4915-5015`)

All run on the app/embedding thread via `performBindingAction`.

* **`.start_search`** (`4915`): deliberately does **not** start any search
  work. It just forwards to the GUI:
  ```zig
  return try self.rt_app.performAction(
      .{ .surface = self }, .start_search,
      .{ .needle = "" },
  );
  ```
  "To save resources, we don't actually start a search here, we just notify the
  apprt. The real thread will start when the first needles are set."

* **`.search_selection`** (`4926`): reads the current selection text
  (`selectionString`) and forwards it as `.start_search` with the selection as
  the needle. If there is no selection it returns `false` (action not
  performed).

* **`.end_search`** (`4936`): stops the search thread (if any), then always
  notifies the GUI with `.end_search` so "GUIs can clean up stale stuff". It
  returns whether a search was actually active (`performed`).

* **`.search => |text|`** (`4956`): the heart of it.
  1. If `self.search == null` and `text.len == 0`, do nothing (return false).
  2. Otherwise lazily **spawn the search thread**:
     ```zig
     self.search = .{
         .state = try .init(self.alloc, .{
             .mutex = self.renderer_state.mutex,
             .terminal = self.renderer_state.terminal,
             .event_cb = &searchCallback,
             .event_userdata = self,
         }),
         .thread = undefined,
     };
     s.thread = try .spawn(.{}, terminal.search.Thread.threadMain, .{&s.state});
     ```
     The thread is given the renderer-state mutex and a pointer to the
     `Terminal`; it will grab that mutex itself when reading terminal data.
  3. If `text.len == 0`, stop the search (`s.deinit()`, `self.search = null`).
  4. Otherwise push a `change_needle` message to the thread's mailbox and wake
     it.

* **`.navigate_search => |nav|`** (`5004`): if no search is active, returns
  `false`. Otherwise pushes `.select` (`.{ .next | .prev }`) to the thread
  mailbox and wakes the thread.

### 3.3 Search events back to the surface (`searchCallback`, `1426-1541`)

**This callback runs on the search thread.** It must not touch anything mutable
on the surface except via mailboxes. It receives `terminal.search.Thread.Event`
and routes:

| Event | Where it goes |
|---|---|
| `.viewport_matches` | **renderer thread** mailbox as `.search_viewport_matches` (matches are cloned into an arena owned by the message). |
| `.selected_match` (some match) | **renderer thread** mailbox `.search_selected_match` + **surface** mailbox `.search_selected = idx`. |
| `.selected_match` (null) | renderer `.search_selected_match = null` + surface `.search_selected = null`. |
| `.total_matches` | **surface** mailbox `.search_total = total`. |
| `.quit` | renderer reset (null selected + empty viewport matches) + surface `.search_total = null` / `.search_selected = null`. |

The surface mailbox messages are handled in the surface's own message switch at
`Surface.zig:1166-1180`, which re-forwards to the GUI:

```zig
.search_total => |v| {
    _ = try self.rt_app.performAction(
        .{ .surface = self }, .search_total,
        .{ .total = v },
    );
},
.search_selected => |v| {
    _ = try self.rt_app.performAction(
        .{ .surface = self }, .search_selected,
        .{ .selected = v },
    );
},
```

So the full "upward" path is:

```
search thread Event ──> searchCallback ──> surface mailbox ──> Surface.handle
  ──> rt_app.performAction(.search_total / .search_selected) ──> GUI handler
```

---

## 4. The libghostty C API (`include/ghostty.h`)

The public C header is the **ABI contract** for embedding Ghostty. C consumers
(such as the macOS Swift app, via Swift C bridging) interact with search
through two entry points.

### 4.1 Upward structs (`ghostty.h:868-881`)

```c
// apprt.action.StartSearch.C
typedef struct {
  const char* needle;
} ghostty_action_start_search_s;

// apprt.action.SearchTotal
typedef struct {
  ssize_t total;   // -1 if null
} ghostty_action_search_total_s;

// apprt.action.SearchSelected
typedef struct {
  ssize_t selected; // -1 if null
} ghostty_action_search_selected_s;
```

These are members of the big `ghostty_action_u` union, tagged by
`ghostty_action_tag_e` with entries `GHOSTTY_ACTION_START_SEARCH`,
`GHOSTTY_ACTION_END_SEARCH`, `GHOSTTY_ACTION_SEARCH_TOTAL`,
`GHOSTTY_ACTION_SEARCH_SELECTED` (generated from `apprt.Action.Key`; the
compile-time test at `action.zig:424` verifies sync).

### 4.2 Receiving events (core → C app)

The embedded `ghostty_app_t` has a `perform_action` callback. The callback
receives `(app, target, action: ghostty_action_s)`, where `action.action` is
the union. The consumer switches on `action.tag`. The macOS app does exactly
this (see §7).

### 4.3 Sending actions (C app → core)

```c
// src/apprt/embedded.zig:1963
export fn ghostty_surface_binding_action(
    ptr: *Surface,
    action_ptr: [*]const u8,
    action_len: usize,
) bool {
    const action_str = action_ptr[0..action_len];
    const action = input.Binding.Action.parse(action_str) catch ...;
    return ptr.core_surface.performBindingAction(action) catch ...;
}
```

The string form means the C API does not need a struct for downward search
actions — `"search:hello"`, `"end_search"`, `"navigate_search:next"` are all
valid. This is how macOS sends needles. The return value is whether the action
was performed.

---

## 5. The Search Engine (`src/terminal/search/`)

All search algorithms live here. Read in this order: `sliding_window.zig` →
`active.zig` / `viewport.zig` / `screen.zig` / `pagelist.zig` → `Thread.zig`.

### 5.1 `Thread.zig` — the dedicated search thread

Spawned lazily by the surface (§3.2). Owns an `xev.Loop` and several async
handles: `wakeup`, `stop`, and a `refresh` timer.

* **`Options`** (`Thread.zig:445`): the `renderer_state` mutex (must be held to
  read the terminal), the `Terminal` pointer, and an optional event callback +
  userdata. All thread-safety hinges on this mutex.

* **`Mailbox`** (`Thread.zig:463`): `BlockingQueue(Message, 64)`.

  ```zig
  pub const Message = union(enum) {
      change_needle: WriteReq,        // WriteReq = MessageData(u8, 255), max 255 bytes
      select: ScreenSearch.Select,    // .next | .prev
  };
  ```

* **`Event`** (`Thread.zig:482`) — what the thread emits via the event callback:

  ```zig
  pub const Event = union(enum) {
      quit,
      complete,                    // search complete on all screens
      total_matches: usize,        // total on active screen changed
      selected_match: ?SelectedMatch, // { idx: usize, highlight: FlattenedHighlight }
      viewport_matches: []const FlattenedHighlight, // owned by thread, valid only during callback
  };
  ```

* **`REFRESH_INTERVAL = 24`** ms (`Thread.zig:45`, "40 FPS") — the timer at
  which the thread re-checks the terminal for changes. Comment notes the design
  goal: responsive but not hammering the terminal lock.

* **Main loop** (`threadMain_`, `Thread.zig:141-247`): interleaves xev loop
  runs with search progress. While a search is active:
  1. Call `s.notify(...)` to flush pending events to the callback.
  2. If complete, block on the loop (`loop.run(.once)`).
  3. Otherwise `s.tick()`; if all searches are `.blocked` (need fresh terminal
     data), grab the mutex and `s.feed(alloc, terminal)`.

* **`changeNeedle`** (`Thread.zig:289`): if the new needle equals the current
  one (case-insensitive), do nothing. Otherwise tear down the old search and
  emit a reset burst (`total_matches = 0`, `selected_match = null`,
  `viewport_matches = &.{}`) so the UI clears, then start fresh.

* **`select`** (`Thread.zig:259`): feeds terminal data, calls
  `screen_search.select(.next|.prev)`, grabs the selected flattened match, and
  **scrolls the screen to the match** (`screen.scroll(.{ .pin = ... })`) if it
  is not already in the viewport. This is what makes "navigate to next match"
  move the terminal viewport.

### 5.2 `sliding_window.zig` — the substring matcher

`SlidingWindow` (`sliding_window.zig:38`) searches for the needle across a
**set of contiguous pages**, handling matches that straddle page boundaries.

* `data` is a circular buffer of encoded page text; `meta` is a parallel
  circular buffer of per-page metadata (`node`, `serial`, `rows`,
  `cell_map` — the mapping from encoded byte offsets back to grid
  coordinates). `cell_map` is what lets the search turn a byte-range match
  into a `FlattenedHighlight` with real `point`s.
* `overlap_buf` (size `needle.len * 2`) covers cross-page overlaps.
* Supports both `forward` and `reverse` search directions, though current
  callers only use `forward`.
* Callers append pages with `append`/`appendIfWrapped` while **holding the
  terminal lock** (the window copies the data it needs so the lock can be
  released before the actual matching). `next()` yields matches as
  `FlattenedHighlight`.

### 5.3 `active.zig` — `ActiveSearch`

Searches only the **active area** (the mutable part of a `PageList` — usually
just the bottom row(s)). Because the active area is the only part of the
terminal that changes, it is the only part that must be re-searched as the
screen scrolls or new lines arrive. `update(list)` copies the active area into
the window while the lock is held, and returns the last (oldest) page covered
so a history search can start from there and overlap without double-counting
results.

### 5.4 `viewport.zig` — `ViewportSearch`

Searches only the **viewport** (what the user can see). This is where results
must be highlighted, so it re-searches whenever the viewport or content
changes, but not otherwise.

* A **`Fingerprint`** of the visible pages is kept. `update()` returns `false`
  (no re-search) if the fingerprint is unchanged AND the viewport does not
  overlap the active area; otherwise it re-searches.
* `active_dirty` (`viewport.zig:33`) is a dirty flag the caller (the thread)
  sets whenever it thinks the active area may have changed; the viewport search
  re-runs if either the fingerprint changed or the active area got dirtied and
  intersects the viewport.

### 5.5 `screen.zig` — `ScreenSearch`

Searches an entire **screen** (history + active area). This is the "full"
search used for totals and for the selected match navigation.

* Holds an `active: ActiveSearch` plus an optional `history: HistorySearch`.
* Results are cached in two lists (`history_results`, `active_results`) so
  history (immutable) results don't need recomputation while the active area
  churns (`screen.zig:59-63`).
* `Select` enum (`screen.zig:786`): `.next` / `.prev` with **wrap-around**
  arithmetic (`selectNext` / `selectPrev`, `screen.zig:812`). The tests at
  `screen.zig:1189` ("Next match (wrap)") and `1422` ("Prev match (wrap)")
  pin the wrap behavior.
* `selectedMatch()` returns the currently selected `FlattenedHighlight`.

### 5.6 `pagelist.zig` — `PageListSearch`

An older/whole-`PageList` search variant. Where `ScreenSearch` covers history +
active explicitly, `PageListSearch` just searches a `PageList` in the requested
direction. Used by tests and as a building block.

### 5.7 Highlights (`src/terminal/highlight.zig`)

Search results are represented as `FlattenedHighlight` values — flattened
(rectangularized) ranges with a `tag`. The renderer's `HighlightTag` enum
(`renderer/generic.zig:237-238`) has two search tags:

```zig
search_match,            // any match
search_match_selected,   // the currently selected match
```

The search thread tags matches appropriately; the renderer colors them (see
§6).

---

## 6. Rendering Search Matches (`src/renderer/`)

### 6.1 `renderer/message.zig`

Two messages carry search data to the renderer thread (`message.zig:58-78`):

```zig
search_viewport_matches: SearchMatches, // { arena, matches: []const Flattened }
search_selected_match: ?SearchMatch,     // { arena, match: Flattened }
```

Each owns an `ArenaAllocator` — the renderer thread frees the previous value's
arena when replacing it, so there is no shared allocator between threads.

### 6.2 `renderer/Thread.zig` mailbox handling (`Thread.zig:482-496`)

```zig
.search_viewport_matches => |v| {
    if (self.renderer.search_matches) |*m| m.arena.deinit();
    self.renderer.search_matches = v;
    self.renderer.search_matches_dirty = true;
},
.search_selected_match => |v| {
    if (self.renderer.search_selected_match) |*m| m.arena.deinit();
    self.renderer.search_selected_match = v;
    self.renderer.search_matches_dirty = true;
},
```

### 6.3 `renderer/generic.zig` cell highlighting (`generic.zig:1304-1347`)

During a frame where `search_matches_dirty` or the terminal is dirty:

1. Clear all per-cell highlights.
2. Apply the **selected** match first with
   `updateHighlightsFlattened(alloc, HighlightTag.search_match_selected, ...)`.
3. Apply all **viewport** matches with
   `updateHighlightsFlattened(alloc, HighlightTag.search_match, ...)`.

"Highlights added earlier will take priority" (`generic.zig:1324`), so the
selected match wins when it overlaps another match.

Per-cell coloring happens later in the cell loop (`generic.zig:2740-2772`): a
cell can be `selection`, `search`, or `search_selected`, resolved by checking
the selection first, then the search highlight tags. The colors come from the
derived config (`generic.zig:559-560, 636-637`):

```zig
search_selected_background: configpkg.Config.TerminalColor,
search_selected_foreground: configpkg.Config.TerminalColor,
```

pulled from config keys `search-background`, `search-foreground`,
`search-selected-background`, `search-selected-foreground` (config defaults in
`Config.zig:1108-1124`: black on yellow for matches, black on salmon for the
selected match).

---

## 7. macOS GUI

Files: `macos/Sources/Ghostty/Ghostty.App.swift`,
`macos/Sources/Ghostty/Surface View/{OSSurfaceView,SurfaceView,SurfaceView_AppKit}.swift`.

### 7.1 Receiving events from the core (`Ghostty.App.swift`)

The app's `perform_action` callback switches on the action tag
(`Ghostty.App.swift:651-661`):

```swift
case GHOSTTY_ACTION_START_SEARCH:
    startSearch(app, target: target, v: action.action.start_search)
case GHOSTTY_ACTION_END_SEARCH:
    return endSearch(app, target: target)
case GHOSTTY_ACTION_SEARCH_TOTAL:
    searchTotal(app, target: target, v: action.action.search_total)
case GHOSTTY_ACTION_SEARCH_SELECTED:
    searchSelected(app, target: target, v: action.action.search_selected)
```

The handlers (`Ghostty.App.swift:2111-2208`) all take `GHOSTTY_TARGET_SURFACE`,
resolve the `SurfaceView` from the surface pointer, and dispatch to the main
queue:

* `startSearch`: if a `SearchState` already exists, update its needle; else
  create `Ghostty.SurfaceView.SearchState(from: startSearch)`. Then post the
  `.ghosttySearchFocus` notification so the find field gains focus
  (`GhosttyPackage.swift:379`).
* `endSearch`: `surfaceView.endSearch()` → sets `searchState = nil`, which
  hides the overlay (and, via the `didSet` below, sends `end_search` back to
  the core).
* `searchTotal`: `surfaceView.searchState?.total = v.total >= 0 ? UInt(v.total) : nil`.
* `searchSelected`: same pattern for `selected` (decoding the C `ssize_t`/`-1`
  convention).

### 7.2 `SearchState` (`OSSurfaceView.swift:119-156`)

```swift
@MainActor class SearchState: ObservableObject {
    private let pasteboard: OSPasteboard   // NSPasteboard.find

    @Published var needle: String = ""
    @Published var selected: UInt?
    @Published var total: UInt?
    @Published var needleSelection: Range<String.Index>?
    ...
}
```

Two interesting details:

* The needle is persisted to the **`.find` pasteboard** so it syncs with the
  system and other find bars. On init, a non-empty incoming needle is written
  to the pasteboard; an empty one *reads* the pasteboard back. The overlay
  re-reads on `didBecomeActiveNotification` and writes on every needle change
  (`SurfaceView.swift:408-419`).
* `@Published var searchState: SearchState?` on the surface view — non-nil
  means "show the overlay" (rendered by the SwiftUI body in
  `SurfaceView.swift:163-171`).

### 7.3 Sending the needle down (debounced, `SurfaceView_AppKit.swift:43-74`)

`SurfaceView_AppKit.swift` overrides `searchState` with a `didSet` that wires
up a Combine pipeline on `searchState.$needle`:

```swift
searchState.$needle
    .removeDuplicates()
    .map { needle in
        if needle.isEmpty || needle.count >= 3 {
            return Just(needle).eraseToAnyPublisher()
        } else {
            return Just(needle)
                .delay(for: .milliseconds(300), scheduler: DispatchQueue.main)
                .eraseToAnyPublisher()
        }
    }
    .switchToLatest()
    .sink { needle in
        guard let surface = self?.surface else { return }
        let action = "search:\(needle)"
        ghostty_surface_binding_action(surface, action, UInt(action.lengthOfBytes(using: .utf8)))
    }
```

* Needles **shorter than 3 chars are debounced 300ms** to "avoid kicking off
  expensive searches"; 3+ chars and empty needles go through immediately.
* When `searchState` goes nil (`oldValue != nil`), it sends `end_search` back
  to the core (cleanup, see the §2.3 note).

### 7.4 The overlay UI (`SurfaceView.swift:366-459`)

`SurfaceSearchOverlay` is a SwiftUI view: a search text field
(`BackportSelectionTextField`) bound to `searchState.needle` and
`needleSelection`, a match counter overlay, and prev/next/close buttons.

* Counter: `"\(selected + 1)/\(searchState.total, default: "?")"` — the C
  index is 0-based, display is 1-based (`selected + 1`).
* Prev/next buttons call `surfaceView.navigateSearchToNext()` /
  `navigateSearchToPrevious()` (`OSSurfaceView.swift:158-181`), which send
  `"navigate_search:next"` / `"navigate_search:previous"` via
  `ghostty_surface_binding_action`. (`onSubmit` / Return → next; Shift+Return →
  previous.)
* Close button and `onExitCommand`: if needle is empty, `onClose()` (hides
  overlay); otherwise move focus back to the surface without closing.

### 7.5 Menu shortcuts (`AppDelegate.swift:1167-1172`)

```swift
syncMenuShortcut(config, action: "start_search", menuItem: self.menuFind)
syncMenuShortcut(config, action: "end_search", menuItem: self.menuHideFindBar)
syncMenuShortcut(config, action: "search_selection", menuItem: self.menuSelectionForFind)
syncMenuShortcut(config, action: "navigate_search:next", menuItem: self.menuFindNext)
syncMenuShortcut(config, action: "navigate_search:previous", menuItem: self.menuFindPrevious)
```

---

## 8. GTK / Linux GUI

Files: `src/apprt/gtk/class/{search_overlay,surface,application}.zig`,
`src/apprt/gtk/ui/1.2/search-overlay.blp`.

### 8.1 The overlay widget class (`search_overlay.zig`)

`SearchOverlay` is a GObject class (`Adw.Bin` root) defined by the Blueprint
template `search-overlay.blp`. The template wires:

* A `Gtk.SearchEntry` (`search_entry`) with `stop-search`, `search-changed`,
  `next-match`, `previous-match` signals, plus a capture-phase
  `EventControllerKey` so `Return`/`Shift+Return` and Escape are handled before
  the entry. (`GtkSearchEntry` itself debounces `search-changed`.)
* A `Label` showing the match counter, bound to a closure:
  ```
  label: bind $match_label_closure(template.has-search-selected, template.search-selected,
                                    template.has-search-total, template.search-total)
  ```
* Prev/next buttons and a close button. Curious detail: in the current blp the
  click handlers are **swapped relative to the icons/tooltips** —
  `prev_button` (icon `go-up-symbolic`, tooltip "Previous Match") is wired to
  `$next_match()`, while `next_button` (icon `go-down-symbolic`, tooltip "Next
  Match") is wired to `$previous_match()`.

**Properties** (`search_overlay.zig:58-114`): `active`,
`search-total` / `has-search-total`, `search-selected` / `has-search-selected`.
The `has-*` properties exist so the label closure can distinguish "0 matches"
from "unknown yet". `setSearchTotal`/`setSearchSelected`
(`search_overlay.zig:261-282`) only `notifyByPspec` when the value actually
changed, and additionally notify the `has-*` property when presence flips.

**Signals** (`search_overlay.zig:158-172`): `stop-search` (Escape/close),
`search-changed` (debounced on the Gtk side too — see `setSearchActive`),
`next-match`, `previous-match`.

### 8.2 Surface integration (`surface.zig`)

* `setSearchActive(active, needle)` (`surface.zig:2235`): sets the `active`
  property (shows/hides the overlay), sets the entry contents if needle is
  non-empty, and `grabFocus()`s the entry when activating.
* `setSearchTotal` / `setSearchSelected` (`surface.zig:2254-2260`): thin
  forwards to the overlay.
* Template callbacks (`surface.zig:3585-3611`) translate widget signals back
  into binding actions on the core surface:

  ```zig
  fn searchStop(...) { _ = surface.performBindingAction(.end_search) ...; }
  fn searchChanged(_, needle, self) {
      _ = surface.performBindingAction(.{ .search = std.mem.sliceTo(needle orelse "", 0) }) ...;
  }
  fn searchNextMatch(...) { _ = surface.performBindingAction(.{ .navigate_search = .next }) ...; }
  fn searchPreviousMatch(...) { _ = surface.performBindingAction(.{ .navigate_search = .previous }) ...; }
  ```

### 8.3 App-level action dispatch (`application.zig`)

The core's upward actions are dispatched at `application.zig:783-786`:

```zig
.start_search => Action.startSearch(target, value),
.end_search => Action.endSearch(target),
.search_total => Action.searchTotal(target, value),
.search_selected => Action.searchSelected(target, value),
```

which resolve to (`application.zig:2702-2728`):

```zig
pub fn startSearch(target: apprt.Target, value: apprt.action.StartSearch) void {
    switch (target) {
        .app => {},
        .surface => |v| v.rt_surface.surface.setSearchActive(true, value.needle),
    }
}
pub fn endSearch(target: apprt.Target) void {
    switch (target) {
        .app => {},
        .surface => |v| v.rt_surface.surface.setSearchActive(false, ""),
    }
}
pub fn searchTotal(...) { .surface => |v| v.rt_surface.surface.setSearchTotal(value.total), }
pub fn searchSelected(...) { .surface => |v| v.rt_surface.surface.setSearchSelected(value.selected), }
```

`app` targets are no-ops for search actions (same as the macOS warning logs).

---

## 9. End-to-End Walkthroughs

### 9.1 Typing a needle (Linux)

1. User hits `super+f` → core `Surface.performBindingAction(.start_search)` →
   `rt_app.performAction(.start_search, needle="")` →
   `application.zig Action.startSearch` → `surface.setSearchActive(true, "")` →
   overlay shown and search entry focused.
2. User types "hello" → `search_changed` signal → `searchChanged` callback →
   `surface.performBindingAction(.{ .search = "hello" })` → `Surface.zig:4956`.
3. No search thread yet → spawned with `event_cb = &searchCallback`. Then
   `change_needle("hello")` pushed to the thread mailbox + wakeup.
4. Thread ticks: feeds terminal (under renderer mutex), runs
   `ActiveSearch`/`ScreenSearch`/`ViewportSearch` over history, active area,
   and viewport.
5. Events flow back: `viewport_matches` → renderer mailbox → renderer paints
   `search_match` highlights; `total_matches` → surface mailbox →
   `rt_app.performAction(.search_total)` → `application.zig searchTotal` →
   `setSearchTotal` → GObject property notify → the blp `match_label_closure`
   updates the counter to "N".
6. User presses Return → `searchEntryKeyPressed` → `next-match` signal →
   `searchNextMatch` → `performBindingAction(.{ .navigate_search = .next })` →
   thread `.select(.next)` → `ScreenSearch.selectNext` (wraps) →
   `screen.scroll(pin)` moves the viewport to the match → `selected_match`
   event → `search_selected` action → overlay counter shows "K+1/N".

### 9.2 Typing a needle (macOS)

Same core path; the differences are all in step 1–2:

1. `super+f` keybinding → core `.start_search` → `Ghostty.App.swift startSearch`
   → creates `SearchState` (reads `NSPasteboard.find` if needle empty) → posts
   `.ghosttySearchFocus`.
2. User types → SwiftUI `$searchState.needle` → Combine pipeline (300ms debounce
   if < 3 chars) → `ghostty_surface_binding_action(surface, "search:hello")` →
   same core path as §9.1.
3. `search_total`/`search_selected` actions → `Ghostty.App.swift` handlers →
   update `searchState` on main thread → SwiftUI counter re-renders.

### 9.3 Search for a selection

`search_selection` binding (default `super+e`) → `Surface.zig:4926` reads
`selectionString` → forwards as `.start_search` with the selected text as the
needle → GUI shows the overlay pre-filled and focused.

### 9.4 Ending a search

`end_search` (Escape / `super+shift+f`, or closing the GUI) →
`Surface.zig:4936` stops the thread (join) and always notifies the GUI →
macOS `endSearch` clears `searchState` (overlay disappears) / GTK
`setSearchActive(false)`.

---

## 10. Configuration & Defaults

Colors (`Config.zig:1108-1124`):

| Key | Default |
|---|---|
| `search-foreground` | black |
| `search-background` | `#FFE082` (yellow) |
| `search-selected-foreground` | black |
| `search-selected-background` | `#F2A57E` (salmon) |

Default keybindings (`Config.zig:7150-7183`):

| Key | Action |
|---|---|
| `super+f` | `start_search` |
| `super+e` | `search_selection` |
| `super+shift+f`, `escape` | `end_search` |
| `super+g` | `navigate_search:next` |
| `super+shift+g` | `navigate_search:previous` |

All search actions are marked `.performable = true`, so they are exposed to the
command palette (`input/command.zig:186-211`) and can be invoked by name from
CLI/config.

---

## 11. Threading Model & Gotchas

1. **Search runs on its own thread**, spawned lazily on first needle and joined
   on `end_search` / surface teardown. The surface `deinit` calls
   `Search.deinit` which joins the thread before freeing state — do not free
   the thread's state from another thread.

2. **The search thread reads the terminal under the renderer-state mutex.**
   It only copies the bytes/pages it needs while holding the lock (the sliding
   window's "update while locked, match unlocked" design), which is what keeps
   the terminal responsive during a search.

3. **`searchCallback` runs on the search thread.** It may only touch the
   surface through mailboxes. Renderer messages carry their own arenas; the
   renderer thread frees the old arena when it installs a new one.

4. **Renderer highlights are asynchronous** — `search_viewport_matches` may
   describe a slightly older viewport than what's on screen. The renderer must
   tolerate this ("may be off for a frame or two", `message.zig:56-57`).

5. **Empty needle vs. end_search**: an empty needle cancels the search but
   keeps the GUI visible; only `end_search` hides the GUI. macOS exploits this
   so clearing the field doesn't close the overlay, while closing the overlay
   sends `end_search` back to clean up core state.

6. **Null encoding**: C uses `ssize_t` with `-1` for "unknown/no value".
   macOS decodes to `UInt?`; GTK uses `?usize` internally and the
   `has-search-total`/`has-search-selected` properties for the label binding.

7. **Match index is 0-based in the API, 1-based in the UI.** The GUIs add 1
   when displaying (`selected + 1`).

---

## 12. Testing

* Search engine tests live alongside the code (each `search/*.zig` file has a
  `test { ... }` block with many cases):
  * `screen.zig`: wrap-around next/prev, history + active interactions, clear
    screen, failure paths.
  * `active.zig` / `viewport.zig` / `pagelist.zig`: simple substring, viewport
    change detection, allocator-failure paths.
  * `Thread.zig`: mailbox-driven behavior.
* Run targeted:
  ```sh
  zig build test -Dtest-filter=screen  # or active / viewport / pagelist / sliding_window
  ```
  or the whole suite with `zig build test`.
* The C ABI sync is itself tested: `apprt/action.zig:424` "ghostty.h
  Action.Key" runs `lib.checkGhosttyHEnum` against `include/ghostty.h`.
* C API types for the VT library also carry a sentinel rule: every enum in
  `include/ghostty/vt/` must end with `_MAX_VALUE = GHOSTTY_ENUM_MAX_VALUE`
  (see AGENTS.md).

---

## 13. Suggested Learning Path

Read in this order; each step builds on the previous:

1. **The contract**: `src/apprt/action.zig:320-344` (Action union) →
   `:997-1040` (StartSearch/SearchTotal/SearchSelected + C mirrors). This is
   the single source of truth.
2. **The downward side**: `src/input/Binding.zig:409-433` (search action
   members) → `:1253-1317` (`parse`). Then `src/apprt/embedded.zig:1963`
   (`ghostty_surface_binding_action`).
3. **The core**: `src/Surface.zig:4915-5015` (search handlers) →
   `:1426-1541` (`searchCallback`) → `:1166-1180` (mailbox → apprt).
4. **The engine**: `src/terminal/search/Thread.zig` (lifecycle + loop) →
   `sliding_window.zig` (matcher) → `active.zig` → `viewport.zig` → `screen.zig`
   → `pagelist.zig`.
5. **The renderer**: `src/renderer/message.zig:55-78` → `renderer/Thread.zig:
   482-496` → `renderer/generic.zig:1304-1347` (highlight application) and
   `:2740-2850` (cell coloring).
6. **The C ABI**: `include/ghostty.h:868-881` and the `GHOSTTY_ACTION_*` tag
   enum; read it against the `// Sync with:` comments in `action.zig`.
7. **One GUI end-to-end**: pick GTK (`search_overlay.zig` →
   `surface.zig:3585-3611` → `application.zig:2702-2728`) or macOS
   (`Ghostty.App.swift:2111-2208` → `OSSurfaceView.swift` SearchState →
   `SurfaceView_AppKit.swift` debounce → `SurfaceView.swift` overlay), then do
   the other for the contrast.
