# Signal Matrix — คู่มือระบบฉบับเต็ม (อัพเดตล่าสุด)

> เอกสารนี้สรุปทั้งระบบ (frontend + backend) ให้ chat ใหม่เริ่มงานได้ทันทีโดยไม่ต้องอธิบายซ้ำ
> วางไฟล์นี้ไว้ที่ root ของ repo `signal_supabase` เป็น `README.md`
> **อัพเดตรอบนี้ (สำคัญที่สุด):** แก้บั๊ก sign ของ weight เสร็จสมบูรณ์ทั้งวงจร (คำนวณ weight + กลับทิศทางเปรียบเทียบใน 10 จุด) — ก่อนหน้านี้แก้แค่ครึ่งเดียวแล้วเกิดบั๊กใหม่ (STX ร้อนสุดกลับได้ BUY, ONDS เย็นสุดกลับได้ SELL) วันนี้แก้ครบวงจรแล้วยืนยันด้วยข้อมูลจริง
> **อัพเดตล่าสุด (2026-09-10):** เพิ่ม Fear Streak Badge (มิติใหม่ ไม่แตะของเดิม) + สร้างระบบ ML Forward Model 7D/14D ทดลองคู่ขนาน (ยังไม่ใช้งานจริง — ดูหัวข้อ "ML Forward Model") + แก้บั๊ก Leader Tag timing (บั๊กที่ 3 ของสาย Leader Tag ขัดกับ Action)

## หลักการออกแบบ (อ่านก่อนอย่างอื่น)

ระบบช่วยตัดสินใจซื้อขายหุ้น ~34-38 ตัว ใช้ ML ช่วยสอบทาน **คนตัดสินใจสุดท้ายเสมอ ไม่มี auto-trade**

**3 แกนที่ตั้งใจแยกอิสระจากกัน:** ML แยกกัน (Momentum vs Value) · ข้อมูลจริง (seed vs live) · การตัดสินใจของคน

**ML คือฐานที่จำเป็นแต่ไม่พอเพียง:** ML ช่วยชี้ว่าฟีเจอร์ไหนสำคัญ (เช่น GroupRS ที่ยืนยันซ้ำหลายรอบ train ทั้ง 2 โมเดล) แต่ไม่รู้จักบริบทปัจจุบันเสมอไป — ต้องเช็คคู่กับสถิติจริงเสมอผ่าน Convergence

**ทุกอย่างต้องมองผ่านภาพรวมตลาดก่อน:** สัญญาณรายตัวเป็นแค่ "ภาพซูมเข้า" — ต้องรู้ก่อนว่าอยู่ช่วงไหนของรอบตลาดใหญ่ ไม่งั้นตีความสัญญาณย่อยผิดทางได้ง่าย

**หลักการตัดสินใจที่ยึดจริง (ยืนยันจากการใช้งานจริงหลายวัน):** ไม่ตัดสินใจจาก P&L เป็นหลัก แต่เช็คว่า **"รูปแบบที่เกิดขึ้นตรงกับสมมติฐานที่ตั้งไว้ตอนเข้าไหม"** — SL/TP ทั้งระบบออกแบบมาตอบคำถามนี้ ไม่ใช่ตอบว่า "กำไร/ขาดทุนเท่าไหร่"

**ความสัมพันธ์ 2 แอป:** backend train seed จากประวัติ (ฐานมั่นคง) → frontend refine ด้วย live snapshot (ทันเหตุการณ์) → ระบบใช้ live ก่อนเสมอ fallback seed ถ้ายังไม่เคย train

**กฎที่ยึดตลอดการพัฒนา:**
1. แต่ละส่วนทำหน้าที่เดียว แก้จุดหนึ่งไม่กระทบจุดอื่น
2. Fail gracefully — ไม่ error ไม่ block ส่วนอื่น
3. อย่าเชื่อ correlation/weight ที่ยังอ่อน (r<0.15) หรือ median ที่มาจาก sample น้อย (<5 รอบ) ว่าเป็นค่าจริงนิ่งแล้ว
4. ทุก error ต้องเช็คของจริงจาก DB ก่อนฟันธง ไม่เดาจาก error type
5. **ก่อนแก้ไฟล์ backend/frontend ทุกครั้ง ต้องขอไฟล์ปัจจุบันจริงจากผู้ใช้ก่อนเสมอ**
6. **เวลาถอด UI ส่วนไหนออก ต้องเช็คว่ามีโค้ดจุดอื่นเรียกใช้ element แบบไม่มีเงื่อนไขไหม**
7. **เพิ่มเงื่อนไข/ฟีเจอร์ใหม่เฉพาะที่ให้ข้อมูลมิติใหม่จริง** ไม่เพิ่มเพื่อกรองซ้ำมิติเดิม
8. **ข้อมูลนิ่ง (คำนวณจากประวัติจริง) ต้องซิงค์ให้เหมือนกันทั้ง 2 แอป ผ่าน Supabase — ข้อมูลต่อยอดสดใช้ตัดสินใจในตัวเองได้เลย ไม่ต้อง sync กลับ**
9. **เวลาแก้ทิศทาง/สเกลของค่าตัวไหน ต้อง grep หาทุก occurrence ของ field name ที่เกี่ยวข้องทั้งไฟล์ ไม่ใช่ไล่ตามแค่ชื่อฟังก์ชันที่จำได้** — บทเรียนจากบั๊ก sign-direction ที่แก้ไม่ครบรอบแรก (ดูรายละเอียดด้านล่าง)

---

## ⚠️⚠️ บั๊กใหญ่ที่สุดที่เจอ — Sign ของ weight/composite score (แก้ครบแล้ว 2026-09-09)

### ต้นตอ
ทุกจุดที่ train ML (`runMLTrain`/`mlStep3` ที่ backend, `trainLive`/`trainValueLive` ที่ frontend) คำนวณ weight ด้วย `Math.abs(corr)/total` — **ทิ้งเครื่องหมายของ correlation ไปเก็บแค่ขนาด** ทำให้ feature ที่มี correlation จริงเป็นลบ (เช่น Heat -0.08, RSI -0.04 — ยืนยันจากตลาดหมียาวที่ train มา) ถูกบวกเข้า composite score ในทิศทางบวกเสมอ **ผลคือหุ้นร้อนจัด (RSI 87+, EXTREME) กลับได้คะแนนสูงราวกับเป็นหุ้นดี**

### แก้ขั้นที่ 1 — คำนวณ weight ให้เก็บ sign (เสร็จ)
5 จุด: backend `runMLTrain()`, `mlStep3()`, `trainValueModel()`; frontend `trainLive()`, `trainValueLive()` — เปลี่ยนจาก `Math.abs(corr)/total` เป็นเก็บ sign ไว้ (`corr/total`) ยัง normalize ด้วยผลรวม "ขนาด" เหมือนเดิม

**ผล:** composite score กลับทิศทันที — หุ้นร้อนจัดได้คะแนนต่ำ/ติดลบมาก (สมเหตุสมผลแล้ว) หุ้นเย็น/กลุ่มแข็งแรงได้คะแนนสูงกว่า

### แก้ขั้นที่ 2 — กลับทิศทางเปรียบเทียบทุกจุดที่ใช้ threshold (เสร็จ — สำคัญที่สุด)
**นี่คือจุดที่พลาดตอนแรก** — แก้แค่ขั้นที่ 1 แล้วคิดว่าจบ แต่ลืมว่าโค้ดทุกจุดที่ "แปลคะแนนเป็น action" ยังเทียบทิศเดิม ("คะแนนต่ำ = ปลอดภัย ซื้อได้") ซึ่งเคยถูกตอน scale เป็นบวกล้วน แต่ scale กลับทิศแล้ว **ทำให้หุ้นร้อนสุด (STX RSI92) ถูกแนะนำ BUY และหุ้นเย็นสุด (ONDS RSI58) ถูกแนะนำ SELL สลับกันไปหมด** — เจอจาก user ทดสอบเทียบ 2 ตัวจริงแล้วสังเกตความขัดแย้ง

ต้องกลับทิศเปรียบเทียบครบ **10 จุด**:

| # | ไฟล์ | จุด | อะไรเปลี่ยน |
|---|---|---|---|
| 1 | Backend | `autoTrain()` percentile threshold | tBuy=percentile สูง(ปลอดภัย) / tSell=percentile ต่ำ(ร้อนสุด) — สลับจากเดิม |
| 2 | Backend | `mlStep3()` percentile threshold | เหมือนข้อ 1 (ปุ่ม TRAIN ML) |
| 3 | Backend | `aiLabel()` | ป้าย BUY/SELL/STRONG BUY บนการ์ดหลัก SIGNAL MATRIX |
| 4 | Backend | `portfolioAction()` | คำแนะนำ "ซื้อเพิ่ม/ขายบางส่วน/ขายหมด" ใน Portfolio backend |
| 5 | Backend | `compClr()` + `gradeLabel()` | สีตัวเลขคะแนน + ป้าย "FRESH BUY/MOMENTUM" — จุดที่ทำให้ STX ดูน่าซื้อผิดๆ |
| 6 | Backend | `renderThresholds()` + `renderScoreChart()` zone | ข้อความ/สี/zone chart อธิบาย threshold ในหน้า ML Analyzer |
| 7 | Frontend | `compositeToAction()` | action ที่บันทึกเข้า `stock_signals` (Supabase) |
| 8 | Frontend | `leaderTag` inline logic | **จุดที่พลาดไปรอบแรกจริงๆ** — logic คำนวณ "Leader Tag" แยกก้อนจาก action/signal ใช้ `comp`/`mlConfig.thresholds` ตัวเดียวกันแต่เขียนเงื่อนไขซ้ำเอง ไม่มีชื่อฟังก์ชันแยกให้ grep เจอง่าย ทำให้ Leader Tag กับ Action ขัดกันเอง |
| 9 | Frontend | สีคอลัมน์ "ML Score" ในตาราง SIGNAL MATRIX | เจอตอน grep ซ้ำหา `thresholds.buy_add`/`sell_all` ทั้งไฟล์ |
| 10 | — | (สำรอง — ตรวจสอบด้วย `grep -n "thresholds\.\|mlThresholds\."` ทั้ง 2 ไฟล์ทุกครั้งที่แก้ scale) | |

**บทเรียนสำคัญ:** จุดที่ 8-9 คือเหตุผลที่เพิ่มกฎข้อ 9 ด้านบน — ต้อง `grep` หา **field name** (`.buy`/`.sell`/`.partial`/`thresholds.buy_add` ฯลฯ) ทั้งไฟล์เสมอเวลากลับทิศ/เปลี่ยน scale ไม่ใช่ไล่ตามแค่ชื่อฟังก์ชันหลักที่จำได้ เพราะ logic อาจถูกเขียนซ้ำแบบ inline โดยไม่มีชื่อฟังก์ชันให้ grep เจอ

### ยืนยันผลถูกต้องแล้วจากข้อมูลจริง
เทียบ COHR (RSI84.6 EXTREME, GroupRS **+3.13** Memory) vs MRVL (RSI86.4 EXTREME ใกล้เคียงกัน, GroupRS **-3.15** Other) — COHR ได้ ML Score -7.6 (BUY/ADD), MRVL ได้ -18.9 (HOLD) — GroupRS (feature เดียวที่ทิศถูกมาตั้งแต่แรก ไม่เคยมีบั๊ก) ดึงผลลัพธ์ให้สมเหตุสมผลได้จริง ยืนยันว่าการกลับทิศทำถูกแล้ว

### ขั้นตอนที่ต้องทำหลัง deploy โค้ดที่แก้แล้ว
1. Deploy backend → กด **Train ML** + **Train Value** ใหม่ (คำนวณ threshold ทิศใหม่ + weight sign ใหม่)
2. กด **Save ML Config → Supabase**
3. Deploy frontend → กด **Train Live** + **Train Value Live** ใหม่
4. เช็คว่า action/Leader Tag/สี ตรงทิศกันหมดในทุกตาราง

### สิ่งที่ยืนยันแล้วว่า "ไม่กระทบ" ไม่ต้องแก้
- **Score (Screener filter, 0-8 point)** — มาจากกฎ rule-based ง่ายๆ (price>EMA20 ฯลฯ) คนละสูตรกับ composite/ML ไม่เคยผ่าน weight ที่มีบั๊กเลย เกณฑ์ scoreMin เดิม (5/3) ยังใช้ได้ปกติ
- **GroupRS (Screener filter)** — มาจาก `computeGroupRegime()` สูตรสถิติดิบ (group avg - market avg) ไม่ใช่ weighted composite
- **Confidence A/B/C** — กฎ rule-based คงที่ (score 0-8 เทียบ threshold hardcode 7/5/0) ไม่เกี่ยวกับ ML เลย
- **Screener ทั้งหมด (pct50/RSI/Trend ดิบ)** — ไม่เคยแตะ composite/weight เลยตั้งแต่ต้น

---

## สถาปัตยกรรม 2 แอป

| แอป | Deploy ที่ | ไฟล์ | หน้าที่หลัก |
|---|---|---|---|
| **Backend** | GitHub Pages, repo `signal_supabase` | `index_backend.html` | Train ML จากประวัติ (seed), Portfolio (มุมมองวิเคราะห์อิสระ ไม่ sync มา frontend), ML Analyzer, สรุป Stock ล่าสุด |
| **Frontend** | Cloudflare Workers | `index_front.html` | Dashboard, ML Pick, Ranking, Screener, **Portfolio (จุดตัดสินใจ/ลงมือขายจริง)**, Train Live |

ทั้งคู่ต่อ Supabase project เดียวกัน (ID `dhnlnvppveotthhxkcdu`)

---

## Frontend — Screener (หัวใจของการตัดสินใจ) — 6 preset

| ปุ่ม | เงื่อนไข | ใช้เมื่อ |
|---|---|---|
| 🟢 พื้นฐาน | Trend UP, Conf A/B, Score≥5, Upside≥15, GroupRS≥0, สอดคล้อง | หาจุดเข้าเข้มสุด |
| 🟡 ปานกลาง | ผ่อน Score≥3, Upside≥10 | หลักฐานแน่นสุดที่ยังผ่อนได้ |
| 🔴 สูง | ผ่อน Conf A/B/C, Upside≥5, GroupRS≥-2 | ปลด Score/Convergence |
| 🔥 ร้อนเกิน | Trend UP, pct50≥8%, RSI≥70 | เตือนพิจารณาขาย |
| 🧊 ดิ่งหนักสะสม | Trend DOWN, pct50≤-5%, RSI≤30 | เตือนพิจารณาตัดขาดทุน |
| 💎 เด้งยืนยันแล้ว | pct50≤-2% + ราคาเพิ่มขึ้นต่อเนื่อง 3 รอบติด (hardcode) | กันหุ้นร่วงต่อเนื่องไม่มีวี่แววเด้งติดอันดับ |

**📍 กล่องบริบทตลาด (dynamic)** — โชว์ "กระทิง/หมีวันที่เท่าไหร่ เทียบ median" 3 ชั้น (ปกติ/จับตา±1วัน/เกินจริง) แค่โชว์ ไม่ auto-ปรับ filter

---

## Frontend — Portfolio (จุดตัดสินใจ/ลงมือขายจริง)

### 🧊 SL ใหม่ (RSI Recovery-based) — automate จริง
เคยแตะ RSI≤30 → ฟื้นเหนือ 30 ได้จริง → **ร่วงกลับต่ำกว่า 30 อีกครั้ง = 🔴 TRIGGER** ข้อมูล 2 ชั้น: ฐานนิ่งจาก backend (`recoveryTroughSince()`) + สดจาก session tracker ใน browser (รีเซ็ตเมื่อปิด/รีโหลดหน้า)

### 💎 TP Confidence — ยกระดับจาก "ตัวเลขดิบ" เป็น "สถานะความมั่นใจ" (ไม่ automate)
```
peak = max(momentum_peaks จาก backend, RSI สดตอนนี้ถ้า≥70)
rsiDropFromPeak = peak - currentRSI

🟢 ยืดแบบมีเหตุผล: rsiDropFromPeak≤0 (ยัง new high) + GroupRS≥0 + Volume=HIGH (ครบ 3 สัญญาณอิสระ)
🔴 อ่อนแรงหลายสัญญาณ: rsiDropFromPeak>5 + (GroupRS<0 หรือ Volume≠HIGH)
🟡 ทดสอบแนวต้าน: ที่เหลือ (ฟันธงไม่ได้ — เปิด modal เช็คเอง)
```
**เจตนา:** แทนที่การเดา ("จะทะลุแนวต้านไหม") ด้วยการรอสัญญาณจริงยืนยัน — ใช้ปรัชญาเดียวกับ SL (ไม่ทำนาย รอ pattern ยืนยัน) แต่ตั้งใจไม่ automate เพราะทำนายทิศทางราคาไม่ได้จากข้อมูลที่มี

**ผลทดสอบจริง (2026-09-08/09):** DELL ขาย +9.5% ตาม Capital Rotation แนะนำ (Upside บีบแคบ) ตัวกลุ่ม EXTREME (RSI88-95) ขึ้นแดงพร้อมกันหลายตัวเช้าวันถัดมา ตรงกับสมมติฐาน "ปลายกระทิง/พักตัว" ที่ผู้ใช้คาดไว้ล่วงหน้า (breadth กว้างผิดปกติ + day 3 ตรง median + ราคาเริ่มผ่อนจาก peak จริง) — ยืนยันว่า TP Confidence ใช้เป็นเครื่องมือ "ยืนยันสมมติฐาน" ได้ผลจริง ไม่ใช่แค่ทฤษฎี

### Capital Rotation banner — แยกคำ SL กับ Upside ชัดเจนแล้ว
เดิมใช้คำว่า "ถึง SL" ปนกันทั้ง SL trigger จริงกับ Upside ต่ำ (คนละเรื่อง) แก้เป็น: 🔴 "SL Trigger จริง" / 🟠 "Upside บีบแคบมาก" / 🟡 "Upside เริ่มน้อย"

---

## Backend — 4 Tabs (ถอด UI ที่ไม่ใช้แล้วออก 2026-09-09)

Signal Matrix (การ์ดหลัก + High/Low, ไม่มี Timeline table แล้ว) / Portfolio (มุมมองวิเคราะห์อิสระ — `portfolioAction()`, ไม่ sync มา frontend) / สรุป Stock ล่าสุด (7D/14D/30D + High/Low table เท่านั้น, ไม่มี action/rotation table แล้ว) / ML Analyzer

### ถอดออกแล้ว (ไม่ใช้งานจริง ผู้ใช้ยืนยันแล้ว)
- **TIMELINE table** (วันที่/Symbol/Action/Score/ราคา ใน SIGNAL MATRIX tab) — ใช้ `aiLabel()`
- **สรุป Stock ล่าสุด — Action/Rotation table** (KPI cards, filter BUY/TAKE PROFIT/WATCH, Rotate IN/OUT) — filter นับจาก `r.action` (field เก่า) แต่แถวโชว์จาก `aiLabel()` (คนละตัว) ทำให้ filter กับแถวขัดกันเอง (BUY(38) ทั้งที่แถวโชว์ SELL หมด) เป็นอีกเหตุผลที่ถอดออก ไม่ใช่แค่ไม่ได้ใช้

โค้ดลดจาก 4,459 → 4,198 บรรทัด `aiLabelPill()` ยังเก็บไว้ (การ์ดหลักยังใช้อยู่)

### 📤 กลไกส่งข้อมูลไป Supabase (`config_type='pricerange'`)
ปุ่ม "ส่ง 14D → Supabase" hardcode 14 วัน ไม่ผูกกับปุ่มดูตาราง ส่ง 3 ชุด: `prices` (High/Low), `momentum_peaks` (RSI≥70, สำหรับ TP), `recovery_troughs` (เคยแตะ RSI≤30, สำหรับ SL)

**ต้องรัน SQL นี้ครั้งแรกก่อนใช้:**
```sql
ALTER TABLE ml_config DROP CONSTRAINT ml_config_config_type_check;
ALTER TABLE ml_config ADD CONSTRAINT ml_config_config_type_check
  CHECK (config_type IN ('value', 'backend', 'live', 'value-live', 'pricerange'));
```

---

## บั๊กสำคัญที่เจอและแก้แล้ว (ประวัติ — อย่าทำซ้ำ)

| บั๊ก | สาเหตุ | แก้ยังไง |
|---|---|---|
| Bounce Score เป็น 0 เสมอ | อ่าน history ผิด localStorage key | เปลี่ยน key |
| Heat filter กรองไม่ได้เลย | เทียบ string มี emoji กับเปล่า | strip emoji ก่อนเทียบ |
| R:R หลอก | สูตรเก่ากลับด้าน | เปลี่ยนเป็น RSI-based |
| Backend Upside 90-130% ผิดปกติ | สูตรปลอม | เปลี่ยนเป็น RSI+Risk |
| ราคาไม่อัพเดตข้ามแอป | `run_seq` วนซ้ำ | เปลี่ยนเป็นวินาทีเต็ม |
| ส่ง pricerange ไป Supabase ไม่ได้ | CHECK constraint ไม่รู้จักค่าใหม่ | รัน SQL เพิ่มค่า |
| Save stock_signals ไม่ได้ (`stock_signals_action_check`) | `compositeToAction()` return `"SELL ALL"` ซึ่งไม่อยู่ใน constraint (มีแค่ 7 ค่า) | เปลี่ยนเป็น `"TAKE PROFIT"` + เพิ่ม defensive fallback ใน `cleanAction()` ให้ default WATCH เสมอถ้าไม่รู้จักค่า |
| **สลับขั้ว BUY/SELL ทั้งระบบ (STX ร้อนสุด=BUY, ONDS เย็นสุด=SELL)** | แก้บั๊ก sign ของ weight แล้ว แต่ลืมกลับทิศเปรียบเทียบใน 10 จุดที่ใช้ threshold | ไล่ grep หา field name `mlThresholds.`/`thresholds.buy_add` ฯลฯ ทั้ง 2 ไฟล์ กลับทิศครบ (ดูหัวข้อด้านบน) |
| "Leader Tag" ขัดกับ "Action" ในตารางเดียวกัน (บั๊กที่ 1-2) | logic คำนวณ leaderTag แยกก้อน ซ้ำ logic กับ action แต่ไม่ได้กลับทิศตอนแก้รอบแรก | grep หา field name ซ้ำทั้งไฟล์เจอจุดที่ตกหล่น |
| "Leader Tag" ขัดกับ "Action" อีกครั้ง (บั๊กที่ 3, คนละสาเหตุ) | `leaderTag` คำนวณก่อนรู้ group (ไม่มี GroupRS) แต่ `action` คำนวณหลังรู้ group — ค่าเดียวกันแต่คำนวณคนละจังหวะ | ย้าย leaderTag ไปคำนวณครั้งเดียวพร้อม action ที่จุดเดียวกัน (comp6D) ลบจุดคำนวณก่อนหน้าออก |

---

## ข้อจำกัดที่รู้อยู่

- Correlation ส่วนใหญ่ยังอ่อน ยกเว้น Value GroupRS/Drawdown, Momentum GroupRS
- Market regime median ผันผวนสูง (sample <5 รอบ) — ห้ามพยากรณ์แม่นยำ (แต่ใช้ประกอบกับสัญญาณอื่นได้ — ดูเคส "ปลายกระทิง" ที่ยืนยันถูกจากหลายสัญญาณพร้อมกัน)
- SL ใหม่ฝั่งสด (`sessionTroughTracker`) เป็น in-memory รีเซ็ตทุกครั้งที่ปิด/รีโหลดหน้าเว็บ
- TP ไม่มี trigger อัตโนมัติโดยตั้งใจ — ใช้เป็นเครื่องมือ "ยืนยันสมมติฐาน" ไม่ใช่ตัวทำนาย
- "จุดเข้า" พื้นฐานมักได้ 0/38 ตัว = ทำงานถูกต้อง ไม่ใช่บั๊ก

## สิ่งที่รอทำ

- Win-rate tracking มีอยู่แล้วจริงที่ backend (Train ML modal, 79+ รายการ) — ควรเริ่มผูก bScore/composite เข้ากับสถิติจริงอย่างเป็นระบบ (ตอนนี้ sign แก้ถูกแล้ว ทำได้)
- ผูก Portfolio SL (backend เอง, `portfolioAction()`) เข้ากับ regime timing
- เก็บ median regime ให้ครบ ≥5 รอบทั้งกระทิง/หมี
- สังเกตผล SL ใหม่ (RSI Recovery) ใช้งานจริงสักพัก — ยังไม่เคยเห็น trigger จริงเกิดขึ้นเลยแม้แต่ครั้งเดียว (ข้อมูล ณ 2026-09-09)
- "Leader Tag" — ถอดหรือรวมกับ Action ให้เป็นตัวเดียวกันไปเลยดีไหม (แก้บั๊ก timing แล้ว แต่ยังเป็น 2 field คู่ขนานที่ต้องระวังไม่ให้หลุด sync กันอีกในอนาคต)
- ML Forward Model 7D/14D — รอสะสม snapshot 7-14 วัน แล้วดึงย้อนหลังมาเทียบราคาจริง เพื่อวัด accuracy ที่แท้จริง (ต้องกด Train Forward + Convergence สม่ำเสมอทุกวัน)
- `autoTrain()`/`mlStep3()` เดิมยังมีบั๊ก pct50 ซ้ำ EMAMom (แก้แล้วเฉพาะใน trainForwardModel() ทดลอง) — รอตัดสินใจว่าจะแก้ของจริงด้วยไหม (ต้อง retrain ทั้งระบบใหม่)

---

## 🆕 Fear Streak Badge (2026-09-09) — มิติใหม่ของ Market Fear Index

**ปัญหาที่เจอ:** `updateFearTrend()` เดิม (กล่อง "FEAR TREND ENGINE") วัดแค่ "ขนาด" (diff ของค่าสูงสุด-ต่ำสุดใน 5 ค่าล่าสุด ≥8 ถึงจะเปลี่ยนสถานะ) — คืนที่ทดสอบจริง Fear ไหลลงต่อเนื่อง 5 รอบติด (49→48→47→46→45, diff=-4) **แต่กล่องเดิมยังโชว์ "STABLE" เพราะ diff ไม่ถึง 8** ทั้งที่ทิศทางชัดเจนมาก

**แก้ด้วยมิติใหม่ที่ไม่ซ้ำของเดิม:** เพิ่ม `calcFearStreak()`/`renderFearStreakBadge()` — นับ **"ก้าวติดทิศเดียวกัน"** แทนขนาด (ไม่สนว่าก้าวใหญ่แค่ไหน นับแค่ทิศทาง) badge ขึ้นเมื่อ ≥3 ก้าวติด (ใช้เลข 3 เดียวกับ 💎 เด้งยืนยันแล้ว) กฎ: ค่าเท่าเดิม=พัก(ไม่รีเซ็ต) / กลับทิศ=รีเซ็ตเริ่มนับ 1 ใหม่ ไม่แตะ `fearHistory`/`updateFearTrend()` เดิมเลย เป็น badge แยกที่วางข้างตัวเลข Fear ในการ์ดเดิม

**ทดสอบจริง:** เจอ badge นี้ทำงานถูกต้องตอน Fear ไหลลง 4 ก้าวติด (จับได้ ของเดิมจับไม่ได้) แล้วดีดกลับขึ้นเต็มที่ภายในคืนเดียว (49→45→50) — ยืนยันสมมติฐาน "พักตัวเพราะขึ้นเกิน 10%" มากกว่า "หมียาว" ตามที่ผู้ใช้คาดจากประสบการณ์รอบก่อน

---

## 🆕 ML Forward Model 7D/14D — ระบบทดลองคู่ขนาน (2026-09-10, "การบ้าน ML")

**ที่มา:** ผู้ใช้ให้คะแนนคุณภาพ ML เดิมแค่ 5/10 เพราะ "ไม่มี algorithm เรียนรู้และพัฒนาตัวเองที่ดี" — ยืนยันจากตัวเลข accuracy จริง (BUY 28%, TAKE PROFIT 33%) เสนอให้ใช้ข้อมูลสรุป 7/14/30 วันที่เก็บไว้แล้ว (ใช้กับ TP/SL อยู่แล้ว) มาพัฒนา ML ด้วย เป้าหมายสุดท้ายคือ "C" = เทรน 2 โมเดลแยก (7D/14D) แล้วดู Convergence ข้าม horizon

**สถานะ: ระบบทดลองคู่ขนาน 100% — ยังไม่เชื่อมเข้า `composite()`/`aiLabel()`/`portfolioAction()`/Screener เลยแม้แต่จุดเดียว** ถอดปุ่มพวกนี้ออกพรุ่งนี้ได้โดยไม่กระทบระบบที่ใช้งานจริงเลย

### สถาปัตยกรรม
```
buildForwardTrainingSetByDays(days, tolerance=1)  — สร้าง (X,Y) จาก "วันปฏิทินจริง" (ต่าง
  จาก buildForwardTrainingSet(horizon) เดิมที่นับ "จำนวนรอบ/session" ซึ่งมีปัญหาเวลาไม่คงที่
  เหมือน SlopeRec เดิม) มี tolerance ±1 วันกันวันหยุดตลาด/ข้อมูลขาด
       ↓
trainForwardModel(days) — เทรนจาก 5 feature "point-in-time ปลอดภัย" เท่านั้น
  (RSI, Heat, pct50, EMAMom, Score) — ตัด GroupRS/Slope ออกโดยเจตนา เพราะสองตัวนั้น
  คำนวณจากข้อมูล "ตอนนี้" เท่านั้น ไม่มีเวอร์ชันย้อนเวลา เอามาจับคู่กับแถวอดีตจะเกิด
  look-ahead bias (เอาอนาคตมาอธิบายอดีต) — เก็บผลใน mlForwardModels{7:{},14:{}} แยกจาก
  mlWeights/mlThresholds เดิมเด็ดขาด
       ↓
renderForwardConvergence() — คำนวณคะแนนของทุกตัว ณ ปัจจุบันด้วย weight ที่เทรนไว้ เทียบ
  ทิศทาง (บวก/ลบ) ระหว่าง 7D กับ 14D — 3 หมวด: ✅ตรงกัน/❌ขัดกันจริง/⚪ไม่มีสัญญาณ
  (ทั้ง 2 ค่า |score|<0.05 ไม่นับขัดกัน กันเคส noise ใกล้ศูนย์ เช่น AVGO/IONQ)
       ↓
saveForwardSnapshot(rows) — INSERT อัตโนมัติทุกครั้งที่กด Convergence เข้า ml_config
  (config_type='forward-snapshot') ใช้ pattern เดียวกับ exportMLConfig() เป๊ะ (ไม่ทับ
  ของเก่า มี trigger ปิด is_active ของแถวเก่าแต่ไม่ลบ) เก็บไว้เทียบผลจริงในอนาคต
```

**ต้องรัน SQL เพิ่มก่อนใช้ auto-save (ครั้งเดียว):**
```sql
ALTER TABLE ml_config DROP CONSTRAINT ml_config_config_type_check;
ALTER TABLE ml_config ADD CONSTRAINT ml_config_config_type_check
  CHECK (config_type IN ('value','backend','live','value-live','pricerange','forward-snapshot'));
```

### ผลทดสอบจริง (2026-09-10)
- Sample: 7D=684-722, 14D=532-570 (ผ่านเกณฑ์ ≥50 ขาดลอย, เฉลี่ยคลาดเคลื่อน 0.32-0.36 วัน = ข้อมูลต่อเนื่องดี)
- ทุก feature ทั้ง 2 horizon เป็นลบสม่ำเสมอ (ไม่สลับทิศเหมือนโมเดลเดิมที่เคยมีบั๊ก) — **แต่ต้องตีความด้วยความระวัง: ข้อมูล ~2 เดือนที่มีอยู่คือ "หมียาว + peak สั้นๆ" เป็นหลัก ผลลบอาจสะท้อนช่วงข้อมูลนี้ ไม่ใช่กฎสากล เหมือนบทเรียนเดิมกับโมเดลหลัก**
- RSI อ่อนกำลังเมื่อ horizon ยาวขึ้น (7D w=-16.5% → 14D w=-2.0%) ส่วน EMAMom แรงขึ้น (7D -21.1% → 14D -33.6%) — สมเหตุสมผล (RSI เป็นตัวชี้วัดระยะสั้น)
- **Convergence: ✅ตรงกัน 33 · ❌ขัดกันจริง 0 · ⚪ไม่มีสัญญาณ 5 (จาก 38 ตัว)** — ไม่มีตัวไหนขัดกันจริงเลย

### บั๊กที่เจอและแก้ระหว่างทาง
`pctArr` ใน `trainForwardModel()` copy สูตรจาก `autoTrain()` เดิมมาผิด — ใช้ `ema20/ema50` ซ้ำกับ EMAMom เป๊ะ (ป้ายชื่อ "pct50" แต่ไม่ได้วัด pct50 จริง) แก้เป็น `r.pct_ema50` ตรงๆ **หมายเหตุ: `autoTrain()`/`mlStep3()` เดิมที่ใช้งานจริงยังมีบั๊กเดียวกันอยู่ ไม่ได้แก้ เพราะคนละขอบเขต ต้อง retrain ทั้งระบบใหม่ถ้าจะแก้ตรงนั้น**

### ขั้นตอนใช้งานที่แนะนำ
กด **Train Forward 7D/14D** ก่อนเสมอทุกครั้งที่เปิดแอปใหม่ (weight เก็บใน memory ของ browser เท่านั้น หายเมื่อรีเฟรช) แล้วค่อยกด **Forward Convergence** — **วันละ 1 ครั้งพอ ไม่ต้องกดถี่ระหว่างวัน** (ข้อมูลพื้นฐานยังไม่ทันเปลี่ยนมากในไม่กี่ชั่วโมง) แต่ต้องกดสม่ำเสมอทุกวันเพื่อสะสม snapshot ให้พอเทียบผลจริงได้ในอนาคต (7-14 วันข้างหน้า)

### ขั้นต่อไปที่ยังไม่ทำ (ต้องรอเวลา)
รอสะสม snapshot 7-14 วัน → ดึงย้อนหลังมาเทียบราคาจริง → ได้ accuracy ที่แท้จริงครั้งแรกของโมเดลนี้ (ไม่ใช่แค่ "สอดคล้องกันเอง") → ถ้า accuracy น่าเชื่อกว่าเดิมมาก ค่อยคุยเรื่องเชื่อมเข้าการตัดสินใจจริง

---

## 🐛 Leader Tag Timing Bug (2026-09-10) — บั๊กที่ 3 ของสาย "Leader Tag ขัดกับ Action"

หลังแก้บั๊ก sign-direction (10 จุด) และแก้ inline logic ที่ตกหล่น (2 จุด) เมื่อคืนก่อน วันถัดมาเจอ **MU/SNDK/STX/WDC (กลุ่ม Memory, GroupRS บวก) ขึ้น 🔴 SELL ใน Leader Tag แต่ Action เป็นแค่ PARTIAL SELL** — คนละบั๊กจากที่เคยแก้ทั้งหมด

**ต้นตอ:** `leaderTag` คำนวณจาก `calcCompositeScore(baseResult)` **ก่อนรู้จัก group** (ใน `fetchData()`, ไม่มี GroupRS ประกอบ) — ส่วน `Action`/`ML Score` คำนวณจาก `calcCompositeScore(d, group)` **หลังรู้ group แล้ว** (ในลูปจัดกลุ่ม) `d.action`/`d.signal` ถูกคำนวณซ้ำด้วยค่าที่ถูกต้องทีหลัง **แต่ `d.leaderTag` ไม่เคยถูกคำนวณซ้ำเลย** ค้างค่าเก่าที่ไม่มี GroupRS ตลอดมา

**แก้โดยย้ายการคำนวณ `leaderTag` ทั้งหมดไปทำครั้งเดียว ณ จุดที่มี `comp6D`** (ถูกต้อง รู้ group แล้ว) ลบการคำนวณ premature ออกจาก `fetchData()` ทั้งหมด รับประกันว่า Leader Tag กับ Action มาจากค่า composite เดียวกันเป๊ะ ไม่มีทางขัดกันได้อีก — ระวังไว้ตอนแก้: เกือบเผลอเพิ่ม `else` branch ทับ 🏆LEADER/🚀MOMENTUM tag เดิมในโซนกลางโดยไม่ตั้งใจ (เกินขอบเขตบั๊กจริง) ตรวจพบและแก้คืนก่อนส่ง

**บทเรียนสำหรับบั๊กสาย "ค่าเดียวกัน คำนวณคนละที่":** ไม่ใช่แค่ทิศทาง (sign) ที่ต้องตรวจ — ต้องเช็ค**จังหวะ/context ที่ใช้คำนวณ**ด้วยว่าครบถ้วนเหมือนกันไหม ค่าที่ควรจะเท่ากันแต่มาจากฟังก์ชันเรียกคนละจุดคนละเวลา มีความเสี่ยงหลุด sync กันได้เสมอ

---

## Troubleshooting

- Save/Refresh error บนมือถือ → มักเป็นเน็ตสะดุด เช็ค DB ก่อน
- แก้โค้ดไม่เห็นผล → Private tab ใหม่
- ราคาไม่ตรงข้ามแอป → เช็ค `run_seq`
- ก่อนแก้ไฟล์ → ขอไฟล์ปัจจุบันเสมอ
- ส่ง config ใหม่ไป Supabase แล้ว error `violates check constraint` → เช็คค่าที่ส่งอยู่ใน constraint ที่อนุญาตหรือยัง (`SELECT pg_get_constraintdef(oid) FROM pg_constraint WHERE conname='...'`)
- **แก้ scale/ทิศทางของค่าไหนก็ตาม → grep หา field name ทั้ง 2 ไฟล์ก่อนเสมอ** อย่าไล่ตามแค่ชื่อฟังก์ชันที่จำได้ — logic อาจถูกเขียนซ้ำแบบ inline ไม่มีชื่อฟังก์ชันแยก
