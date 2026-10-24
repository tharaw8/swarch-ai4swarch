Agent: Claude Code, Sonnet 5.5 (claude-sonnet-5-5). Run a, context: none (เขียนจาก requirement อย่างเดียว ไม่ได้อ่านโค้ดหรือเอกสารของระบบ)

# ADR-0002: Publish an "OrderPlaced" event to a message broker for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-24
- **Deciders:** ยังไม่ระบุ (ดู Open questions)

## Context and problem

หลังลูกค้าสั่งซื้อสำเร็จ Online Boutique ต้องทำ 3 อย่างใหม่ตาม requirement:

1. ส่งอีเมลยืนยันคำสั่งซื้อผ่าน MailFast (ผู้ให้บริการอีเมลภายนอกรายใหม่) แทน logic ของ `emailservice` ที่เป็น mock
2. ให้ loyalty points ผ่าน `loyaltyservice` ซึ่งต้องสร้างใหม่
3. บันทึก analytics event "order placed" ให้ทีม marketing

นอกจากนี้ marketing คาดว่าปีหน้าจะมี post-order action เพิ่ม (SMS, fraud scoring, การอัปเดต recommendations)

ปัญหาที่ต้องตัดสินใจ: ส่วนที่รับคำสั่งซื้อควรเรียกแต่ละ action เองโดยตรง หรือควรประกาศ event แล้วให้แต่ละ action ทำงานแยกกัน

ข้อเท็จจริงที่รู้จาก requirement เท่านั้น: มี `emailservice` (mock), ต้องสร้าง `loyaltyservice` ใหม่, MailFast เป็น external dependency, และจะมี action เพิ่มอีก. ส่วนอื่น (service ไหนรับ order อยู่ตอนนี้, protocol, มี broker อยู่แล้วหรือไม่, ปริมาณ order) Not evidenced in the provided files (ไม่ได้อ่านไฟล์ใดของระบบตามเงื่อนไข context: none).

## Decision drivers

เรียงตามความสำคัญ (ลำดับนี้เป็น assumption ของผู้เขียน ต้องให้ผู้มีส่วนได้เสียยืนยัน):

- **Availability / การแยกความล้มเหลว:** MailFast ล่มหรือช้า ต้องไม่ทำให้การสั่งซื้อล้มเหลวหรือค้าง
- **Modifiability:** เพิ่ม action ใหม่ (SMS, fraud scoring, recommendations) โดยแก้ส่วนรับ order น้อยที่สุด
- **Performance (latency ของการสั่งซื้อ):** ไม่บวกเวลารอของ action หลังสั่งซื้อเข้าไปใน response ให้ลูกค้า
- **Data consistency / ความถูกต้อง:** ลูกค้าได้ points และอีเมลครบ ไม่ซ้ำ ไม่หาย (ระดับที่ยอมรับได้ยังไม่ระบุ)
- **Cost / ความซับซ้อนในการ operate:** ทีมต้องดูแลของใหม่ได้
- **Testability:** ทดสอบ action แต่ละตัวแยกกันได้

## Considered options

1. **Option A: Synchronous direct calls** ส่วนที่รับ order เรียก MailFast (ผ่าน `emailservice`), `loyaltyservice` และ analytics ทีละตัวโดยตรง
2. **Option B: Event publish ผ่าน message broker** ส่วนที่รับ order ประกาศ event `OrderPlaced` ไปที่ broker; `emailservice` (ต่อ MailFast), `loyaltyservice` และ analytics consumer subscribe และทำงานแยกกันแบบ asynchronous
3. **Option C: Event ผ่าน broker + transactional outbox** เหมือน B แต่บันทึก event ลงตาราง outbox ใน transaction เดียวกับการบันทึก order แล้วมี relay ส่งไป broker

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A | ง่ายที่สุด ไม่มี infrastructure ใหม่; flow อ่านและ debug ตรงไปตรงมา; รู้ผลทันทีว่า action สำเร็จหรือไม่ | MailFast ช้า/ล่มกระทบ latency และ availability ของการสั่งซื้อ (availability, performance); ทุก action ใหม่ต้องแก้โค้ดส่วนรับ order (modifiability); coupling สูง; ต้องออกแบบ retry และ partial failure เอง (เช่น points ได้แต่อีเมลไม่ออก) |
| B | แยกความล้มเหลวของแต่ละ action (availability); เพิ่ม consumer ใหม่โดยไม่แก้ producer (modifiability); response ลูกค้าไม่รอ action หลังสั่งซื้อ (performance); consumer retry ได้อิสระ | ต้องมี broker เพิ่ม (cost, operability); eventual consistency; ต้องจัดการ message ซ้ำ (idempotent consumer); ถ้า publish ล้มหลัง order บันทึกแล้ว event อาจหาย; debug ข้ามหลาย service ยากขึ้น |
| C | แก้ปัญหา event หายจาก B ได้ (consistency); ได้ข้อดีของ B ทั้งหมด | ซับซ้อนที่สุด: ต้องมี outbox และ relay เพิ่ม (cost, operability); ต้องรู้ว่าที่เก็บข้อมูลของ order รองรับ transaction แบบนี้หรือไม่ (ยังไม่รู้); ถ้าความเสี่ยง event หายยอมรับได้ ก็เกินความจำเป็น |

## Decision

เลือก **Option B**: ส่วนที่รับคำสั่งซื้อ publish event `OrderPlaced` ไปที่ message broker และให้ `emailservice` (ใช้ MailFast), `loyaltyservice` และ analytics consumer แต่ละตัว subscribe แยกกัน. consumer ทุกตัวต้อง idempotent (ใช้ order id เป็น key กันการทำซ้ำ).

เหตุผลเทียบกับ decision drivers:

- Availability: MailFast เป็น external dependency ที่เราควบคุมไม่ได้ การแยกด้วย broker ทำให้ความล้มเหลวของอีเมลไม่ทำให้ order ล้ม
- Modifiability: requirement บอกว่าจะมี SMS, fraud scoring, recommendations เพิ่ม; ใน Option B แต่ละอย่างเป็น consumer ใหม่ ไม่ต้องแก้ producer. ใน Option A ต้องแก้ producer ทุกครั้ง
- Performance: action หลังสั่งซื้อออกจาก critical path

ไม่เลือก C ตอนนี้เพราะยังไม่มีข้อมูลว่า event หายระดับไหนยอมรับได้ และโครงสร้างข้อมูลของ order เป็นอย่างไร; ถ้าตอบ open question ข้อ 4 ว่า "ยอมรับไม่ได้" ให้ยกระดับเป็น C. หมายเหตุ: ข้อสรุปนี้ถ้า fraud scoring ต้องบล็อกก่อนยืนยัน order (ต้อง synchronous) จะต้องทบทวน เพราะ fraud scoring อาจไม่ใช่ post-order action แบบ asynchronous ได้ **[Inferred]**

## Consequences

- **Positive:** ความล้มเหลวหรือความช้าของ MailFast ไม่กระทบการสั่งซื้อ (quality attribute: availability)
- **Positive:** เพิ่ม consumer ใหม่ (SMS ฯลฯ) ได้โดยไม่แก้ producer (quality attribute: modifiability)
- **Positive:** latency ของการสั่งซื้อไม่รวมเวลาของ action หลังสั่งซื้อ (quality attribute: performance)
- **Positive:** ทดสอบ consumer แต่ละตัวแยกกันได้ (quality attribute: testability)
- **Negative:** ต้องมี broker เพิ่ม ต้อง deploy เฝ้าดู และดูแล (quality attribute: cost, operability)
- **Negative:** ผลของ action เป็น eventual consistency เช่น ลูกค้าอาจเห็นอีเมลหรือ points ช้ากว่าหน้ายืนยัน (quality attribute: data consistency)
- **Negative:** การ trace ปัญหาข้าม service ยากขึ้น ต้องมี correlation id และ monitoring (quality attribute: observability / modifiability)
- **Risks and mitigations:**
  - Event หายหากบันทึก order สำเร็จแต่ publish ล้ม: ลดด้วยการยกระดับเป็น Option C (outbox) ถ้าความเสี่ยงยอมรับไม่ได้
  - Message ซ้ำทำให้ได้ points สองครั้งหรืออีเมลซ้ำ: ทำ consumer ให้ idempotent โดยใช้ order id
  - Broker ล่ม: ต้องกำหนดพฤติกรรมของ producer (ยังไม่ตัดสินใจ ดู Open questions)
  - Message ที่ consumer ประมวลผลไม่สำเร็จซ้ำๆ: ใช้ retry แบบจำกัดจำนวนและ dead-letter queue พร้อม alert
  - Event มีข้อมูลส่วนบุคคล (อีเมลลูกค้า): กำหนดว่า payload มีอะไรบ้าง และส่งข้อมูลให้ MailFast เท่าที่จำเป็น

## Validation

- ทดสอบโดยจำลอง MailFast ล่ม/ช้า: การสั่งซื้อต้องสำเร็จและ latency ไม่เปลี่ยนอย่างมีนัยสำคัญ; อีเมลต้องถูกส่งเมื่อ MailFast กลับมา
- ทดสอบส่ง `OrderPlaced` ซ้ำ: points และอีเมลต้องไม่ซ้ำ
- วัด end-to-end delay จาก order ถึงอีเมล/points ให้อยู่ในเกณฑ์ที่ตกลง (ค่ายังไม่ระบุ)
- เพิ่ม consumer ตัวที่สี่ (เช่น SMS แบบ prototype) โดยไม่แก้ producer เพื่อยืนยัน modifiability
- ทบทวน ADR นี้เมื่อมีการตัดสินใจเรื่อง fraud scoring หรือหลังใช้งานจริงประมาณหนึ่งไตรมาส (วันที่ทบทวนให้ทีมกำหนด)

## Open questions

ข้อมูลที่ไม่ได้ให้มาและไม่ได้แต่งเอง:

1. **ระบบปัจจุบัน:** service ใดรับ/ยืนยัน order ตอนนี้ และเรียก `emailservice` ด้วย protocol อะไร? มี message broker ในระบบแล้วหรือไม่? (Not evidenced in the provided files)
2. **ปริมาณและเป้าหมาย:** จำนวน order ต่อวินาที/วัน, เป้า latency ของการสั่งซื้อ, ความหน่วงสูงสุดที่ยอมรับได้ของอีเมลและ points
3. **MailFast:** API/SLA/rate limit/ราคา/วิธี authenticate, ข้อกำหนดเรื่องข้อมูลส่วนบุคคลและสัญญา
4. **ความเสี่ยงยอมรับได้:** ยอมให้ event หาย หรืออีเมล/points ซ้ำหรือไม่ (กำหนดว่าต้อง Option C หรือไม่)
5. **`loyaltyservice`:** กฎการให้ points, data store, ต้องการ consistency ระดับใด, ใครเป็นเจ้าของ
6. **Analytics:** schema ของ event, ปลายทางที่ marketing ใช้, ต้องไม่ซ้ำหรือไม่, ข้อมูลส่วนบุคคลที่อนุญาต
7. **Fraud scoring:** ต้องเกิดก่อนยืนยัน order (synchronous) หรือหลัง (asynchronous)?
8. **ทีมและ operation:** ทีมมีทักษะและงบสำหรับ operate broker หรือไม่; เลือกเทคโนโลยี broker ใด (ADR นี้ยังไม่เลือกยี่ห้อ)
9. **Deciders:** ใครมีอำนาจตัดสินใจและอนุมัติ ADR นี้
10. **ลำดับความสำคัญของ quality attributes:** ลำดับใน Decision drivers เป็น assumption ต้องให้ stakeholder ยืนยัน
