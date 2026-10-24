**ภาษา:** ภาษาไทย | [English](bonus-recover-decision.en.md)

# โบนัส (ไม่บังคับ ทำที่บ้าน): กู้คืนการตัดสินใจที่ไม่ได้บันทึกไว้ (ประมาณ 15 นาที)

> **เชื่อมโยงกับการบรรยาย:** T5 (Dhar et al. สร้างการตัดสินใจจากโค้ดและแผนภาพเพื่อลด *การสูญหายของความรู้* หรือ knowledge vaporization), AICH4
> **คำถาม:** เมื่อ AI อธิบายว่า*ทำไม*ระบบจึงถูกสร้างแบบนี้ คำอธิบายส่วนใดมาจากหลักฐาน และส่วนใดเป็นการเดาอย่างมั่นใจ

## ลงมือทำ

เริ่มแชตหรือ session ใหม่ แล้วส่งข้อความนี้ให้ agent

```
Follow prompts/bonus-recover-decision.md
decision: Why do all services communicate synchronously over gRPC, using one shared demo.proto?
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/src/checkoutservice/main.go, system/online-boutique/src/frontend/main.go
language: Thai
```

บรรทัด `language: Thai` ทำให้ ADR ที่กู้คืนเป็นภาษาไทย หากต้องการภาษาอังกฤษให้ลบบรรทัดนี้

agent จะเขียน ADR ที่กู้คืนมา พร้อมตาราง **Claim check** ไว้ที่ [`outputs/bonus-recovered-adr.md`](outputs/bonus-recovered-adr.md)

## ตรวจสอบตาราง claim check

เลือกข้อกล่าวอ้าง**ห้าข้อ**จากตารางของ agent โดยเป็นข้อที่ agent ระบุว่า *Evidenced* (มีหลักฐาน) อย่างน้อยสองข้อ และ *Inferred* (อนุมานเอา) อย่างน้อยสองข้อ แล้วตรวจสอบแต่ละข้อด้วยตัวเอง

| # | ข้อกล่าวอ้าง (ย่อ) | AI ระบุว่า | นิสิตระบุว่า | เหตุผล (path ของไฟล์หรือการให้เหตุผล) |
|---|---|---|---|---|
| 1 | #8 frontend อ่าน `SHOPPING_ASSISTANT_SERVICE_ADDR` แต่ไม่ได้เชื่อมต่อด้วย gRPC | Evidenced | Evidenced | `src/frontend/main.go:139` มีแค่ `mustMapEnv` แต่ไม่มี `mustConnGRPC` (141–147); `src/frontend/handlers.go:464` เรียกด้วย `"http://"` จึงเป็น HTTP ซึ่ง AI ยืนยันไม่ได้เพราะ `handlers.go` อยู่นอกหลักฐานที่อนุญาต |
| 2 | #15 ไม่มี timeout ต่อ call: `convertCurrency` ใช้ `context.TODO()` และ context ที่มี timeout ใน `mustConnGRPC` ถูกทิ้ง | Evidenced | Evidenced | `src/checkoutservice/main.go:360` ใช้ `context.TODO()`; `:212` เขียน `_, cancel := context.WithTimeout(...)` |
| 3 | #16 ไม่มีการคืนเงินเมื่อ `shipOrder` ล้มหลังจาก `chargeCard` สำเร็จ | Evidenced | Evidenced | `src/checkoutservice/main.go:252–260` หลัง `chargeCard` ถ้า `shipOrder` ล้มจะ return error ทันที โดยไม่มีโค้ดคืนเงิน |
| 4 | #18 เลือกแบบ sync เพราะ `PlaceOrder` ต้องใช้ผลลัพธ์ทันที | Inferred | Inferred | data dependency ระหว่างขั้นมีหลักฐานจริง (ข้อ #7) แต่ไม่มี comment หรือเอกสารบอกว่านี่คือ*เหตุผล*ของการตัดสินใจ |
| 5 | #20 payment หรือ shipping ล้ม ทำให้ order ล้ม | Inferred | Evidenced | `src/checkoutservice/main.go:252–260` เขียน `return nil, status.Errorf(...)` ชัดเจน มีเพียงคำว่า "coupling ระดับระบบ" เท่านั้นที่เป็นการอนุมาน |

**นับ:** จากข้อกล่าวอ้างทั้งหมด agent ระบุว่า *Evidenced* กี่ข้อ 16 / 27 ตามที่นิสิตตรวจ 17 (16 ข้อเดิมยืนยันได้ทั้งหมดตามที่สุ่มตรวจ และ #20 ควรเป็น Evidenced) ไม่พบกรณีที่ AI ระบุการเดาว่าเป็น Evidenced

## สะท้อนคิด

1. agent เคยระบุการเดาว่าเป็น *Evidenced* หรือไม่ ยกตัวอย่าง หรืออธิบายว่าตรวจสอบอย่างไรว่าไม่มี
  - คำตอบคือไม่พบ ที่ตรงข้ามคือ AI ระวังเกินไป ข้อ #20 ติดป้ายไว้แค่ Inferred แต่ main.go:252-260 มีหลักฐานชัด
  - ตรวจโดยเปิดไฟล์และดูบรรทัดที่ AI อ้างไว้ทีละข้อ
2. หากนักพัฒนาใหม่อ่านเฉพาะ ADR ที่กู้คืนมานี้ เขาอาจเข้าใจผิดในเรื่องใด
  - เข้าใจว่า "ทุก service ใช้ gRPC" ทั้งที่ frontend เรียก shoppingassistant ผ่าน HTTP (handlers.go:464)
  - เข้าใจว่าเหตุผล เช่น "ใช้ sync เพราะ checkout ต้องได้ผลลัพธ์ทันที" เป็นเจตนาของผู้ออกแบบ ทั้งที่จริงเป็นแค่ Inferred
  - อาจคิดว่า Option 2–4 เคยถูกพิจารณาจริง ทั้งที่ AI สร้างขึ้นมาเพื่อเปรียบเทียบ
3. AI กู้คืนอะไรจากโค้ดได้บ้าง และอะไรที่มีเพียงสถาปนิกผู้ออกแบบเดิมเท่านั้นที่รู้ ยกตัวอย่างหนึ่งตัวอย่างจาก ADR ของนิสิต
  - ที่ AI กู้คืนได้: โครงสร้าง (ใครเรียกใคร, unary RPC, ไม่มี timeout, ไม่มีการคืนเงิน)
  - ที่ AI กู้คืนไม่ได้: เหตุผล ทางเลือกที่เคยพิจารณา และข้อจำกัดตอนนั้น
  - ตัวอย่างจาก ADR: เหตุผลที่ loyalty ต้องเป็น sync มาจากแคมเปญที่สัญญาว่าลูกค้าจะ "เห็นแต้มทันที" ถ้าอีกปีมีคนอ่านแค่โค้ด จะไม่มีทางรู้เหตุผลนี้ ซึ่งเป็นเหตุผลว่าทำไมต้องเขียน ADR
