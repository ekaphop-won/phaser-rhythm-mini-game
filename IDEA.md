# 🐰 Bunny Tea Parade — Rhythm Mini Game (Phaser 3)

## เกี่ยวกับ
เกมจับจังหวะ (Rhythm Game) 4 เลน สร้างด้วย Phaser 3.80.1
เล่นบนมือถือได้ ใช้เพลง "Bunny Tea Parade" โดย crunchymocha (Suno AI)

## ไฟล์
- `index.html` — ไฟล์เดียวจบ เกมทั้งหมด
- `audio/bunny-tea-parade-adjusted.mp3` — เพลงประกอบ

## วิธีเล่น (มือถือ)
1. `python3 -m http.server 8080` ใน project root
2. เปิด http://[IP]:8080/ ในมือถือ (network เดียวกัน)
3. รอเพลงโหลด → แตะเริ่ม → แตะเลนตรงโน๊ตที่ถึงเส้น

## ระบบ
- 4 เลน (แดง-ส้ม-เขียว-น้ำเงิน)
- Web Audio API sync กับเพลงจริง
- Perfect/Great/Good scoring
- Combo system
- 104 โน๊ต (91 onset beats + 13 chord doubles)
- Pattern แบ่งเป็น 5 sections: intro → verse → pre-hook → **hook** → outro
- Hook: dense chords (2 โน๊ตพร้อมกัน) + rapid alternation
- VFX: Phaser 3.80 particle emitter (star/spark ระเบิดเวลากด) + lane flash
- Rating SS/S/A/B/C/D

## Dev Notes
- Phaser 3.80.1 (CDN, ไม่ต้อง install)
- Web Audio API สำหรับทั้งเพลง + ซาวด์เอฟเฟกต์
- SFX: noise burst click (เสียงเดียว ไม่กวนเพลง)
- Beatmap จาก audio onset detection + beat tracking → snap to actual audio peaks
- ไม่ใช้ fixed BPM grid — timing ตามจังหวะจริงของเพลง (variable interval)
- Logic ใช้ได้กับเพลงอื่น (general-purpose onset detection)
- Mobile-first: Phaser.Scale.FIT + touch-action:none
