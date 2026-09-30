# 10 — Media Engine

## Overview

Deterministic video processing. Consumes an `EditPlan` and produces a finished vertical MP4. No AI, no creative decisions — pure execution. The hard boundary between intelligence and output.

---

## Requirements

- `EditPlan` is the only input — media engine makes no creative decisions
- Supported operations in MVP: trim, concatenate, crop/reframe (to 9:16), speed adjustment (0.5x–2.0x), transitions (cut and crossfade only), music overlay, basic beat alignment, export
- No timeline editor exposed to user
- No generative AI, no frame synthesis
- Export time: < 15 seconds for a 5–15 second Story on iPhone
- Output: H.264 vertical MP4, 1080×1920, suitable for direct posting to Instagram/TikTok
- Music library: 5–10 curated licensed tracks, one mapped to each style template
- If a segment references a missing asset: fail the export with a clear error — do not silently skip

---

## Design

**Implementation:** AVFoundation `AVMutableComposition` on-device. All processing on iPhone — no server-side video in MVP.

**Processing pipeline:**
```
EditPlan
        ↓
For each segment:
  Load asset (AVURLAsset)
  Apply trim (CMTimeRange)
  Apply speed (scaleTimeRange)
  Apply crop/reframe (AVVideoComposition transform)
        ↓
Concatenate (AVMutableComposition)
        ↓
Apply transitions (AVVideoCompositionInstruction crossfade = 0.3s dissolve)
        ↓
Mix music (AVMutableAudioMix)
        ↓
Beat sync (snap cuts to nearest beat marker within ±0.15s)
        ↓
Export (AVAssetExportSession → MP4, 1080×1920, H.264)
```

**Crop to 9:16:** Center crop for landscape source. Use explicit `EditPlan.segment.crop` offsets when provided.

**Transitions:** `cut` = frame-accurate. `crossfade` = 0.3-second dissolve.

**Music tracks:** Stored in app bundle. Each track has AAC audio file + beat marker JSON (array of timestamps). Default track selected by matching `CreativeIntent.style`.

---

## Tasks

**Phase 1:**
- [ ] Implement basic AVMutableComposition (trim + concatenate)
- [ ] Implement 9:16 center crop
- [ ] Implement static music overlay (no beat sync yet)
- [ ] Implement AVAssetExportSession → MP4 1080×1920
- [ ] Measure export time on iPhone (target < 15s)
- [ ] Run end-to-end: hard-coded EditPlan → finished MP4 on device

**Phase 9 (full implementation):**
- [ ] Implement speed adjustment
- [ ] Implement crossfade transition
- [ ] Implement explicit crop from EditPlan segment
- [ ] Implement beat marker loading and cut snapping
- [ ] Curate and license 5–10 music tracks; produce beat marker JSON for each
- [ ] Map tracks to style templates
- [ ] Add export error handling with clear user-facing messages
- [ ] Test with: landscape clips, portrait clips, mixed orientations, slow/fast clips
