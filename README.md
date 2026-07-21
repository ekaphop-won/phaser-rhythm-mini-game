# 🐰 Bunny Tea Parade — Rhythm Mini Game

เกมจับจังหวะ (Rhythm Game) 4 เลน สร้างด้วย Phaser 3.80.1
เล่นบนมือถือได้ ใช้เพลง "Bunny Tea Parade" โดย crunchymocha (Suno AI)

## ⚡ Quick start

```bash
# เปิด static server ใน project root
python3 -m http.server 8080

# เปิด browser
#   เกม:   http://localhost:8080/
#   editor: http://localhost:8080/beatmap-editor.html
```

> **ทำไมต้อง static server?** เกมโหลด beatmap จาก `audio/bunny-tea-parade-adjusted.json` ผ่าน `fetch()` ซึ่ง CORS-block ใน `file://` ถ้าเปิด `file://` โดยตรงเกมจะใช้ embedded fallback 149 notes แทน

## 🎮 Controls

| Input | Action |
|---|---|
| แตะเลน | กดโน้ต |
| **D / F / J / K** | กดโน้ตเลน 1-4 (desktop keyboard) |
| **Space / Esc / P** | pause / resume |
| **Tab hidden / window blur** | auto-pause |
| **⛉ (มุมขวาบน)** | settings (offset, SFX volume) |

## 🛠 Beatmap workflow

1. เปิด `beatmap-editor.html` ผ่าน static server
2. กด **↻ โหลดจาก index.html** → โหลด beatmap + waveform เข้า editor
3. คลิกเพลง Lane 1-4 เพื่อเพิ่ม note / ลากจุดเพื่อปรับเวลา-เลน
4. กด **Export JSON** เพื่อบันทึก beatmap ใหม่
5. วาง `beatmap-adjusted.json` ทับ `audio/bunny-tea-parade-adjusted.json` (หรือแก้ `AUDIO_PATH` ใน `index.html` ให้ชี้ไฟล์ใหม่)

> snap default 10ms · ปรับได้ใน checkbox "snap 10ms"

## 📐 Sections (canonical)

| Section | Time gate (ms) | Banner |
|---|---|---|
| **Intro** | 0 → 10645 | 🎬 Intro |
| **Verse** | 10645 → 23989 | 🎵 Verse |
| **Build Up** | 23989 → 27317 | ⚡ Build Up |
| **Hook** | 27317 → 43946 | 🔥 Hook (flash + hit zone) |
| **Outro** | 43946 → 60000 | 🌅 Outro |

> Section gates กำหนดใน `G.SECTIONS` ใน `index.html` (single source of truth) และ sync ใน `beatmap-editor.html` แล้ว

## 📁 ไฟล์

| ไฟล์ | บทบาท |
|---|---|
| `index.html` | เกมทั้งหมดในไฟล์เดียว (Phaser scene + AudioManager + beatmap embedded) |
| `beatmap-editor.html` | standalone editor: waveform + drag/drop notes + export |
| `audio/bunny-tea-parade-adjusted.mp3` | เพลงประกอบ (60s) |
| `audio/bunny-tea-parade-adjusted.json` | external beatmap (149 notes, 149 markers) |
| `IDEA.md` | design notes (legacy — superseded by this README) |

## 🎯 ไฟล์ไหนทำอะไร

### `index.html` (single-file game)

- **Phaser config** (line ~1060): resolution = `devicePixelRatio`, SRI-pinned Phaser 3.80.1
- **`G` config** (~line 80): window 400×720, 4 lanes, hit zone 620, timing windows (Perfect 80ms / Great 150ms / Good 250ms)
- **`buildBeatmap()`** (~line 110): rebuilds `BEATMAP_NOTES` from `BN_L/BN_T` — called once at init **and** after `loadExternalBeatmap()` succeeds
- **`AudioManager`**: Web Audio API wrapper (decode-on-start, noise burst + sine 2kHz hit SFX, sawtooth miss SFX)
- **`GameScene`**:
  - `create()`: init scene, bind keyboard, register `visibilitychange`/`blur` for auto-pause
  - `update()`: spawn notes, tick positions, detect misses, fire section banners (data-driven from `G.SECTIONS`)
  - `handleTap(pointer)` / `handleLaneInput(lane)`: hit detection
  - `togglePause()`: pause/resume with audio context sync
  - `endGame()`: rating + high-score persistence
  - `openSettings()`: offset (-300..+300ms) + SFX volume sliders
- **`loadExternalBeatmap()`**: fetches `audio/bunny-tea-parade-adjusted.json`, replaces `BN_L/BN_T`, calls `buildBeatmap()`

### `beatmap-editor.html`

- **`parseIndex()`** (line 27): regex รับ `const|let` เพื่อ parse beatmap จาก `index.html`
- **render()** วาด waveform, lane strips, section bands (Intro/Verse/Build Up/Hook/Outro), notes, markers
- **drag/select/nudge** ใช้ pointer events + snap 10ms
- **Export JSON / Export JS** สำหรับ round-trip กลับ `index.html`

## 🐛 Troubleshooting

| อาการ | สาเหตุ | วิธีแก้ |
|---|---|---|
| โน้ตออกผิดจังหวะ | audio latency / device mismatch | เปิด settings (⛉) → ปรับ **Offset** (negative = notes later) |
| เกมเงียบ | SFX volume = 0 | เปิด settings → ปรับ **SFX volume** |
| เพลงไม่เล่น auto | browser block user-gesture | ต้องแตะเริ่มเกมก่อน (iOS Safari policy) |
| `❌ Failed to load audio` | mp3 fetch fail | ตรวจ network + audio file path |
| เปิด `file://` แล้วใช้ beatmap เก่า | CORS block fetch | ใช้ static server (`python3 -m http.server`) |
| Editor โหลดจาก index ไม่ได้ | regex mismatch | ใน `parseIndex` ใช้ `(?:const\|let)` แล้ว |

## 💾 Persistence (localStorage)

| Key | ใช้เก็บ |
|---|---|
| `btp.settings` | `{ offset: number, sfx: number }` |
| `btp.highscore` | best score ตัวเลข |

ลบ key ใน DevTools → Application → Local Storage ถ้าต้องการ reset

## 🧰 Tech stack

- **Phaser 3.80.1** (CDN with SRI)
- **Web Audio API** (decode + scheduling + SFX synthesis)
- **No build step** · **No bundler** · **No framework**
- **No external state** (saves to localStorage only)

## 📝 Known limitations

- ไม่มี difficulty/level system
- ไม่มี reduced-motion accessibility option
- ไม่มี colorblind palette
- ไม่มี CI/lint/tests (single-file static project)
- editor ไม่ save state — reload หาย
- texture leak เมื่อ restart หลายครั้ง (Phaser ไม่ destroy texture เดิม)
