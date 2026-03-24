# Changelog

All notable changes to the Submission Video Player are documented here.  
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

---

## [1.0.0.4] — 2026-03-24

### Fixed
- `mvhd` v1 timescale was reading `modification_time` field at `pos+20` instead of `timescale` at `pos+28` — broke FPS Path 2 fallback for all v1 mvhd containers
- `walkBoxes` `pos++` byte-crawl replaced with `break` — oversized box no longer burns 20,000 iteration budget before reaching metadata atoms
- Trak handler identification now explicitly excludes non-AV handler types (`hint`, `tmcd`, `text`, `subp`, `sbtl`, `clcp`) — wrong track was being flagged as video in some containers
- `_lastMdtaKey` state leak — key now cleared on any non-`data` box encountered, proximity guard (512 bytes) added to prevent cross-atom metadata corruption
- `hasMoov` false positive — size validation now checks `sz <= fileSize` to filter out coincidental `moov` byte sequences in raw video data
- `findMoovOffset` now starts scan at `i=0` — was missing moov at the very start of returned buffers
- `readFullMoov` now returns `probeBuf.slice(moovPos, moovPos + moovSize)` — buffer passed to parser always starts exactly at moov, eliminating silent truncation from pre-moov data
- `fsVolBtn` (fullscreen mute) now saves `volBeforeMute` and syncs `volRange` slider — previously caused UI/state desync in fullscreen mode
- Pulse rings now cleared before creating new one on rapid `togglePlay` calls — prevented minor DOM node accumulation

### Changed
- `stts` entry cap raised from `2000` → `10000` — improves FPS accuracy on high-variance VFR screen recordings
- Trak fallback (no `hdlr` found) now verifies `stbl` exists inside the trak before assuming video — reduces false identification of hint/metadata traks

---

## [1.0.0.3] — 2026-03-24

### Added
- Keyboard shortcut: `←` / `→` seek ±10 seconds
- Keyboard shortcut: `Shift + ←` / `Shift + →` seek ±30 seconds
- Keyboard shortcut: `↑` / `↓` volume ±10%
- Keyboard shortcut: `M` mute/unmute with toast confirmation
- Memory leak fix — `URL.revokeObjectURL()` now called on previous blob URL before creating new one

### Changed
- Initial volume set to 20% unmuted on every new file load — audible enough to detect speed anomalies, safe for open-plan environments
- Speed resets to 1x on new file load — playback rate and UI label now always in sync
- Speed dropdown simplified — removed 1.25x and 1.5x, retained: 1x, 2x, 3x, 4x, 5x

### Fixed
- Speed dropdown label showing stale speed (e.g. 5x) after loading a new video while UI showed previous session's speed

---

## [1.0.0.2] — 2026-03-24

### Added
- Samsung vendor `udta` blob parser — **Vendor Blobs v0.1**
  - `auth` atom: binary blob scan for `0xA9` (©) byte, reads model string after it
  - `smta` atom: secondary fallback scanning for `mdln` substring
  - Confirmed: SM-S9060, SM-G991B (S21 5G), SM-A166P (A16 5G)
  - Versioned section in code (`VENDOR UDTA BLOBS v0.1`) — structured for future vendor additions
- MediaInfo.js two-phase metadata architecture
  - Phase 1: native parser runs instantly — FPS and dates populated immediately
  - Phase 2: MediaInfo.js loads lazily from CDN — vendor metadata populated async
  - MediaInfo fed only the moov slice — typically < 500KB regardless of file size
- CDN failure handling — 4-second timeout, `_miFailed` flag prevents all subsequent attempts, queue flushed with `null` to unblock waiting callers
- `populateFromMediaInfo()` maps `Writing_hardware` and `Writing_operating_system` fields
- FPS override from MediaInfo when native parser returns nothing

### Fixed
- Second video stuck on `reading…` — CDN failure state now resets cleanly, no hanging promises
- MediaInfo CDN switched from `jsdelivr` to `unpkg` (subsequently handled via timeout fallback)

---

## [1.0.0.1] — 2026-03-23

### Added
- `readBoxName()` using `latin1` encoding for all 4-byte atom name reads — critical fix for `0xA9` (©) byte preservation. UTF-8 was silently corrupting `©wrm`, `©wrt`, `©nam`, `©ART` atom names causing all vendor metadata to return `undefined`
- `keys` box key strings now also read with `latin1` — same `0xA9` corruption was affecting Samsung key string matching
- `meta` box child offset auto-detection — QuickTime `meta` (children at `pos+8`) vs MPEG-4 `meta` (children at `pos+12`) now correctly distinguished. Previously always used `pos+12` which skipped 4 bytes into the first child of QuickTime meta boxes, breaking all Samsung and iPhone metadata
- `findMoovOffset()` brute-force byte scan replacing `findBox()` for moov location — `findBox()` relied on box structure which fails in tail probe buffers starting mid-`mdat`
- `hasMoov()` brute-force byte scan for same reason
- Keyboard shortcuts — spot-check positions
  - `1` → 0% (start)
  - `2` → 25%
  - `3` → 50%
  - `4` → 75%
  - `5` → 100% (end)
- Keyboard shortcuts — frame stepping
  - `,` → one frame back
  - `.` → one frame forward (auto-pauses, frame duration calculated from resolved FPS)
- `Space` → play/pause keyboard shortcut
- Toast notification for all keyboard shortcuts

### Fixed
- Samsung device model and OS not populating — root cause was `meta` box offset error and `latin1`/UTF-8 encoding mismatch
- iPhone device model and OS not populating — same `meta` box offset error
- `locateMoov` returning `null` for all files — `findBox()` used sequential box walk which immediately stopped on garbage sizes in mid-`mdat` tail probe buffers
- `©wrm` (Samsung Writing hardware) and `©wrt` (Writing OS) never matched — both `0xA9` encoding bug and wrong metadata location (keys/ilst vs udta)
- `©nam` and `©ART` not populating for Lavf/YouTube re-encoded files — plain iTunes `ilst` without `keys` box now handled
- Trak handler detection fallback — reads full handler string for non-standard encoders, falls back to first trak with `stbl` when `hdlr` not found

### Changed
- `mdta` legacy key reading switched to `latin1`
- `udta` handler now maps `©wrm` → `_device`, `©wrt` → `_software`, `©ART` → author
- `keys`/`ilst` matcher now explicitly handles bare `©wrm`/`©wrt` key strings (Samsung `keys` structure)
- Apple QuickTime keys matched exactly before pattern matching (`com.apple.quicktime.model`, `.software`) to prevent false positives

---

## [1.0.0.0] — 2026-03-22

### Added
- Full video player UI — dark theme, Outfit Regular font, 1.5rem rounded corners
- `<video>` element with `createObjectURL` file loading — no upload, no server
- File picker (`Open File` button) with filename badge
- Play/pause with centre button, auto-hide during playback, reappears on hover
- Click anywhere on video area to toggle play/pause
- Seek bar — click or drag to scrub with live timestamp
- Volume slider — expands on hover, click icon to mute/unmute with memory
- Playback speed dropdown — 1x, 2x, 3x, 4x, 5x
- Resolution/quality dropdown — 480p, 540p, 720p, 1080p, Auto (cosmetic in POC)
- Fullscreen — native browser fullscreen with overlay controls
- Fullscreen overlay — seek bar, play/pause, mute, timestamp, speed label, exit button, resolution badge
- Resolution + Aspect Ratio badges — hidden until video loads, auto-detected from `videoEl.videoWidth/Height`
- Submission Info panel (`i` button) — Task ID, Date of Upload, Date Created, Frame Rate, Writing Device, Device OS, Title, Author
- Native MP4/MOV box parser written from scratch
  - `locateMoov` — multi-step probe strategy (64KB head → 512KB head → 512KB tail → 8MB scan)
  - `walkBoxes` — recursive container walker with container list: `moov`, `trak`, `mdia`, `minf`, `stbl`, `udta`, `ilst`, `meta`
  - `findBoxInRange` — deep recursive box search
  - `mvhd` — creation time (Mac 1904 epoch + Unix epoch conversion), movie timescale
  - `mdhd` — media timescale for video track (v0 and v1)
  - `tkhd` — track duration (v0 and v1)
  - `stts` — sample-to-time table, all entries collected for weighted average
  - `stsz` — total sample count
  - `keys` + `ilst` — Android/Apple QuickTime metadata pairing
  - `udta` — QuickTime freeform text atoms
  - `stsd` → `avcC` H.264 SPS NAL VUI timing_info parsing (custom `BitReader` + Exp-Golomb)
  - `stsd` → `hvcC` H.265 `avgFrameRate` field + SPS NAL VUI timing parsing
  - iTunes-style `©nam`, `©ART`, `©day`, `©swr`, `©mod`, `©mak` atoms
- FPS resolution — 4-path chain with 1–300 FPS sanity gate
  - Path 1: `stts` weighted average ÷ `mdhd` timescale (CFR + VFR)
  - Path 2: `stsz` sample count ÷ `tkhd` duration
  - Path 3: H.264 `avcC` SPS VUI `timing_info`
  - Path 4: H.265 `hvcC` `avgFrameRate` + SPS VUI
- Device metadata coverage
  - iPhone — `com.apple.quicktime.model`, `.software`, `.make`, `.creationdate` via `keys`/`ilst`
  - Samsung — `©wrt` OS via `meta`/`keys`/`ilst`
  - Xiaomi — `©wrt` OS only (no device model written to file)
  - OBS/screen recordings — `©swr` writing application
  - Lavf/FFmpeg re-encoded — creation date from `mvhd` only (device info stripped)
- Idle cursor hide — cursor hidden after 3s inactivity during playback
- Centre play button auto-hide during playback with hover reveal

---

## Architecture Notes

### Current ceiling (local `file://` host)
The player has reached maximum capability without a web server. All parsing is done via the File API + ArrayBuffer with no external dependencies. CDN-hosted libraries (MediaInfo.js, ffmpeg.wasm) are blocked by browser security policy on `file://` protocol.

### What unlocks on a web server
- MediaInfo.js WASM — full vendor atom coverage (Tecno, Vivo, Realme, GoPro, DJI, all others)
- Self-hosted WASM — eliminates CDN dependency, persistent cache
- Server-side `ffprobe` at upload time — gold standard metadata, zero client-side parsing needed
- HTTP Range requests — fetch only moov box from remote video URL, no full file download

### Vendor Blob Versioning
Native parser udta blob mappings are versioned independently under `VENDOR UDTA BLOBS` in source.

| Version | Vendors Covered |
|---------|----------------|
| v0.1 | Samsung (`auth`, `smta` atoms) — SM-S9060, SM-G991B, SM-A166P |

### Production Handoff (ENG)
- Replace file picker with video URL from submission API
- Implement `ffprobe -v quiet -print_format json -show_streams -show_format` at upload time
- Store metadata JSON per submission
- Player fetches `/api/submissions/{id}/metadata` → populate panel
- Wire resolution dropdown to pre-transcoded rendition URLs
- HTTP Range request for moov box (avoids full file download for metadata)
