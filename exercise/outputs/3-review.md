Agent: Claude Code, Sonnet 5.5 (claude-sonnet-5-5)

# Review: ADR-0002 Publish OrderPlaced events via RabbitMQ

หลักฐานที่ใช้: `docs/adr/0002-publish-order-placed-events-via-rabbitmq.md`, `docs/scenario.md` เท่านั้น

## 1. Quality-attribute scenarios

| # | Source · Stimulus · Environment | Response | Response measure |
|---|---|---|---|
| S1 Performance | ลูกค้า · `PlaceOrder` · peak 200 orders/s (11.11) | ตัด MailFast ออกจาก critical path แล้ว publish event | p95 `PlaceOrder` ≤ 500 ms (ปัจจุบันประมาณ 300 ms จึงเหลือ headroom ประมาณ 200 ms) |
| S2 Availability | MailFast, `loyaltyservice` หรือ RabbitMQ ล่ม · มีคนสั่งซื้อ · ช่วงปกติและ peak | order สำเร็จ แล้วชดเชยทีหลัง | order สำเร็จ 100 %; loyalty ได้ภายใน 1 นาที exactly once เมื่อระบบกลับมา |
| S3 Modifiability/Rate limit | marketing ขอ SMS หรือ fraud scoring · ปีหน้า · และ burst 200/s ชน MailFast 50/s | เพิ่ม consumer ใหม่ผูก exchange เดิม; email ถูก throttle | แก้ `checkoutservice` 0 จุด; MailFast ไม่เกิน 50 emails/s รวมทุก replica |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Non-risk | แยก queue ต่อ consumer ผ่าน topic exchange | email backlog ไม่บล็อก loyalty และเพิ่ม action ใหม่ได้โดยไม่แก้ checkout (S3) |
| Non-risk | Option 1 และ 2 ถูกตัดออกด้วยเหตุผลที่ตรวจสอบได้: MailFast p95 1.5 s เกินงบ 500 ms อยู่แล้ว และ 200/s เป็น 4 เท่าของ 50/s | การตัดสินใจไม่ใช่แค่ความชอบ |
| Sensitivity point | publisher confirm timeout 100 ms อยู่ใน critical path ของทุก order ที่ 200/s ADR ยอมรับเองว่าเป็น design parameter ไม่ใช่ข้อมูลจริง และไม่มีตัวเลข latency ของ confirm บน quorum queue 3 node | ถ้า confirm ช้าเกินกว่า headroom 200 ms ที่เหลือ p95 จะเกิน 500 ms (S1) ต้องวัดก่อนเลือกค่า |
| Sensitivity point | rate limit email "ต่อ pod = 50 / จำนวน replica" | autoscale หรือ pod restart ทำให้ผลรวมเกิน 50/s ได้ หรือต่ำกว่าที่ควร; ชน rate limit คือเงื่อนไขของ S3 |
| Trade-off point | publish แบบ async หลัง `shipOrder` แลก availability/latency กับ eventual consistency | ลูกค้าอาจไม่ได้อีเมลหรือ points ทันที ต้องตกลง SLA กับ marketing (ยังเป็น open question 1) |
| Risk | producer buffer อยู่ใน memory ของ `checkoutservice` ถ้า pod ตายหรือ RabbitMQ ล่มนาน event หายและ points หาย | ขัดกับ requirement "exactly once" และ "ภายใน 1 นาที" ใน scenario; failure mode เดียวกับ Option 2 (goroutine) ที่ ADR ใช้เป็นเหตุผลตัดทิ้ง |
| Risk | buffer ไม่มีขนาดจำกัดที่ระบุ และ retry แบบ backoff อาจเกิน 1 นาที | RabbitMQ ล่มช่วง 11.11 ที่ 200/s ทำให้ memory โต หรือ points ช้ากว่า SLA |
| Risk | email backlog: ส่วนเกิน 150/s ยังไม่มีการคำนวณ | ตัวอย่าง peak 10 นาที = ประมาณ 90,000 ฉบับ ใช้เวลาระบายประมาณ 30 นาทีที่ 50/s; ถ้า peak ยาวกว่านี้อีเมลช้าเป็นชั่วโมง และ queue ต้องมีขนาดรองรับ |
| Risk | retry ของ email กับ at-least-once อาจส่งอีเมลซ้ำ (MailFast idempotency ยังไม่ทราบ, open question 6) | ลูกค้าได้อีเมลซ้ำ และ retry เพิ่มภาระกับ limit 50/s |
| Risk | ความเสี่ยงด้าน operability: cluster 3 node quorum queues สำหรับทีม 4 คน ที่มีเพียง 2 คนเคยใช้ RabbitMQ และยังไม่มีผู้ operate (open question 4) | ล่มแล้วไม่มีคนแก้ได้ อาจกระทบ S2 |

## 3. Missing or weak

- **Options ที่ไม่ได้พิจารณา:** (a) แบบผสม: เรียก `loyaltyservice` แบบ sync (เป็น service ภายใน เร็วและต้อง exactly once) แล้ว async เฉพาะ email/analytics; (b) เพิ่ม order store เล็กๆ พร้อม outbox ตั้งแต่ ADR นี้ แทนที่จะเลื่อนไป ADR ถัดไป; (c) broker เบากว่าหรือ broker ที่ persist ง่ายกว่า (เช่น NATS JetStream) ADR ไม่ได้เทียบ
- **ข้อเรียกร้องที่ไม่มีหลักฐาน:** "ไม่มีใครในทีมเคยใช้ Kafka" — scenario บอกแค่ว่า 2 คนเคยใช้ RabbitMQ ไม่ได้พูดถึง Kafka (Not evidenced in the provided files)
- **ความไม่สอดคล้อง:** ADR อ้างว่า exactly once ทำได้ด้วย idempotency ฝั่ง consumer แต่ requirement คือ end-to-end; ฝั่ง producer ยังรับความเสี่ยงสูญหาย จึงไม่ตรง scenario ตามที่เขียน
- **Consequence ที่ไม่ได้ระบุ:** concurrency ที่ MailFast ต้องรองรับ (50/s × p95 1.5 s ≈ 75 request พร้อมกัน), ความลึก queue สูงสุด, การจัดการ order ที่ publish ซ้ำตอน retry (ใช้ `order_id` เป็น key ได้ แต่ไม่ได้ระบุว่า dedupe ที่ใด)
- **Loyalty DB, MailFast protocol:** ยัง "Not decided"/"Not evidenced" ในแผนภาพ ทั้งที่ loyalty exactly once ขึ้นกับ DB นี้โดยตรง
- **PII:** event มีอีเมลลูกค้า แต่ไม่มีข้อสรุป (open question 5)

## 4. Verdict

**Accept with changes.** ทิศทางถูก (RabbitMQ pub/sub ตอบ S1, S3) แต่ producer side ยังไม่ตอบ S2

Top 3 changes:
1. **ปิดช่องว่าง producer:** ตัดสินใจก่อน Accept ว่า loyalty จะใช้ sync call, order store + outbox, หรือให้ business ลงนามรับความเสี่ยง points หาย พร้อมกำหนดขนาด buffer สูงสุดและพฤติกรรมเมื่อเต็ม
2. **วัดและกำหนดค่า sensitivity:** ทดสอบ latency ของ publisher confirm บน cluster จริงที่ 200/s เพื่อยืนยัน timeout 100 ms และเปลี่ยนเป็น global rate limiter (เช่น shared token bucket หรือ consumer จำนวนคงที่) แทนการหารต่อ pod
3. **คำนวณ backlog และตกลง SLA อีเมลกับ marketing** พร้อมแก้ข้อความ Kafka ที่ไม่มีหลักฐาน และระบุเจ้าของการ operate RabbitMQ
