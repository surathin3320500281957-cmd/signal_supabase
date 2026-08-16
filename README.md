# Signal Matrix — คู่มือระบบฉบับเต็ม (อัพเดต)

> เอกสารนี้สรุปทั้งระบบ (frontend + backend) ให้ chat ใหม่เริ่มงานได้ทันทีโดยไม่ต้องอธิบายซ้ำ
> วางไฟล์นี้ไว้ที่ root ของ repo `signal_supabase` เป็น `README.md`

## หลักการออกแบบ (อ่านก่อนอย่างอื่น)

ระบบช่วยตัดสินใจซื้อขายหุ้น ~34-38 ตัว ใช้ ML ช่วยสอบทาน **คนตัดสินใจสุดท้ายเสมอ ไม่มี auto-trade**

**3 แกนที่ตั้งใจแยกอิสระจากกัน:** ML แยกกัน (Momentum vs Value) · ข้อมูลจริง (seed vs live) · การตัดสินใจของคน

**ความสัมพันธ์ที่แท้จริงระหว่าง 2 แอป:** frontend ดูครบเครื่องกว่า (UI/ฟีเจอร์เยอะ) แต่**รากฐานที่ทำให้ตัวเลขมีความหมายยังมาจาก backend** — backend train seed จากประวัติหุ้นย้อนหลัง (ข้อมูลเยอะ นิ่ง) ให้ weights เริ่มต้น ส่วน frontend refine ต่อจาก snapshot สดหน้างาน (ทันเหตุการณ์กว่า แต่ข้อมูลน้อยกว่า) ระบบเลือกใช้ live ก่อนเสมอ ถ้ายังไม่เคย train live ในเซสชันนั้น fallback ไป seed ของ backend

**กฎที่ยึดตลอดการพัฒนา:**
1. แต่ละส่วนทำหน้าที่เดียว (single responsibility) — แก้จุดหนึ่งไม่กระทบจุดอื่น
2. Fail gracefully — ข้อมูลไม่พอ → fallback หรือขึ้น "ยังไม่พอ" ไม่ error ไม่ block ส่วนอื่น
3. อย่าเชื่อ correlation/weight ที่อ่อน (r<0.15) ว่าเป็นความจริงตายตัว
4. ทุก error message ต้องเช็คของจริงจาก DB ก่อนฟันธง ไม่เดาจาก error type
5. **ก่อนแก้ไฟล์ backend/frontend ทุกครั้ง ต้องขอไฟล์ปัจจุบันจริงจากผู้ใช้ก่อนเสมอ** ห้ามใช้ไฟล์เก่าที่แชทเก็บไว้เอง (เคยลบฟีเจอร์ 7D/14D/30D ทิ้งไปเพราะใช้ไฟล์เก่า)
6. **เวลาถอด UI ส่วนไหนออก ต้องเช็คว่ามีโค้ดจุดอื่นเรียกใช้ element ของมันแบบไม่มีเงื่อนไขไหม** (เคยถอด tab แล้วลืมจุดที่เรียก render ทุกครั้งที่ refresh ทำให้ทั้งระบบพัง)

---

## สถาปัตยกรรม 2 แอป

| แอป | Deploy ที่ | ไฟล์ | หน้าที่หลัก |
|---|---|---|---|
| **Backend** | GitHub Pages, repo `signal_supabase` | `index.html` | Train ML จากประวัติ (seed), Portfolio, ML Analyzer |
| **Frontend** | Cloudflare Workers (`frontsupabase...workers.dev`) | `index_front.html` | Dashboard, ML Pick, Ranking, Screener, Portfolio, Train Live |

ทั้งคู่ต่อ Supabase project เดียวกัน (`SIGNAL MATRIX`, ID `dhnlnvppveotthhxkcdu`) — ตาราง `ml_config`, `stock_signals` ใช้ร่วมกัน

---

## ML Architecture — คู่ขนาน 2 แกน

```
Yahoo (ราคาจริง real-time)
    ├── ML Momentum: backend(seed) → Live(ใช้จริง) → ตารางบน "ML Pick"
    └── ML Value:     backend(seed) → Value-Live(ใช้จริง) → ตารางล่าง "ของดีราคาถูก"
```

- Features Momentum: RSI, Heat, pct50, EMAMom, Score, Slope, GroupRS
- Features Value: Drawdown, RSILow, DropHigh, SlopeRec, GroupRS
- Weight = `|correlation| / sum(|correlation|)`
- ML Analyzer (backend) มีการ์ดกราฟแยกทั้ง 2 โมเดล (Correlation table + Weight bars สไตล์เดียวกัน) ไม่ใช่ text บรรทัดเดียวเหมือนเดิม

---

## Frontend — รายละเอียดแต่ละ Tab (7 tabs)

### 📊 Dashboard
Market Regime (BULL/BEAR/NEUTRAL), median วัน/รอบจาก episode 90 วัน, ดัชนีราคา (median % เปลี่ยนแปลงราคาหุ้นทั้งหมด/วัน ทบต้นสะสม) — ต้องรอตลาดพลิก regime อย่างน้อย 1 ครั้งถึงเห็น % ของ episode

### 🎯 ML Pick (ตารางบน)
Filter: Confidence A/B + Trend UP + Heat ไม่ EXTREME → จัดอันดับด้วย momTier (accel/stable/slow จาก history หลายรอบ) — **R:R ตรงนี้ตั้งใจไม่แก้ให้เหมือน Ranking/Screener** เพราะละเอียดกว่า (มี history จริง)

### 💎 ของดีราคาถูก (ตารางล่าง)
Filter: "ตกจากตารางบน" + ต้องมี Bounce Score (`calcBounceScore`, pct50<0 เท่านั้น) — **แก้บั๊กสำคัญ:** เคยอ่าน history ผิด key (`mompick_price_` ซึ่งมีแค่หุ้นที่ผ่าน ML Pick) ทำให้ DropHigh/SlopeRec เป็น 0 เสมอสำหรับหุ้นร่วงจริง แก้เป็น `valuepick_pricehist_` ที่เก็บทุกตัวแล้ว

### 📈 Ranking วันนี้
ไม่กรองเงื่อนไข เห็นหุ้นทั้งหมด คลิกหัวคอลัมน์ sort ได้ทุกตัว
- **R:R**: สูตรต่อเนื่อง `clamp(3-(RSI-50)×0.1, 0.3, 6)` — ผ่านการแก้ 3 รอบ (pct50-based ตอนแรกกลับด้านกับความจริง → ขั้นบันได 3 ระดับติดก้อน → ต่อเนื่องพร้อมเพดานปรับจาก 3→6)
- **🔥 จุดเปลี่ยน**: (เทรนด์ UP มาไม่เกิน 3 รอบ) + RSI<60 + Confidence ขยับขึ้น — เดิมเข้มกว่านี้ (ต้องจับจังหวะ DOWN→UP เป๊ะ) ผ่อนแล้วเพื่อเพิ่มโอกาสจับได้ 3 เท่า
- **🎯 Convergence box**: เทียบ "โอกาสชนะ (Technical+สถิติจริง)" กับ "ML Score" — สแกนทั้งฝั่ง 🧠 Momentum (จาก lastPicks) และ 💎 Value (จาก calcBounceScore) พร้อมกัน, flag เมื่อทั้งคู่ ≥60% และห่างกันไม่เกิน 12 จุด — ต้อง re-render หลัง Train Live/Train Value Live เสร็จด้วย (เคยลืม ทำให้ผลค้าง)
- ปุ่ม BUY/SELL ในตาราง (เพิ่มทีหลัง, logic เดียวกับ ML Pick)
- **บั๊กที่เจอและแก้แล้ว**: Heat filter เทียบ string ผิด (`d.heat` เก็บเป็น `"🟢 COOL"` มี emoji แต่ chip ส่ง `"COOL"` เปล่าๆ — ต้อง strip emoji ก่อนเทียบ)

### 🔍 Screener
ตั้งเงื่อนไขเอง หลายเงื่อนไขพร้อมกัน (AND logic): Trend/Confidence/Heat (chip เลือกได้หลายค่า), RSI/pct50/EMAMom/GroupRS/Score (ช่วง min-max), R:R ขั้นต่ำ, **🎯 ML vs ข้อมูลจริง** (สอดคล้อง/ไม่สอดคล้อง — ใช้ threshold เดียวกับ modal คือ gap≤12 ไม่ต้อง≥60% เหมือน Convergence box) — บันทึก/โหลด preset ได้ (localStorage) — มีปุ่ม BUY/SELL ในตารางด้วย

### 💼 Portfolio
เห็นทุกตัวที่ถืออยู่ไม่ว่าจะผ่าน ML Pick หรือไม่ — SL = `avg_cost × (1-slPct)` (LOW=3%, MEDIUM/HIGH=5%, fixed % ไม่ผูก regime) — **เพิ่ม KPI cards** (ถือทั้งหมด/AVG P&L/AVG UPSIDE/SL ALERT) + **กล่อง "แนะนำ Capital Rotation"** (แสดงตัวที่ Upside<15% หรือติด SL) + **คอลัมน์ UPSIDE** ในตาราง — Upside ใช้สูตรเดียวกับ Ranking/Screener (RSI-based ต่อเนื่อง)

### 🔍 กล่องวิเคราะห์ (modal)
ค้นข้อมูลจาก `lastPicks` → `lastValuePicks` → `allResults` (fallback ครอบคลุมทุกตัว รวมที่ไม่ผ่านตารางไหนเลย) — R:R ใช้สูตรเดียวกับ Ranking/Screener

---

## Backend — รายละเอียดแต่ละ Tab (4 tabs — ถอด Summary History แล้ว)

### Signal Matrix / Portfolio / สรุป Stock ล่าสุด
7D/14D/30D performance ranking, KPI cards, **Upside column แก้สูตรแล้ว** (เดิมใช้ `100-CompositeScore` ซึ่งไม่เกี่ยวกับราคา/เป้าหมายจริงเลย ทำให้ตัวเลข 90-130% ไม่สมเหตุสมผล → เปลี่ยนเป็นสูตร RSI+Risk เดียวกับ frontend R:R แล้ว ได้ตัวเลขจริง ~1-30%)

### 🧠 ML Analyzer
Train ML / Train Value / Save ML Config → Supabase — Step 2-4 Momentum cards + **Value Model card ใหม่** (Correlation table + Weight bars) — Save ใช้ verify-after-save pattern (เช็คของจริงจาก DB ไม่เดาจาก error type)

### ~~📋 Summary History~~ (ถอดออกแล้ว)
ถอดแค่ UI (nav button + content div) **ฟังก์ชันเบื้องหลังยังอยู่ครบ** (`parseSummaryRows`, `computeAccuracy`, `renderAccBars`, `renderScoreChart`) เพราะ ML Analyzer (version number, STEP 4 accuracy) ยังใช้อยู่ — **ต้องระวัง**: มีจุดที่เคยเรียก `renderSummaryTab()` แบบไม่มีเงื่อนไขทุกครั้งที่ข้อมูลโหลด (comment เดิม "render ล่วงหน้าไม่ต้องรอกด tab") ลบจุดนั้นออกแล้ว ไม่งั้นจะ error เพราะ DOM element ไม่มีแล้ว

---

## Save Pattern (ml_config) — สำคัญมาก อย่าย้อนกลับไปทำแบบเก่า

**Insert-only เสมอ ห้าม PATCH ปิด active เก่าเอง** — มี DB trigger `ensure_single_active_ml_config` จัดการปิด row เก่าอัตโนมัติแบบ atomic

**ตรวจสอบสถานะ save จริงเสมอ** (query กลับไปเช็คว่า version ที่ insert เข้าไปจริงไหม ไม่เดาจาก error type เช่น "Load failed" ซึ่งอาจเป็นแค่เน็ตสะดุด)

---

## บั๊กสำคัญที่เจอและแก้แล้ว (ประวัติ — อย่าทำซ้ำ)

| บั๊ก | สาเหตุ | แก้ยังไง |
|---|---|---|
| Bounce Score เป็น 0 เสมอสำหรับหุ้นร่วงจริง | อ่าน history ผิด localStorage key | เปลี่ยนเป็น key ที่เก็บทุก symbol |
| Heat filter ใน Screener กรองไม่ได้เลย | เทียบ string มี emoji (`"🟢 COOL"`) กับ string เปล่า (`"COOL"`) | strip emoji ก่อนเทียบทุกจุด |
| R:R หลอก (pct50-based) | หุ้นเพิ่งตัดขึ้น (room เยอะ) ดูไม่คุ้ม, หุ้นวิ่งไกลแล้วดูคุ้มมาก — กลับด้านกับความจริง | เปลี่ยนเป็น RSI-based |
| R:R ติดก้อนเป็นแถว | เพดาน clamp ตั้งไว้แคบเกิน (3) ชนกันเยอะ | ขยายเพดานเป็น 6 |
| Backend "offline TEST DATA" หลังถอด tab | เรียก `renderSummaryTab()` แบบไม่มีเงื่อนไข ชี้ไปยัง DOM ที่ถูกลบ ทำให้ error กลางฟังก์ชันโหลดข้อมูลหลัก | ลบจุดเรียกที่ไม่จำเป็นออก |
| **ราคาไม่อัพเดตข้ามแอป (backend ค้างราคาเก่า)** | `run_seq: Date.now()/1000 % 100000` วนซ้ำทุก ~27.8 ชม. ทำให้ข้อมูลเก่ากว่าบางครั้งได้เลขมากกว่าข้อมูลใหม่ | เปลี่ยนเป็น `Math.floor(Date.now()/1000)` ไม่วนซ้ำ ไม่เกินขนาด int4 |
| 7D/14D/30D chart หายไปจากไฟล์ backend | แก้ไฟล์จากสำเนาเก่าที่แชทเก็บไว้เอง ไม่ได้ขอไฟล์ปัจจุบันจากผู้ใช้ก่อน | กู้จาก git history บน GitHub, ตั้งกฎห้ามใช้ไฟล์เก่าอีก |
| Backend Upside 90-130% ไม่สมเหตุสมผล | สูตร `100 - CompositeScore` ไม่เกี่ยวกับราคา/เป้าหมายจริงเลย | เปลี่ยนเป็นสูตร RSI+Risk เดียวกับ frontend |

---

## ข้อจำกัดที่รู้อยู่ (อย่าแก้จนกว่าจะมีข้อมูลพอ)

- Correlation ส่วนใหญ่ยังอ่อน (r<0.15) ยกเว้น Value model ที่เริ่มแข็งขึ้น (r=0.34 ที่ 1315 samples) ต้องติดตามว่าคงที่ไหม
- Market regime median มาจากไม่กี่ episode — รอ ~180 วันก่อนเชื่อถือได้
- Portfolio SL เป็น fixed % ไม่รู้จักบริบทตลาด — รอข้อมูล regime timing พอค่อยทำ dynamic
- 🔥 จุดเปลี่ยน หายากโดยตั้งใจ (กันสัญญาณหลอก)

## สิ่งที่รอทำ (หลังมีข้อมูลพอ)

- Win-rate tracking เทียบ ML score กับผลจริงย้อนหลัง
- ผูก Portfolio SL เข้ากับ market regime timing แบบ dynamic
- สัญญาณ "จังหวะเข้า-ออก" ระดับตลาดรวม — รอข้อมูล ~180 วัน

## Troubleshooting ที่เจอบ่อย

- **Save/Refresh error บนมือถือ/แท็บเล็ต** → มักเป็นเน็ตสะดุดชั่วคราว เช็คของจริงจาก DB ก่อนตื่นตระหนก ลองบน PC ถ้าซ้ำบ่อย
- **แก้โค้ดแล้วไม่เห็นผล** → เปิด Private/Incognito tab ใหม่กันแคช
- **ราคา/ข้อมูลไม่ตรงกันระหว่าง 2 แอป** → เช็ค `run_seq`/logic การเลือก "แถวล่าสุด" ก่อน ไม่ใช่แค่เช็ค connection
- **ก่อนแก้ไฟล์ backend/frontend** → ขอไฟล์ปัจจุบันจากผู้ใช้เสมอ ห้ามใช้ไฟล์เก่าที่เก็บไว้เอง
