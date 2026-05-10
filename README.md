<div align="center">
<h1>video-fixtures</h1>
</div>

Shared video fixtures for findit-studio's media stack
([`mediadecode`](https://github.com/findit-ai/mediadecode),
[`scenesdetect`](https://github.com/findit-ai/scenesdetect), …).

The clips are real footage — two source pieces (an airport
walk and a panda zoo clip) re-encoded into the container /
codec combinations the upstream consumers need to demux and
decode. Format coverage is the point: every consumer should
exercise its decode + demux paths against representative
content rather than synthetic test patterns.

Video gets its own repo (separate from
[`audio-fixtures`](https://github.com/findit-ai/audio-fixtures))
so audio-only consumers don't pull video bytes they'll never
use, and vice versa.

All clips are checked in directly (no Git LFS); the repo
size budget assumes the set grows slowly. The MXF airport
clip was re-encoded to 14 Mbps mpeg2video (down from 22 Mbps
in the source) so it fits under GitHub's 100 MiB single-file
limit while preserving the codec representation.

## Layout

Fixtures are grouped by **video codec** — one directory per
codec — so a consumer that only cares about, say, h264
demux/decode can pull just `h264/` without faulting in the
rest. The codec name follows FFmpeg's `AVCodecID`-string
convention (`h264`, `mpeg2video`, `msmpeg4v3`, `vp9`, …).
`manifest.json` at the repo root lists every codec
directory with per-file metadata so tests can pick fixtures
programmatically.

```
h264/             16 clips, 8 containers (mp4, mov, avi,
                  mkv, flv, m4v, ts, mts) × 2 source clips
mpeg2video/        2 clips (mxf × 2)
msmpeg4v3/         2 clips (wmv × 2)
vp9/               2 clips (webm × 2)
manifest.json      machine-readable index, grouped by codec
```

Each clip is offered in two source variants:

- **`*_airport.*`** — ~40 s of airport walking footage,
  720×1270 portrait. Use as a representative "real-world"
  short clip.
- **`*_pandas_muted.*`** — ~10 s of panda footage,
  1280×720 landscape, audio track muted. Good for tighter
  smoke tests and codec-only paths that don't care about
  audio.

## `h264/`

Most-broadly-supported codec; this directory is the one to
hit when you want demux coverage across containers without
spinning the codec dial. All clips are H.264 High / Main
profile, 30 fps.

| File | Container | Source | Duration |
| --- | --- | --- | ---: |
| `01_airport.mp4` | MP4 | airport | 40.5 s |
| `01_pandas_muted.mp4` | MP4 | pandas (muted) | 10.0 s |
| `02_airport.mov` | QuickTime | airport | 40.5 s |
| `02_pandas_muted.mov` | QuickTime | pandas (muted) | 10.0 s |
| `04_airport.avi` | AVI | airport | 40.6 s |
| `04_pandas_muted.avi` | AVI | pandas (muted) | 10.1 s |
| `05_airport.mkv` | Matroska | airport | 40.5 s |
| `05_pandas_muted.mkv` | Matroska | pandas (muted) | 10.0 s |
| `07_airport.flv` | Flash ⚠ | airport | 40.6 s |
| `07_pandas_muted.flv` | Flash ⚠ | pandas (muted) | 10.1 s |
| `09_airport.m4v` | MPEG-4 | airport | 40.5 s |
| `09_pandas_muted.m4v` | MPEG-4 | pandas (muted) | 10.0 s |
| `10_airport.ts` | MPEG-TS | airport | 40.5 s |
| `10_pandas_muted.ts` | MPEG-TS | pandas (muted) | 10.0 s |
| `11_airport.mts` | AVCHD | airport | 40.5 s |
| `11_pandas_muted.mts` | AVCHD | pandas (muted) | 10.0 s |

Format invariants: H.264 (High / Main), 30 fps, no B-frame
reordering beyond what the encoders default to. `.ts` and
`.mts` files use Annex B; the rest use AVCC (length-prefixed
NALUs).

## `mpeg2video/`

| File | Container | Source | Duration |
| --- | --- | --- | ---: |
| `03_airport.mxf` | MXF | airport | 40.5 s |
| `03_pandas_muted.mxf` | MXF | pandas (muted) | 10.0 s |

Format invariants: MPEG-2 Main profile, 30 fps. The MXF
container packs PCM audio alongside; consumers that want
audio-free decode should mux to MOV or feed straight to a
video-stream-only demux path.

## `msmpeg4v3/`

| File | Container | Source | Duration |
| --- | --- | --- | ---: |
| `06_airport.wmv` | ASF | airport | 40.6 s |
| `06_pandas_muted.wmv` | ASF | pandas (muted) | 10.1 s |

Format invariants: Microsoft MPEG-4 v3 (msmpeg4v3, FOURCC
`MP43`), 30 fps. Legacy codec; included for compatibility
testing on the WMV decode path.

## `vp9/`

| File | Container | Source | Duration |
| --- | --- | --- | ---: |
| `08_airport.webm` | WebM | airport | 40.5 s |
| `08_pandas_muted.webm` | WebM | pandas (muted) | 10.0 s |

Format invariants: VP9 Profile 0, 30 fps. Use when
exercising WebCodecs `VideoDecoder` against `vp09.*` codec
strings or libvpx behind FFmpeg.

## Picking a fixture

| If your test wants… | Pick |
| --- | --- |
| A fast smoke test, broadly compatible | `h264/01_pandas_muted.mp4` (10 s) |
| Demux coverage across containers | the full `h264/` set |
| A long-running stress run | any `*_airport.*` (40 s) |
| A non-H.264 codec | `vp9/08_*` (modern), `mpeg2video/03_*` (legacy IBP), `msmpeg4v3/06_*` (legacy WMV) |
| Audio-track-free | any `*_pandas_muted.*` |

## Consuming from another repo

Recommended pattern is a **git submodule** pinned at a
specific SHA — gives bisectable, reproducible test runs and
works the same on every CI runner without extra tooling.

```sh
# one-time, in the consumer repo
git submodule add https://github.com/findit-ai/video-fixtures \
  tests/fixtures/video
git commit -m "test: pin video-fixtures submodule"
```

CI checkouts then need `submodules: recursive` (GitHub
Actions) or `git submodule update --init --depth=1` (other
runners). Tests look up files by codec directory:

```rust
let path = std::path::Path::new(env!("CARGO_MANIFEST_DIR"))
  .join("tests/fixtures/video/h264/01_pandas_muted.mp4");
```

A flat `git clone` works too if you don't need exact-revision
pinning — point a `VIDEO_FIXTURES_DIR` env var at the local
path and have tests fall back to that.

## Adding fixtures

1. If the file is a new codec, create a `<codec>/`
   directory at the repo root using FFmpeg's `AVCodecID`
   string (`h264`, `hevc`, `av1`, …). Otherwise drop it
   into the existing matching directory.
2. Use a sortable, human-readable name
   (`NN_short_topic.<ext>`).
3. Add a row to that codec's table in this README and an
   entry in `manifest.json` (under the matching
   `codecs[].files[]` array). Run
   `ffprobe -v error -show_entries format=duration -show_entries stream=codec_name,width,height,r_frame_rate -of default=nw=1 <file>`
   for the manifest values.
4. PR. CI is intentionally minimal — there's no Rust crate
   here, just binaries — but please attribute the source.

## License

Fixtures are licensed Apache-2.0 / MIT (your choice) at the
repo level. Individual recordings retain their upstream
license.

> ⚠ `.flv` (07) — Flash support ended in 2020; preserved
> here for legacy-system compatibility testing only.
