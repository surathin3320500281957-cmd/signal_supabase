# Signal Matrix — คู่มือระบบฉบับเต็ม

> เอกสารนี้สรุปทั้งระบบ (frontend + backend) ให้ chat ใหม่เริ่มงานได้ทันทีโดยไม่ต้องอธิบายซ้ำ
> วางไฟล์นี้ไว้ที่ root ของ repo `signal_supabase` เป็น `README.md` หรือ upload เข้า Project knowledge ของ Claude ก็ได้

## หลักการออกแบบ (อ่านก่อนอย่างอื่น)

ระบบช่วยตัดสินใจซื้อขายหุ้น ~34-38 ตัว ใช้ ML ช่วยสอบทาน **คนตัดสินใจสุดท้ายเสมอ ไม่มี auto-trade**

**3 แกนที่ตั้งใจแยกอิสระจากกัน:** ML แยกกัน (Momentum vs Value) · ข้อมูลจริง (seed vs live) · การตัดสินใจของคน — แต่ละส่วนทำงานได้เองแม้ส่วนอื่นยังไม่พร้อม ไม่มี single point of failure

**กฎที่ยึดตลอดการพัฒนา:**
1. แต่ละส่วนทำหน้าที่เดียว (single responsibility) — แก้จุดหนึ่งไม่กระทบจุดอื่น
2. Fail gracefully — ข้อมูลไม่พอ → fallback หรือขึ้น "ยังไม่พอ" ไม่ error ไม่ block ส่วนอื่น
3. **อย่าเชื่อ correlation/weight ที่อ่อน (r<0.15) ว่าเป็นความจริงตายตัว** — ระบบยังสะสมข้อมูลอยู่ ต้อง cross-check เสมอ
4. ทุก error message ต้องเช็คของจริงจาก DB ก่อนฟันธง ไม่เดาจาก error type (เช่น "Load failed" อาจเป็นแค่เน็ตสะดุด ไม่ใช่ save fail จริง)

---

## สถาปัตยกรรม 2 แอป

| แอป | Deploy ที่ | ไฟล์ | หน้าที่หลัก |
|---|---|---|---|
| **Backend** | GitHub Pages, repo `signal_supabase` | `index.html` | Train ML จากประวัติ (seed), จัดการ stock_history/summary, ML Analyzer |
| **Frontend** | Cloudflare Workers (`frontsupabase...workers.dev`) | `index_front.html` | Dashboard, ML Pick, Ranking, Portfolio, Train Live |

ทั้งคู่ต่อ Supabase project เดียวกัน (`SIGNAL MATRIX`) — ตาราง `ml_config` ใช้ร่วมกัน

---

## ML Architecture — คู่ขนาน 2 แกน

```
Yahoo (ราคาจริง real-time)
    ├── ML Momentum: backend(seed) → Live(ใช้จริง) → ตารางบน "ML Pick"
    └── ML Value:     backend(seed) → Value-Live(ใช้จริง) → ตารางล่าง "ของดีราคาถูก"
```

- **seed** (backend/value): train จากประวัติหุ้นย้อนหลัง ปุ่ม "Train ML" / "Train Value" (backend app)
- **live** (live/value-live): train จาก snapshot สดหน้าบ้าน ปุ่ม "Train Live" / "Train Value Live" (frontend app)
- โหลด config เสมอ: **ลอง live ก่อน → fallback seed** ถ้ายังไม่เคย train live
- Features Momentum: RSI, Heat, pct50, EMAMom, Score, Slope, GroupRS
- Features Value: Drawdown, RSILow, DropHigh, SlopeRec, GroupRS
- Weight = `|correlation| / sum(|correlation|)`
- ณ ล่าสุด (2026-08-06): **Value model correlation แข็งขึ้นชัดเจน** (DropHigh r=+0.341, 1315 samples) ในขณะที่ Momentum ยังอ่อน (r~0.1) — Value model เริ่มน่าเชื่อถือกว่า แต่ต้องติดตามต่อว่าคงที่ไหม

---

## Frontend — รายละเอียดแต่ละ Tab

### 📊 Dashboard
- ดึงราคาสด 34-38 ตัวจาก Yahoo (ปุ่ม Refresh)
- **Market Regime panel**: BULL/BEAR/NEUTRAL รายวัน (`market_regime` table), NEUTRAL นับต่อ streak เดิม
- Median วัน/รอบ จาก episode ย้อนหลัง 90 วัน + **ดัชนีราคา** (median % เปลี่ยนแปลงราคาหุ้นทั้งหมด/วัน จาก `stock_signals` ทบต้นสะสม เริ่มที่ 100)
- % ของ episode จะเห็นได้ต่อเมื่อตลาดพลิก regime อย่างน้อย 1 ครั้ง (มี baseline เทียบ) — แนะนำรอข้อมูล **~180 วัน** ก่อนเชื่อถือ median

### 🎯 ML Pick (ตารางบน)
- Filter: Confidence A/B + Trend UP + Heat ไม่ EXTREME → จัดอันดับด้วย Live weights (momTier: accel=1:3, stable=1:2, slow=1:1.5 — วัดจากอัตราเร่ง/แผ่วของ pct50/EMAMom เทียบ history หลายรอบ ละเอียดกว่า Ranking tab จึงตั้งใจไม่แก้ให้เหมือนกัน)
- ปุ่ม BUY/SELL = ทางลัดตามสถานะถือครอง (ถือแล้ว→SELL, ยังไม่ถือ→BUY) **ไม่ใช่สัญญาณ ML**

### 💎 ของดีราคาถูก (ตารางล่าง)
- Filter: **"ตกจากตารางบน"** (symbol ที่ไม่ผ่าน ML Pick) + ต้องมี Bounce Score (pct50<0 เท่านั้น มาจาก `calcBounceScore`)
- จัดอันดับด้วย Value-Live weights (fallback Value seed)

### 📈 Ranking วันนี้
- ไม่กรองเงื่อนไข เห็นหุ้นทั้งหมดพร้อมกัน (เสริม ML Pick ที่กรองออกไปเยอะ)
- คลิกหัวคอลัมน์ sort ได้ทุกคอลัมน์
- **R:R column**: สูตรต่อเนื่อง `clamp(3 - (RSI-50)×0.1, 0.3, 6)` — RSI ต่ำ=room เยอะ=R:R สูง, RSI สูง=ใกล้หมดแรง=R:R ต่ำ (**คนละสูตรกับ ML Pick โดยตั้งใจ** — ML Pick ใช้ momTier ละเอียดกว่า, Ranking/modal ใช้ RSI เพราะไม่มี history)
- **🔥 จุดเปลี่ยน column**: auto-flag เมื่อ (เทรนด์ UP มาไม่เกิน 3 รอบ) + (RSI<60) + (Confidence ขยับขึ้นหรือไม่ใช่ C) — ต้อง Refresh ≥2 รอบถึงเริ่มเทียบได้ เพราะเป็นเหตุการณ์เกิดไม่บ่อย (EMA cross จริงไม่กี่ครั้ง/สัปดาห์/หุ้น)
- ระวัง: เพดาน R:R เคยตั้งที่ 3 แล้วชนก้อนตอน RSI<50 ทั้งหมด → ยกเป็น 6 แล้ว

### 💼 Portfolio
- เห็นทุกตัวที่ถืออยู่ ไม่ว่าจะผ่านเงื่อนไข ML Pick หรือไม่ (ต่างจาก ML Pick ที่กรองทิ้ง)
- SL = `avg_cost × (1 - slPct)` โดย slPct: LOW=3%, MEDIUM/HIGH=5% — **fixed % ตายตัว ไม่ผูกกับ regime** (รอข้อมูลสะสมพอค่อยพัฒนาให้ dynamic)
- แถวที่ราคาต่ำกว่า SL ไฮไลต์แดง — **แค่แสดงสถานะ ไม่มี auto-sell**
- อัพเดตอัตโนมัติ: ตอนเปิดแอป, ทุก Refresh, ทันทีหลัง BUY/SELL

### 🔍 กล่องวิเคราะห์ (modal, กดแว่นขยาย)
- ค้นข้อมูลจาก `lastPicks` → `lastValuePicks` → **`allResults` (fallback สุดท้าย ครอบคลุมทุกตัว)**
- R:R ใช้สูตรเดียวกับ Ranking tab (RSI-based, เพดาน 0.3-6)

---

## Backend — รายละเอียดแต่ละ Tab

### Signal Matrix / Portfolio / สรุป Stock ล่าสุด
- แสดง 7D/14D/30D performance ranking ของทุกตัว (bar chart % เปลี่ยนแปลงราคา)

### 🧠 ML Analyzer
- ปุ่ม **Train ML** (Momentum seed) / **Train Value** (Value seed) / **Save ML Config → Supabase**
- **Step 2-4 การ์ด Momentum**: Correlation table, Weight bars (ปรับ manual ได้), Thresholds, Adaptive accuracy per action
- **💎 Value Model card (เพิ่มใหม่ 2026-08-06)**: Correlation table + Weight bars แยกต่างหาก สไตล์เดียวกับ Momentum อยู่ใต้กราฟ Momentum ในหน้าเดียวกัน — ก่อนหน้านี้ Value weights โชว์เป็น text บรรทัดเดียวอ่านยาก ตอนนี้เป็นการ์ดกราฟอ่านง่ายแล้ว

### 📋 Summary History
- Score Chart by Action (composite score เทียบ zone ML ตอน trade จริง)
- Validate ML ด้วยตาเอง เทียบกับผลจริงที่เกิดขึ้น

---

## Save Pattern (ml_config) — สำคัญมาก อย่าย้อนกลับไปทำแบบเก่า

**Insert-only เสมอ ห้าม PATCH ปิด active เก่าเอง:**
- มี DB trigger `ensure_single_active_ml_config` (BEFORE INSERT/UPDATE) จัดการปิด row เก่าให้อัตโนมัติแบบ atomic ในทรานแซกชันเดียวกับ insert
- เหตุผล: กันเคส "ปิดเก่าสำเร็จ แต่ insert ใหม่ fail" ที่จะทำให้ config_type นั้นไม่มี active row เลยชั่วขณะ

**ตรวจสอบสถานะ save จริงเสมอ (ไม่เดาจาก error type):**
```js
// หลัง POST เสมอ ไม่ว่าจะดูเหมือนสำเร็จหรือ fail
const check = await fetch(`${SUPABASE_URL}/rest/v1/ml_config?select=id&version=eq.${versionKey}&config_type=eq.X`);
const actuallySaved = (await check.json()).length > 0;
// รายงานผลตาม actuallySaved จริง ไม่ใช่ตาม response ของ POST
```
เหตุผล: "Load failed" ที่ browser โยนมา อาจเป็นแค่เน็ตสะดุดตอนรอ response ทั้งที่ insert เข้า DB สำเร็จแล้วจริง — ต้อง verify ไม่ใช่เชื่อ error message ตรงๆ

---

## ข้อจำกัดที่รู้อยู่ (อย่าแก้จนกว่าจะมีข้อมูลพอ)

- **Correlation ทุกโมเดลยังอ่อน** (ส่วนใหญ่ r<0.15) ยกเว้น Value model ล่าสุดที่เริ่มแข็ง (r=0.34) — ต้องติดตามว่าคงที่ข้ามเวลาไหม
- **Market regime median** มาจากแค่ 2 episode (ข้อมูล ~1 เดือน) — รอ ~180 วัน หรือผ่านรอบพลิกหลายๆ ครั้งก่อนเชื่อถือได้
- **Portfolio SL เป็น fixed % ตายตัว** ไม่รู้จักบริบทตลาด (หมี/กระทิง/ระยะเวลาถือ) — ตั้งใจไม่ทำให้ฉลาดกว่านี้จนกว่าจะมีข้อมูล regime timing พอ
- **🔥 จุดเปลี่ยน column** หายาก โดยตั้งใจ (เกณฑ์เข้ม กันสัญญาณหลอก) — ตัดสินใจไว้แล้วว่าไม่ผ่อนเพิ่ม

## สิ่งที่รอทำ (หลังมีข้อมูลพอ — อย่าเริ่มก่อนกำหนด)

- Win-rate tracking เทียบ ML score กับผลจริงย้อนหลัง (ต้องรอ ML/ข้อมูล/การตัดสินใจแยกอิสระ+นิ่งก่อน)
- ผูก Portfolio SL เข้ากับ market regime timing แบบ dynamic
- สัญญาณ "จังหวะเข้า-ออก" ระดับตลาดรวม (ต้นกระทิง/ปลายหมี) — รอข้อมูล ~180 วัน

---

## Troubleshooting ที่เจอบ่อย

- **Save ขึ้น error แต่จริงๆ เข้าไปแล้ว** → ปกติแล้ว (verify-after-save pattern จัดการให้แล้ว) มักเกิดจากเน็ตสะดุดบนมือถือ/แท็บเล็ต ลองบน PC ถ้าซ้ำบ่อย
- **แก้โค้ดแล้วไม่เห็นผล** → เบราว์เซอร์แคช ให้เปิด **Private/Incognito tab ใหม่** เข้า URL เดิม (เร็วสุด บังคับโหลดไฟล์ใหม่)
- **คอลัมน์ตารางหายไปเงียบๆ** → ตารางกว้างเกินจอ ไม่มี scroll (เพิ่ม `overflow-x:auto` + `min-width` ให้ wrapper แล้วในตารางใหม่ๆ)
- **GitHub Pages deploy ดูค้าง** → เช็ค job `deploy` (ไม่ใช่ `report-build-status`) ว่าเขียวหรือยัง ถ้าเขียวแล้วคือใช้งานได้แล้วจริง ไม่ต้องรอ job อื่น
