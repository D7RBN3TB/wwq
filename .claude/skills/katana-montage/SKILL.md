---
name: katana-montage
description: Build a fast-cut katana / sword-fight montage from an ingested Montage video — finds the blade action beats (draws, clashes, slashes, finishing strikes), orders them into a hook → build → climax edit, and creates it as a single hook_montage moment with an editor link. Use when the user asks for a katana montage, sword montage, samurai/blade edit, fight highlight reel, or "cut all the sword scenes together".
---

# Katana montage

Turn the sword action in an ingested video into one multi-segment montage
moment. Everything runs on the Montage MCP server. This skill builds on
`ingest-video` (to get the footage in) and `create-moment` (the creation
mechanics) — follow those for anything not covered here.

Argument: a project name/id, or a video URL. With a URL, run the
`ingest-video` skill first and continue once its status is `completed`.

## 1. Resolve the project

- `list_projects` → `get_project` for the project the user means.
- Ingestion must be complete. If an upload is in flight, poll
  `get_upload_status(workflow_id)` (every 30s, not faster than 15s) until
  `completed`.
- `get_moments(project_id)` — if a katana montage already exists, tell the user
  and ask whether to make another variant instead of duplicating it.

## 2. Check the footage can carry a montage

`get_video_validation(project_id)` first:

- `duration_seconds` is the hard upper bound for every cut.
- If `is_valid` is false, report `error_summary` and stop.
- Note `fps`: action reads best at 24+ fps. Below that, warn that fast cuts
  may look choppy, but continue.
- Unlike talking-head clips, a montage does **not** need usable speech. Empty
  speech detection is fine; empty person detection is a warning (no one on
  screen holding a blade) — say so before continuing.

## 3. Find the blade beats

Read all three visual/verbal sources; the montage is driven by the visuals.

- `get_video_analysis(project_id)` — primary source. Walk the scene breakdown
  and clip candidates for sword action: unsheathing/drawing, stances and
  standoffs, clashes/parries, slashes, sparks, a decisive strike, sheathing
  (the classic ending beat), slow-motion moments, kills/defeats.
- `get_video_corpus(project_id, view="rollup")` — confirm the action is on
  screen: who is visible, screen-text inventory, and **hard-veto ranges** (never
  cut inside them). Use `query_video_corpus_range(project_id, start, end)` to
  check a specific stretch when the rollup is ambiguous. Only use
  `view="full"` if both are not enough.
- `get_transcript(project_id, timestamp_format="seconds")` — supporting only:
  battle cries, a one-line taunt or vow for the hook, or a closing line.
  Collect the utterances that overlap the beats you keep.

Low `coverage_fraction` means unanalysed windows, not empty ones. Tell the
user which stretches you could not see rather than treating them as having no
action.

## 4. Choose and order the cuts

Pick 4–10 segments, each **1.5–6 seconds** (a held standoff or slow-mo shot can
run to ~8s). Target total length:

| Platform the user mentions | Total |
| --- | --- |
| TikTok / Reels / Shorts, or unspecified | 20–45s |
| YouTube / longer edit | 45–90s |

Structure them as an arc, in this order (not necessarily source order):

1. **Hook (first 1–3s)** — the single most striking frame: a blade draw, a
   clash with sparks, or a strong line of dialogue. Never open on setup.
2. **Build** — escalating exchanges; alternate wider and tighter shots when the
   analysis distinguishes them, and vary who is attacking.
3. **Climax** — the decisive strike or the biggest clash.
4. **Resolution (optional)** — sheathing the katana, a bow, or the fallen
   opponent. A slow beat here lands the ending.

Cutting rules:

- Start a cut just **before** the motion begins and end it just **after** the
  impact/follow-through — never mid-swing.
- If a cut contains dialogue, snap its edges to transcript sentence boundaries.
- Do not reuse overlapping source ranges; every segment satisfies
  `0 <= start < end <= duration_seconds` and avoids hard-veto ranges.
- If the video has fewer than 4 usable blade beats, say so and offer either a
  shorter montage or a single best-fight `content` moment instead.

Show the user the planned cut list (order, `[start, end]`, one-line beat
description, total runtime) before creating, unless they asked you to just
make it.

## 5. Create the moment

```
create_moment(
  project_id,
  title="Katana Montage — <short subject>",
  segments=[[start, end], ...],      # in montage order from step 4
  utterances=[{"start", "end", "text", "speaker"}, ...],
  summary="<one sentence: who fights, the climax beat>",
  kind="hook_montage",
  file_id=<only if the project has multiple files>
)
```

- `utterances`: transcript entries overlapping the chosen segments, mapping
  `start_time`→`start`, `end_time`→`end`, `speaker_id`→`speaker` in absolute
  video seconds. If no one speaks in any segment, pass an empty list and tell
  the user speaker detection will be skipped (expected for pure action).
- Requires the `moments:write` scope.

## 6. Wait and deliver

- If `preprocessing.status` is `"processing"`, poll
  `get_workflow_status(preprocessing.workflow_id)` every ~30s until
  `completed` (usually 2–10 min). On `failed`/`timed_out`, report it — the
  moment still exists.
- `"skipped"`: relay `preprocessing.reason` (`no_utterances` is normal for a
  dialogue-free montage).
- Give the user the `editor_url` verbatim, plus the final cut list and total
  runtime.
- Suggest finishing touches to apply in the Montage editor (not via these
  tools): a percussive or taiko-style music bed with cuts on the beat, speed
  ramps into impacts, a 9:16 reframe for vertical platforms, and a title card.
