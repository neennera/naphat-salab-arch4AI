# Tutorial 6 worksheet: serving patterns

**Name / ชื่อ: ณภัทร เสรีรักษ์**

**Student ID / รหัสนิสิต: 6632061321**

This is **individual work**: each student completes and submits their own copy, in their own fork.

_งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง_

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1-3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run      | p50 ms | p95 ms | accuracy | large calls |
| -------- | ------ | ------ | -------- | ----------- |
| t0-large | 507    | 612    | 93.5%    | 100%        |
| t0-small | 58     | 105    | 72.5%    | 0%          |

**Q0.** What does the large model buy, and what does it cost?

_large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร_

> large model แม่นกว่ามาก (accuracy 93.5% เทียบกับ 72.5% ของ small)
>
> แต่เราต้องแลกด้วย latency ที่สูงกว่า (p50 507 ms เทียบกับ 58 ms)
>
> ทุกรีวิวต้องเรียก large model 100% ซึ่งมีค่าใช้จ่ายทุก call และรับได้แค่ 20 calls ต่อวินาที

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run       | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
| --------- | ------------ | ------ | ------ | ------ | ------- | -------- |
| t1-before | 26.5         | 240    | 370    | 18     | 0       | 93.4%    |
| t1-after  | 9.9          | 422    | 1024   | 0      | 22      | 93.3%    |

**Q1.1** In `t1-before`, p50 can look _better_ than the baseline. Why is p50 misleading here? Which column tells the real story?

_ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง_

> p50 ของ t1-before (240 ms) ดูดีกว่า baseline (507 ms) เพราะ load test ไม่นับ request ที่ล้มเหลวเข้าไปใน p50/p95
>
> อีกอย่าง call ที่ค้างยังไม่เข้าไปแย่งคิวของ large model ทำให้ request ที่เหลือเร็วขึ้น
>
> column ที่บอกความจริงคือ `errors` (18 รายการ) และ run time (26.5 s เทียบกับประมาณ 10 s ของ baseline)

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

_การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway_

> request ที่รออยู่ยังถือ connection และ slot ของ client ไว้ใน review service และ gateway จนกว่าจะได้คำตอบ ถ้าไม่มี timeout ก็อาจรอนานถึง 20 s
>
> เมื่อ call ที่ค้างสะสมมากขึ้น slot จะหมด และ request ปกติก็ต้องรอตามไปด้วย
>
> ความล้มเหลวจึงลามไปทั้งระบบ ไม่ใช่แค่ 10% ที่ช้า (run time ของ t1-before เพิ่มเป็น 26.5 s)

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

_รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น_

> เราควรบันทึกรีวิวเป็น pending ไว้ก่อน แล้วให้ batch job มาติด label ทีหลังเมื่อ large model กลับมาปกติ
>
> วิธีนี้คือ topology แบบ Asynchronous / batch (เข้าคิวผ่าน message broker แล้วประมวลผลทีหลัง) ซึ่งใช้คู่กับ pattern fallback

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

_เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที_

> **ถ้าตั้ง 0.1 s:** large model ใช้เวลาประมาณ 200 ms อยู่แล้ว เกือบทุก call จะ timeout และกลายเป็น pending ทั้งที่ model ปกติดี อีกทั้ง call ที่ถูกทิ้งก็ยังกิน capacity ของ large model อยู่
>
> **ถ้าตั้ง 10 s:** request ที่ค้างจะถือ resource นานถึง 10 s ซึ่งผิด QR1/QR2 (ต้องตอบภายใน 1 s) และแทบไม่ต่างจากการไม่มี timeout เลย

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run    | threshold | p50 ms | p95 ms | accuracy | large calls |
| ------ | --------- | ------ | ------ | -------- | ----------- |
| t2-0.6 | 0.6       | 23     | 524    | 91.5%    | 38%         |
| t2-0.7 | 0.7       | 355    | 573    | 93.0%    | 53%         |
| t2-0.8 | 0.8       | 477    | 668    | 93.5%    | 64%         |
| t2-0.9 | 0.9       | 466    | 604    | 93.5%    | 77%         |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

_จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป_

> เราเลือก threshold 0.7 เพราะ accuracy 93.0% ยังเกิน QR3 (90%) และเรียก large model แค่ 53% (ลดจาก 100% เกือบครึ่ง)
>
> ส่วน 0.6 ถูกกว่า (38%) แต่ accuracy 91.5% ใกล้เส้น 90% มากเกินไป และ 0.8/0.9 แม่นเท่า large (93.5%) แต่ยังเรียก large ถึง 64-77%
>
> สิ่งที่เรายอมเสียคือ accuracy ประมาณ 0.5 จุดเมื่อเทียบกับ large (รีวิวที่ small มั่นใจแต่ตอบผิด)

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

_ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก_

> ที่ 0.7 p50 ลดจาก 507 เหลือ 355 ms แต่ p95 แทบไม่ลด (612 เหลือ 573 ms)
>
> เพราะรีวิวที่ small ไม่มั่นใจต้องรอ small แล้วค่อยไปรอ large อีก รีวิวที่ช้าที่สุด 5% จึงเป็นกลุ่มที่ถูกส่งต่อไป large เสมอ
>
> cascade ช่วยได้แค่รีวิวที่ small ตอบเองได้

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

_cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้_

> small model อาจมั่นใจสูงแต่ตอบผิด เช่นประโยคปฏิเสธหรือมี "but" ("not cold", "good but ...") ที่ bag of words อ่านผิด
>
> ในกรณีนี้ cascade จะไม่ส่งต่อไป large เจ้าของร้านหรือผู้ใช้จึงได้ label ผิดโดยที่ไม่มีใครรู้ (small ตัวเดียวแม่นแค่ 72.5%)

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run      | p50 ms | p95 ms | accuracy | large calls | cache hits |
| -------- | ------ | ------ | -------- | ----------- | ---------- |
| t3-cache | 24     | 551    | 93.0%    | 38%         | 35%        |

**Q3.1** Which reviews benefit from the cache, and which never will?

_รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์_

> **ได้ประโยชน์:** รีวิวที่ซ้ำกัน (หลัง normalise ตัวพิมพ์และช่องว่าง) เช่นรีวิวสั้นๆ ที่พบบ่อยอย่าง "Delicious!" (t3-cache มี cache hit 35%)
>
> **ไม่มีวันได้ประโยชน์:** รีวิวที่เป็นข้อความเฉพาะและไม่ซ้ำใคร ซึ่งเป็นส่วนใหญ่ เพราะ miss ทุกครั้ง

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

_สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร_

> เราจะต้องล้าง cache เพราะคำตอบเก่ามาจาก model ตัวเก่า
>
> วิธีออกแบบคือใส่ model version ไว้ใน cache key (และตั้ง TTL ด้วย) เมื่อ deploy model ใหม่ key เดิมจะไม่ match เอง ไม่ต้องล้างด้วยมือ และถ้า rollback ก็กลับไปใช้ cache เดิมได้

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

_ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture_

> เราเห็นว่า p50 ของ cascade ไม่ได้ลดลงเรื่อยๆ ตาม threshold แต่มันเป็นขั้นๆ ที่ threshold 0.6 p50 เหลือแค่ 23 ms แต่ที่ 0.7 กลับเป็น 355 ms
>
> เพราะ p50 จะดีก็ต่อเมื่อ small model ตอบเองได้เกินครึ่งของรีวิว ที่ 0.6 มีแค่ 38% ที่ไปถึง large model (request ตรงกลางจึงเป็นของ small) แต่ที่ 0.7 เป็น 53% (request ตรงกลางต้องรอ large) ก่อนวัดเราคิดว่า threshold ที่เปลี่ยนเล็กน้อยจะทำให้ latency เปลี่ยนเล็กน้อยตาม
