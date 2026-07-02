# Genie3 MCTS Collection Status

Date: 2026-07-02

## Current State

The Codex in-app browser is on:

```text
https://labs.google/fx/projectgenie/zh
```

Tab title:

```text
Project Genie
```

Browser connection check:

- `browser.tabs.list()` succeeded and found the Project Genie tab.
- Reading the visible DOM timed out.
- Taking a viewport screenshot timed out.
- A second lightweight check on 2026-07-02 again found the Project Genie tab, but even a tiny page `document.readyState` read timed out.
- A later retry on 2026-07-02 again found the `Project Genie` tab through `browser.tabs.list()`, but binding the tab and reading URL/title by id timed out.
- A system-level screenshot succeeded but returned a black frame, so it could not be used to verify the Genie3 viewport.
- A local Chrome read-only check found no direct Project Genie tab in Chrome, but did find a Google Accounts sign-in tab whose OAuth query references Project Genie. The readiness script intentionally ignores query strings for Project Genie detection, so the current Chrome status is `google_signin_tab_present`. The sanitized status is written to `manifests/genie3_browser_readiness.json`.
- No action replay was started.
- No keys, mouse drags, prompt submission, generation trigger, or download action was performed.

## Interpretation

The browser/page is present, but the page is too heavy or the browser automation connection is not stable enough for safe replay right now. Because action replay would send 72 seconds of keyboard/mouse events, it should not be started until the page can be observed reliably.

This is an automation readiness issue, not a data-design issue.

## Current Genie3 MCTS Output Status

The 10 MCTS Genie3 output slots exist under:

```text
runs/<protocol_id>/genie3/
```

But no MCTS Genie3 videos are currently present:

```text
manifests/genie3_postprocess_index.csv
```

All rows are `missing_video`.

## Safe Next Attempt

Before running `scripts/genie3_replay_actions.mjs`, ensure:

1. Genie3 page is loaded and idle.
2. The generated world is visible and interactive.
3. The target viewport is focused by a click.
4. The page can be observed by Codex browser automation with either DOM or screenshot.
5. Screen recording or Genie3 download is ready.
6. A dry run prints all 36 action rows.

Then use:

```js
var replay = await import("file:///Users/hugo/Documents/世界模型面试/记忆一致性实验/06_synthetic_mcts/scripts/genie3_replay_actions.mjs");
var rows = await replay.readMatrixActionCsv("/Users/hugo/Documents/世界模型面试/记忆一致性实验/06_synthetic_mcts/matrix_game3_csv/huaiqiu_mcts_01_multi_glance_yaw_right_return.csv");
await replay.replayMatrixActionCsv(globalThis.tab, rows, { dryRun: true });
```

Only after the dry run is verified:

```js
await replay.replayMatrixActionCsv(globalThis.tab, rows, {
  dryRun: false,
  secondsPerRow: 2,
  center: { x: 640, y: 360 },
  focusPoint: { x: 640, y: 360 },
  dragPixels: 220,
  keyRepeatMs: 350
});
```

## Fallback

If browser automation remains unstable, collect Genie3 manually with the same semantic action protocol:

```text
genie3_protocols/<protocol_id>_genie3_action_list.md
```

and record in:

```text
runs/<protocol_id>/genie3/notes.md
```

that the run is:

```text
manual_semantic_action_matched
```

This is still valid for the benchmark as long as prompt, action semantics, duration, and evaluation windows are preserved.
