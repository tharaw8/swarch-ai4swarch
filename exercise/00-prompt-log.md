**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชื่อ / รหัสนิสิต:** ธรา วรานุเศรษฐ์ / 6630132921
**agent ที่ใช้:** Claude code
**โมเดล:** Opus 5.5 และ Sonnet 5.5
**แพ็กเกจที่ใช้:** Pro

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code + Opus 5.5 | `prompts/1-…` run A | prompts/1-recover-architecture.md system/online-boutique/protos/demo.proto  | ใช่ | 2 | ถูกหมด แต่ไม่มีหลักฐาน |
| 2 | 1-B | Claude Code + Sonnet 5.5 | `prompts/1-…` run B | prompts/1-recover-architecture.md system/online-boutique/protos/demo.proto In system/online-boutique/kubernetes-manifests/: README.md, kustomization.yaml, adservice.yaml, cartservice.yaml, checkoutservice.yaml, currencyservice.yaml, emailservice.yaml, frontend.yaml, loadgenerator.yaml, paymentservice.yaml, productcatalogservice.yaml, recommendationservice.yaml, shippingservice.yaml | ใช่ | 4 | ขาดเส้น frontend→shoppingassistantservice (HTTP) และวาง loadgenerator ไว้ในขอบเขตระบบ |
| 3 | 2-a | Claude Code + Sonnet 5.5 | `prompts/2-…` run a | AGENTS.md, prompts/2-write-adr.md, docs/adr/template.md | ใช่ | 2 | ไม่มีตัวเลขจาก scenario เลย ใส่ Open questions 10 ข้อแทน |
| 4 | 2-b | Claude Code + Sonnet 5.5 | `prompts/2-…` run b | AGENTS.md, prompts/2-write-adr.md, docs/adr/template.md, docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go | ใช่ | 4 | เลือก RabbitMQ, ใช้ตัวเลขจาก scenario ครบ |
| 5 | 3 | Claude Code + Sonnet 5.5 | `prompts/3-…` | AGENTS.md, prompts/3-review-adr.md, docs/adr/0002-publish-order-placed-events-via-rabbitmq.md, docs/scenario.md | ใช่ | 4 | verdict: accept with changes; จับได้ว่า ADR อ้างเรื่อง Kafka โดยไม่มีหลักฐาน |
| 6 | bonus | Claude Code + Sonnet 5.5 | `prompts/bonus-…` | AGENTS.md, prompts/bonus-recover-decision.md, docs/adr/template.md, system/online-boutique/protos/demo.proto, system/online-boutique/src/checkoutservice/main.go, system/online-boutique/src/frontend/main.go | ใช่ | 3| claim 27 ข้อ (Evidenced 16 / Inferred 5 / Assumption 6); ข้อ #20 ควรเป็น Evidenced แต่ AI ให้แค่ Inferred |
| 7 | | | | | | | |
