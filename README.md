# cl-shrink

Reclaim disk space in [gclient](https://www.chromium.org/developers/how-tos/depottools/)-managed
Chromium checkouts by rebuilding the git delta chains of their `third_party` repositories.

Measured **50-65% reclaim** across ~750 dependency repos in three Chromium checkouts
(100 GB of `.git` down to roughly 40 GB).

```
$ cl-shrink --dry-run ~/chromium
scanning for git repositories...
found 278 repositories

  chromium/src/third_party/skia                     1.03 GB  repack only (local commits)
  chromium/src/third_party/swiftshader              1.17 GB  repack+prune
  chromium/src/third_party/tflite/src               1.57 GB  repack+prune
  chromium/src/v8                                   1.55 GB  repack+prune
  SKIP  chromium/src                                needs 64 GB free, have 31 GB
  ...

175 repositories processed, 214 skipped, 0m elapsed
candidate .git total: 51.6 GB -- expect 50-65% reclaim (25-33 GB)
```

## Why checkouts bloat

`gclient sync` updates dependencies with incremental `git fetch`. Each fetch appends a
*thin pack* whose deltas are never recomputed against the rest of the history, and nothing
ever prunes. After a few months a dependency ends up with dozens of packs holding
objects that barely share any deltas:

| repo | packs | `.git` |
|---|---|---|
| `src/third_party/skia` | 50 | 1.01 GB |
| `src/third_party/tflite/src` | 16 | 1.57 GB |

`git gc` does not fix this: it concatenates the packs but *reuses* the existing deltas.
You need `git repack -f`, which throws them away and searches for new ones.

| step | skia `.git` |
|---|---|
| untouched | 1.01 GB |
| `git gc --prune=now` | 0.90 GB (-13%) |
| `git repack -a -d -f` | **0.40 GB (-61%)** |
| `git repack -a -d -f --window=250 --depth=50` | 0.33 GB (-68%) |

On top of that, dependency repos carry refs nobody needs -- `tflite` ships 452 tags and
7721 remote-tracking refs -- and DEPS pins every git dependency by SHA, so the tags only
keep otherwise-unreachable objects alive. `cl-shrink` deletes them before repacking.

Everything it removes is restored by `gclient sync`.

## Install

```bash
curl -fsSL https://raw.githubusercontent.com/hjanuschka/cl-shrink/main/cl-shrink \
  -o ~/.local/bin/cl-shrink && chmod +x ~/.local/bin/cl-shrink
```

Requires bash 4+ and git 2.20+. No other dependencies.

## Usage

```bash
cl-shrink --dry-run ~/chromium              # report, change nothing
cl-shrink ~/chromium ~/chromium_2           # shrink, multiple roots are fine
cl-shrink -a ~/chromium                     # --window=250: ~7% smaller, ~4x the CPU
cl-shrink --top-level ~/chromium            # include src/ itself (see below)
```

| option | effect |
|---|---|
| `-n`, `--dry-run` | report sizes and exit without touching anything |
| `-a`, `--aggressive` | `--window=250 --depth=50` |
| `-f`, `--force` | re-process repos already shrunk since their last fetch |
| `--include-local` | also prune tags/reflogs in repos holding local commits |
| `--min-size MB` | skip repos with a `.git` smaller than this (default 20) |
| `--max-pack GB` | skip repos whose largest pack exceeds this |
| `--top-level` | also shrink the checkout's own solution repos (`src`, `angle`, ...) |

Budget roughly **4 minutes per GB** of `.git` on 16 cores, so a full checkout takes a
couple of hours. It is interruptible; re-running picks up where it stopped.

## Safety

- Refuses to start while `gclient`, `autoninja` or `siso` is running.
- Per-repo free-space guard. `git repack` writes the new pack before unlinking the old
  one, so it needs headroom equal to the largest existing pack; repos that do not fit are
  skipped rather than filling the disk.
- Repos holding local-only commits or stashes are **repacked but never pruned**, so your
  branches, tags and reflogs survive. `--include-local` opts out of that protection.
- A failed `repack` leaves the repository untouched and cleans up `tmp_pack_*`.
- A stamp file per repo means a re-run skips anything that has not been fetched since.

## `--top-level`: the checkout's own repos

The gclient solution roots (`<checkout>/src`, and `angle`, `skia`, ... if you have them)
are the biggest object stores you own -- ~65 GB for Chromium `src` -- and they are the
ones holding your branches. They are **opt-in**, and they are processed **last**, after
the dependency pass has freed space, so that they can clear the headroom check within a
single run.

```
$ cl-shrink ~/chromium                     # default
  skip  chromium/src   solution repo (--top-level to include)
```

Full `repack -a` has to reproduce every pack before dropping the originals, so it needs
free space equal to the largest existing pack. For a 63 GB clone pack that is 63 GB of
headroom, which is exactly what you do not have when you reach for this tool. Rather than
give up, `--top-level` falls back to a **geometric repack** (`repack --geometric=2 -d`):
it leaves the giant clone pack alone and rolls up only the small per-fetch packs, so peak
usage is a couple of GB.

That fallback is precisely aimed at the "grows and never shrinks" problem, because the
per-fetch packs *are* the growth. Measured on a real `src` with six months of syncs on it:

```
BEFORE: 68049 MB, packs=29
git repack --geometric=2 -d      1m38s
AFTER:  66579 MB, packs=4        -1469 MB
```

Not dramatic in percentage terms, but it is bounded, it takes 98 seconds, and it keeps the
pack count from creeping upward forever. Once the dependency pass has freed enough space,
the same invocation does the full `-f` repack instead and reports a much larger number.

Solution repos are detected as holding local commits, so they are never pruned: your
branches, tags and reflogs survive. Verified on a 65 GB `src` -- 133 branches and 38,897
tags intact, `git fsck` clean.

### The one thing it will not do

If you keep several checkouts, each one stores a full copy of the same `src` history --
197 GB for three. Sharing it via `objects/info/alternates` saves far more than repacking
ever will. But a `gc` in the donor repository can corrupt the borrowers and gclient knows
nothing about the arrangement, so `cl-shrink` deliberately stays away from it.

## License

MIT
