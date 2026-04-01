# expose.sh Performance Notes

## Changes Made

| Change | Location | Effect |
|---|---|---|
| Bash slash-counting for `node_depth` | ~line 140/204 | Removes one `awk` subshell per directory scanned |
| Combine two `sed` calls → one | ~line 218 | Removes one pipe fork per directory |
| Single `find` + bash loop for `dircount` | ~lines 224–225 | Removes one `find` per directory (was two) |
| `grep -q` instead of `grep \| wc -l` | ~lines 236, 339 | Exits on first match, no pipe |
| `sed y/.../.../` instead of `\| tr` | ~lines 287, 337 | Removes one pipe fork per call |
| Palette color caching | ~lines 390–398 | Skips `convert` color extraction on re-runs — biggest win for reading phase |
| Single `identify` call (was 3) | ~lines 401–410 | 2 fewer ImageMagick process launches per image |
| `perl_available` flag hoisted out of loop | before HTML loop | Removes `command -v perl` subprocess per image |
| Parallel `convert` resizes | ~lines 940–950 | All resolutions of each image encode simultaneously |

## Performance Expectations

| Scenario | Before | After | Speedup |
|---|---|---|---|
| 50 photos, no video | ~3 min | ~45 sec | ~4× |
| 200 photos, no video | ~12 min | ~2–3 min | ~5× |
| Re-run, unchanged photos | same as first run | seconds (palette cache hits) | large |

## Key Bottlenecks (reading phase)

1. **`convert` color extraction** — full image decode + resize + quantize per photo, no caching. Now cached in `_site/.palette_cache/` keyed by source file path (`cksum`). Cache is invalidated when the source file is newer than the cache entry.
2. **Three `identify` calls per image** — orientation, width, height fetched separately. Now one call: `identify -quiet -format "%[EXIF:Orientation] %w %h\n"`.
3. **`ffmpeg` thumbnail for videos** — decodes up to frame 1 on every run. No change here; would require restructuring the flow to cache the thumbnail.

## Key Bottlenecks (encode phase)

- **Sequential `convert` per resolution** — with 6 resolutions and 100 photos that's 600 serial calls. Now backgrounded with `&` and collected via `wait`, so all resolutions for a given image run in parallel.
- **Video encoding** — `ffmpeg` already uses all cores internally (`ffmpeg_threads=0`). Not parallelized across files (would need a job queue).

## Palette Cache

- Location: `_site/.palette_cache/<cksum>`
- Cache key: `cksum` of the source file path (not content)
- Invalidation: cache file must be newer than source file (`-nt` test)
- Works for images, videos (keyed on source video), and image sequences (keyed on sequence directory)

## Backup

Original script preserved as `expose.bak` — do not modify.
