# Signal Matrix — คู่มือระบบฉบับเต็ม (อัพเดตล่าสุด)

> เอกสารนี้สรุปทั้งระบบ (frontend + backend) ให้ chat ใหม่เริ่มงานได้ทันทีโดยไม่ต้องอธิบายซ้ำ
> วางไฟล์นี้ไว้ที่ root ของ repo `signal_supabase` เป็น `README.md`
> **อัพเดตรอบนี้:** เพิ่ม Screener preset ใหม่ 2 ปุ่ม, กล่องบริบทตลาด, กลไกส่ง High/Low+SL/TP ผ่าน Supabase, และเปลี่ยนระบบ SL ทั้งหมดใน Portfolio (frontend) จาก % คงที่ → RSI Recovery-based

## หลักการออกแบบ (อ่านก่อนอย่างอื่น)

ระบบช่วยตัดสินใจซื้อขายหุ้น ~34-38 ตัว ใช้ ML ช่วยสอบทาน **คนตัดสินใจสุดท้ายเสมอ ไม่มี auto-trade**

**3 แกนที่ตั้งใจแยกอิสระจากกัน:** ML แยกกัน (Momentum vs Value) · ข้อมูลจริง (seed vs live) · การตัดสินใจของคน

**ML คือฐานที่จำเป็นแต่ไม่พอเพียง:** ML ช่วยชี้ว่าฟีเจอร์ไหนสำคัญ (เช่น GroupRS ที่ยืนยันซ้ำหลายรอบ train ทั้ง 2 โมเดล) แต่ไม่รู้จักบริบทปัจจุบันเสมอไป (เคย IONQ/OKLO ที่ ML มองบวกเกินจริง) — ต้องเช็คคู่กับสถิติจริงเสมอผ่าน Convergence

**ทุกอย่างต้องมองผ่านภาพรวมตลาดก่อน:** สัญญาณรายตัว (Score/Conf/R:R/Convergence) เป็นแค่ "ภาพซูมเข้า" — ต้องรู้ก่อนว่าอยู่ช่วงไหนของรอบตลาดใหญ่ (Dashboard: regime, median, breadth) ไม่งั้นตีความสัญญาณย่อยผิดทางได้ง่าย

**ความสัมพันธ์ 2 แอป:** backend train seed จากประวัติ (ฐานมั่นคง) → frontend refine ด้วย live snapshot (ทันเหตุการณ์) → ระบบใช้ live ก่อนเสมอ fallback seed ถ้ายังไม่เคย train

**กฎที่ยึดตลอดการพัฒนา:**
1. แต่ละส่วนทำหน้าที่เดียว แก้จุดหนึ่งไม่กระทบจุดอื่น
2. Fail gracefully — ไม่ error ไม่ block ส่วนอื่น
3. อย่าเชื่อ correlation/weight ที่ยังอ่อน (r<0.15) หรือ median ที่มาจาก sample น้อย (<5 รอบ) ว่าเป็นค่าจริงนิ่งแล้ว — ต้องเห็น pattern ซ้ำหลายรอบก่อนเชื่อ ไม่ใช่ปรับกลยุทธ์ตามรอบล่าสุดรอบเดียว
4. ทุก error ต้องเช็คของจริงจาก DB ก่อนฟันธง ไม่เดาจาก error type
5. **ก่อนแก้ไฟล์ backend/frontend ทุกครั้ง ต้องขอไฟล์ปัจจุบันจริงจากผู้ใช้ก่อนเสมอ**
6. **เวลาถอด UI ส่วนไหนออก ต้องเช็คว่ามีโค้ดจุดอื่นเรียกใช้ element แบบไม่มีเงื่อนไขไหม**
7. **เพิ่มเงื่อนไข/ฟีเจอร์ใหม่เฉพาะที่ให้ข้อมูลมิติใหม่จริง** ไม่เพิ่มเพื่อกรองซ้ำมิติเดิม
8. **ข้อมูลนิ่ง (คำนวณจากประวัติจริง) ต้องซิงค์ให้เหมือนกันทั้ง 2 แอป ผ่าน Supabase — ข้อมูลต่อยอดสด (เช่น ราคาที่เพิ่ง refresh) ใช้ตัดสินใจในตัวเองได้เลย ไม่ต้อง sync กลับ** (กฎใหม่จากเซสชันนี้ — ดูหัวข้อ "ความเร็วข้อมูลไม่เท่ากัน" ด้านล่าง)

---

## สถาปัตยกรรม 2 แอป

| แอป | Deploy ที่ | ไฟล์ | หน้าที่หลัก |
|---|---|---|---|
| **Backend** | GitHub Pages, repo `signal_supabase` | `index_backend.html` | Train ML จากประวัติ (seed), Portfolio (มุมมองวิเคราะห์อิสระ ไม่ใช่จุดตัดสินใจ), ML Analyzer, สรุป Stock ล่าสุด |
| **Frontend** | Cloudflare Workers | `index_front.html` | Dashboard, ML Pick, Ranking, Screener, **Portfolio (จุดตัดสินใจ/ลงมือซื้อขายจริง)**, Train Live |

ทั้งคู่ต่อ Supabase project เดียวกัน (ID `dhnlnvppveotthhxkcdu`)

**สำคัญ:** Portfolio มีอยู่ทั้ง 2 แอป แต่คนละหน้าที่ — **backend = มุมมองวิเคราะห์เสริม (portfolioAction) ไม่ต้องแตะ ไม่ต้อง sync มา frontend / frontend = จุดตัดสินใจและลงมือขายจริงเพียงจุดเดียว**

---

## ML Architecture

```
Yahoo (ราคาจริง) → ML Momentum: backend(seed) → Live → "ML Pick"
                 → ML Value:    backend(seed) → Value-Live → "ของดีราคาถูก"
```

**สิ่งที่ยืนยันซ้ำหลายรอบ train (เชื่อถือได้):** **GroupRS ติดอันดับ 1-2 ของทั้ง Momentum และ Value model แทบทุกรอบ** — เป็นฟีเจอร์เดียวที่ 2 โมเดลอิสระเห็นตรงกันสม่ำเสมอที่สุด สะท้อนว่า "กลุ่มเป็นยังไง" สำคัญพอๆ กับ "ตัวหุ้นเป็นยังไง"

**สิ่งที่ยังไม่นิ่ง (อย่ารีบเชื่อ):** อันดับ 1/2 ระหว่าง GroupRS กับ Drawdown (ฝั่ง Value) สลับกันไปมาทุกรอบ, SlopeRec เคยเปลี่ยนเครื่องหมาย r — Momentum ยังคง r อ่อนกว่า Value เสมอ (0.05-0.08 vs 0.12-0.17)

### ⚠️ บั๊กที่รู้อยู่แต่ไม่เร่งด่วน — weight ทิ้ง sign ของ correlation

ทั้ง 4 จุด (`trainLive`/`trainValueLive` ที่ frontend, `runMLTrain`/`trainValueModel` ที่ backend) คำนวณ weight ด้วย `Math.abs(corr)/total` — **ทิ้งเครื่องหมายของ correlation ไปเก็บแค่ขนาด** แล้วตอนคิดคะแนนจริง (`calcBounceScore` ฯลฯ) บวกทุก feature เข้าไปในทิศทางบวกเสมอ ถ้า feature ไหนมี correlation จริงเป็นลบ (เช่น SlopeRec ที่เคยเปลี่ยนเครื่องหมาย) คะแนนจะผิดทิศเงียบๆ

**ทำไมยังไม่ต้องรีบแก้:** จุดตัดสินใจจริงทั้งหมด (Screener, Conf/สอดคล้อง ในบริบท Trend UP, ตาราง ML Pick ที่อ่านจาก tier-text ไม่ใช่อันดับ/เลข BOUNCE) **หลบเส้นทางที่มีบั๊กนี้เกือบทั้งหมด** เพราะใช้ข้อมูลดิบ (pct50/RSI/Trend) เป็นหลัก ไม่ใช้ bScore/mlScore โดยตรง — **ควรกลับมาแก้ก่อนเริ่มทำ win-rate tracking แบบผูก bScore เข้ากับสถิติจริง** (ดูหัวข้อ Win-rate ด้านล่าง — ตอนนี้มีข้อมูลจริงพร้อมแล้ว)

---

## Frontend — 7 Tabs

### Dashboard
Market Regime, median วัน/รอบ (90 วัน), Timeline 60 วันล่าสุด (scroll ได้), ดัชนีราคาสะสม — **median ยังผันผวนมาก จากตัวอย่างน้อย (<5 รอบ)** ห้ามใช้พยากรณ์ล่วงหน้าแม่นยำ ใช้แค่เป็นกรอบคร่าวๆ

### ML Pick / ของดีราคาถูก
ตารางบน=Momentum (momTier history-based), ตารางล่าง=Value (`calcBounceScore`, pct50<0 เท่านั้น) — **อ่านคอลัมน์ tier-text/ทิศทางราคาเป็นหลัก ไม่ใช่อันดับหรือเลข BOUNCE** (บั๊ก sign ด้านบน)

### Ranking วันนี้
ไม่กรอง เห็นทุกตัว + R:R (RSI-based ต่อเนื่อง) + 🔥 จุดเปลี่ยน + 🎯 Convergence box (Momentum+Value พร้อมกัน) + ปุ่ม BUY/SELL

### 🔍 Screener — หัวใจของการตัดสินใจ
ตั้งเงื่อนไขเอง (Trend/Conf/Heat chip, RSI/pct50/EMAMom/GroupRS/Score range, R:R min, Upside min, ML vs ข้อมูลจริง chip) + preset ส่วนตัว (localStorage) + **สูตรมาตรฐาน hardcode 6 ปุ่ม:**

| ปุ่ม | สี | เงื่อนไข | ใช้เมื่อ |
|---|---|---|---|
| 🟢 พื้นฐาน | เขียว | Trend UP, Conf A/B, Score≥5, Upside≥15, GroupRS≥0, สอดคล้อง | หาจุดเข้าเข้มสุด |
| 🟡 ปานกลาง | เหลือง | ผ่อน Score≥3, Upside≥10 (คง GroupRS/Convergence) | หลักฐานแน่นสุดที่ยังผ่อนได้ |
| 🔴 สูง | แดง | ผ่อน Conf A/B/C, Upside≥5, GroupRS≥-2 | ปลด Score/Convergence |
| 🔥 ร้อนเกิน | ส้ม | Trend UP, pct50≥8%, RSI≥70 | เตือนพิจารณาขาย (แพงเกิน) — เป็นแค่ "เริ่มร้อน" ไม่ใช่สัญญาณขายจริงของ backend (ต้อง RSI≥80+Heat EXTREME ถึงจะตรงเกณฑ์ TAKE PROFIT จริง) |
| 🧊 ดิ่งหนักสะสม | ฟ้า | Trend DOWN, pct50≤-5%, RSI≤30 | เตือนพิจารณาตัดขาดทุน (รายตัว ไม่พึ่งภาพรวมตลาด) — คู่ตรงข้าม 🔥 |
| 💎 เด้งยืนยันแล้ว | ชมพู | pct50≤-2% + **ราคาเพิ่มขึ้นต่อเนื่อง 3 รอบติด (hardcode ไม่มีช่องปรับ)** | แก้ปัญหา calcBounceScore ที่มีแค่ SlopeRec ตัวเดียวจับ "เด้งจริง" ตัวอื่นจับแค่ "ถูกแค่ไหน" — กันหุ้นร่วงต่อเนื่องแบบไม่มีวี่แววเด้ง (falling knife) ติดอันดับ |

**หลักออกแบบ 3 ระดับเสี่ยง:** ผ่อนเฉพาะเงื่อนไขที่หลักฐานอ่อนกว่าก่อน (Score, Upside) เก็บเงื่อนไขที่ยืนยันซ้ำที่สุดไว้นานสุด (GroupRS, Convergence)

**📍 กล่องบริบทตลาด (dynamic)** — อยู่ใต้ปุ่ม preset แยกกล่องชัดเจน (ขอบเส้นประ) ดึงจาก `regimeCtx` ที่ Dashboard คำนวณอยู่แล้ว (ไม่สร้างสูตรใหม่ซ้ำ) โชว์ "กระทิง/หมีวันที่เท่าไหร่ เทียบ median" แบบ 3 ชั้น (ปกติ/จับตา ±1 วัน buffer/เกินจริง) — **แค่โชว์ ไม่ auto-ปรับเลข filter ให้** ตามหลัก "คนตัดสินใจสุดท้ายเสมอ"

**⚠️ ทุกครั้งที่ผ่านเกณฑ์ ต้องเปิด modal เช็คซ้ำก่อนตัดสินใจจริง** — ราคาเปลี่ยนเร็วกว่าที่ตารางแสดง

### 💼 Portfolio — จุดตัดสินใจ/ลงมือขายจริง

ทุกตัวที่ถืออยู่ + KPI cards + กล่อง "แนะนำ Capital Rotation" + คอลัมน์ EMAMom/pct50/Upside (สูตรเดียวกับ Ranking/Screener)

**🧊 SL ใหม่ (RSI Recovery-based) — แทนที่ระบบ SL แบบ % จากทุนเดิมทั้งหมด:**
- Trigger: เคยแตะ RSI≤30 (oversold) → ฟื้นขึ้นเหนือ 30 ได้จริง → **ร่วงกลับต่ำกว่า 30 อีกครั้ง = 🔴 TRIGGER**
- ข้อมูล 2 ชั้น: **ฐานนิ่งจาก backend** (`recoveryTroughConfig`, มาจาก `recoveryTroughSince()`) + **ข้อมูลสดที่ frontend เห็นเอง session นี้** (`sessionTroughTracker`, in-memory, รีเซ็ตเมื่อปิด/รีโหลดหน้า — ไม่ persist)
- เหตุผลที่เลิกใช้ % จากทุน: SL ควรอิงโมเมนตัมจริงของหุ้น ไม่ใช่แค่ทุนที่เข้าซื้อ (ทุนต่างกันแต่ราคาเดียวกัน ควรได้สัญญาณเดียวกัน)

**💎 TP อ้างอิงเฉยๆ (ไม่ automate — ตัดสินใจแล้วว่าจะไม่สร้าง trigger):**
- โชว์ High(14D จาก backend) + RSI peak(≥70 จาก `momentumPeakSince()`) เป็นข้อมูลประกอบ
- เหตุผล: ทำนายไม่ได้ว่าจะทะลุแนวต้านไหม (ข้อมูล regime ยังไม่พอ) — ให้คนดูข้อมูลแล้วตัดสินใจเอง ไม่ฝืนทำเป็นกฎ

### กล่องวิเคราะห์ (modal)
fallback: `lastPicks` → `lastValuePicks` → `allResults` — โชว์ "โอกาสชนะ(ข้อมูลจริง)" vs "ML Score" พร้อมเตือนถ้าห่างกันเกิน 12 จุด

---

## Backend — 4 Tabs

Signal Matrix / Portfolio (มุมมองวิเคราะห์อิสระ ดู `portfolioAction()` ด้านล่าง — ไม่ sync มา frontend) / สรุป Stock ล่าสุด (7D/14D/30D + High/Low table ใหม่) / ML Analyzer (Momentum + Value model card แยก)

### Backend Portfolio — engine วิเคราะห์ที่มีอยู่แล้ว (คนละระบบกับ frontend SL/TP)

`portfolioAction()` รวม 3 scores ตัดสินใจ action แนะนำ:
- `healthScore()` — "ติดลบอยู่ ซื้อเพิ่มหรือขายทิ้ง" (EMA trend + RSI zone + slope + confidence, -60 ถึง +100)
- `momentumScore()` — "กำไรแล้ว ไปต่อหรือหมดแรง" (RSI<75/<70, heat≠EXTREME, action≠TAKE PROFIT)
- `pctFromHigh()` — %ร่วงจาก high 10 sessions ล่าสุด (exit override ถ้าร่วง≥8% ตอนกำไร / ≥12% เสมอ)
- `trendSlope()` — ความชันแนวโน้ม (exit override ถ้าอ่อนแรงเร็ว slope≤-8)

**สถานะ:** ทำงานอิสระ ไม่ต้องแตะ ไม่ต้อง sync มา frontend — เป็นมุมมองเสริมของ backend เอง

### 🎯 High/Low N-Day table (ใหม่)

ในแท็บ "สรุป Stock ล่าสุด" — ใช้ตัวเลือก 7D/14D/30D ร่วมกับกราฟ Performance เดิม แต่คำนวณต่างกัน: **ไล่ดูทุก session ในช่วงจริงหา max/min** (ต่างจากกราฟ Performance ที่เทียบแค่จุดต้น-จุดปลาย) โชว์ราคาปัจจุบัน/สูงสุด/ต่ำสุด/ห่าง High-Low %

**เลือกใช้ 14 วัน** เป็นค่าอ้างอิงหลัก (มากกว่า median รอบหมี 12 วันนิดหน่อย — มีโอกาสครอบคลุมรอบเต็มโดยไม่ยาว/สั้นเกิน แต่ยังเป็นค่าจาก sample น้อย ไม่ใช่ค่าฟันธง)

### 📤 ส่งข้อมูลไป Supabase (`config_type='pricerange'`)

ปุ่ม "📤 ส่ง 14D → Supabase" — hardcode 14 วันตายตัว **ไม่ผูกกับปุ่ม 7D/14D/30D ที่ใช้ดูตาราง** (ปุ่มนั้นไว้แค่ดู สลับได้อิสระไม่กระทบสิ่งที่ส่ง) ส่ง 3 ชุดข้อมูลในก้อนเดียว (เก็บใน `ml_config.thresholds`):

| key | มาจาก | ใช้ทำอะไร |
|---|---|---|
| `prices` | `computeNDayHighLow(14)` | High/Low/current ต่อ symbol |
| `momentum_peaks` | `momentumPeakSince()` — เฉพาะ RSI≥70 | 💎 TP อ้างอิง (ไม่ automate) |
| `recovery_troughs` | `recoveryTroughSince()` — เฉพาะเคยแตะ RSI≤30 แล้วกำลังฟื้น | 🧊 SL ใหม่ ฐานนิ่ง (frontend เอาไป automate ต่อ) |

**⚠️ ต้องรัน SQL นี้ใน Supabase ก่อนใช้งานครั้งแรก** (ค่า `config_type` มี CHECK constraint จำกัดชุดคำที่อนุญาตไว้):
```sql
ALTER TABLE ml_config DROP CONSTRAINT ml_config_config_type_check;
ALTER TABLE ml_config ADD CONSTRAINT ml_config_config_type_check
  CHECK (config_type IN ('value', 'backend', 'live', 'value-live', 'pricerange'));
```

กลไก save/check ตาม pattern เดียวกับ `exportMLConfig()` เดิม — POST insert แล้วเช็คย้อนกลับจาก Supabase จริงว่าเข้าไปหรือยัง (ไม่เดาจาก error type, ตามกฎข้อ 4)

### ⚠️ ความเร็วข้อมูลไม่เท่ากัน — ทำไม peak/trough ต้องมี 2 ชั้นเสมอ

Backend บันทึกเป็น session (ไม่บ่อย) ส่วน frontend refresh ได้ถี่กว่ามาก **ไม่มีทางรู้ว่า backend ส่งข้อมูลล่าสุดเมื่อไหร่เทียบกับตอนนี้** ถ้าใช้ค่าจาก backend เป็น peak/trough ตรงๆ โดยไม่เทียบกับสิ่งที่ frontend เห็นสด อาจพลาดจังหวะที่ราคาทำ new high/low ไปแล้ววันนี้แต่ backend ยังไม่ทันบันทึก

**กฎ (ข้อ 8 ด้านบน):** ข้อมูลนิ่ง (อดีตที่ปิดจบแล้ว) → sync ผ่าน Supabase / ข้อมูลสด (กำลังเกิดขึ้นตอนนี้) → คำนวณเองที่ frontend ไม่ต้อง sync กลับ → **trigger จริง = ใช้ค่าที่ครอบคลุมกว้างกว่าเสมอระหว่าง 2 แหล่ง**

---

## บั๊กสำคัญที่เจอและแก้แล้ว (ประวัติ — อย่าทำซ้ำ)

| บั๊ก | สาเหตุ | แก้ยังไง |
|---|---|---|
| Bounce Score เป็น 0 เสมอ | อ่าน history ผิด localStorage key | เปลี่ยน key ที่เก็บทุก symbol |
| Heat filter กรองไม่ได้เลย | เทียบ string มี emoji กับเปล่า | strip emoji ก่อนเทียบ |
| R:R หลอก | สูตรเก่ากลับด้านกับความจริง | เปลี่ยนเป็น RSI-based |
| Backend "offline TEST DATA" | เรียก `renderSummaryTab()` ชี้ DOM ที่ถูกลบ | ลบจุดเรียกออก |
| ราคาไม่อัพเดตข้ามแอป | `run_seq` วนซ้ำทุก ~27.8 ชม. | เปลี่ยนเป็นวินาทีเต็ม ไม่วนซ้ำ |
| Backend Upside 90-130% ผิดปกติ | สูตร `100-CompositeScore` ปลอม | เปลี่ยนเป็น RSI+Risk |
| ALAB/NBIS ซ้ำในตาราง | ดึง symbols ไม่กันซ้ำ | เพิ่ม Set กันซ้ำ |
| ไฟล์เก่าทับไฟล์ใหม่ | แก้จากสำเนาเก่า ไม่ขอไฟล์ปัจจุบัน | กู้จาก git history |
| ส่ง pricerange ไป Supabase ไม่ได้ | CHECK constraint `ml_config_config_type_check` ไม่รู้จักค่าใหม่ | รัน SQL เพิ่มค่าเข้า constraint (ดูด้านบน) |

---

## ข้อจำกัดที่รู้อยู่

- Correlation ส่วนใหญ่ยังอ่อน ยกเว้น Value GroupRS/Drawdown
- **Weight ทิ้ง sign ของ correlation ทั้ง 4 จุด train** (ดูหัวข้อ ML Architecture) — รู้อยู่ ไม่กระทบ workflow ปัจจุบัน ควรแก้ก่อนทำ win-rate tracking แบบผูก bScore
- Market regime median ผันผวนสูง (sample <5 รอบ) — ห้ามพยากรณ์แม่นยำ
- SL ใหม่ฝั่งสด (`sessionTroughTracker`) เป็น in-memory รีเซ็ตทุกครั้งที่ปิด/รีโหลดหน้าเว็บ — ถ้าอยากทนต่อการปิดเปิดมากขึ้น ต้องย้ายไป localStorage (แลกกับความเสี่ยง "นับรอบไม่ใช่นับเวลา" แบบที่เจอปัญหา SlopeRec มาก่อน — ยังไม่ทำเพราะเลือกความเรียบง่ายไว้ก่อน)
- TP ไม่มี trigger อัตโนมัติโดยตั้งใจ — เพราะทำนายไม่ได้ว่าจะทะลุแนวต้านไหมจากข้อมูลที่มี
- "จุดเข้า" พื้นฐานมักได้ 0/38 ตัว = ทำงานถูกต้อง ไม่ใช่บั๊ก (เช่นเดียวกับ 🧊/💎 ที่มักได้ตัวน้อยมาก)

## สิ่งที่รอทำ

- ~~Win-rate tracking เทียบ ML score กับผลจริง~~ **มีอยู่แล้วจริงที่ backend** (Train ML modal โชว์ 79+ รายการปิดสถานะจริงพร้อม PnL/WIN-LOSS) — README เก่าเขียนผิดว่ายังไม่ทำ แก้ไขแล้วรอบนี้
- แก้บั๊ก sign ของ correlation ก่อนเริ่มผูก win-rate จริงเข้ากับ bScore
- ผูก Portfolio SL (backend เอง) เข้ากับ regime timing — ยังเป็นแผนเดิม
- เก็บ median regime ให้ครบ ≥5 รอบทั้งกระทิง/หมี ก่อนพิจารณาทำ trailing-exit ที่ผูกกับ regime context จริงจัง
- สังเกตผล SL ใหม่ (RSI Recovery) ใช้งานจริงสักพักก่อนค่อยพิจารณาปรับ threshold (30/50/% จาก peak — ตอนนี้เลือกสมมาตรกับ TP คือหลุด 30 อีกครั้ง)

## Troubleshooting

- Save/Refresh error บนมือถือ → มักเป็นเน็ตสะดุด เช็ค DB ก่อน
- แก้โค้ดไม่เห็นผล → Private tab ใหม่
- ราคาไม่ตรงข้ามแอป → เช็ค `run_seq`
- ก่อนแก้ไฟล์ → ขอไฟล์ปัจจุบันเสมอ
- ส่ง config ใหม่ไป Supabase แล้ว error `violates check constraint` → เช็คว่า `config_type` ใหม่อยู่ใน constraint ที่อนุญาตหรือยัง (ดู SQL ด้านบน)
