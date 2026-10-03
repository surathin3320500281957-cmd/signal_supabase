# Signal Matrix — คู่มือระบบฉบับเต็ม (อัพเดตล่าสุด)

> เอกสารนี้สรุปทั้งระบบ (frontend + backend) ให้ chat ใหม่เริ่มงานได้ทันทีโดยไม่ต้องอธิบายซ้ำ
> วางไฟล์นี้ไว้ที่ root ของ repo `signal_supabase` เป็น `README.md`
> **อัพเดตรอบนี้ (สำคัญที่สุด, 2026-09-21/22):** เพิ่ม Convergence/Divergence 4 ภาวะ (แท็บ Recovery Forecast) + Forward Model 1D + Price Timeline คาดการณ์ + **⚙️ Config Control panel** (แท็บใหม่ที่ backend — ผูก threshold 5 ตัวเข้าโค้ดจริงครบแล้วตามภาวะตลาด) + แก้บั๊ก order/limit ร้ายแรงที่ backend (pattern เดียวกับ frontend) + เพิ่มหัวข้อ **Supabase schema** และ **Deployment (Cloudflare/GitHub Pages)** เต็มรูปแบบ — ดูหัวข้อ "🗳️ Convergence/Divergence", "🔮 Forward Model 1D", "⚙️ Config Control", "🗄️ Supabase", "☁️ Deployment" ด้านล่าง
> **อัพเดตก่อนหน้า (2026-09-18/19):** เพิ่มแท็บใหม่ **"🌊 จุดเปลี่ยนรอบ" (Turning Point Monitor)** ฝั่ง frontend — รวม Gate + Breadth Saturation (2 แท่ง) + Divergence Trend คำนวณสดจาก Supabase ล้วนๆ ไม่พึ่ง backend เลย + แก้บั๊ก dedupe หลาย session/วัน 3 จุด + Forward Model แยกป้าย 4 กรณี (++/--/+-/-+) พร้อมปุ่มเช็ค accuracy แยกตามคู่สัญญาณ
> **อัพเดตก่อนหน้า:** แก้บั๊ก sign ของ weight เสร็จสมบูรณ์ทั้งวงจร (คำนวณ weight + กลับทิศทางเปรียบเทียบใน 10 จุด) — ก่อนหน้านี้แก้แค่ครึ่งเดียวแล้วเกิดบั๊กใหม่ (STX ร้อนสุดกลับได้ BUY, ONDS เย็นสุดกลับได้ SELL) วันนี้แก้ครบวงจรแล้วยืนยันด้วยข้อมูลจริง
> **อัพเดตล่าสุด (2026-09-10):** เพิ่ม Fear Streak Badge (มิติใหม่ ไม่แตะของเดิม) + สร้างระบบ ML Forward Model 7D/14D ทดลองคู่ขนาน (ดูหัวข้อ "ML Forward Model" — ตอนนี้ผ่านเกณฑ์ >55% แล้ว ดูหัวข้อ Turning Point Monitor) + แก้บั๊ก Leader Tag timing (บั๊กที่ 3 ของสาย Leader Tag ขัดกับ Action)

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
| **Frontend** | Cloudflare Workers | `index_front.html` | Dashboard, ML Pick, Ranking, Screener, **Portfolio (จุดตัดสินใจ/ลงมือขายจริง)**, Train Live, **🌊 จุดเปลี่ยนรอบ (คำนวณสดล้วนๆ ไม่พึ่ง backend)** |

ทั้งคู่ต่อ Supabase project เดียวกัน (ID `dhnlnvppveotthhxkcdu`)

---

## Frontend — Screener (หัวใจของการตัดสินใจ) — 6 preset

| ปุ่ม | เงื่อนไข | ใช้เมื่อ |
|---|---|---|
| 🟢 พื้นฐาน | Trend UP, Conf A/B, Score≥5, Upside≥15, GroupRS≥0, สอดคล้อง, **ConDiv = Bull Conv** (เพิ่ม 2026-10-03) | หาจุดเข้าเข้มสุด |
| 🟡 ปานกลาง | ผ่อน Score≥3, Upside≥10, **ConDiv = Bull Conv/Neutral** | หลักฐานแน่นสุดที่ยังผ่อนได้ |
| 🔴 สูง | ผ่อน Conf A/B/C, Upside≥5, GroupRS≥-2, **ConDiv = Bull Conv/Neutral** | ปลด Score/Convergence |
| 🔥 ร้อนเกิน | Trend UP, pct50≥8%, RSI≥70 | เตือนพิจารณาขาย |
| 🧊 ดิ่งหนักสะสม | Trend DOWN, pct50≤-5%, RSI≤30 | เตือนพิจารณาตัดขาดทุน |
| 💎 เด้งยืนยันแล้ว | pct50≤-2% + ราคาเพิ่มขึ้นต่อเนื่อง 3 รอบติด (hardcode) | กันหุ้นร่วงต่อเนื่องไม่มีวี่แววเด้งติดอันดับ |

> 🆕 (2026-10-03) Screener มี **🧭 โหมดตลาด** (ปกติ / ↔️ ไซด์เวย์) และ chip **📡 CONDIV** — ดูหัวข้อ "🧭 Screener โหมดตลาด + ตัวกรอง ConDiv" ด้านล่าง

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
| **Breadth/Divergence นับเกินจำนวน symbol จริงมาก (2026-09-18)** | ทุกครั้งที่ Refresh INSERT แถวใหม่เข้า `stock_signals` เสมอ ไม่ทับของเดิม — query ตรงๆ นับซ้ำทุก session/วัน | dedupe ด้วย `run_seq` (group by symbol_id+signal_date เก็บ run_seq สูงสุด) ก่อนคำนวณเสมอ — แก้ 3 จุด (`computeBreadthSaturation`, `computeDivergenceHistoryLive`, `buildPriceIndex`) |

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
- ML Forward Model 7D/14D — **ผ่านเกณฑ์แล้ว** (7D accuracy 66% จาก 38-76 ครั้งที่เช็คได้จริง, เกณฑ์ ≥55-60%) แต่ n ยังไม่มากพอมั่นใจ 100% — เริ่มใช้เป็นตัวช่วยกรอง/ยืนยันสัญญาณอื่นได้แล้ว ยังไม่ควรตัดสินใจเทรดเดี่ยวๆ ดูหัวข้อ "🌊 Turning Point Monitor" สำหรับการต่อยอดล่าสุด
- Accuracy แยกตามคู่สัญญาณ (4 quadrant) — สร้างเครื่องมือแล้ว รอสะสม n≥5 ต่อกลุ่มถึงจะฟันธงได้ว่า `-/+` พิเศษกว่า `+/-` จริงไหม
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

## 🔮 Recovery Forecast + 📊 Forward Model แยกเป็น 2 แท็บ (2026-09-11) — โครงสร้างสุดท้าย

**Forward Model** ย้ายออกจาก ML Analyzer มาเป็นแท็บแยก (เหตุผล: มีหลายขั้นตอน Preview→Train→Convergence→Snapshot Trend→Check Accuracy ปนกับโมเดลจริงแล้วรกเกินไป) — ตอนนี้มี **6 แท็บ**: Signal Matrix / Portfolio / สรุป Stock ล่าสุด / 🧠 ML Analyzer / 📊 Forward Model / 🔮 Recovery Forecast

**Recovery Forecast** — ต่อยอดจากไอเดีย "อีกกี่วันสัญญาณหมดแรงจะเกิดจริง" ผสม `detectRSIDivergence()` (แบ่งช่วง 20 วันเป็น 2 ครึ่ง เช็ค price lower-low + RSI higher-low) + `calcHistoricalRecoveryStats()` (สแกนทุกครั้งในอดีตที่ RSI≤30→ฟื้น>50 จริง หา median/average) — ตารางกดเรียงคอลัมน์ได้ (Symbol/Group/Price/RSI/Divergence)

**ผลทดสอบจริง (15 ก.ย.):** สถิติสะสม 72 ครั้ง median 3 วัน เฉลี่ย 4.1 วัน — **แยก "oversold ธรรมดา" ออกจาก "มี divergence จริง" ได้ชัดเจน** (เช่น 7/38 ตัวมี divergence ขณะ 31 ตัวแค่ RSI ต่ำเฉยๆ) ตัวเลขประมาณเวลาเป็นค่ารวมทั้งตลาด (ไม่แยกรายหุ้น/กลุ่ม เพราะ sample จะเล็กเกินไป) — ตัดสินใจคงไว้แบบนี้ ใช้ดูประกอบกว้างๆ พอ

---

## 📡 Market Signal Log (2026-09-15) — เชื่อม Fear/Breadth/Divergence

ต่อยอดคำถาม "Divergence เยอะขึ้น = สัญญาณกระทิงไหม" — backend คำนวณ Fear-equivalent (`avg(smartRank)/10*100`, สูตรเดียวกับ frontend) + Breadth (`trend` field) + Divergence % เองได้จากข้อมูลที่มีอยู่แล้ว ไม่ต้องพึ่ง frontend บันทึกอัตโนมัติทุกครั้งที่กด (config_type='market-signal-log', ต้องรัน SQL เพิ่มก่อนใช้เหมือน forward-snapshot)

**🐛 บั๊กที่เจอ:** `allRows[i].trend` เก็บเป็น `'🟢 UP'` (มี emoji) ไม่ใช่ `'UP'` ดิบๆ — เทียบ `===` ตรงๆ ได้ Bullish/Bearish=0 เสมอ แก้เป็น `.includes()` แทน **บทเรียน: ต้อง grep เช็ค field ก่อนสมมติรูปแบบข้อมูลเสมอ แม้จะดูเหมือนชัดเจนแค่ไหนก็ตาม**

**ยังไม่มีข้อสรุปเรื่อง correlation จริง** (ข้อมูลมีแค่ไม่กี่วัน) — เป็นสมมติฐานที่กำลังสะสมข้อมูลเช็คต่อเนื่อง

---

## 🎯 TP Filter ใน Screener (2026-09-15) — จุดค้นพบใหม่จากเคส INTC

**ที่มา:** INTC ผ่าน preset 🟢 พื้นฐาน (GroupRS=+1.04 เฉียดขอบ ≥0) แต่ TP Confidence ขึ้น 🔴 อ่อนแรงตั้งแต่แรก — เข้าไม้ทดสอบแล้ว SL Trigger จริงตามที่ TP เตือนไว้ (ภายในไม่กี่ชม.) **เหตุผล:** Screener เดิมดูแค่ค่าปัจจุบัน ไม่มี "ความจำ" อดีต (peak ที่เคยร้อนมาก่อน) TP มีมิตินี้อยู่แล้ว

**แก้:** เพิ่ม `excludeTpWeak` เข้า 4 preset "จุดเข้า" เท่านั้น (🟢🟡🔴💎) — ตัดตัวที่ `slTp.tpLabel === '🔴 อ่อนแรง'` ออก **ไม่แตะ 2 preset "จุดเตือน"** (🔥🧊 เป็นคำเตือนอยู่แล้วในตัว ไม่ต้องมี TP ช่วยตีความซ้ำ) เพิ่ม checkbox ใน filter panel ให้เปิด/ปิดเองได้ด้วย **ตัดสินใจไม่ใส่ SL เข้าเกณฑ์** (SL คือสัญญาณ "ขายจริง" ใช้ตอนมีตำแหน่งแล้ว ไม่ใช่ตอนหาจุดเข้า — ผิดจุดประสงค์ถ้าเอามากรอง)

---

## 📚 บทเรียนสำคัญจากการเทรดจริง (Sept 10-16)

1. **Timeline หมี/กระทิงจาก median ในอดีต + สัญญาณแนวต้านไม่ผ่าน → ทำนายจังหวะเข้าหมีได้แม่น** (ขาย port ก่อนหมีจริง 1 วันพอดี จาก breadth+แนวต้านไม่ผ่าน แม้ไม่รู้ผลจริงล่วงหน้า)
2. **Fear Index หลอกได้ (แกว่งแรงใน noise) — Fear Streak Badge (นับก้าวต่อเนื่อง ไม่สนขนาด) กรอง noise ได้แม่นกว่าดูค่าดิบเฉยๆ** พิสูจน์ซ้ำ 2 รอบ (45→50→ร่วง วันที่ 9, 29→44→ร่วง วันที่ 14)
3. **สัญญาณรายตัว (Screener ผ่านเกณฑ์) ไม่ชนะบริบทมหภาค (หมียังไม่ถึง median)** — เคส INTC เข้าทั้งที่รู้ว่าไม่ควร (ทดสอบ) → ขาดทุนตามที่ TP เตือนไว้ตั้งแต่แรก
4. **⚠️ Event risk (FOMC/CPI ฯลฯ) ทำให้สัญญาณ technical ทั้งหมดใช้ไม่ได้ชั่วคราว** — INTC ที่เพิ่ง SL Trigger ไปกลับ +6% หลัง FOMC ประกาศผล (dovish) วันถัดมา **ไม่ใช่ระบบผิด แค่ event risk ที่ไม่มีใครรู้ล่วงหน้าได้** บทเรียนคือ "ระวังถือข้ามคืนก่อน event ใหญ่" ไม่ใช่ "อย่าเชื่อ SL"
5. **Premarket เขียว/แดงทั้งกระดานแบบไม่มี technical รองรับ (Divergence ต่ำ) → เช็คปฏิทินเศรษฐกิจก่อนเชื่อ** อาจเป็นการรอข่าวไม่ใช่สัญญาณกลับตัวจริง

---

## 🌊 Turning Point Monitor (2026-09-18/19) — แท็บใหม่ฝั่ง frontend

**ที่มา:** จากบทสนทนายาวเรื่อง "สัญญาณเปลี่ยนรอบตลาด" — ผู้ใช้สังเกตว่า Divergence เดิม (มีอยู่แล้วใน Market Signal Log) ใช้ไม่ได้ตอนตลาดไปทางเดียวกันเกือบหมด (Divergence=0% เพราะไม่เหลือหุ้นให้สวนทาง) และ bull/bear count เป็นแค่ตัวยืนยันหลังเหตุการณ์ (coincident) ไม่ใช่ตัวเตือนล่วงหน้า (leading) — ต้องการสัญญาณที่เตือนได้**ก่อน**ทั้งฝั่งหมีถึงจุดหมดรอบ (ก่อนกระทิง) และกระทิงถึงจุดพีค (ก่อนหมี)

**สถาปัตยกรรม (สำคัญ — ยึดตามกฎข้อ 8):** คำนวณสดฝั่ง frontend ล้วนๆ จากข้อมูลที่มีอยู่แล้วใน Supabase (`stock_signals`, `market_regime`) **ไม่พึ่ง backend เลยแม้แต่จุดเดียว** — เหตุผลตรงตามที่ผู้ใช้ยืนยัน: "เป็นจุดที่ไม่ต้องพึ่ง back" เพราะเคยเจอปัญหา Market Signal Log (เก่าจาก backend) ต้องรอมีคนกดปุ่มที่ backend ก่อนถึงจะเห็นค่าล่าสุด ทำให้ไม่ตรงกับ Dashboard สดๆ

### 3 สัญญาณที่รวมกันในแท็บนี้

**1) Gate — ตำแหน่งในรอบ.** ใช้ `regimeCtx` ที่มีอยู่แล้ว (จาก `loadRegimeStats()`) ไม่สร้างสูตรใหม่ซ้ำ — โชว์ "กระทิง/หมีวันที่เท่าไหร่ เทียบ median" **สีของแถบไล่โทนต่อเนื่อง** (ไม่ใช่ 3 ช่วงคงที่แบบเดิม) จากคะแนนผสม:
```
compositeScore = (ตำแหน่งในรอบ % × 60%) + (breadth ที่ตรงทิศทางรอบ % × 40%)
  BULL → breadth ร้อน (HOT+EXTREME%)     — เตือนก่อนหมี
  BEAR → breadth เย็นจัด (oversold%)      — เตือนก่อนกระทิง (สมมาตร)
```
เหตุผลที่ต้องสลับตามทิศทาง: ตอนแรกใช้ breadth ร้อนเสมอไม่ว่ารอบไหน ทำให้เตือนได้แค่ "ใกล้จบขาขึ้น" ไม่เคยเตือน "ใกล้จบขาลง" เลย — ผู้ใช้ชี้ว่า "ก่อนขึ้นก็มีสัญญาณร่วงหนักสุดขีดเหมือนกัน" (capitulation ก่อนกระทิง) จึงแก้ให้สมมาตร 2 ทิศทาง

**2) Breadth Saturation — 2 แท่งแนวนอนตรงข้ามกัน.** ใช้เกณฑ์เดียวกับคอลัมน์ Heat ในตาราง Signal Matrix เป๊ะ (`mlConfig.heat_boundary`, ไม่ hardcode 70/30 แยกต่างหากเหมือนตอนแรกที่ทำผิดจนตัวเลขไม่ตรงกับตาราง):
- **แท่งบน (🔥):** 3 สี COOL/HOT/EXTREME — เตือนก่อนหมี
- **แท่งล่าง (🧊, ตรงข้าม):** OVERSOLD (RSI≤30) vs ปกติ — เตือนก่อนกระทิง (เพิ่มกลับเข้ามาหลังผู้ใช้ชี้ว่าตอนปรับให้ตรงกับ Heat ทำให้ตัวชี้วัด oversold เดิมหายไปโดยไม่ตั้งใจ)

ทั้งคู่มี percentile เทียบ 90 วันย้อนหลัง เตือนเมื่อ ≥85 — **satFlag ข้าม Gate ได้เลย** ถ้าฝั่งใดฝั่งหนึ่งเตือน (ไม่ต้องรอ Gate เปิดก่อน)

**3) Divergence Trend.** พอร์ตสูตรจาก backend (`detectRSIDivergence`, `computeMarketSignalSnapshot`) มาคำนวณสดฝั่ง frontend ทั้งชุด — ดึง `stock_signals` ย้อนหลัง 45 วัน (7 วันแสดงผล + เผื่อ lookback 20 วันสำหรับตรวจ divergence ของวันท้ายสุด)

### แถบ "LIVE" — ต้องเข้าใจ 2 ชั้นข้อมูลที่อัพเดตคนละจังหวะ
- **Dashboard Bullish/Bearish Count:** คำนวณสดในเบราว์เซอร์ทันทีทุกครั้งที่ Refresh (ไม่ต้องเขียน DB ก่อน)
- **แท็บนี้:** อ่านจาก `stock_signals` ซึ่งอัพเดตเฉพาะตอนกดปุ่ม **"💾 Save to Supabase"** เท่านั้น — อาจล่าช้ากว่า Dashboard ถ้ายังไม่มีใครกด Save หลังราคาขยับต่อ

เดิมติดป้าย "🔴 LIVE" ทำให้เข้าใจผิดว่าเท่ากับ Dashboard — แก้เป็น **"💾 ล่าสุดที่ Save"** พร้อมข้อความเตือนต่อท้าย **ลำดับใช้งานที่ถูกต้อง: กด Save to Supabase ที่ Dashboard ก่อนเสมอ แล้วค่อยมากด Refresh ที่แท็บนี้**

### 🐛 บั๊ก dedupe หลาย session/วัน (แก้แล้ว 3 จุด)
ทุกครั้งที่กด Refresh จะ **INSERT แถวใหม่เข้า `stock_signals` เสมอ ไม่ทับของเดิม** — 1 วันมีได้หลาย session ต่อ 1 symbol ถ้าไม่ dedupe ก่อนนับ ตัวเลขจะพองเกินจำนวน symbol จริง (เจอ 🐂63+🐻165=228 ทั้งที่มีแค่ 38 ตัว) แก้ด้วย `run_seq` (unix timestamp ตอน insert, มีอยู่แล้วในตาราง) — group by `symbol_id+signal_date` เก็บแค่ `run_seq` สูงสุด ก่อนคำนวณทุกครั้ง

จุดที่แก้: `computeBreadthSaturation()`, `computeDivergenceHistoryLive()`, `buildPriceIndex()` (ตัวหลังกระทบเบากว่า เพราะแค่เลือกราคาไม่ชัวร์ ไม่ได้นับซ้ำ) — **ถ้าเพิ่มฟังก์ชันใหม่ที่ query `stock_signals` ในอนาคต ต้องเช็ค dedupe ด้วย run_seq เสมอ ไม่งั้นซ้ำบั๊กนี้แน่นอน**

### Forward Model — แยกป้าย 4 กรณี (++/--/+-/-+) แทน agree/conflict แบบเดิม
ที่มา: ChatGPT เสนอไอเดียว่า `-/+ (7D ลบ, 14D บวก)` ควรมีความหมายพิเศษกว่า `+/- (7D บวก, 14D ลบ)` — ตรวจสอบด้วยข้อมูลจริงก่อนเชื่อ (ไม่ใช่เชื่อทฤษฎีเฉยๆ):
- ตาราง Forward Convergence (backend) แยก badge เป็น `🐂🐂 ตรงกันขึ้น` / `🐻🐻 ตรงกันลง` / `🐂🐻 +/-` / `🐻🐂 -/+` แทนแค่ ✅/❌
- Snapshot Trend (ชั้น 3) แยกนับ 4 กรณีต่อวันแทนนับรวม agree/conflict
- **ปุ่มใหม่ "🧭 Accuracy แยกตามคู่สัญญาณ (4 กรณี)"** — จัดกลุ่ม snapshot ตาม sign(7D)/sign(14D) เช็คความแม่นยำแยกอิสระ พร้อมเทียบ `-/+` vs `+/-` ตรงๆ ให้อัตโนมัติ สรุปเป็นภาษาคน (ใกล้เคียงกัน=ยังไม่มีหลักฐาน / ต่างกันจริง=บอกทิศที่ถูก) — **ต้องมี n≥5 ต่อกลุ่มถึงจะเชื่อผลได้**

**บทเรียน:** ไอเดียจากภายนอก (ChatGPT หรือแหล่งอื่น) ที่ฟังดูสมเหตุสมผลทางทฤษฎี ต้องพิสูจน์กับข้อมูลจริงของระบบนี้โดยเฉพาะก่อนเชื่อ — ระบบนี้แต่ละหุ้นมีนิสัยต่างกัน (เช่น SNDK แกว่งขึ้นซ้ำๆ ผ่านแนวต้าน vs CRWV ยังไม่เคยพิสูจน์ตัวว่ากลับตัวได้) กฎเดียวใช้กับทุกหุ้นไม่ได้เสมอไป

### บทเรียนสำคัญเรื่อง "ไม่ใช่ทุกข้อสังเกตต้องเป็นฟีเจอร์ใหม่"
ผู้ใช้สังเกตว่าขนาดรอบกระทิงกำลังอ่อนลง (median ดัชนีจาก ~10% เหลือ 7.8%) แต่ตัดสินใจ**ไม่**สร้างตัวชี้วัดใหม่ เพราะ ML Score (backend) และ FW Model (7D/14D) ควรสะท้อนแรงถดถอยนี้อยู่แล้วในตัวเลขของมันเอง — เพิ่มแท่งใหม่มาวัดเรื่องเดียวกันจะเป็นการนับซ้ำ (double-counting) ไม่ใช่มิติใหม่จริง **ตรงกับกฎข้อ 7 ที่ยึดมาตั้งแต่ต้น**

### ผลทดสอบจริงจากการใช้งาน (18-19 ก.ย.)
- Gate พลิกกลับไปกลับมาในเวลาไม่ถึง 1 วัน (หมีวันที่ 1 → กระทิงวันที่ 2) — ยืนยันว่า Gate เป็นแค่กรอบสถิติ ไม่ใช่คำทำนาย
- Divergence ไต่ขึ้นต่อเนื่อง 2 วันติด (0%→3%→5%) ตอนเข้าสู่หมีใหม่ — ตรงกับที่เคยพิสูจน์ตัวจากรอบก่อน (21%→24% ก่อนพลิกจริง) แต่ n ยังน้อยเกินฟันธง
- ผู้ใช้ตัดสินใจขาย RXRX ก่อน median จะ "ครบ" โดยใช้ Gate+Breadth ประกอบ แทนรอสัญญาณยืนยัน 100% — ตรงตามเจตนาการออกแบบว่าเป็นเครื่องมือช่วยตัดสินใจเร็วขึ้น ไม่ใช่ทำนายแม่นยำ

---

## 🗄️ Supabase — โครงสร้างฐานข้อมูล + บทเรียนสำคัญ (2026-09-22)

### ตารางหลักที่ใช้จริง
| ตาราง | ใช้ทำอะไร | เขียนจากไฟล์ไหน |
|---|---|---|
| `stock_signals` | ราคา/RSI/EMA/action ฯลฯ รายหุ้นรายวัน — ต้นทางของแทบทุกฟีเจอร์ | frontend (ปุ่ม Save to Supabase) |
| `market_regime` | Gate — วันที่เท่าไหร่ของรอบกระทิง/หมี (`regime_date`, `regime`, `bullish_ratio`) | frontend (upsert อัตโนมัติทุก Refresh) |
| `ml_config` | **key-value store กลาง** ใช้ `config_type` แยกประเภท — เก็บได้หลายอย่างในตารางเดียว | ทั้ง frontend และ backend |
| `symbols`, `groups`, `portfolio`, `live_quotes`, `trade_journal` | ข้อมูลตั้งต้น/พอร์ตจริง ไม่ใช่ snapshot รายวัน | ตามหน้าที่ |

### 🐛 บั๊กร้ายแรงที่สุดที่เจอทั้งโปรเจกต์ — `order=asc` + ไม่มี `limit`
**อาการ:** Breadth Saturation/ConDiv เห็นข้อมูลเก่ากว่าความจริงมาก ทั้งที่ Save สำเร็จแล้วจริงๆ
**สาเหตุ:** Supabase/PostgREST มี default row limit ต่อ request (มักเป็น 1000) — query เรียง `order=run_seq.asc` (เก่า→ใหม่) พอข้อมูลสะสมเกิน limit จะ**ตัดแถวใหม่สุดทิ้ง**เพราะอยู่ท้ายลำดับ ไม่ใช่ตัดแถวเก่าอย่างที่ควรจะเป็น
**แก้:** เปลี่ยนทุก query ที่ดึง `stock_signals` เป็น `order=run_seq.desc&limit=5000` เสมอ (เรียงใหม่→เก่า ถ้าโดนตัดจะตัดของเก่าแทน) — **เจอบั๊กนี้ซ้ำ 2 รอบ (frontend ก่อน, backend ทีหลัง) เพราะ pattern เดียวกันถูก copy ไปใช้คนละไฟล์โดยไม่เช็คซ้ำ**
**กฎกันซ้ำ:** ทุก query ที่มีโอกาสข้อมูลสะสมเยอะ (`stock_signals`, `ml_config` แบบ log) ต้องมี `order=...desc` + `limit=` ระบุชัดเจนเสมอ ห้ามปล่อย default

### `stock_signals` — UNIQUE constraint ตัวจริง (เช็คแล้วยืนยันด้วย SQL)
```sql
stock_signals_unique_per_save   UNIQUE (symbol_id, signal_date, session, created_at)
```
**`created_at` อยู่ใน key ด้วย** — แปลว่าแทบไม่มีทางชนกันเลย (Save กี่ครั้งก็ insert แถวใหม่ได้เสมอ ไม่ทับกัน) **เคยเข้าใจผิดคิดว่า `ignore-duplicates` เป็นสาเหตุที่ข้อมูลไม่อัพเดต** (สมมติฐานผิด) ก่อนจะรู้ว่าสาเหตุจริงคือบั๊ก order/limit ด้านบน — **บทเรียน: ก่อนเชื่อสมมติฐานเรื่อง "ข้อมูลหาย" ให้เช็คข้อมูลจริงในฐานข้อมูลด้วย SQL ตรงๆ ก่อนเสมอ** อย่าเดาจากพฤติกรรมที่เห็นในแอปอย่างเดียว

### `ml_config` — CHECK constraint ต้องอัปเดตทุกครั้งที่เพิ่ม `config_type` ใหม่
คอลัมน์ `config_type` มี CHECK constraint ล็อครายชื่อที่อนุญาตไว้ (`ml_config_config_type_check`) — ถ้าใช้ค่าใหม่ที่ไม่อยู่ในรายการ **INSERT จะ error ทันที** (`violates check constraint`) ต้องแก้ที่ SQL Editor บน Supabase โดยตรง (Postgres ไม่มีคำสั่งแก้ CHECK ตรงๆ ต้อง DROP แล้ว ADD ใหม่ทั้งก้อน ไม่กระทบข้อมูลเดิม):
```sql
ALTER TABLE ml_config DROP CONSTRAINT ml_config_config_type_check;
ALTER TABLE ml_config ADD CONSTRAINT ml_config_config_type_check
  CHECK (config_type = ANY (ARRAY['value','backend','live','value-live','pricerange',
    'forward-snapshot','market-signal-log','condiv-log','regime-thresholds']::text[]));
```
**รายชื่อ `config_type` ที่ใช้จริงตอนนี้ (9 ตัว):** `value`, `backend`, `live`, `value-live`, `pricerange`, `forward-snapshot`, `market-signal-log`, `condiv-log`, `regime-thresholds`
**กฎกันพลาด:** ทุกครั้งที่จะเพิ่ม `config_type` ใหม่ในโค้ด ต้องรัน `ALTER TABLE` เพิ่มชื่อเข้า constraint ก่อนเสมอ ไม่งั้น Save จะ error เงียบๆ (เจอปัญหานี้ 2 รอบติด — ConDiv และ Config Control panel)

**วิธีเช็ค constraint ปัจจุบันแบบไม่โดนตัดทอน (ข้อความยาวจะโดน UI ตัด):**
```sql
SELECT unnest(regexp_matches(pg_get_constraintdef(oid), '''([^'']+)''', 'g')) AS allowed_value
FROM pg_constraint WHERE conname = 'ml_config_config_type_check';
```

---

## ☁️ Deployment — Cloudflare Worker (frontend) + GitHub Pages (backend)

| | Frontend | Backend |
|---|---|---|
| **โฮสต์ที่** | Cloudflare Workers | GitHub Pages |
| **URL** | `frontsupabase.surathin3320500281957.workers.dev` | `surathin3320500281957-cmd.github.io` |
| **Repo/ที่เก็บไฟล์** | Cloudflare dashboard โดยตรง | GitHub repo ชื่อ `signal_supabase` |
| **ชื่อไฟล์จริงบน repo** | — | **`index.html`** (ไม่ใช่ `index_backend.html`! ชื่อ `index_backend.html` เป็นแค่ชื่อที่ใช้เรียกกันในบทสนทนาเพื่อแยกจาก frontend เท่านั้น) |
| **วิธี deploy** | อัพโหลดผ่าน Cloudflare dashboard | **Upload files** ผ่านหน้า GitHub Code (ไม่ใช้ copy-paste เนื้อไฟล์ยาวๆ เพราะเสี่ยงตัดทอนบนมือถือ) → Commit → GitHub Actions build อัตโนมัติ |
| **เช็คสถานะ deploy** | Cloudflare dashboard → Deployments | GitHub repo → แท็บ Actions → "pages build and deployment" |

### 🐛 บั๊กที่ไม่ใช่บั๊ก — cache ล้าหลัง deploy
ทั้ง Cloudflare และ GitHub Pages มี edge cache ที่บางทีไล่ตามไฟล์ใหม่ไม่ทันทันที (แม้ deploy status ขึ้น "Success" แล้ว) **เจอปัญหานี้ซ้ำหลายรอบตลอดทั้งโปรเจกต์**
**วิธีเช็คให้ชัวร์ก่อนสรุปว่าโค้ดผิด:**
1. เช็คก่อนว่า deploy สำเร็จจริง (Actions log / Cloudflare dashboard)
2. เปิดเว็บใน **Incognito/Private tab ใหม่** (กัน browser cache)
3. ถ้าเป็นไปได้ ลองคนละเบราว์เซอร์ไปเลย (เช่น Safari สดๆ)
4. ถ้ายังไม่ขึ้นทั้งที่ทำครบ → รอ 1-2 นาทีแล้วลองใหม่ (edge cache บางทีต้องใช้เวลาไล่ตามจริงๆ)

### 🐛 เคสพิเศษ — deploy ไฟล์เก่าทับของใหม่โดยไม่ตั้งใจ
เคยเกิดขึ้นจริง: ผู้ใช้อัพโหลดไฟล์ผิดเวอร์ชันทับของที่เพิ่งพัฒนาไปเมื่อวันก่อน (สังเกตจาก UI หน้าตาเปลี่ยนไปเป็นเวอร์ชันเก่าทั้งที่ deploy สำเร็จ) — **วิธีกู้คืนที่ได้ผล:** ให้ AI ส่งไฟล์เวอร์ชันล่าสุดที่เก็บไว้ในเซสชันกลับมาให้ใหม่ แล้ว deploy ทับอีกรอบ ไม่ต้องงมหาผ่าน git history ถ้า AI ยังมีไฟล์ถูกต้องอยู่ในมือ

---

## 🗳️ Convergence/Divergence — 4 ภาวะตลาด (2026-09-21/22, แท็บ Recovery Forecast)

**ที่มา:** อยากรู้ในแต่ละภาวะตลาด (กระทิง/หมี) มีหุ้นกี่ตัวที่ "ยืนยัน" (Convergence) กับกี่ตัวที่ "เริ่มขัดแย้ง" (Divergence) — คล้ายนับโหวตรายวัน ต่อยอดจาก Divergence ตัวเดิมที่วัดได้แค่ทิศทางเดียว (หมี+สัญญาณขึ้น) ให้ครบทั้ง 4 การผสมทิศทาง

**นิยาม:**
```
"ตลาด" (marketDir) = ราคา vs EMA50 รายตัว (นิยามเดียวกับ Gate/Dashboard — พิสูจน์แล้วว่าแม่น
                       เคยลองใช้ EMA20 vs EMA50 ก่อน แต่ตัวเลขไม่ตรงกับ Dashboard เลย เปลี่ยนกลับ)
"สัญญาณ" (signalDir) = ทิศทาง RSI เฉลี่ยครึ่งหลัง vs ครึ่งแรกของ 20 วันล่าสุด
                        ต้องต่าง ≥2 จุด (RSI_NOISE_THRESHOLD) ถึงนับว่ามีนัยสำคัญ ไม่งั้นเป็น Neutral

4 ภาวะ:
  ตลาดขึ้น + สัญญาณขึ้น → Bullish Convergence  (ยืนยันขาขึ้น)
  ตลาดขึ้น + สัญญาณลง   → Bearish Divergence   (เตือนใกล้จบขาขึ้น)
  ตลาดลง   + สัญญาณลง   → Bearish Convergence  (ยืนยันขาลง)
  ตลาดลง   + สัญญาณขึ้น → Bullish Divergence   (เตือนใกล้จบขาลง — ตัวเดิมที่มีอยู่แล้วก่อนหน้านี้)
```

**⚠️ บทเรียนเรื่องตำแหน่ง:** ตอนคุยกันตัดสินใจย้ายจาก Recovery Forecast ไป ML Analyzer แต่**โค้ดจริงไม่เคยถูกย้ายตาม** ยังอยู่ Recovery Forecast เดิม — ทำให้เสียเวลาหากันนานมาก **กฎกันพลาด: หลังคุยตัดสินใจ "จะย้ายไปไว้ตรงไหน" ต้องเช็คโค้ดจริงว่าทำตามจริงหรือยัง ก่อนบอกตำแหน่งให้ผู้ใช้ตามหา**

**ยอดรวมกระทิง/หมี:** `bullTotal = bullish_convergence + bearish_divergence`, `bearTotal = bearish_convergence + bullish_divergence` — ถ้าต่างจากยอด Dashboard (ที่นับจากราคาล้วนๆ ไม่สนสัญญาณ) แปลว่ามีหุ้นที่ Neutral (สัญญาณ RSI ยังไม่ชัด) ปนอยู่ ไม่ใช่บั๊ก — ยิ่งช่องว่างกว้าง ยิ่งบอกว่าตลาดขึ้น/ลงแบบ "ไม่มีแรงหนุนชัดเจน"

---

## 🔮 Forward Model 1D + Price Timeline คาดการณ์ (2026-09-21/22)

**ต่อยอดจาก Forward Model 7D/14D เดิม** — `trainForwardModel(days)` เป็น generic function รับ `days` เท่าไหร่ก็ได้อยู่แล้ว เพิ่ม `trainForwardModel(1)` ก็ใช้งานได้ทันทีไม่ต้องเขียนใหม่

**ลำดับการใช้งานที่ต้องรู้:**
```
1. กด "Train Forward 1D" ก่อนเสมอ (สร้าง mlForwardModels[1])
2. กด "Forward Convergence 7D vs 14D" (ปุ่มเดิม) — ปุ่มนี้แหละที่บันทึก s1 ลง snapshot
   (ไม่ใช่ปุ่ม ConDiv ตามที่เข้าใจผิดตอนแรก)
3. ทำซ้ำทุกวัน สะสม n≥20 คู่ข้อมูล ถึงจะเห็นตัวเลข % คาดการณ์จริง (ก่อนหน้านั้นเห็นแค่ทิศทาง ↑/↓)
```

**Regression:** ผ่านจุดกำเนิด (`b = Σxy/Σx²`, ไม่มี intercept) เหตุผล: คะแนนโมเดล=0 ควรแปลว่า return คาดหวัง=0 ด้วย ลด overfit ตอน sample น้อย — สูตรเดียวกับที่ใช้กับ 7D/14D มาก่อน

**บั๊กที่เจอและแก้แล้ว:** ตอน `n=0` (ยังไม่มี snapshot อายุครบ) `b=0` พอดี ทำให้ fallback ทิศทางเดิม (เช็ค `predReturn>=0`) โชว์ "↑" ผิดๆ ให้ทุกตัวเพราะ `0>=0` เป็นจริงเสมอ — แก้ให้ fallback ใช้เครื่องหมายดิบของ `s1`/`s7`/`s14` เอง (ตัวเดียวกับที่ badge quadrant ใช้) แทน และแยกกรณี `n=0` เป๊ะให้โชว์ "—" ตรงๆ

**ตาราง "+1D คาดการณ์ + ConDiv":** รวม 2 โมเดลเข้าด้วยกัน ไฮไลต์เหลืองอัตโนมัติเมื่อโมเดล +1D กับ ConDiv ขัดกันเอง (เช่น โมเดลทายขึ้นแต่ RSI เป็น Bearish Divergence) — ช่วยกรองหาเคสที่ "ราคาอาจขึ้นได้จริง แต่แรงขับเริ่มอ่อน ต้องระวัง"

---

## ⚙️ Config Control — แผงควบคุม threshold ตามภาวะตลาด (2026-09-22, แท็บใหม่ที่ backend)

**ที่มา:** threshold สำคัญกระจายอยู่คนละไฟล์คนละจุด ไม่มีที่เดียวดู/แก้ตามภาวะตลาด (หมี/กระทิง/ออกข้าง)

**5 threshold ที่รวมไว้ (ค่า Default = ค่าที่ระบบ hardcode มาแต่แรก ยืนยันจากโค้ดจริง):**
| Threshold | Default | อยู่ไฟล์ไหน |
|---|---|---|
| Gate: น้ำหนักตำแหน่งในรอบ (`wCycle`) | 0.6 | frontend |
| Gate: น้ำหนัก Breadth (`wBreadth`) | 0.4 | frontend |
| Breadth Saturation percentile เตือน | 85 | frontend |
| ConDiv RSI delta ขั้นต่ำ | 2 | backend |
| TP Confidence RSI ร่วงจาก peak (อ่อนแรง) | 5 | frontend |

**การตรวจจับภาวะตลาด:** อ่าน `market_regime` แถวล่าสุด → ถ้า `bullish_ratio` อยู่ 0.4-0.6 = **sideways** ไม่ว่า Gate จะบันทึกว่า BULL/BEAR ไว้ก็ตาม นอกนั้นใช้ค่า `regime` ตรงๆ (BULL→bull, BEAR→bear)

**สถานะ: ผูกเข้าโค้ดจริงครบทั้ง 5 ตัวแล้ว (🟢)** — วิธีผูกที่ใช้ (สำคัญ เพราะ 2 ไฟล์มีข้อจำกัดต่างกัน):
- ฟังก์ชันที่ต้องใช้ค่านี้เป็น **sync** (ไม่ใช่ async) แต่ต้องโหลดจาก Supabase (async) → แก้ด้วยการโหลดครั้งเดียวเก็บไว้ใน global variable (`activeRegimeThreshold`) ก่อนเรียกฟังก์ชัน sync พวกนั้น ไม่ต้องแปลงทั้งฟังก์ชันเป็น async (เสี่ยงน้อยกว่า กระทบน้อยกว่า)
- Frontend โหลดตั้งแต่ต้นของ `load()` หลัก **แบบไม่ await** (กันบล็อก UI) — ถ้า TP Confidence คำนวณเสร็จก่อนโหลดเสร็จ รอบนั้น fallback เป็น Default ชั่วคราว แล้วถูกต้องเองรอบ Refresh ถัดไป (ยอมรับ trade-off นี้เพื่อความเร็ว)
- **Fallback ปลอดภัยทุกจุด:** ถ้ายังไม่เคย Save ในแผง หรือโหลดจาก Supabase ไม่สำเร็จ ใช้ค่า Default เป๊ะ — พฤติกรรมเหมือนก่อนมี Config Control ทุกอย่าง ถ้าไม่มีใครไปแก้ค่า

**ป้ายยืนยันการทำงานจริง (ไม่ใช่แค่เชื่อคำพูด):** ทุกจุดที่ผูกแล้วจะมีบรรทัด `🟢 ใช้งานจริงแล้ว — ภาวะตอนนี้: ... (จาก ⚙️ Config Control)` โชว์ค่าที่ใช้จริง ณ ตอนนั้น + ตาราง Config Control เองมีคอลัมน์ "สถานะ" บอกว่าจุดไหนผูกแล้ว (🟢) จุดไหนยัง (⚪)

**วิธีทดสอบว่าผูกจริง:** ตั้งค่าให้สุดโต่งชัดๆ ในแผง (เช่น TP RSI drop = 100) → Save → Refresh → ดูว่าผลลัพธ์เปลี่ยนไปสมเหตุสมผลไหม (ไม่มีตัวไหนขึ้น "อ่อนแรง" เลยถ้า threshold สูงเกินจะแตะ) — เห็นผลต่างชัดเจน = ยืนยันว่าใช้งานจริง ไม่ใช่ UI ลอยๆ

---

## 📋 ConDiv ตารางรายตัว ใน "🌊 จุดเปลี่ยนรอบ" (2026-10-01, frontend)

**ที่มา:** แท็บจุดเปลี่ยนรอบเดิมสรุปแค่ภาพรวมตลาด (Breadth Saturation, Divergence History) ไม่เห็นรายตัว — เพิ่มตารางที่ 4 เข้าไป โชว์สถานะ ConDiv ของทุกหุ้นใน watchlist ณ ขณะนั้น พร้อมคอลัมน์สรุปรวม HOT/OVERSOLD ไว้ท้ายตาราง ให้กดเรียง column ได้แบบตาราง Screener

**คอลัมน์ (10 คอลัมน์, ชิดซ้ายทุกอัน — ต้องบังคับ `text-align:left` เพราะมี CSS กลาง `th,td{text-align:center}` ทับอยู่):**
Symbol, ราคา, Range 14D (reuse `priceRangeConfig` เดิมที่ใช้กับ TP/SL), ตลาด/สัญญาณ (ลูกศรตัวหนา เขียว/เหลือง/แดง), สถานะ ConDiv, ΔRSI, Heat, Group RS (reuse คอลัมน์ `group_rs` ใน `stock_signals` ตรงๆ ไม่คำนวณใหม่), ML Score (reuse `composite_score` ตรงๆ), สรุป (ข้อความสีตาม sentiment)

**หลักการสำคัญ:** ทุกคอลัมน์ reuse ข้อมูลที่ระบบคำนวณ/เก็บไว้แล้วจากจุดอื่น (Dashboard, TP/SL, Save to Supabase) ไม่มีจุดไหนคำนวณซ้ำหรือ query ข้อมูลใหม่หนักๆ — ดึงจาก `stock_signals` ครั้งเดียว (`order=run_seq.desc&limit=5000` ตามกฎ dedupe run_seq)

**ClassifyConDiv ฝั่ง front (`classifyConDivFE`)** — พอร์ตจาก backend ตรงๆ: `marketDir` จาก `price vs ema50` (ตรงกับนิยาม Gate/Dashboard), `signalDir` จากทิศทาง RSI เฉลี่ยครึ่งหลังเทียบครึ่งแรกของ lookback เทียบ `RSI_NOISE_THRESHOLD` (ดึงจาก Config Control, default 2)

**สี sentiment:** เขียว=ดี (bullish_convergence ปกติ, bullish/bearish_divergence ที่ oversold), แดง=แย่ (bearish_convergence ปกติ, อยู่โซน extreme ผิดทาง), เหลือง=กลางๆ, เทา=ข้อมูลไม่พอ (`not_enough_data`, <10 วัน)

---

## ⚖️ Slope (7th ML dimension) เพิ่มเข้า Front (2026-10-01)

**จุดเริ่มเรื่อง:** ผู้ใช้สังเกตจากภาพ ML Analyzer ของ backend ว่า weight รวม (`Total: 6.8%`) เป็นบวก แต่ `composite_score` ที่โชว์ใน front ติดลบแทบทุกตัว → ตรวจโค้ดแล้วพบว่า**front กับ backend ใช้คนละสูตรกัน**:
- Backend `composite()` (บรรทัด ~3242): 6 มิติ (RSI, Heat, pct50, EMAMom, Score, **Slope**) blend กับ `healthScore()` 85/15
- Front `calcCompositeScore()` (เดิม ก่อนแก้): 6 มิติเหมือนกันแต่ใช้ **GroupRS** แทน Slope และ**ไม่มี** health blend

ผลคือ front ขาด Slope ไปเลย ทำให้ net weight bias เพี้ยนจาก backend's +6.8% เหลือแค่ -1.7% (สูตรคำนวณ bias จาก magnitude ของ weight ที่ยังใช้งานจริง) — เป็นสาเหตุที่ `composite_score` ฝั่ง front ติดลบเป็นระบบ ไม่เกี่ยวกับพื้นฐานหุ้นจริง

**ทางเลือกที่คุยกัน:**
- **Option A (เลือกใช้):** เพิ่ม Slope เข้า front เฉยๆ ไม่เพิ่ม health blend ตาม (เพราะ front แยก `composite_score`/`health_score` ชัดเจนอยู่แล้วใน zone logic ปลายทาง `isHealthy`/`hasUpside`) — ML ยังเป็นแค่ "1 vote" เหมือนเดิม ไม่ให้ front หลงทางตาม ML ฝั่งเดียว
- Option B (ไม่เลือก): ทำ parity เต็มรูปแบบกับ backend (เพิ่มทั้ง Slope และ health blend)

**วิธีหา Slope โดยไม่ต้องยิง query หนัก (Option 1 ที่เลือก):** backend คำนวณ Slope จากประวัติหลาย session ที่มีอยู่แล้วใน DB (`trendSlope()`) แต่ front ไม่มีแบบนั้น จึง**จำลองด้วย localStorage** — เก็บ `quickScore` (สูตรเดียวกับ backend: `(rsiN*0.4 + pctN*0.4 + scrN*0.2) * 100`) ของทุก refresh ไว้ที่ key `quickscore_hist_<symbol>` (เก็บ 5 ค่าล่าสุด, ใช้ `pushHist()`/`getHist()` ที่มีอยู่แล้วในโค้ด) แล้วหา slope จาก `(ล่าสุด - แรกสุด) / จำนวนจุด` เหมือน backend เป๊ะ

**ข้อแตกต่างที่ตั้งใจ (ไม่ใช่บั๊ก):**
- `pctN` ของ front ใช้ fixed scale ±10% จาก EMA50 แทน `maxPct50` แบบ cross-sectional ของ backend เพราะ front ประมวลผลทีละ group ยังไม่รู้ universe เต็มตอนคำนวณ
- Slope ที่ได้ (-1..1) ถูก remap เป็น 0..1 (`(slpN+1)/2`) ก่อนคูณ weight เพื่อให้ scale เดียวกับอีก 6 มิติใน `calcCompositeScore()` เอง — backend ใช้ -1..1 ดิบๆ ตรงๆ เป็นคนละ convention กัน แต่ weight ตัวเลขที่ใช้ (`mlConfig.weights.Slope`) เป็นตัวเดียวกับที่ backend เทรนมา

**Cold-start:** ต้องมี snapshot อย่างน้อย 2 ครั้ง (2 refresh) ถึงจะเริ่มมีผล ก่อนหน้านั้น Slope = 0 เสมอ (ปลอดภัย เพราะ ML เป็นแค่ 1 โหวตใน 7 มิติ) — ไม่ต้องกังวลถ้าเพิ่งเปิดเครื่อง/browser ใหม่แล้วยังไม่เห็นผล

**ผลที่ยืนยันแล้วหลัง deploy:** composite_score บางตัวเริ่มเป็นบวกแล้ว ตรงกับที่คาดไว้ตอนวิเคราะห์ bias

---

## 🔘 ConDiv ปุ่ม Preset 5 ปุ่ม + ช่อง Range 14D แบบบอก % + ซ่อนคอลัมน์ Dashboard (2026-10-02)

**ที่มา:** ตลาดช่วงนี้ sideway (bull ratio ~55% อยู่ในแถบ 0.4–0.6) Screener ที่ออกแบบมาสำหรับตลาดมีทิศ (💎 เด้งยืนยันต้องขึ้นต่อเนื่อง 3 ครั้ง + TP confidence) จึงผ่านน้อย/มีแต่ตัวเสี่ยงสูง — เพิ่มปุ่ม hardcode 4 ปุ่มไว้บนตาราง ConDiv ใช้ข้อมูลเฉพาะในตารางนี้เท่านั้น แนวทางเดียวกับ Screener ("ตะแกรง" บน "แผนที่")

### 5 ปุ่ม (กดซ้ำ = ยกเลิก, แสดงจำนวนตัวที่ผ่านในวงเล็บ, ผล 0 ตัว = ข้อมูลอย่างหนึ่ง ไม่ใช่บั๊ก)
| ปุ่ม | เงื่อนไข (ทั้งหมดต้องผ่าน) | เรียงตาม |
|---|---|---|
| 🟢 Fresh Buy ปกติ | `bullish_convergence` + Heat cool + ΔRSI ≤ 8 + Range ≤ 60% + Group RS ≥ 0 | ΔRSI มาก→น้อย |
| 🟡 เสี่ยงกลาง | `bullish_divergence` + cool + RSI ≤ 50 + ΔRSI ≤ 8 + Range ≤ 40% + Group RS ≥ -1 + ราคาสูงกว่าจุดต่ำสุดของ 3 วันล่าสุด (`bounce3`) | Range ต่ำ→สูง |
| 🔴 เสี่ยงสูง | oversold (RSI ≤ 30) + (Bull Div หรือ Neutral) + Range ≤ 20% + ถ้า Bull Div ต้อง ΔRSI ≤ 8 | Range ต่ำ→สูง |
| 🔪 ช้อนมีดตก | `bearish_convergence` + oversold + Range ≤ 25% + ΔRSI ≥ -12 (ยังไม่ดิ่งแรงสุด) | ΔRSI มาก→น้อย |
| 🧱 แนวต้าน | แนวต้าน = High 14D (ตัวเดียวกับช่อง Range) · `distHigh` = (ราคา−high)/high×100 ≥ −3% (`CONDIV_RESIST_NEAR_PCT`) ครอบ 3 จังหวะในปุ่มเดียว: เกือบถึง / ถึง / ทะลุผ่านแล้ว · **ไม่ผูกกับ ConDiv state/Heat โดยตั้งใจ** (ดูแรงขับ/ความร้อนจากคอลัมน์อื่นประกอบ) | `distHigh` มาก→น้อย (ทะลุ/ใกล้สุดก่อน) |

- 🧱 เป็นปุ่มวัด "โซนราคาบน" คล้าย TP แต่คนละมุม (TP = แรงขึ้นอ่อนลงหรือยัง, 🧱 = ราคาอยู่ที่เพดานเดิม) — ข้อมูลย้อนหลังชี้ว่าตัวที่ขึ้นแรงสุดมักย่อ ใน sideway แนวต้านมักผลักกลับ ใช้ดู "ใครกำลังทดสอบเพดาน" ไม่ใช่สัญญาณซื้อไล่ · 2026-10-02 เห็น 15 จาก 38 ตัวอยู่ใกล้เพดานพร้อมกัน ไม่มีตัวไหนทะลุ (เกณฑ์ 3% อาจกว้างสำหรับหุ้นผันผวน)
- เกณฑ์ ΔRSI ≤ 8 (`CONDIV_FRESH_DELTA_MAX`) กับ ≥ -12 (`CONDIV_KNIFE_DELTA_MIN`) เป็น**ค่าเดา**ยังไม่ได้ทดสอบกับข้อมูลจริง — แก้ได้ที่เดียวตรงค่าคงที่ใกล้ตัวแปร `CONDIV_PRESETS`
- เข้าใจวงจร: ตัวหนึ่งเลื่อนปุ่มตามจังหวะ RSI — 🔪 (ราคาลง RSI ลง) → 🔴 (RSI หยุดลงที่ oversold) → 🟡 (ราคาเด้ง ยืนได้) → 🟢 แต่**ไม่ได้ผ่านครบทุกครั้ง** บางตัวหยุดแค่ 🔴 หรือกลับไป 🔪
- ΔRSI ใน 🔪 เป็นบวกไม่ได้**โดยนิยาม** (Bear Conv = RSI ลดเกิน noise ถ้า RSI เพิ่มจะกลายเป็น Bull Div แล้วย้ายไป 🔴) — ไม่ใช่บั๊ก
- ลำดับปุ่มบนจอคือ 🟢🟡🔴🔪 (ซ้าย→ขวา) ซึ่งสวนกับลำดับวงจรด้านบน ผู้ใช้ตัดสินใจคงไว้

### ช่อง Range 14D — บอก % ทั้งสองทาง (สีแดง = ฝั่ง low/ลง, เขียว = ฝั่ง high/ขึ้น)
- อยู่ในกรอบ: `↑ อีก +x.x% ถึง high` (เขียว) + `↓ อีก -x.x% ถึง low` (แดง) — ฐาน % = ราคาปัจจุบัน
- ทะลุ low: `⬇️ ทะลุ low -x.x%` (แดงตัวหนา) + บรรทัดอีกกี่ % ถึง high — ฐาน % ของ "ทะลุ" = ระดับ low/high เดิม (ตั้งใจให้ฐานต่างจาก "อีกกี่ %")
- ทะลุ high: `⬆️ ทะลุ high +x.x%` (เขียวตัวหนา) + บรรทัดอีกกี่ % ถึง low
- ราคาเท่า low/high พอดี (0%): `● ที่ low พอดี` / `● ที่ high พอดี` — หมายถึงเพิ่งทำจุดสุดขอบใหม่ ไม่ใช่ "ทะลุ" (ทะลุ = เกิน**ระดับที่ save ไว้**เท่านั้น)
- ตัวเลขช่วง `$low–$high (xx%)` เปลี่ยนเป็นสีเทากลาง (เดิม ≥80% แดง / ≤20% ฟ้า สวนกับโทนเขียว/แดงที่อื่นในตาราง)
- ทะลุ high เปลี่ยนจากส้มเป็นเขียว ตามคำขอให้สีสอดคล้อง (เขียว = ขึ้น)
- คำนวณทั้งหมดจาก `priceRangeConfig` เดิม ไม่ query เพิ่ม — ผู้ใช้ Save วันละหลายรอบ และใช้ range 14D

### สีสรุปให้สอดคล้อง
`bearish_convergence` + oversold เดิมเป็นเขียว (ดี) → เปลี่ยนเป็น**เหลือง (กลางๆ)** เพราะเขียวถูกมองว่า "ดี" แต่ตัวนี้ยังเป็นมีดตก (บรรทัด `bearish_convergence && oversold → 'good'` ใน `getConDivSentiment` ถูกลบ)

### Dashboard — ซ่อนคอลัมน์ Action และ News (UI อย่างเดียว)
ผู้ใช้ไม่ได้ใช้สองคอลัมน์นี้ ถอดจากตารางเพื่อเพิ่มพื้นที่ (`colspan` ของแถวกลุ่ม 18 → 17) **ข้างหลังไม่แตะ** — `d.action` ยังคำนวณและถูกบันทึกลง Supabase / export เหมือนเดิม

### ข้อค้นพบ: ML ทับ action Heat-EXTREME → TAKE PROFIT บน Dashboard
ใน `fetchData` กฎ Heat (RSI ≥ `extreme_min` = 80) ตั้ง `TAKE PROFIT` ก่อน แล้ว `compositeToAction(comp)` ทับอีกที (รวมถึง Leader Tag ที่คำนวณจาก `comp6D`) ทำให้บางตัว RSI สุดโต่งขึ้น BUY/ADD (เคส INTC) เพราะ composite แบบบวกให้รางวัล RSI/Heat สูง ผู้ใช้**ตัดสินใจคงไว้** เพราะ Screener มีระบบ TP hardcode แยกอิสระ (`tpConfidence`) และ Forward Model ไม่ได้ใช้ — ถ้าจะแก้ในอนาคต: ห้าม ML ทับ Heat = EXTREME

---

## 📤 Backend — ปุ่ม Export CSV + ผลทดสอบย้อนหลังจากข้อมูลจริง (2026-10-02)

**ปุ่ม `⬇ Export CSV`** (ข้างปุ่ม ↻ Refresh ใน Status bar, ฟังก์ชัน `exportStockSignalsCSV()`) ดึง `stock_signals` **ทั้งหมด** (ไม่จำกัด 30 วันแบบ Refresh) ทีละ 1,000 แถว เรียง `run_seq → symbol_id → created_at` (ถ้าไม่มี `symbol_id` fallback อัตโนมัติ) โหลดเป็น `stock_signals_YYYY-MM-DD.csv` (UTF-8 BOM) — ใช้วิเคราะห์ย้อนหลังนอกแอป เพราะ environment ที่ Claude ทำงานเชื่อม `supabase.co` ไม่ได้ (403) ต้อง export มาให้

**ผลทดสอบจากข้อมูลจริง** (10,203 แถว, 3 มิ.ย.–2 ต.ค., 38 ตัว, เอาแถวสุดท้ายของวันต่อตัว, 7D = 5 วันทำการ):
- **ไม่มีเกณฑ์ "7D ลบกี่ % แล้วจะเด้ง" ที่ใช้ได้ตายตัว** — ลบเกิน -15% เด้ง +5.9% (ชนะตลาด +1.7%) แต่ -10% ถึง -7% กลับลงต่อ ไม่เป็นขั้นบันได
- **ฝั่งบนชัดกว่าฝั่งล่าง:** 7D ขึ้นเกิน +10% → 7D ถัดไปเฉลี่ย -3.8% (บวกแค่ 29%, แพ้ตลาด -1.1%) ตัวที่ขึ้นแรงสุดของแต่ละวันถัดมา -3.2%
- **กำไรฝั่งล่างส่วนใหญ่เป็นตลาดเด้งทั้งกระดาน** (สหสัมพันธ์ 7D ตลาดกับ 7D ถัดไป = -0.39) ชนะตลาดจริงแค่ ~+0.6–1.7%
- **🔪:** บวก 50% (ฐาน 43%) ชนะตลาด ~+1.9% แต่ค่ากลางเกือบ 0; ตอน ΔRSI ยังดิ่งแรง (≤ -12) ค่ากลางติดลบ; ตอน RSI หยุดลงที่ oversold ดีกว่า (+4.1%, บวก 59%) = ฝั่งปุ่ม 🔴
- **Rank IC (ข้อมูลชุดเดียวกัน, แบบเดี่ยวๆ ข้ามหุ้นทุกตัว):** ML Score ≈ **-0.05** (อ่อน ติดลบเล็กน้อย), Group RS ≈ **+0.004** (ไม่มีความสัมพันธ์) — Group RS มีค่าไม่ซ้ำแค่ ~10 ค่าสำหรับ 38 ตัว (เป็นค่าระดับกลุ่ม) · ⚠️ นี่วัด "เดี่ยวๆ" เท่านั้น **ไม่ได้ทดสอบแบบมีเงื่อนไข** (เช่น Group RS ติดลบมาก + ติดปุ่ม 🧱) ซึ่งผู้ใช้สังเกตว่าให้ผลดี — ยังเป็นสมมติฐาน ต้องรอ snapshot แล้วนับตามกลุ่ม (10 กลุ่ม) ไม่ใช่รายตัว
- **ข้อควรระวัง:** ตัวอย่างน้อย (ไม่ซ้อนทับเหลือ ~15–16 สัปดาห์, ผลชุดหนึ่งใน 5 ชุดเป็นลบ), ผลแกว่งตามช่วงตลาด (🔪 ครึ่งแรก -0.9% ครึ่งหลัง +4.0%), กระจุกตัวที่หุ้นผันผวนสูง (EOSE, ALAB, IONQ, PLTR, QBTS), ราคาเป็นของรอบ save สุดท้ายของวัน ไม่ใช่ราคาปิดแท้ — **ยังไม่ใส่เป็นเงื่อนไขในปุ่ม/ตาราง** ใช้ดูประกอบเท่านั้น

**Forward Model (ทดลองก่อนหน้า):** ผู้ใช้ยืนยันว่าค่าไม่น่าเชื่อถือจึงเลิกใช้ และใช้ ConDiv แทน (ConDiv วัดความสอดคล้องของทิศราคากับ RSI จากแถวที่ save จริง ไม่ได้ทำนายผลตอบแทน)

**🙈 ซ่อน Forward Model ใน UI backend (2026-10-03):** ปุ่มแท็บ `📊 Forward Model` (`onclick="switchTab('forward')"`, ไม่มี id) ใส่ `style="display:none;"` (มี HTML comment อธิบาย) และปุ่ม `#tomorrowBtn` ในแท็บ Recovery ใส่ `display:none;` — **โค้ด, เนื้อหาแท็บและฟังก์ชันทั้งหมดอยู่ครบ** เอาคืนได้โดยลบ style ที่เติม · ⚠️ ห้ามซ่อน/แตะ "forward return" ใน Step 2/3 ของ ML Analyzer เพราะเป็นแกนของการเทรน ML จริง ไม่เกี่ยวกับแท็บ Forward Model

**ลองแล้วเอาออก:** บรรทัด `EMA50 ±x.x%` ในช่อง Range — ซ้ำกับไอคอน 🐂/🐻 (ซึ่งก็คือราคาเหนือ/ใต้ EMA50) และ pct50 ที่อยู่ใน ML Score อยู่แล้ว เหลือข้อมูลใหม่แค่ "ห่างกี่ %" ผู้ใช้เลือกถอดออก

---

## 📸 Snapshot — ประวัติ ConDiv ลง Supabase + แท็บกราฟ (2026-10-02)

**เหตุผล:** อยากวิเคราะห์ย้อนหลังว่าแต่ละปุ่ม (🔪, 🧱 ฯลฯ) ได้ผลจริงไหม — แต่ high/low 14D (`pricerange` ใน `ml_config`) เก็บแค่ค่าล่าสุด ทับทุกครั้งที่ Save ส่วนสถานะ ConDiv ตามเกณฑ์ปัจจุบันก็คำนวณย้อนหลังด้วยเกณฑ์ที่แก้ภายหลังได้ (เสี่ยงหลอกตัวเอง) → เก็บ "ภาพถ่ายของตาราง ConDiv ตามที่หน้าจอแสดงจริงในวันนั้น" เป็นรายวัน ผลตอบแทนข้างหน้าไม่ต้องเก็บ คำนวณทีหลังจากราคาใน `stock_signals`

### ตาราง `condiv_snapshots` (สร้างด้วย SQL ใน Supabase SQL Editor — รันแล้ว)
```sql
CREATE TABLE IF NOT EXISTS condiv_snapshots (
  id            bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  snap_date     date        NOT NULL,
  symbol        text        NOT NULL,           -- ticker ตรงๆ ไม่ผูก FK กับ symbols (ไม่ต้องรู้ชนิด id)
  price numeric, range_low numeric, range_high numeric, range_days integer,
  range_pct numeric,                            -- ตำแหน่งในกรอบ 0-100
  dist_high numeric,                            -- (ราคา-high)/high*100 ติดลบ=ยังไม่ถึง บวก=ทะลุ
  range_break   text CHECK (range_break IN ('low','high')),
  state text, rsi numeric, rsi_delta numeric, heat_flag text, group_rs numeric, ml_score numeric,
  presets       text[] NOT NULL DEFAULT '{}',   -- ปุ่มที่ติด เช่น {knife},{resist},{}
  created_at    timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT condiv_snapshots_unique_day UNIQUE (snap_date, symbol)
);
CREATE INDEX IF NOT EXISTS condiv_snapshots_symbol_date_idx ON condiv_snapshots (symbol, snap_date DESC);
ALTER TABLE condiv_snapshots ENABLE ROW LEVEL SECURITY;
-- policy: SELECT / INSERT / UPDATE ให้ anon, authenticated (upsert ต้องใช้ทั้ง insert และ update) · ไม่มี DELETE policy
```
- **ไม่ต้องแก้ CHECK constraint ของ `ml_config`** — เป็นตารางแยก ไม่ใช่ `config_type` ใหม่
- เขียนด้วย publishable (anon) key ฝั่งหน้าเว็บ — ไม่ควรเก็บของสำคัญในตารางนี้

### ปุ่ม 💾 บันทึก snapshot (แท็บ 🌊 จุดเปลี่ยนรอบ ขวาสุดของแถวปุ่ม, ฟังก์ชัน `saveConDivSnapshot()`)
- เก็บ**ทุกตัวในตาราง** (ไม่ใช่เฉพาะที่ติดปุ่ม) พร้อมรายชื่อปุ่มที่ติดใน `presets` (คำนวณจาก `CONDIV_PRESETS[id].test(row)` ตัวเดียวกับที่ใช้กรอง)
- upsert `on_conflict=snap_date,symbol` → 1 วัน × 1 ตัว = 1 แถว กดซ้ำวันเดียวกันแถวล่าสุดทับ
- **`snap_date` = "วันตลาดสหรัฐ" ของข้อมูลที่ Save ล่าสุด (แก้ 2026-10-03)** คำนวณจาก `created_at` ของแถว `stock_signals` ล่าสุด (ค่ามากสุดข้ามทุกตัว) → แปลงเป็นเวลา New York → **ลบ 4 ชม.** → เอาวันที่ผลลัพธ์ (ถ้าไม่มี `created_at` fallback เป็น `signal_date`) ข้อความหลังบันทึกแสดง `✓ บันทึกแล้ว N ตัว · วันตลาด YYYY-MM-DD` ใช้ยืนยันได้ทันที
  - **ทำไมไม่ใช้ `signal_date` เดิม:** มันคือวัน UTC ตอน Refresh ซึ่งพลิกวันที่ 07:00 น. ไทย กดเช้าไทยจะได้ป้ายวันถัดไปของข้อมูลปิดตลาดเมื่อคืน → วันเดียวมี 2 แถว/ป้ายผิด (เจอจริง 2026-10-03: ป้าย 10-03 = วันเสาร์ ตลาดปิด เกิดจากแท็บเก่าที่เปิดค้างยังรันโค้ดเก่า)
  - ตัวอย่าง: 21:17 / 23:53 / 03:30 / 08:00 / 11:30 ไทยของคืนวันศุกร์ต่อเสาร์ → ป้าย **ศุกร์** ทั้งหมด · เช้าวันจันทร์ไทย → ป้าย**อาทิตย์** (ข้อมูลคือปิดวันศุกร์ เป็นข้อจำกัดที่รับรู้) · กดบ่ายเสาร์ไทยเป็นต้นไปจะเป็นป้ายเสาร์
- **ลำดับใช้ประจำวัน:** Save to Supabase ที่ Dashboard → Refresh แท็บ 🌊 → กด 💾 (ควรกดเวลาใกล้เคียงกันทุกวัน; วันที่ลืมกด = ช่องว่างถาวร เติมย้อนหลังไม่ได้) · **กดตอนเช้าไทยหลังตลาดปิด (~04:00 ไทยเป็นต้นไป) = ได้ราคาปิดจริงและทับแถวที่กดไว้ระหว่างคืน** · ถ้าข้ามขั้น Save ที่ Dashboard ก่อน จะได้ภาพของ run เก่า
- **ป้ายวันที่ 2 ตารางไม่ตรงกัน (ยังไม่แก้ ตั้งใจ):** `stock_signals.signal_date` = วัน UTC · `condiv_snapshots.snap_date` = วันตลาดสหรัฐ → เช้าไทยต่างกัน 1 วัน เวลา join ข้ามตารางให้ใช้ `created_at` หรือเลื่อนวันให้ตรง (การเปลี่ยน `signal_date` จะกระทบสถิติเดิมที่ dedupe ตามวัน — ให้ผู้ใช้ตัดสินใจ)
- เช็คผล: `SELECT snap_date, count(*), count(*) FILTER (WHERE array_length(presets,1) > 0) FROM condiv_snapshots GROUP BY snap_date ORDER BY snap_date DESC;`

### แท็บ 📸 Snapshot (ใหม่, ต่อจาก 🌊 จุดเปลี่ยนรอบ — `tabSnapshot`/`contentSnapshot`, `loadSnapshotTab()`)
- **ปรับให้อ่านง่าย (2026-10-03, ตามที่ผู้ใช้ขอ):** ฟอนต์ใหญ่ ตัวอักษรสว่าง · พื้นกราฟสีน้ำเงินเข้มต่างจากพื้นการ์ด + เส้นตารางแนวนอน/แนวตั้ง (แกนตั้งหาร 4 ลงตัว เส้นตรงกับเลขจำนวนเต็ม) · แกนมีชื่อ, วันที่มีตัวย่อวัน (วันเสาร์/อาทิตย์เป็นสีเหลือง + ⚠) · ตารางตัวเลขจริงย้อนหลังไม่เกิน 7 วัน + ▲▼ เทียบวันก่อน · กล่อง "📖 วิธีอ่าน" ใต้ทั้งสองกราฟ · legend สีครบทั้งสองกราฟ · แถบเตือนสีส้มเมื่อมีวันเสาร์/อาทิตย์ปนในข้อมูล (จับป้ายวันที่เพี้ยน)
- **กราฟ 1:** จำนวนตัวที่ติดแต่ละปุ่มรายวัน (เส้น 5 สีตามสีปุ่ม, SVG ล้วน ไม่ใช้ไลบรารี) — 🧱 เพิ่ม = ชนเพดานเป็นกลุ่ม · 🔪/🔴 เพิ่ม = ลงมาก้นกรอบ เตือนเมื่อข้อมูล < 10 วันว่าอย่าสรุปแนวโน้ม
- **กราฟ 2:** แถบรายตัว หุ้น × วัน สีตามปุ่ม (ลำดับสี 🔪 > 🔴 > 🟡 > 🟢 > 🧱, ติดหลายปุ่มมีขอบในช่อง, เทา = ไม่ติด, เส้นประ = วันนั้นไม่ได้บันทึก, tooltip มีสถานะ/RSI/Range/ระยะถึง high) แสดงเฉพาะตัวที่เคยติดปุ่ม — ไว้ดูการเลื่อนปุ่มของตัวเดียวกัน (🔪→🔴→🟡)
- query: `snap_date>=cutoff(90 วัน)&order=snap_date.desc&limit=5000` ตามกฎกัน silent truncation (38 ตัว × 90 วัน ≈ 3,420 แถว)

### แผนวิเคราะห์เมื่อข้อมูลพอ (ยังไม่ทำ — ต้องสะสมหลายสัปดาห์ + ผ่านตลาดที่เปลี่ยนสภาพอย่างน้อยหนึ่งรอบ)
ผลตอบแทน 7 วันถัดไปแยกตามปุ่มเทียบค่าเฉลี่ยตลาด (join กับราคาใน `stock_signals`) · ตัวที่ทะลุ 🧱 จริง vs ถูกผลักกลับ ต่างกันที่ Heat/ΔRSI/Group RS ไหม · 🔪 ที่รอด vs ร่วงต่อ · ตัวที่เลื่อน 🔪→🔴 ดีกว่าอยู่ 🔪 เฉยๆ จริงไหม · **ต้องนับตัวอย่างแบบไม่ซ้อนทับ และแยก "ชนะตลาด" ออกจาก "ตลาดเด้งทั้งกระดาน"** (ดูบทเรียนด้านล่าง)

---

## 🧭 Screener โหมดตลาด + ตัวกรอง ConDiv (2026-10-03, frontend)

**ที่มา:** ใน sideway ปุ่ม 🟢 พื้นฐานแทบไม่เคยผ่าน — ตรวจสูตรในโค้ดแล้วพบว่า **Upside = risk% × clamp(3 − (RSI−50)×0.1, 0.3, 6)** ผูกกับ **RSI อย่างเดียว** (ไม่เกี่ยวกับระยะถึง high/แนวต้านเลย): ความเสี่ยง MEDIUM (5%) → Upside ≥15% ≡ **RSI ≤ 50**, ≥10% ≡ RSI ≤ 60, ≥8% ≡ RSI ≤ 64 · LOW (3%) เข้มกว่ามาก (≥10% ต้อง RSI ≤ 46) → "Trend UP + RSI ≤ 50 + TP ไม่อ่อนแรง" ชนกันเองในตลาดออกข้าง (หุ้นที่ RSI ต่ำจริงมักอยู่ในกลุ่มที่ร่วงและโดนตัวกรอง TP ตัดอีกชั้น) เคส SYM: ติด 🟢 ใน ConDiv (Bull Conv, COOL, Range 53%, Group RS +2.14) แต่ไม่ติด 🟢 พื้นฐานของ Screener เพราะ RSI 61.1 → Upside ≈ 9.5% (<15) — **สองปุ่มชื่อคล้ายกันแต่เงื่อนไขคนละชุด มีร่วมกันข้อเดียวคือ Group RS ≥ 0**

### 🧭 โหมดตลาด (เลือกเอง ไม่ auto — ตรงหลัก "แค่โชว์ คนตัดสินใจเอง" ของกล่องบริบทตลาด)
- `SCREENER_MODE_OVERRIDES` + `screenerMarketMode` (จำใน `localStorage` key `screener_market_mode`) · แถบเลือกอยู่เหนือปุ่มสูตรมาตรฐาน (`renderScreenerModeBar()`, `setScreenerMarketMode()`)
- **ปกติ** = Upside เดิมทุกปุ่ม · **↔️ ไซด์เวย์** = ตัด Upside ของ 🟢 กับ 🟡 ออก แล้วใช้เพดาน RSI ตรงๆ แทน: 🟢 RSI ≤ 60, 🟡 RSI ≤ 65 (เงื่อนไขอื่นเหมือนเดิม) · 🔴 🔥 🧊 💎 ไม่แตะ (🔥/🧊 ไม่ติดในไซด์เวย์ = ข้อมูลจริง ไม่ใช่ปัญหา)
- ⚠️ **เลข 60/65 เป็นค่าตั้งต้นที่ยังไม่ผ่านการทดสอบกับข้อมูลจริง** ปรับได้ที่ `SCREENER_MODE_OVERRIDES` จุดเดียว
- สลับโหมดตอนปุ่มสูตรมาตรฐานค้างอยู่ → โหลดปุ่มเดิมซ้ำด้วยเกณฑ์โหมดใหม่ (`screenerLastBuiltin`) · โหลดชุดที่บันทึกเอง/กด "ล้างเงื่อนไข" → ล้าง `screenerLastBuiltin` กันสลับโหมดแล้วปุ่มเก่าทับของที่เลือกไว้

### 📡 ตัวกรอง ConDiv (chip หมวดที่ 5 ต่อจาก Trend/Conf/Heat/ML-vs-จริง)
- chip: Bull Conv / Bull Div / Neutral / Bear Div / Bear Conv — ใช้ `computeConDivTableFE()` ตัวเดียวกับตาราง 🌊 (state + ΔRSI ตรงกันเป๊ะ) ผ่าน `getScreenerCondivMap()` (cache 2 นาที, โหลด `mlConfig` + `loadActiveRegimeThresholdFE()` ก่อนให้ noise threshold ตรงกับตาราง) · เลือก chip → มีคอลัมน์ **📡 ConDiv** (สถานะ + ΔRSI)
- ข้อมูลไม่มี/ไม่พอ (`not_enough_data`) = **ไม่ผ่าน** (ยืนยันสัญญาณไม่ได้ หลักเดียวกับปุ่มในตาราง 🌊) · โหลดไม่สำเร็จ → แสดงข้อความเตือน ไม่แสดง "ไม่มีตัวเข้าเงื่อนไข" ที่ทำให้เข้าใจผิด
- **ผูกกับปุ่ม (ทั้งสองโหมด):** 🟢 = Bull Conv · 🟡 และ 🔴 = Bull Conv หรือ Neutral (ตัด Bear Div = ราคาเหนือ EMA50 แต่ RSI ลดลง) · 🔥 🧊 💎 ไม่ใช้ — **เลือกโดย Claude จากหลักเหตุผล (Trend UP อยู่แล้ว ConDiv เหลือแค่ตัดตัวที่สัญญาณอ่อนลง) ยังไม่ได้พิสูจน์ด้วยข้อมูล** ผู้ใช้กด chip เปลี่ยนเองได้
- ⚠️ ConDiv คำนวณจาก `stock_signals` ที่ **Save ไว้ล่าสุด** ไม่ใช่ราคาสด → ถ้าราคาขยับหลัง Save สถานะอาจต่างจาก Dashboard เล็กน้อย
- 🟢🟡🔴 ใน "โหมดปกติ" ตอนนี้เข้มกว่าก่อน 2026-10-03 (เพิ่มเงื่อนไข ConDiv) · ชุดที่ผู้ใช้บันทึกไว้เองก่อนหน้าไม่มี chip นี้ → พฤติกรรมเดิม
- ผู้ใช้ให้ความเห็นว่าการมีตัวกรองนี้ทำให้ "ชัดขึ้น" — เป็นความรู้สึกจากการอ่านจอ **ยังไม่ใช่หลักฐานว่าผลดีขึ้น** (ต้องรอ snapshot) · ไอเดียต่อยอดที่ยังไม่ได้ทำ: เก็บในตาราง snapshot ว่าวันนั้นผ่านปุ่มไหนของ Screener เพื่อเทียบผลย้อนหลัง · มุมมอง "ตัวที่เพิ่งติดปุ่มวันนี้เทียบเมื่อวาน" ในแท็บ 📸

### ข้อสังเกตที่ยังเป็นสมมติฐาน (ยังไม่ได้พิสูจน์)
- **Group RS ติดลบมาก + ติด 🧱 → ผลตอบแทนดี?** ผู้ใช้สังเกตเอง · เดี่ยวๆ Group RS ไม่มี IC (+0.004) แต่ไม่เคยทดสอบแบบมีเงื่อนไข · ข้อควรระวัง: ตัวอย่างอิสระจริงมีแค่ ~10 กลุ่ม, หุ้นกลุ่มเดียวกันขยับด้วยกัน, ตลาดช่วงเดียว · วิธีทดสอบเมื่อข้อมูลพอ: เทียบผลตอบแทนข้างหน้า (เช่น 5 วัน) ของ 🧱∧GroupRS ติดลบมาก vs 🧱∧GroupRS ≥ 0 vs ตลาด **นับตามกลุ่ม** (`condiv_snapshots` เก็บ `group_rs`, `presets`, `price` ไว้แล้ว)
- **🔴/🔪 รวมหุ้นที่หลุดต่ำกว่า low 14D แล้ว** (เช่น WDC) เพราะ `rangePct` ถูก clamp ที่ 0 (มอง "ทะลุ low" ไม่เห็น) — ยังไม่เปลี่ยน รอดูข้อมูลสะสมก่อนตัดสินใจว่าจะแยกหรือไม่

---

## 📌 บทเรียนสะสม (Lessons learned)

- แก้โค้ดไม่เห็นผล → Private tab ใหม่
- ราคาไม่ตรงข้ามแอป → เช็ค `run_seq`
- ก่อนแก้ไฟล์ → ขอไฟล์ปัจจุบันเสมอ
- ส่ง config ใหม่ไป Supabase แล้ว error `violates check constraint` → เช็คค่าที่ส่งอยู่ใน constraint ที่อนุญาตหรือยัง (`SELECT pg_get_constraintdef(oid) FROM pg_constraint WHERE conname='...'`)
- **แก้ scale/ทิศทางของค่าไหนก็ตาม → grep หา field name ทั้ง 2 ไฟล์ก่อนเสมอ** อย่าไล่ตามแค่ชื่อฟังก์ชันที่จำได้ — logic อาจถูกเขียนซ้ำแบบ inline ไม่มีชื่อฟังก์ชันแยก
- **เพิ่มฟังก์ชันใหม่ที่ query `stock_signals` → ต้อง dedupe ด้วย `run_seq` เสมอ** (group by symbol_id+signal_date เก็บ run_seq สูงสุด) ไม่งั้นนับเกินจำนวน symbol จริงจากหลาย session/วัน
- **แท็บ "จุดเปลี่ยนรอบ" ตัวเลขไม่ตรง Dashboard สดๆ** → ปกติ ไม่ใช่บั๊ก เพราะแท็บนี้อ่านจาก `stock_signals` ที่อัพเดตเฉพาะตอนกด "Save to Supabase" — ต้องกด Save ที่ Dashboard ก่อนเสมอ แล้วค่อยกด Refresh ที่แท็บนี้
- **UI ไม่อัพเดตหลัง deploy ทั้งที่ไฟล์ถูกต้อง** → เช็ค Cloudflare/เบราว์เซอร์ cache ก่อน (ลอง Incognito) อาจไม่ใช่บั๊กโค้ด แค่มาช้า
- **ตัวเลขจากคนละแหล่งไม่ตรงกันทั้งที่ควรจะสะท้อนตลาดเดียวกัน** → เช็คนิยามที่ใช้ก่อน (เช่น EMA20vsEMA50 ≠ ราคาvsEMA50) อย่าสรุปว่าเป็นบั๊กข้อมูลทันที บางทีเป็นบั๊ก "นิยามไม่ตรงกัน" ระหว่าง 2 ระบบ
- **เพิ่ม `config_type` ใหม่ใน `ml_config` → ต้องรัน SQL `ALTER TABLE` เพิ่มเข้า CHECK constraint ก่อนเสมอ** ไม่งั้น Save error ทันที (เจอ 2 รอบ: ConDiv, Config Control) — เช็ค constraint ปัจจุบันด้วย query `regexp_matches` ก่อนเขียน ALTER กันพลาดค่าเดิม
- **Query ที่ดึง `stock_signals`/`ml_config` แบบ log ต้องมี `order=...desc&limit=` เสมอ** ห้ามปล่อย default — ไม่งั้นเสี่ยง silent truncation ตัดข้อมูลใหม่ทิ้งเงียบๆ (เจอบั๊กร้ายแรงสุดของโปรเจกต์จากจุดนี้ ซ้ำ 2 ไฟล์)
- **ชื่อไฟล์บน GitHub repo backend คือ `index.html`** ไม่ใช่ `index_backend.html` (ชื่อหลังเป็นแค่ชื่อเรียกในบทสนทนา) — อย่าสับสนเวลาหาไฟล์บน repo จริง
- **คุยตัดสินใจ "จะย้ายฟีเจอร์ไปแท็บไหน" แล้วต้องเช็คโค้ดจริงว่าทำตามหรือยัง** ก่อนบอกตำแหน่งให้ผู้ใช้ไปตามหา — การคุยกันด้วยคำพูดไม่ได้แปลว่าโค้ดถูกย้ายจริงเสมอไป
- **เจอปัญหา "ข้อมูลไม่อัพเดต/หาย" → เช็คข้อมูลจริงในฐานข้อมูลด้วย SQL ตรงๆ ก่อนเสมอ** อย่าเดาสาเหตุจากพฤติกรรมที่เห็นในแอปอย่างเดียว (เคยเดาผิดว่าเป็น `ignore-duplicates` ทั้งที่สาเหตุจริงคือ order/limit)
- **2 ไฟล์มีฟังก์ชันคำนวณ composite/ML score คนละตัว คนละสูตรกัน (`composite()` ฝั่ง backend vs `calcCompositeScore()` ฝั่ง front)** ทั้งที่ดูเผินๆ เหมือนเป็น "ML ตัวเดียวกัน" — ก่อนเชื่อว่าตัวเลข 2 ฝั่งควรตรงกัน ต้อง grep ชื่อฟังก์ชันจริงทั้ง 2 ไฟล์เทียบ dimension ต่อ dimension ก่อนเสมอ อย่าเดาจากชื่อ/ที่มาของ weight เพียงอย่างเดียว
- **เพิ่ม dimension ใหม่เข้าสูตรที่มีอยู่แล้ว (เช่น Slope) → เช็ค convention การ normalize ของฟังก์ชันปลายทางก่อนเสมอ** (front normalize ทุกมิติเป็น 0..1 ก่อนคูณ weight, backend ใช้ -1..1 ดิบๆ ได้บางมิติ) — copy สูตรจากอีกไฟล์ตรงๆ โดยไม่เช็ค convention จะทำให้ weight ตัวเดียวกันส่งผลต่าง scale กันระหว่าง 2 แอป
- **ตัวชี้วัดใหม่ที่จะเพิ่ม → เช็คก่อนว่ามาจากวัตถุดิบเดียวกับที่มีอยู่แล้วไหม** (ราคา/RSI/EMA/ตำแหน่งในกรอบ) — สัญญาณที่ตรงกันหลายตัวอาจเป็นข้อมูลชุดเดียวกันจัดเรียงต่างกัน ไม่ใช่การยืนยันอิสระ (เคส EMA50 ซ้ำกับ 🐂/🐻 และ pct50)
- **เกณฑ์ที่ดูดีใน sideway จะพังเมื่อตลาดมีทิศ** (และกลับกัน: Screener ที่เข้มแบบยืนยันต่อเนื่องจะผ่านน้อยใน sideway) — ปุ่ม/ตัวกรองทุกตัวผูกกับสภาพตลาดช่วงนั้น ไม่ใช่กฎถาวร ระวังเอาผลทดสอบช่วงเดียวมาตั้งเป็นเกณฑ์
- **อ้างผลย้อนหลังต้องแยก "ชนะตลาด" ออกจาก "ตลาดเด้งทั้งกระดาน"** และนับตัวอย่างแบบไม่ซ้อนทับ (รายวันที่หน้าต่าง 7D ซ้อนกันทำให้ตัวอย่างดูเยอะเกินจริง)
- **ตารางใหม่ใน Supabase ที่แอปต้องเขียน → ต้องตั้ง RLS policy (SELECT/INSERT/UPDATE) ให้ anon ด้วย** ไม่ใช่แค่สร้างตาราง ไม่งั้นอ่านได้/เขียนไม่ได้เงียบๆ · ถ้าไม่รู้ชนิดของ id ตารางอื่น อย่าผูก FK เก็บเป็น ticker ข้อความแทน
- **เก็บ "สัญญาณตามที่หน้าจอแสดงจริง" เป็นรายวัน ดีกว่าคำนวณย้อนหลังด้วยเกณฑ์ปัจจุบัน** — แก้เกณฑ์ภายหลังไม่ทำให้ผลย้อนหลังเปลี่ยน กันหลอกตัวเองว่าเกณฑ์ใหม่ดูดี
- **ป้ายวันที่ของข้อมูลที่เก็บเป็นรายวัน ต้องผูกกับ "วันของตลาดต้นทาง" ไม่ใช่วันที่/เวลาของเครื่องหรือ UTC** — ตลาดสหรัฐปิด ~03:00–04:00 ไทย ส่วน UTC พลิกวัน 07:00 ไทย กดเช้าไทยจึงเสี่ยงได้ป้ายวันถัดไป/วันซ้ำ แล้วสถิติเพี้ยนเงียบๆ · ตรวจด้วยวันเสาร์/อาทิตย์ในข้อมูล (ตลาดปิด = ป้ายผิดแน่)
- **อัปไฟล์ใหม่ขึ้น Cloudflare แล้ว ต้องปิดแท็บเก่าเปิดใหม่ (หรือ reload ล้างแคช) ก่อนใช้เสมอ** — แท็บที่เปิดค้างยังรันโค้ดเก่า และเขียนข้อมูลผิดเงียบๆ ได้ (เคส snapshot ป้าย 10-03) · เช็คจากข้อความหลังกด 💾 ว่าเขียน `วันตลาด` ตามที่ควร
- **เกณฑ์ที่ชื่อปุ่มคล้ายกันในคนละตาราง/แท็บ (🟢 Fresh Buy ใน ConDiv vs 🟢 พื้นฐานใน Screener) ไม่ใช่เกณฑ์เดียวกัน** — ก่อนสงสัยว่าระบบขัดแย้ง ให้เทียบเงื่อนไขทีละข้อจากโค้ด · และตัวชี้วัดที่ดูเหมือนบอกว่า "มีที่ให้ขึ้น" (Upside) อาจเป็นแค่ฟังก์ชันของ RSI ตัวเดียว ไม่เกี่ยวกับระยะถึงแนวต้านจริง
- **ผลวัดแบบเดี่ยวๆ (IC ข้ามหุ้นทุกตัว) ปฏิเสธข้อสังเกตแบบมีเงื่อนไขไม่ได้** — "ไม่มีความสัมพันธ์โดยรวม" ≠ "ไม่มีความสัมพันธ์ในบางกลุ่มย่อย" · แต่การเจอผลในกลุ่มย่อยที่เลือกดูหลังเห็นข้อมูล ก็ต้องระวังหลอกตัวเอง (ตัวอย่างอิสระน้อย/กระจุกตามกลุ่ม) · ต้องพิสูจน์ด้วยข้อมูลที่เก็บไว้ล่วงหน้า
