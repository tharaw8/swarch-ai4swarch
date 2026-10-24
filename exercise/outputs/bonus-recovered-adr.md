Agent: Claude Code, model Sonnet 5.5 (claude-sonnet-5-5)

# ADR-NNNN: ให้ทุก service สื่อสารกันแบบ synchronous ผ่าน gRPC โดยใช้ `demo.proto` ไฟล์เดียวร่วมกัน

- **Status:** Reconstructed
- **Date:** Not evidenced in the provided files
- **Deciders:** Not evidenced in the provided files

> หมายเหตุ: ADR นี้สร้างกลับจากโค้ดเท่านั้น เหตุผล (rationale) ส่วนใหญ่เป็น **Inferred** ดูตาราง Claim check ท้ายไฟล์ ตัวเลขในวงเล็บ [n] อ้างถึงแถวในตารางนั้น

## Context and problem

- `demo.proto` ไฟล์เดียว (package `hipstershop`) นิยาม service 9 ตัว: `CartService`, `RecommendationService`, `ProductCatalogService`, `ShippingService`, `CurrencyService`, `PaymentService`, `EmailService`, `CheckoutService`, `AdService` [1]. ทุก rpc เป็น unary (ไม่มี `stream`) [2].
- `frontend` รับ HTTP จากเบราว์เซอร์ผ่าน `gorilla/mux` [3] แล้วเรียก backend 7 ตัวผ่าน gRPC (`currency`, `productCatalog`, `cart`, `recommendation`, `shipping`, `checkout`, `ad`) [4].
- `checkoutservice` ถือ gRPC client ไป 6 service (`shipping`, `productCatalog`, `cart`, `currency`, `email`, `payment`) [5]. `PlaceOrder` เรียกต่อกันเป็นลำดับแบบ blocking: อ่าน cart, ดึงสินค้า, แปลงสกุลเงิน, ขอ quote ค่าส่ง, charge บัตร, ship, ล้าง cart, ส่งอีเมล [6] และต้องใช้ผลลัพธ์ของแต่ละขั้น (เช่น tracking id ใส่ใน `OrderResult`) [7].
- ข้อสังเกตที่ทำให้คำถาม "ทุก service" ไม่จริงทั้งหมด: `frontend` อ่าน `SHOPPING_ASSISTANT_SERVICE_ADDR` แต่ไม่สร้าง gRPC connection ใน `main.go` และ `demo.proto` ไม่มี service นี้ [8] ดังนั้นการสื่อสารนี้ยืนยันได้เฉพาะ 2 service ที่อ่านเท่านั้น (frontend, checkoutservice)
- ปัญหาที่ต้องตัดสินใจ: จะให้ service หลายตัวคุยกันด้วยวิธีและสัญญา (contract) แบบใด

## Decision drivers

- ความชัดเจนและความสอดคล้องของ interface ระหว่าง service (modifiability) [Inferred]
- ผลลัพธ์ที่ checkout ต้องได้ทันทีเพื่อทำขั้นต่อไป (correctness ของ flow) [7]
- latency ของการสั่งซื้อ (performance) [Assumption]
- observability: trace ต้องผ่านทุก hop [9]
- ความง่ายในการ deploy/config (ที่อยู่ service มาจาก env var) [10]

## Considered options

1. gRPC แบบ synchronous + `demo.proto` ไฟล์เดียว (สิ่งที่ระบบใช้จริง)
2. REST/JSON over HTTP แบบ synchronous + สัญญาแยกต่อ service
3. Asynchronous messaging ผ่าน message broker (event-driven)
4. gRPC แบบ synchronous แต่แยกไฟล์ `.proto` ต่อ service

Option 2-4 ไม่ปรากฏในไฟล์หลักฐาน เป็นทางเลือกที่สร้างขึ้นเพื่อเปรียบเทียบ [24][25][26]

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. gRPC sync + proto เดียว | สัญญาเป็น typed และอยู่ที่เดียว ทุก service generate client/server จากไฟล์เดียวกัน [11][22]; ได้ผลลัพธ์ทันทีสำหรับ flow ของ checkout [7]; trace context ส่งผ่าน gRPC ได้ด้วย `otelgrpc` [9] | caller ผูกกับ callee ตอน runtime (availability); latency สะสมตาม chain [23]; แก้ proto กระทบทุก service [26]; ไม่มี timeout/retry ที่เห็นได้ [15] |
| 2. REST/JSON + สัญญาแยก | เครื่องมือแพร่หลาย debug ง่าย [Assumption]; service เปลี่ยนสัญญาอิสระกว่า | ไม่มี typed contract กลางโดยอัตโนมัติ; payload ใหญ่/ช้ากว่า binary [27] |
| 3. Async ผ่าน broker | ลด coupling เชิงเวลา; ล้มเหลวแยกกันได้ดีกว่า [Assumption] | checkout ต้องการผลลัพธ์ทันที (quote, tx id) จึงซับซ้อนขึ้น [7]; ต้องเพิ่มโครงสร้างพื้นฐาน broker (cost) |
| 4. gRPC sync + proto แยกต่อ service | แต่ละทีมแก้สัญญาตัวเองโดยไม่กระทบอื่น [26] | หลายไฟล์ต้องดูแล message ร่วม (เช่น `Money`, `Address`, `CartItem` ถูกใช้ข้าม service ใน `demo.proto`) อาจซ้ำหรือเพี้ยนกัน [Inferred] |

## Decision

เลือก **Option 1**: gRPC แบบ synchronous และใช้ `demo.proto` ไฟล์เดียว

ที่เห็นจากโค้ด: ทั้ง `frontend` และ `checkoutservice` สร้าง client ด้วย `grpc.NewClient(...)` พร้อม `otelgrpc.NewClientHandler()` [9][12] และ generate stub จาก `genproto` ต่อ service [11].

เหตุผลที่ **Inferred** (ไม่มีเอกสารยืนยัน):
- สัญญาเดียวทำให้ message ร่วมอย่าง `Money`, `Address`, `CartItem` เหมือนกันทุก service และ interface ตรวจสอบได้ตอน build [17]
- Synchronous เหมาะเพราะ `PlaceOrder` ต้องใช้ผลของแต่ละ call ก่อนทำขั้นต่อไป [7][18]
- ไม่พบการอภิปรายเรื่อง polyglot หรือความเร็วในไฟล์ที่ให้ เหตุผลด้านนี้จึงเป็นการคาดเดา [19]

## Consequences

- **Positive:** สัญญา typed และรวมศูนย์ (modifiability) [22]; flow ของ checkout ตรงไปตรงมา อ่านและ debug ตามลำดับได้ (modifiability/testability) [6]; trace ต่อเนื่องข้าม service (observability) [9]
- **Negative:**
  - failure ของ payment/shipping ทำให้ `PlaceOrder` ล้มทันที (availability) [6][20]; มี coupling เชิงเวลา [Inferred]
  - ไม่เห็น timeout/retry ใน call ของ checkout: `convertCurrency` ใช้ `context.TODO()` และ context ที่มี timeout ใน `mustConnGRPC` ถูกทิ้ง (`_`) (availability/performance) [15]
  - ถ้า `shipOrder` ล้มหลัง `chargeCard` สำเร็จ ไม่มีโค้ดคืนเงินใน `main.go` (correctness/availability) [16]
  - latency สะสมจากการเรียกต่อกันหลายครั้งต่อ 1 order (performance) [23]
  - แก้ `demo.proto` ที่เดียวกระทบทุก service ที่ใช้ (modifiability) [25]
- **Risks and mitigations:**
  - บางขั้นถูก "ยอมให้พลาด" แล้ว: `emptyUserCart` ถูกเมิน error และ `sendOrderConfirmation` แค่ log warning [13] ซึ่งลดผลกระทบต่อ availability ของ order
  - ใช้ `insecure.NewCredentials()` (ไม่มี TLS ใน client) [14]; เหตุผลไม่ปรากฏ อาจพึ่งความปลอดภัยของเครือข่ายภายใน [Inferred][21]
  - Mitigation ที่ควรพิจารณา (ข้อเสนอ ไม่ใช่สิ่งที่โค้ดทำ): deadline/retry ต่อ call, ทำ saga/compensation สำหรับ payment-shipping, backward-compatible proto changes

## Validation

Not evidenced in the provided files (ไม่มีการทดสอบหรือเมตริกในหลักฐาน). ข้อเสนอ: load test `PlaceOrder` วัด latency p95, fault injection ปิด `shipping`/`payment` ดูพฤติกรรม, และ review เมื่อมี flow ที่ไม่ต้องการผลทันที (เช่น อีเมล) ว่าควรเป็น async หรือไม่

## Claim check

| # | Claim | Type | Evidence |
|---|---|---|---|
| 1 | `demo.proto` ไฟล์เดียวนิยาม 9 service ใน package `hipstershop` | Evidenced | `system/online-boutique/protos/demo.proto` (`package hipstershop`, `service CartService` ... `service AdService`) |
| 2 | ทุก rpc เป็น unary ไม่มี streaming | Evidenced | `system/online-boutique/protos/demo.proto` (ทุก `rpc ... returns ...` ไม่มี `stream`) |
| 3 | frontend รับ HTTP จากเบราว์เซอร์ | Evidenced | `system/online-boutique/src/frontend/main.go` (`mux.NewRouter`, `http.ListenAndServe`, `r.HandleFunc("/cart/checkout", ...)`) |
| 4 | frontend เชื่อม gRPC ไป 7 service | Evidenced | `system/online-boutique/src/frontend/main.go` (`mustConnGRPC` x7: currency, productCatalog, cart, recommendation, shipping, checkout, ad) |
| 5 | checkoutservice ถือ gRPC client ไป 6 service | Evidenced | `system/online-boutique/src/checkoutservice/main.go` (struct `checkoutService`, `mustConnGRPC` x6) |
| 6 | `PlaceOrder` เรียก dependency แบบ blocking ต่อเนื่องเป็นลำดับ | Evidenced | `system/online-boutique/src/checkoutservice/main.go` (`PlaceOrder`, `prepareOrderItemsAndShippingQuoteFromCart`, `chargeCard`, `shipOrder`) |
| 7 | แต่ละขั้นของ checkout ใช้ผลลัพธ์ของ call ก่อนหน้า (ยอดรวมใช้ตอน charge, tracking id ใส่ใน `OrderResult`) | Evidenced | `system/online-boutique/src/checkoutservice/main.go` (`total`, `chargeCard(ctx,&total,...)`, `shippingTrackingID`, `orderResult`) |
| 8 | `frontend` อ่าน `SHOPPING_ASSISTANT_SERVICE_ADDR` แต่ไม่มี gRPC conn และไม่มีใน `demo.proto` ดังนั้น "ทุก service ใช้ gRPC" ยืนยันไม่ได้ | Evidenced | `system/online-boutique/src/frontend/main.go` (`shoppingAssistantSvcAddr`, ไม่มี `mustConnGRPC` ของมัน); `demo.proto` (ไม่มี service ดังกล่าว) |
| 9 | trace context ส่งผ่าน gRPC ด้วย OpenTelemetry | Evidenced | `system/online-boutique/src/checkoutservice/main.go` (`otelgrpc.NewServerHandler`, `otelgrpc.NewClientHandler`, `propagation.TraceContext{}`); `src/frontend/main.go` (`otelgrpc.NewClientHandler`) |
| 10 | ที่อยู่ของ service อื่นมาจาก env var และ panic ถ้าไม่ตั้ง | Evidenced | `system/online-boutique/src/checkoutservice/main.go` และ `src/frontend/main.go` (`mustMapEnv`, `*_SERVICE_ADDR`) |
| 11 | stub ของ checkout generate จาก `genproto` ภายในแต่ละ service (import `src/checkoutservice/genproto`) | Evidenced | `system/online-boutique/src/checkoutservice/main.go` (import `pb ".../src/checkoutservice/genproto"`) |
| 12 | client ใช้ `grpc.NewClient` กับ `insecure.NewCredentials()` | Evidenced | `system/online-boutique/src/checkoutservice/main.go` และ `src/frontend/main.go` (`mustConnGRPC`) |
| 13 | `emptyUserCart` ถูกเมิน error และ `sendOrderConfirmation` ล้มแล้วแค่ log warning ส่วนขั้นอื่นล้มแล้วคืน error | Evidenced | `system/online-boutique/src/checkoutservice/main.go` (`_ = cs.emptyUserCart(...)`, `log.Warnf("failed to send order confirmation...")`, `return nil, status.Errorf(...)`) |
| 14 | checkoutservice มี gRPC health server | Evidenced | `system/online-boutique/src/checkoutservice/main.go` (`health.NewServer`, `Check`) |
| 15 | ไม่เห็น timeout/retry ต่อ call: `convertCurrency` ใช้ `context.TODO()`; context timeout ใน `mustConnGRPC` ถูกทิ้ง | Evidenced | `system/online-boutique/src/checkoutservice/main.go` (`convertCurrency`, `_, cancel := context.WithTimeout`) |
| 16 | ไม่มีโค้ดคืนเงิน/compensation ถ้า `shipOrder` ล้มหลัง `chargeCard` สำเร็จ | Evidenced | `system/online-boutique/src/checkoutservice/main.go` (`PlaceOrder`: หลัง `chargeCard` ไป `shipOrder` แล้ว `return nil, status.Errorf(codes.Unavailable, ...)`) |
| 17 | เหตุผลของ proto เดียว: ให้ message ร่วม (`Money`, `Address`, `CartItem`) สอดคล้องกันและตรวจสอบตอน build | Inferred | `demo.proto` แสดงว่ามี message ถูกใช้ข้าม service แต่ไม่มีไฟล์ระบุเหตุผล |
| 18 | เหตุผลที่เลือก sync: `PlaceOrder` ต้องใช้ผลทันทีจึงเหมาะกับ request/response | Inferred | `checkoutservice/main.go` แสดง data dependency แต่ไม่มี comment ระบุว่าเป็นเหตุผลของการตัดสินใจ |
| 19 | ไม่มีการพิจารณา polyglot/ความเร็วใน evidence ดังนั้นเหตุผลเหล่านี้เป็นการคาดเดา | Inferred | ทั้งสามไฟล์ไม่มีคำอธิบายเหตุผล (frontend/checkout เป็น Go ทั้งคู่) |
| 20 | ความล้มเหลวของ payment/shipping ทำให้ order ล้ม สร้าง coupling เชิงเวลา/ผลกระทบต่อ availability | Inferred | `checkoutservice/main.go` (`PlaceOrder` return error) แสดงเฉพาะ call-path ของ checkout ไม่ได้วัดผลระดับระบบ |
| 21 | การใช้ insecure credentials น่าจะพึ่งความปลอดภัยของเครือข่ายภายใน | Inferred | `mustConnGRPC` ใช้ `insecure.NewCredentials()` แต่ไม่มีไฟล์ระบุเหตุผล |
| 22 | ประโยชน์ทั่วไปของ typed contract กลาง: ตรวจ interface ตอน build | Assumption | ความรู้ทั่วไปเกี่ยวกับ protobuf/gRPC |
| 23 | latency สะสมเมื่อเรียกต่อกันหลาย hop | Assumption | ความรู้ทั่วไปของ synchronous call chain |
| 24 | REST/JSON เป็นทางเลือก (Option 2) ที่ debug ง่ายและเครื่องมือแพร่หลาย | Assumption | ความรู้ทั่วไป ไม่ใช่เฉพาะระบบนี้ |
| 25 | broker แบบ async (Option 3) ลด coupling เชิงเวลาและแยก failure ได้ดีกว่า | Assumption | ความรู้ทั่วไป |
| 26 | แก้ไฟล์ proto กลางกระทบทุก service ที่ใช้ ส่วน proto แยกต่อ service ให้อิสระกว่า | Assumption | ความรู้ทั่วไป |
| 27 | protobuf binary payload เล็ก/เร็วกว่า JSON | Assumption | ความรู้ทั่วไป |
