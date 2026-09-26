# Subtitle sync checker: design

**Date:** 2026-09-25 · **Status:** draft for review

## In one paragraph

Some sidecar subtitle files in a home media library are out of sync with their videos. The
text is right, but the timing is off, usually because the subtitle was made for a different
version of the video (a clipped intro, a different cut). `subsync` checks each video's
English `.srt` at a handful of spots. At each spot it transcribes a 30-second clip and finds
those words in the subtitle, which gives the offset there. Every file then gets one verdict.
When all the spots agree on the same offset, `subsync apply` shifts the whole file by that
amount, so it plays in sync in both Plex and Jellyfin with no per-playback adjustment. When
the spots disagree, the file needs a different subtitle, and the report says why. Every
change is backed up and can be undone.

## Decisions made while designing (2026-09-25)

| Question | Decision |
|---|---|
| Which subtitles are out of sync? | Sidecar files next to the video. Subtitle tracks built into the video are out of scope for version 1 |
| How do fixes get applied? | Check everything, write a report, then one command applies the confident fixes. Undo is always available |
| Which languages? | English subtitles on English audio. Everything else is listed as **Not checked** |
| How many spots per video? | 5 per episode, 10 per movie |
| Which fixes are allowed? | One shift for the whole file. A file that is "completely off" needs a different subtitle; the tool never re-times a subtitle line by line |
| Cold open right, everything after the intro off ("trimmed intro") | **Replace**, labelled and counted in the report. A fix may come later if it turns out to be common |
| Speech engine | Parakeet, through `parakeet-mlx` with the English model `mlx-community/parakeet-tdt-0.6b-v2` (a 2.47 GB download). It sits behind a swappable interface, with Whisper (`mlx-whisper`) as the fallback |
| Where it runs | On an Apple Silicon Mac, reading the library over a network share (SMB) |

## Scope

**In version 1:** the library folders listed in `config.local.toml`, each marked as `tv` or
`movies`, and their English `.srt` sidecar files. For version 1 these are the English-language
TV and movie folders. The design assumes libraries of several thousand videos on a network
share.

**Not in version 1, but listed in the report as Not checked** so nothing is silently skipped:

- Non-English sidecar files next to videos in the configured folders (`.spa.srt`, `.swe.srt`
  and so on).
- Forced-only subtitles, image-based subtitles (`.sub`/`.idx`) and other formats (`.smi`,
  `.vtt`, `.ass`).
- Foreign-audio titles inside the configured folders whose audio track is tagged with another
  language, such as a German-language film filed with the English movies. If the audio is
  untagged, these end up as **No match** instead.

**Not looked at at all:**

- Folders not listed in `config.local.toml`, such as other-language libraries (see the anime
  pass under *Later*).
- Subtitle tracks built into the video.
- Subtitle files that don't share their video's name, such as `subs/English.srt`. Plex and
  Jellyfin probably ignore these too.

The command refuses paths outside the configured folders.

## How one file is checked

### 1. Pairing

A video is any file ending in `.mkv .mp4 .m4v .avi .mov .wmv .ts .m2ts .webm .mpg .mpeg .flv
.divx .ogm`. Its sidecars are files in the same folder named `<video name>.<tags>.srt`, where
`<tags>` is empty or a dot-separated list, compared case-insensitively:

| Tags | Result |
|---|---|
| none, `en`, `eng`, `english`, optionally with `sdh`, `cc`, `hi` or `default` | Checked, as long as the text passes the English check below |
| anything containing `forced` | Not checked: forced subtitles only |
| a non-English ISO 639 language code, such as `spa` or `pt-BR` | Not checked: subtitle language (tag) |
| anything else | Not checked: unrecognised name |

Sidecars in other formats get a Not checked row with the format as the reason.

**English check.** After cleanup (step 4.1 below), at least 25% of the subtitle's words must
come from a fixed list of the ~100 most common English words. Otherwise the result is **Not
checked: subtitle text isn't English**. The threshold is confirmed during calibration. The
check applies to tagged files too, so an `.eng.srt` that is really Romanian is caught.

Folders skipped: `Plex Versions`, `*.trickplay`, `@eaDir`, `#recycle`, and any folder whose
name starts with a dot.

A video with two English sidecars (for example `.eng.srt` and `.eng.sdh.srt`) is transcribed
once, and each sidecar gets its own verdict. Quality versions of the same episode are
separate videos with separate sidecars, so they are checked separately.

### 2. Audio track

1. Drop tracks whose title contains "commentary" or whose disposition is `comment`.
2. If any track is tagged English (`eng`/`en`), use the default one of those, or else the
   first.
3. Otherwise, if some track has a real language tag and none is English, the verdict is
   **Not checked: audio isn't English**.
4. Otherwise (all tracks untagged or `und`), use the default track, or else the first.

### 3. Checkpoints

| Folder type in `config.local.toml` | Checkpoints |
|---|---|
| `movies` | 10 |
| `tv` | 5 |
| Any video shorter than 10 minutes | 3 |

`--checkpoints N` overrides these counts.

For a video of length `D`, the first checkpoint starts at `max(30 s, 3% of D)`, and the last
starts at `92% of D − 30 s` so its clip ends before the credits. The rest are spaced evenly
in between. For a 22-minute episode that means clips starting at 0:40, 5:26, 10:12, 14:58
and 19:44.

Each checkpoint is a 30-second mono 16 kHz clip of the chosen audio track. ffmpeg seeks to
the clip's start before opening the input, so only about those 30 seconds of the file are
read over the network.

**Sliding.** If fewer than 15 words are heard, the checkpoint moves 45 seconds later and tries
again, at most twice. It never passes the next checkpoint's start; for the last checkpoint,
it never passes `95% of D − 30 s`.

### 4. Measuring one checkpoint

Offsets use one sign convention throughout: **offset = when the line appears in the subtitle
− when its words are spoken**. A positive offset means the subtitle is late.

1. **Clean both sides.** Lowercase everything and remove punctuation and apostrophes. From
   the subtitle, also remove `<…>` tags, `{…}` codes, `[…]` and `(…)` sound notes,
   `♪ … ♪` lyrics, speaker labels such as `JOHN:`, and leading dialogue dashes.
2. **Rough location.** Every run of 3 consecutive heard words that also appears in the
   subtitle votes for an offset. The vote is the first word's estimated time in the subtitle
   minus the time it was heard. The estimated time is the line's start plus that word's share
   of the line's duration, by word position. Votes go into 2-second bins. The winning bin
   needs at least 3 votes and more votes than any bin not next to it. Otherwise the clip
   counts as unmatched. The whole subtitle is searched, so any size of offset can be found.
3. **Line-level alignment.** Align the heard words with the subtitle words within ±10 s of
   the winning offset, using `difflib` sequence alignment. For each subtitle line whose first
   word is aligned, record `d = line start − time that word was spoken`.
4. **Result.** The checkpoint's offset is the median of the `d` values, and its spread is
   their median absolute deviation (MAD).

| Outcome | When |
|---|---|
| matched | 3 or more lines, MAD ≤ 0.5 s |
| mixed | 3 or more lines, MAD > 0.5 s (the offset changes inside the clip) |
| unmatched speech | 15 or more words heard, fewer than 3 lines matched |
| no speech | fewer than 15 words heard, even after sliding |

Every checkpoint keeps its evidence: the clip's start, its outcome and offset, the first ~20
words heard, and up to 3 matched subtitle lines with their subtitle times and spoken times.

### 5. Verdict

`N` is the number of checkpoints. The **median offset** is the median of the matched
checkpoints' offsets. The thresholds come from `config.toml`. Until calibration they are:
`bias = 0.0 s`, `sync_tol = 0.3 s`, `agree_tol = 0.4 s`, `unmatched_limit = 2`.

The rules apply in this order, and the first that fits wins:

1. **Not checked:** the pair is out of scope or couldn't be read. The reason is recorded.
2. **No match:** no checkpoint found any subtitle lines (none was matched or mixed) and at
   least 3 had unmatched speech. The report says "the subtitle doesn't match the audio: wrong
   subtitle, wrong episode, or the audio isn't English." Checking stops early once 3
   checkpoints have unmatched speech and none has been matched or mixed.
3. **Unclear:** fewer than `ceil(0.6 × N)` checkpoints matched (3 of 5, 6 of 10, 2 of 3), or
   one half of the runtime has no matched checkpoint.
4. **Replace:** any of these:
   - a checkpoint is **mixed** (reason: offset changes inside a clip)
   - the matched checkpoints don't all lie within `agree_tol` of their median (reasons below)
   - `unmatched_limit` or more checkpoints had unmatched speech (reason: speech with no
     subtitle lines, probably a different cut)
5. **In sync:** `|median offset − bias| ≤ sync_tol`.
6. **Shift:** everything else. The shift to apply is `−(median offset − bias)`, rounded to the
   millisecond.

A single checkpoint with unmatched speech is allowed and is noted in the report.

**Reasons when the matched checkpoints disagree.** A *jump* is a pair of neighbouring matched
checkpoints, in time order, whose offsets differ by more than `agree_tol`. The reasons are
tested in this order:

| Reason | When | Report shows |
|---|---|---|
| drift | A least-squares straight line through (time, offset) fits every matched checkpoint within `agree_tol / 2` | Seconds of drift per 10 minutes, plus the closest frame-rate pair if the implied speed ratio is within 0.1% of 25/23.976, 25/24, 24/23.976 or 30/29.97 (either way round) |
| trimmed intro | A TV episode whose only jump is between checkpoints 1 and 2, both matched. In a movie the same pattern counts as a plain jump | The offset before and after |
| jump | Exactly one jump anywhere else | Both offsets and the time range the jump is in |
| several jumps | Two or more jumps | The offset at each checkpoint |

## The fix

Only files with a **Shift** verdict change.

- Every start and end time moves by the shift.
- A line that now ends at or before 0:00 is removed. This is usually a "previously on" the
  video doesn't have. A line that now starts before 0:00 but ends after it starts at
  `00:00:00,000`.
- If any line was removed, the remaining lines are renumbered from 1. Only the index lines
  change.
- Everything else stays byte-for-byte identical: the text, encoding, byte-order mark, line
  endings, blank lines, and any position codes after the timestamps. Timestamps are written
  as two-digit hours, minutes and seconds plus three-digit milliseconds, keeping the
  original's separator before the milliseconds (`,` or `.`).
- UTF-16 files are decoded and re-encoded with the same byte order and byte-order mark. Files
  in any other encoding are edited as bytes, because timestamps are plain ASCII in every
  ASCII-compatible encoding.

Shifting a file by `+s` and then by `−s` must reproduce the original bytes whenever no line
was removed. This is one of the tests.

## Commands

| Command | What it does |
|---|---|
| `subsync check [PATH …]` | Finds pairs under the given paths, or under all configured folders by default. Checks them, writes a new run's results and report, and prints a summary with the report's path. `--checkpoints N` overrides the checkpoint count. `--limit N` stops after N videos, which is handy for trials. `--pairs FILE` takes a tab-separated list of explicit (video, subtitle) pairs instead, used for calibration and fault tests. `--runs-dir DIR` puts the run somewhere other than the configured state folder, for example a scratch folder |
| `subsync report [--run ID]` | Rebuilds `report.html` from a run's results |
| `subsync apply [--run ID]` | Shifts every **Shift** file from the latest completed run. `--dry-run` lists what would change. `--only TEXT` limits it to paths containing TEXT. `--exclude FILE` skips the paths listed in FILE. Refuses an unfinished run unless given `--allow-incomplete` |
| `subsync undo [--run ID]` | Restores a run's originals, newest first. `--dry-run` shows what would be restored |

Stopping `check` part-way leaves an unfinished run. Running `check` again starts a new run.
Transcripts and video details (duration, audio tracks) are cached. So redoing work already
done costs only the folder walk and the matching: a few minutes for a whole library.

## Where things live

| What | Where |
|---|---|
| Code | This repository. Python managed with uv. The command is `subsync`, run as `uv run subsync …` |
| Shared settings | `config.toml`, committed: checkpoint counts, thresholds and the speech model. Updated after calibration, with the calibration date noted |
| Local settings | `config.local.toml`, git-ignored: the library folders (each marked `tv` or `movies`), the share's mount point, and the state folder. The repository ships `config.local.example.toml` with placeholder paths |
| Cache | `.cache/cache.sqlite`, git-ignored. Video details are keyed by video path, size and modification time. Transcripts are keyed by those plus audio track, clip start, clip length and engine/model |
| Speech model | The standard Hugging Face cache |
| Runs and backups | `<state folder>/runs/<YYYY-MM-DD_HHMMSS>/`, one folder per `check`. The state folder should be on the same share as the library, so backups stay with the media |
| Hiding the state folder from the servers | On first use, `check` creates an empty `.ignore` (for Jellyfin) and a `.plexignore` containing `*` and `*/*` in the state folder |
| Library-specific notes | `docs/private/`, git-ignored (see *Public repository*) |

**Inside a run folder**

| File | Contents |
|---|---|
| `run.json` | Start and finish times (no finish time means unfinished), paths, checkpoint rule, engine and model, thresholds, and the tool's git commit |
| `results.jsonl` | One line per pair: the video (path, size, modification time), the subtitle (path, SHA-256, tags), audio track, duration, checkpoints with evidence, verdict, reason and shift. Written after every pair |
| `report.html` | The report |
| `applied.jsonl` | One line per changed file: path, backup path, SHA-256 before and after, shift, lines removed, whether the read-only flag was set, and the time |
| `undone.jsonl` | One line per restore, or per skip with the reason |
| `backup/` | Originals, at their path relative to the share's mount point |

## The report

`report.html` is a single self-contained file, with no network access, that opens in a
browser straight from the share.

- **Top:** counts per verdict, and counts per reason. That is where the trimmed-intro count
  shows up.
- **Table:** Verdict · Show or movie · File · Offset · Shift to apply · Checkpoints matched
  (for example 5/5) · Reason. It can be sorted by any column and filtered by verdict and by
  text.
- **Row detail:** each checkpoint's time, outcome, offset and number of matched lines, the
  words heard, and the matched subtitle lines with their subtitle times and spoken times.
  This is the evidence for eyeballing a verdict.

## Safety rules

1. Everything is read-only except the `.srt` files being fixed and the state folder. Video
   files, NFOs, artwork and other subtitles are never written.
2. Nothing is renamed, moved or deleted. So no library scans need pausing, and shows with
   Plex optimized versions can be included.
3. Before writing, `apply` checks that the subtitle's SHA-256 still equals the one recorded at
   check time, and that the video's size and modification time are unchanged. If either
   changed, the file is skipped.
4. Writing a file happens in this order:
   1. Copy the original to `backup/`, keeping its timestamps but not its read-only flag, so
      the backups can be cleaned up later.
   2. Write the new content to `.<name>.subsync-tmp` in the same folder. The leading dot
      makes Plex and Jellyfin ignore it.
   3. If the file has the read-only flag (`uchg`), clear it.
   4. `os.replace` the temp file over the original.
   5. Restore the read-only flag if it was set.
   6. Re-read the file and check its hash.
   7. Log it in `applied.jsonl`.
5. `undo` only restores a file whose current hash equals the "after" hash in `applied.jsonl`,
   so it never overwrites a later edit.
6. `apply` refuses unfinished runs unless given `--allow-incomplete`.

## Errors and edge cases

- An error on one file never stops the run. The pair gets **Not checked** with the error as
  the reason.
- Before starting, `check` makes sure the configured mount point is mounted. If it isn't, it
  stops with a clear message.
- Each ffmpeg clip extraction has a 120-second timeout, to cover slow network reads or AVI
  files without an index. A timeout makes the pair **Not checked**.
- Unreadable subtitles, subtitles with no lines, videos with no duration or no audio track,
  and videos shorter than 2 minutes are all **Not checked**, each with its own reason.
- If an interrupted write left a `.<name>.subsync-tmp` file behind, `apply` deletes it before
  writing that file again.
- Ctrl-C leaves the run unfinished. Every `results.jsonl` line already written stays valid.
- All subprocesses get their arguments as lists, never through a shell, so names with
  apostrophes and braces are safe.

## Components

Each piece has one job and can be tested alone.

| Piece | Input → output |
|---|---|
| `config` | `config.toml` + `config.local.toml` → settings, with clear errors for missing or invalid paths |
| `pairs` | Folders → (video, sidecar, tags) pairs, plus Not checked rows for out-of-scope sidecars |
| `media` | Video → duration and chosen audio track (or a Not checked reason). Cached |
| `clips` | Video, track, start, length → 16 kHz mono samples |
| `speech` | Samples → words with start and end times. Parakeet by default, and cached |
| `subtitles` | `.srt` bytes → lines (start, end, cleaned words), encoding and byte spans. Also the English check, and `shift(bytes, seconds) → bytes` |
| `match` | Heard words + subtitle lines → checkpoint outcome, offset, MAD and evidence |
| `verdict` | Checkpoints + thresholds → verdict, reason and shift |
| `report` | `results.jsonl` → `report.html` |
| `apply` | Run → changed files, backups and `applied.jsonl`; also `undo` |
| `cli` | The four commands |

## Testing and calibration

**Unit tests (pytest)** use synthetic data only. They cover:

- reading `.srt` files in UTF-8 with and without a byte-order mark, Windows-1252, and UTF-16,
  with Windows line endings, and malformed spacing
- the byte-for-byte shift, including the +s/−s round trip, removed lines, clamping to 0:00,
  and renumbering
- pairing, including video names that contain dots, such as `a.k.a.` in an episode title:
  the tags are whatever follows the full video name
- text cleanup
- matching with a known offset, with missing and extra words
- every verdict and reason

**Planted faults on real audio.** Take about 10 videos whose built-in English text subtitle
track is known to be in sync with the video. Extract each track into a scratch folder outside
the repository, never into the library or the repository. Then make faulty copies:

| Planted fault | Expected verdict |
|---|---|
| Shifted +2.5 s | Shift −2.5 s |
| Shifted −40 s | Shift +40 s |
| +30 s jump at 40% of the runtime | Replace (jump) |
| Stretched by 25/23.976 | Replace (drift) |
| Another episode's subtitle | No match |
| A Spanish subtitle | Not checked (subtitle text isn't English) |

**Calibration.** Run about 30 known-good pairs made the same way, with `--pairs`, to measure:

- `bias`: the median offset over all matched checkpoints
- the measurement noise: 1.4826 × the median distance between each checkpoint's offset and
  its own file's median offset, pooled over all files

Then set `sync_tol = max(0.3 s, 3 × noise)` and `agree_tol = max(0.4 s, 4 × noise)`. Also
confirm the 25% English-word threshold, and that `unmatched_limit` doesn't trip on
known-good files. The measurements stay in the runs folder; only the resulting thresholds go
into `config.toml`.

If Parakeet's noise is above 0.15 s, try `mlx-whisper` with the `large-v3-turbo` model and
word timestamps, and keep whichever is lower.

**Pass criteria**

- Every planted fault gets its expected verdict. Planted shifts are recovered within ±0.15 s.
- No known-good pair gets Shift, Replace or No match.
- In the pilot, the owner finds the fixed episodes in sync in both Plex and Jellyfin.

## Rollout

1. **Calibrate** and pass the planted-fault tests.
2. **Pilot** `check` on one show with many sidecar files and older `.avi` files, and read the
   report together. The chosen show is recorded in `docs/private/`.
3. **Fix one file** and play it in Plex and in Jellyfin. Settle whether either server needs a
   metadata refresh to show the change, and confirm both can still read the rewritten file.
   If a refresh is needed, `apply` prints the show and movie folders to refresh.
4. **Apply the pilot show.** The owner spot-checks a few episodes.
5. **Whole library:** `check` overnight, then report, apply and spot-check.
6. **Record it:** add the tool, its location and how to undo to the owner's media-setup
   notes, which live outside this repository.

## Public repository

This repository is public, but the library it runs on is private. `CLAUDE.md` has the full
rules. In short:

- **Never published,** in files, commit messages or GitHub text: titles, file names, paths,
  hostnames, counts, or anything extracted from the library (subtitle text, audio,
  transcripts, reports, calibration measurements).
- **Where those things live instead,** all git-ignored:
  - `config.local.toml` for library settings
  - `docs/private/` for notes and test cases
  - `.cache/` and the runs folder for outputs
- **Test fixtures** are synthetic.
- **The library snapshot behind this design** (sizes, counts, example files) is in
  `docs/private/library-notes.md`.

The implementation includes a guard for this. `scripts/check_private.py` scans staged changes
and the commit message against `docs/private/deny-list.txt`, and is installed as local
`pre-commit` and `commit-msg` hooks. The script is generic; the list it reads stays private.

## To verify during the build

These are facts to check, not decisions to make.

1. Do Plex and Jellyfin serve an edited sidecar straight away, or only after a refresh? And
   can both still read it? `os.replace` creates a new file, which takes its permissions from
   the folder. Plex reads the share on the NAS itself, and Jellyfin reads it over NFS from
   another machine. (Rollout step 3.)
2. Which Python version `parakeet-mlx` and MLX ship wheels for. The system Python may be
   newer; uv pins whatever works.
3. How `parakeet-mlx` takes audio: as an in-memory array through its lower-level API, or as a
   temporary WAV file.
4. Throughput over the network share, measured in the pilot, which gives the full run's time
   estimate.

## Later (not version 1)

- **Anime and foreign-audio pass.** Word matching can't work for these, so this pass needs a
  language-independent method, such as comparing speech timing with subtitle timing (alass is
  a candidate). It must also handle subtitle tracks built into the video: extract the track,
  check it, and write a corrected sidecar. The owner has reported two first test cases, both
  off:
  - a series whose episodes have a built-in English ASS track over Japanese audio
  - a film with an untagged English `.srt` sidecar over Japanese audio

  The exact files are in `docs/private/`.
- **Trimmed-intro fix**, if the report shows it is common: shift only the lines after the
  theme.
- **Non-English sidecars** on English shows.
- **Frame-rate stretch fix** for drift, if the owner wants it.
- **Sidecars Plex and Jellyfin can't see:** subtitle files that don't share their video's
  name.
- **Mislabelled languages**, such as an untagged `.srt` that is really Romanian. The report
  lists them; renaming them is a separate cleanup.
