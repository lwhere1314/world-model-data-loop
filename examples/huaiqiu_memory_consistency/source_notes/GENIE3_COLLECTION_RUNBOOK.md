# Genie3 Collection Runbook

This is the short operational path for collecting the 10 missing Genie3 MCTS videos.

## Current Reality

The Project Genie tab is present, but browser automation is not stable enough to safely run a 72s replay at the moment:

- `browser.tabs.list()` can see the `Project Genie` tab.
- Binding the tab and reading URL/title timed out.
- Earlier DOM/screenshot checks also timed out or returned a black system screenshot.
- No replay has been started.

So the current recommended collection mode is:

```text
manual_semantic_action_matched
```

Use Codex/browser replay only after the page can be observed reliably.

The recommended manual helper is:

```bash
python3 scripts/prepare_next_genie3_collection.py
python3 scripts/run_genie3_action_timer.py <protocol_id> --copy-prompt --run --beep
```

## Queue

The collection queue is generated at:

```text
manifests/genie3_collection_queue.csv
```

Each row gives:

- shared prompt;
- Genie3 action list;
- Matrix action CSV for browser replay;
- replay snippet;
- notes path;
- expected output video path.

## Per-Protocol Steps

For each protocol in `manifests/genie3_collection_queue.csv`:

1. Run `python3 scripts/prepare_next_genie3_collection.py` to select the next missing protocol and print commands.
2. Open Genie3 and start a new world.
3. Copy the exact shared prompt from `shared_prompts/<protocol_id>_shared_prompt.txt`.
4. Generate the world and make sure the first view contains the intended Huaiqiu storefront or an accepted equivalent.
5. Start recording or prepare the Genie3 download.
6. Execute `genie3_protocols/<protocol_id>_genie3_action_list.md` at 2s cadence, preferably with `scripts/run_genie3_action_timer.py`.
7. Save or ingest the video as `runs/<protocol_id>/genie3/video.mp4`.
8. Fill `runs/<protocol_id>/genie3/notes.md`.
9. Run `./scripts/refresh_all_manifests.sh`.

## Pilot First

Start with:

```text
huaiqiu_mcts_01_multi_glance_yaw_right_return
```

It is the simplest sanity check for repeated glance and final return:

- initial read: 0-6s;
- short look-away and revisit: 6-18s;
- long offscreen/confuser: 28-48s;
- return path: 48-62s;
- final revisit: 62-72s.

## Browser Replay Safety Gate

Only use browser replay when all of these are true:

1. The page is visible and interactive.
2. Codex can read a screenshot or DOM state without timeout.
3. The generated world viewport is focused.
4. A dry run returns all 36 rows.
5. Recording/download is ready.

Replay snippet for each protocol is already stored at:

```text
runs/<protocol_id>/genie3/browser_replay_snippet.mjs
```

## After Collection

Run:

```bash
cd /Users/hugo/Documents/世界模型面试/记忆一致性实验/06_synthetic_mcts
./scripts/refresh_all_manifests.sh
```

Then check:

```text
manifests/paired_run_status.csv
manifests/paired_metric_table.csv
manifests/benchmark_state_report.json
```

The benchmark is not fully paired until `num_genie3_videos=10`.

## Video Ingest Helper

After recording or downloading a video, use:

```bash
python3 scripts/ingest_genie3_video.py <protocol_id> /absolute/path/to/recording.mp4 --refresh
```

If you are not sure which downloaded file is the recording, scan recent videos:

```bash
python3 scripts/find_genie3_video_candidates.py --write
```

This copies the video to:

```text
runs/<protocol_id>/genie3/video.mp4
```

and appends ingest metadata to `runs/<protocol_id>/genie3/notes.md`.
