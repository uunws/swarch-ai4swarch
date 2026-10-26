Agent: Claude Code — Model: Claude Opus 5.5 (`claude-opus-5-5`)

# ADR-R001: Use synchronous gRPC with one shared `demo.proto` for all inter-service communication

- **Status:** Reconstructed
- **Date:** Not evidenced in the provided files (header ลิขสิทธิ์ระบุ 2018 ใน `main.go` ทั้งสองไฟล์ และ 2020 ใน `demo.proto`)
- **Deciders:** Not evidenced in the provided files (copyright holder คือ Google LLC — **[Inferred]** ว่าเป็นทีมผู้พัฒนาของ Google)

## Context and problem

ข้อเท็จจริงจากไฟล์หลักฐาน:

- ระบบแบ่งเป็น 9 gRPC service ที่นิยามไว้ในไฟล์เดียว `system/online-boutique/protos/demo.proto` ภายใต้ `package hipstershop` ได้แก่ `CartService`, `RecommendationService`, `ProductCatalogService`, `ShippingService`, `CurrencyService`, `PaymentService`, `EmailService`, `CheckoutService`, `AdService`
- ทุก RPC เป็นแบบ unary (request → response) ไม่มี `stream` เลยใน `demo.proto`
- message หลายตัวถูกใช้ร่วมกันข้าม service เช่น `Money` (ใช้ใน `Product`, `GetQuoteResponse`, `ChargeRequest`, `OrderItem`), `Address`, `CartItem`, `Empty`
- `frontend` รับผู้ใช้ผ่าน HTTP (`gorilla/mux`, `http.ListenAndServe`) แล้วเรียก backend 7 ตัวผ่าน gRPC (`frontend/main.go`, `mustConnGRPC` บรรทัด 141–147)
- `checkoutservice` เป็น orchestrator: `PlaceOrder` เรียก 6 service ต่อกันแบบ synchronous ตามลำดับ cart → product catalog (+ currency ต่อสินค้าแต่ละชิ้น) → shipping quote → currency → payment → shipping → cart empty → email (`checkoutservice/main.go`, `PlaceOrder`, `prepareOrderItemsAndShippingQuoteFromCart`)
- service ไม่ได้แชร์ code library กัน มีเพียง contract จาก proto: comment "this method is duplicated in other microservices using Go since they are not sharing packages" (`initProfiling` ทั้งสองไฟล์) และ checkout ใช้ stub ที่ generate ไว้ในโฟลเดอร์ตัวเอง (`import pb ".../src/checkoutservice/genproto"`)

ปัญหาที่ต้องตัดสินใจ: service หลายตัว (ภาษาอาจต่างกัน — **[Inferred]**) จะสื่อสารกันด้วย protocol อะไร และจะนิยามและแชร์ interface contract อย่างไร

## Decision drivers

1. **Interoperability / modifiability ของ contract** — ต้องมี contract ที่มี type ชัดเจนและ generate client/server ได้ในหลายภาษา **[Inferred]**
2. **ความเรียบง่ายในการพัฒนาและอธิบาย** — ระบบเป็น demo ที่ flow อ่านเข้าใจได้แบบ request/response **[Inferred]**
3. **Observability** — trace request ข้าม service ได้ (มี `otelgrpc` ทั้ง client และ server ใน evidence)
4. **Performance (latency)** — การเรียกภายใน cluster ควรมี overhead ต่ำ **[Assumption]**
5. **Consistency ของ order flow** — checkout ต้องรู้ผลการตัดบัตรและการจัดส่งก่อนตอบผู้ใช้ (`PlaceOrder` คืน `OrderResult` ที่มี `ShippingTrackingId`)

## Considered options

1. Synchronous gRPC + **ไฟล์ `demo.proto` เดียวที่แชร์ร่วมกัน** (ทางที่ระบบใช้จริง)
2. Synchronous gRPC + **ไฟล์ `.proto` แยกต่อ service** (แต่ละ service เป็นเจ้าของ contract ของตัวเอง)
3. Synchronous REST/JSON over HTTP พร้อม OpenAPI spec ต่อ service
4. ผสม asynchronous messaging (message broker เช่น RabbitMQ) สำหรับขั้นตอนที่ไม่จำเป็นต้องรอผล เช่น email confirmation

> ตัวเลือก 2–4 เป็นทางเลือกที่สร้างขึ้นใหม่ ไม่มีหลักฐานในไฟล์ว่าทีมเคยพิจารณา — **[Inferred]**

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. gRPC + shared `demo.proto` | contract เดียวเห็นภาพรวมทั้งระบบ; reuse type กลาง (`Money`, `Address`) ทำให้ไม่ต้อง map ข้อมูล; generate stub ได้หลายภาษา; binary protobuf + HTTP/2 overhead ต่ำ | ทุก service ผูกกับไฟล์เดียว (shared-contract coupling) แก้ message กลางกระทบหลาย service; ไม่มี ownership ของ contract ต่อ service; release ต้องประสานกัน |
| 2. gRPC + proto แยกต่อ service | ownership ชัด; versioning อิสระต่อ service; ลด blast radius ของการเปลี่ยน contract | ต้องจัดการ import/แชร์ type กลางเช่น `Money`; จัดการ repo/build ซับซ้อนขึ้นสำหรับ demo |
| 3. REST/JSON + OpenAPI | debug ง่ายด้วย curl/browser; tooling แพร่หลาย; ไม่ต้องใช้ HTTP/2 | payload ใหญ่และ parse ช้ากว่า; typing อ่อนกว่า, codegen ไม่เป็นมาตรฐานเท่า; ต้องเขียน serialization เอง |
| 4. ผสม async messaging | ลด temporal coupling: email ล่มไม่กระทบ checkout; รองรับ load spike ด้วยคิว | ต้องเพิ่ม broker (ต้นทุนการ operate); eventual consistency; flow ยากต่อการ trace และอธิบาย |

## Decision

เลือก **Option 1**: ทุก service สื่อสารด้วย synchronous unary gRPC ตาม contract ใน `demo.proto` ไฟล์เดียว; ผู้ใช้ภายนอกคุยกับ `frontend` ผ่าน HTTP เท่านั้น

เหตุผลที่สร้างขึ้นใหม่ (ส่วนใหญ่เป็น **[Inferred]**):

- ไฟล์เดียวทำให้ contract ของทั้งระบบอ่านได้ในที่เดียว เหมาะกับระบบที่ใช้เป็นตัวอย่าง/demo
- type กลางเช่น `Money` (พร้อมกฎ `units`/`nanos` ที่อธิบายใน comment) ใช้ร่วมกันได้โดยไม่ต้องแปลงระหว่าง service
- order flow ต้องการผลทันที (transaction id, tracking id) จึงเข้ากับ request/response แบบ synchronous
- gRPC เข้ากับ OpenTelemetry instrumentation (`otelgrpc`) และ gRPC health protocol (`healthpb.RegisterHealthServer`) ที่ใช้อยู่

## Consequences

- **Positive:** contract มี type ชัดเจนและ generate stub ได้ (`pb.NewCartServiceClient`, `pb.RegisterCheckoutServiceServer`) (quality attribute: modifiability, interoperability)
- **Positive:** trace ข้าม service ได้ เพราะทุก connection ใส่ `otelgrpc.NewClientHandler()` / `NewServerHandler()` และตั้ง propagator `TraceContext` (quality attribute: observability)
- **Positive:** flow ของ checkout อ่านเป็นลำดับเดียวในฟังก์ชัน `PlaceOrder` (quality attribute: understandability / modifiability)
- **Negative:** availability ของ checkout ขึ้นกับ service ปลายทางทั้ง 6 ตัว — cart, product catalog, currency, shipping หรือ payment ตัวใดล้มก็ทำให้ `PlaceOrder` ล้ม (quality attribute: availability)
- **Negative:** latency สะสมตามจำนวน call ต่อเนื่อง; `prepOrderItems` เรียก `GetProduct` และ `Convert` ต่อสินค้าทีละชิ้นในลูป (N+1 calls) (quality attribute: performance)
- **Negative:** แก้ message กลางใน `demo.proto` (เช่น `Money`, `Address`) ต้อง regenerate และ redeploy หลาย service (quality attribute: modifiability, deployability)
- **Negative:** ไม่มี compensation: ถ้า `shipOrder` ล้มหลัง `chargeCard` สำเร็จ จะคืน `codes.Unavailable` ทั้งที่ตัดเงินแล้ว (quality attribute: reliability / data consistency)
- **Negative:** ใช้ `insecure.NewCredentials()` — traffic ระหว่าง service ไม่เข้ารหัสที่ระดับ application (quality attribute: security)
- **Risks and mitigations:**
  - ไม่มี deadline ต่อ call: `context.WithTimeout` ใน `mustConnGRPC` ถูกทิ้ง (`_, cancel := ...`) และ `convertCurrency` ใช้ `context.TODO()` แทน `ctx` ทำให้ deadline/trace ของ request ไม่ถูกส่งต่อ → mitigation: ส่ง `ctx` ต่อและตั้ง deadline ต่อ RPC **[Inferred]**
  - ไม่พบ retry หรือ circuit breaker ใน evidence → mitigation: ใช้ gRPC retry policy หรือ service mesh **[Inferred]**
  - email เป็น side effect ที่ไม่ควร block order: โค้ดปัจจุบันแค่ log เมื่อ `sendOrderConfirmation` ล้ม; `emptyUserCart` ก็ทิ้ง error (`_ =`) → อาจย้ายไปเป็น async (Option 4) ในอนาคต **[Inferred]**
  - การเปลี่ยน contract → mitigation: ใช้กฎ backward-compatible ของ protobuf (เพิ่ม field ใหม่, ไม่ reuse field number) **[Assumption]**

## Validation

Not evidenced in the provided files (ไม่พบ load test, SLO หรือ metric ที่ใช้ยืนยัน). ข้อเสนอ **[Inferred]**:

- วัด p95/p99 latency ของ `PlaceOrder` จาก trace ของ `otelgrpc` ภายใต้ load test
- fault-injection test: ปิด `currencyservice` หรือ `paymentservice` แล้ววัด error rate ของ checkout
- contract check ใน CI: ตรวจ breaking change ของ `demo.proto` ก่อน merge

## Claim check

| # | Claim | Type | Evidence |
|---|---|---|---|
| 1 | มี 9 gRPC service นิยามในไฟล์เดียวภายใต้ `package hipstershop` | Evidenced | `protos/demo.proto` (`package hipstershop`, `service CartService` … `service AdService`) |
| 2 | ทุก RPC เป็น unary ไม่มี streaming | Evidenced | `protos/demo.proto` (ไม่มีคำว่า `stream` ในทุก `rpc`) |
| 3 | message กลาง (`Money`, `Address`, `CartItem`, `Empty`) ใช้ร่วมข้าม service | Evidenced | `protos/demo.proto` (`Product.price_usd`, `GetQuoteResponse.cost_usd`, `ChargeRequest.amount`, `OrderItem.cost`) |
| 4 | `frontend` รับผู้ใช้ผ่าน HTTP และเรียก backend 7 ตัวผ่าน gRPC | Evidenced | `src/frontend/main.go` (`mux.NewRouter`, `http.ListenAndServe`, `mustConnGRPC` บรรทัด 141–147) |
| 5 | protocol ของ shopping assistant ไม่ใช่ส่วนหนึ่งของ `demo.proto` และไม่ทราบ protocol | Evidenced (ว่าไม่มี gRPC conn) / Not evidenced (protocol จริง) | `src/frontend/main.go` (`shoppingAssistantSvcAddr` ไม่มี `mustConnGRPC`); ไม่มี service นี้ใน `demo.proto` |
| 6 | checkout เรียก 6 service แบบ synchronous ตามลำดับ | Evidenced | `src/checkoutservice/main.go` (`PlaceOrder`, `prepareOrderItemsAndShippingQuoteFromCart`) |
| 7 | service ไม่แชร์ code library กัน แชร์แค่ proto contract | Evidenced | comment ใน `initProfiling` (ทั้งสองไฟล์); `import pb ".../src/checkoutservice/genproto"` |
| 8 | service เขียนด้วยหลายภาษา จึงต้องการ contract ที่ language-neutral | Inferred | หลักฐานแสดงเฉพาะ Go สองตัว; ภาษาของ service อื่น Not evidenced |
| 9 | ไฟล์เดียวเลือกเพื่อให้เห็น contract ทั้งระบบในที่เดียว เหมาะกับ demo | Inferred | ไม่มีเอกสารเหตุผลใน evidence |
| 10 | เลือก synchronous เพราะ order flow ต้องได้ผลทันที | Inferred | สอดคล้องกับ `PlaceOrderResponse` ที่มี `shipping_tracking_id` แต่ไม่มีเหตุผลเขียนไว้ |
| 11 | gRPC/protobuf มี overhead ต่ำกว่า REST/JSON | Assumption | ความรู้ทั่วไป |
| 12 | ตัวเลือก 2–4 เคยถูกพิจารณา | Inferred | ไม่มีหลักฐานในไฟล์ |
| 13 | ข้อดี/ข้อเสียของ REST และ async messaging ในตาราง | Assumption | ความรู้ทั่วไป |
| 14 | ทุก gRPC connection มี OpenTelemetry instrumentation และ propagate trace context | Evidenced | `mustConnGRPC` (`otelgrpc.NewClientHandler`), `grpc.NewServer(grpc.StatsHandler(otelgrpc.NewServerHandler()))`, `otel.SetTextMapPropagator` |
| 15 | checkout ลงทะเบียน gRPC health service | Evidenced | `src/checkoutservice/main.go` (`healthpb.RegisterHealthServer`) |
| 16 | availability ของ checkout ขึ้นกับ service ปลายทางทุกตัวใน path | Evidenced | `PlaceOrder` คืน error เมื่อ cart/catalog/currency/shipping/payment ล้ม (`codes.Internal`, `codes.Unavailable`) |
| 17 | เกิด N+1 calls ต่อสินค้าในตะกร้า | Evidenced | `prepOrderItems` (ลูป `cl.GetProduct` + `cs.convertCurrency`) |
| 18 | แก้ message กลางต้อง regenerate/redeploy หลาย service | Inferred | type ถูกใช้ร่วม (claim 3) แต่ build/deploy process Not evidenced |
| 19 | ไม่มี compensation เมื่อ shipping ล้มหลังตัดเงิน | Evidenced | `PlaceOrder` (`chargeCard` แล้ว `shipOrder` คืน `codes.Unavailable` โดยไม่มี refund) |
| 20 | traffic ระหว่าง service ใช้ insecure credentials | Evidenced | `mustConnGRPC` (`insecure.NewCredentials()`) ทั้งสองไฟล์ |
| 21 | timeout ใน `mustConnGRPC` ไม่มีผลกับ RPC | Evidenced | `_, cancel := context.WithTimeout(ctx, ...)` (ctx ใหม่ถูกทิ้ง) |
| 22 | `convertCurrency` ไม่ส่ง deadline/trace context ของ request ต่อ | Evidenced | `convertCurrency` ใช้ `context.TODO()` แทน `ctx` |
| 23 | ไม่มี retry หรือ circuit breaker | Evidenced (ในขอบเขตไฟล์ที่อ่าน) | ไม่พบ retry policy ใน `grpc.NewClient` options; infra อื่น (เช่น service mesh) Not evidenced |
| 24 | email ล้มไม่ทำให้ order ล้ม; error ของ `EmptyCart` ถูกทิ้ง | Evidenced | `PlaceOrder` (`log.Warnf` เมื่อ `sendOrderConfirmation` ล้ม, `_ = cs.emptyUserCart(...)`) |
| 25 | ควรย้าย email ไปเป็น async ในอนาคต | Inferred | ข้อเสนอ ไม่ใช่ข้อเท็จจริงของระบบ |
| 26 | กฎ backward compatibility ของ protobuf ช่วยลดความเสี่ยงการเปลี่ยน contract | Assumption | ความรู้ทั่วไป |
| 27 | ไม่มี validation (load test/SLO) ที่บันทึกไว้ | Evidenced (ในขอบเขตไฟล์ที่อ่าน) | ไม่พบใน 3 ไฟล์ evidence |
| 28 | วันที่และผู้ตัดสินใจ | Not evidenced | มีเพียง copyright 2018/2020, Google LLC |
