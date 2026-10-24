**ภาษา:** ภาษาไทย | [English](02-write-adr.en.md)

# งานที่ 2: เขียน ADR สำหรับความต้องการใหม่ (ประมาณ 20 นาที)

> **เชื่อมโยงกับการบรรยาย:** T3 (GPT-4 แบบ zero-shot ให้ผลที่สอดคล้องกัน แต่ *ครบถ้วนน้อยกว่ามนุษย์* ส่วน few-shot ช่วยลดช่องว่างได้มาก), AICH3 (บริบท), สัปดาห์ที่ 10 (message broker)
> **คำถาม:** ADR ที่ AI เขียนดีขึ้นแค่ไหน เมื่อให้ตัวอย่างและบริบทจริงแก่ AI

อ่าน [สถานการณ์ (ภาษาไทย)](../docs/scenario.th.md) ก่อน: ความต้องการ **R-12** ว่าด้วยสิ่งที่เกิดขึ้นหลังจากลูกค้าสั่งซื้อสินค้า

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่าง โดยแทนที่ `<paste R-12 here>` ด้วยข้อความ R-12 ภาษาอังกฤษที่อยู่ในกรอบอ้างอิงของ scenario

**Run a: ไม่มีตัวอย่าง (zero-shot)** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
language: Thai
```

**Run b: มีตัวอย่าง + บริบท** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
language: Thai
```

> บรรทัด `language: Thai` ทำให้ agent เขียน ADR เป็นภาษาไทย (หัวข้อของ template และชื่อทางเทคนิคยังคงเป็นภาษาอังกฤษ) หากต้องการภาษาอังกฤษให้ลบบรรทัดนี้ **ใช้ภาษาเดียวกันทั้ง Run a และ Run b** เพื่อให้เปรียบเทียบกันได้

> ใน Run a ให้ตรวจว่า agent ไม่ได้อ่านไฟล์อื่นนอกจาก template จริง ๆ หาก agent เปิด `docs/scenario.md` ให้บันทึกไว้ใน prompt log เพราะจะทำให้การเปรียบเทียบเปลี่ยนไป

## เปรียบเทียบ a กับ b

| เกณฑ์ | Run a | Run b |
|---|---|---|
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast) | ไม่ใช้เลย เพราะไม่เห็น scenario | ใช้ครบ: 500 ms / 300 ms, 200 orders/s, MailFast 1.5 s / 8 s / 0.5 % / 50/s, loyalty 1 นาที, ทีม 4 คน |
| ทางเลือกที่พิจารณา (ระบุ) | 3 ทาง: sync call, broker (ไม่ระบุชื่อ), broker + outbox | 4 ทาง: sync gRPC, goroutine, RabbitMQ, Kafka (ไม่มี outbox เป็นทางเลือก) |
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร | ใช้ idempotent consumer โดยใช้ order id เป็น key (ไม่รู้ข้อกำหนด 1 นาที); ไม่พูดถึง rate limit เลย | loyalty ใช้ unique `order_id`; email ใช้ queue เป็น buffer และจำกัดที่ 50/s แต่ไม่ได้คิดกรณีมีหลาย replica; ปัญหา event หายตอน publish ล้มโยนไปไว้ใน Open questions |
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) | แทบไม่มี ส่วนที่ไม่รู้ใส่ไว้ใน Open questions 10 ข้อ (ADR จึงกว้างและทั่วไป) | มีอ้างว่า "หลัง commit order" ทั้งที่ `checkoutservice` ไม่มี data store; เอา p95 มาบวกกันตรง ๆ (300 + 1,500 ms); เล่าสถานะปัจจุบันไม่ครบ (ขาด cart, catalog, currency); HTTPS ของ MailFast ติดป้าย Assumption ไว้แล้ว |

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| # | สิ่งที่เปลี่ยนจากร่างของ AI | เหตุผล |
|---|---|---|
| 1 | ระบุว่า `checkoutservice` ไม่มี data store และเขียน "ข้อจำกัดที่ยอมรับ" ว่าฝั่ง producer ยังไม่ใช่ exactly once | ร่างของ AI อ้างถึงการ "commit order" ซึ่งไม่มีอยู่จริงใน `main.go` ถ้าไม่เขียนไว้ outbox จะดูเหมือนทำได้ง่าย |
| 2 | เพิ่ม publisher confirms, timeout 100 ms และ buffer สำหรับ retry พร้อม metric | ร่างเดิมไม่บอกว่าจะกันไม่ให้ publish กิน latency เกิน headroom 200 ms อย่างไร |
| 3 | กำหนดให้ rate limit 50/s เป็นค่ารวมทุก replica ของ `emailservice` | ถ้าแต่ละ pod จำกัด 50/s เอง ค่ารวมจะเกิน rate limit ของ MailFast |
| 4 | เขียนสถานะปัจจุบันให้ครบ (cart, catalog, currency) พร้อมเลขบรรทัด และแก้แผนภาพให้มี `frontend` | ทำให้หลักฐานตรวจสอบได้ และแผนภาพถูกต้อง |
