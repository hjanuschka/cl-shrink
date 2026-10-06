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
```

| option | effect |
|---|---|
| `-n`, `--dry-run` | report sizes and exit without touching anything |
| `-a`, `--aggressive` | `--window=250 --depth=50` |
| `-f`, `--force` | re-process repos already shrunk since their last fetch |
| `--include-local` | also prune tags/reflogs in repos holding local commits |
| `--min-size MB` | skip repos with a `.git` smaller than this (default 20) |
| `--max-pack GB` | skip repos whose largest pack exceeds this |

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

### The one thing it will not do

Your main `src/.git` is usually the biggest single object store (~65 GB for Chromium) and
it is excluded in practice: repacking it needs ~65 GB of free space, which is exactly what
you do not have when you reach for this tool. Run the dependency pass first, then point
`cl-shrink --include-local` at it once there is headroom.

If you keep several checkouts, note that each one stores a full copy of the same
`src` history. Sharing it via `objects/info/alternates` saves far more than repacking
ever will -- but a `gc` in the donor repository can corrupt the borrowers and gclient
knows nothing about it, so `cl-shrink` deliberately stays away from that.

## License

MIT
