**ภาษา:** ภาษาไทย | [English](01-recover-architecture.en.md)

# งานที่ 1: กู้คืนสถาปัตยกรรม (ประมาณ 15 นาที)

> **เชื่อมโยงกับการบรรยาย:** T7 การกู้คืนสถาปัตยกรรม, AICH2 (การตามรอย), AICH3 (บริบท) และข้อค้นพบจาก JSS ว่า 93% ของงานวิจัยไม่เคยตรวจสอบผลลัพธ์จาก AI
> **คำถาม:** AI กู้คืนสถาปัตยกรรมได้มากแค่ไหน และขึ้นอยู่กับหลักฐานที่ AI ได้อ่านมากเพียงใด

**ระหว่างทำงานนี้ ห้ามเปิด repository ต้นฉบับของ Online Boutique** เพราะ README ของต้นฉบับมีแผนภาพสถาปัตยกรรมที่นิสิตกำลังกู้คืนอยู่

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่างให้ agent โดย agent จะทำตาม [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) และเขียนผลลัพธ์ไว้ที่ `exercise/outputs/1<run>-architecture.md`

**Run A: เฉพาะ interface**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interface + ไฟล์ตั้งค่าการ deploy**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

ระหว่างที่ agent ทำงาน ให้สังเกตว่า agent เปิดไฟล์ใดบ้าง หาก agent อ่านไฟล์นอกเหนือจากหลักฐานที่อนุญาต ให้บันทึกไว้ใน prompt log

## ผลลัพธ์

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

เปิดแต่ละไฟล์ด้วย **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) เพื่อดูแผนภาพ

## ตรวจสอบ Run B ด้วยตัวเอง (ส่วนที่สำคัญที่สุด)

ตรวจสอบ dependency ในตารางของ Run B **อย่างน้อยห้ารายการ** เทียบกับไฟล์จริง โดยเปิดไฟล์ที่ AI อ้างถึงและหาหลักฐาน นิสิตสามารถตรวจซอร์สโค้ดของ service ได้ด้วย เช่น `src/checkoutservice/main.go`

| ผู้เรียก → ผู้ถูกเรียก | หลักฐานที่ AI อ้าง | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ |
|---|---|---|---|
| frontend → cartservice | frontend.yaml CART_SERVICE_ADDR | ✅ | src/frontend/rpc.go:58 เรียก GetCart |
| checkoutservice → paymentservice | checkoutservice.yaml PAYMENT_SERVICE_ADDR | ✅ | src/checkoutservice/main.go:370 เรียก Charge |
| checkoutservice → emailservice | EMAIL_SERVICE_ADDR | ✅ | main.go:380 เรียก SendOrderConfirmation |
| recommendationservice → productcatalogservice | PRODUCT_CATALOG_SERVICE_ADDR | ✅ | recommendation_server.py:131-136 |
| cartservice → redis-cart | REDIS_ADDR=redis-cart:6379 | ✅ | src/cartservice/src/Startup.cs:34-41 ใช้ Redis cache |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย):
  - ขาดเส้น frontend → shoppingassistantservice ซึ่งเป็น HTTP ไม่ใช่ gRPC (src/frontend/handlers.go:464-472) เปิดใช้ก็ต่อเมื่อตั้ง ENABLE_ASSISTANT=true
  - loadgenerator ถูกวางไว้ในขอบเขตระบบ แต่จริง ๆ เป็นเครื่องมือทดสอบ Locust (src/loadgenerator/Dockerfile:52)
  - ไม่ได้พูดถึงเส้นที่เปิดใช้ได้ตามตัวเลือก ได้แก่ tracing collector, packaging service และการเก็บตะกร้าบน Spanner/AlloyDB แทน Redis
  - redis-cart เก็บข้อมูลแบบ emptyDir ข้อนี้ AI บอกถูกว่าตะกร้าหายเมื่อรีสตาร์ต

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด
  - .proto บอกแค่ว่าแต่ละ service ให้บริการอะไร แต่ไม่บอกว่าใครเรียกใคร ใช้ data store อะไร และ frontend มีอยู่หรือไม่ เพราะ frontend ไม่มี proto ข้อมูล wiring พวกนี้อยู่ใน env var ของไฟล์ deploy
2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ
   - ถ้าแผนภาพไม่มี shoppingassistantservice สมาชิกใหม่จะคิดว่าทุกเส้นเป็น gRPC และระบบไม่มีการเรียกออกภายนอก แต่ถ้าเปิดฟีเจอร์ assistant ระบบจะเรียก HTTP ไปยัง AlloyDB และ Google AI ด้วย
