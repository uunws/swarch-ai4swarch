Agent: Claude Code · Model: Claude Opus 5.5 (`claude-opus-5-5`)

# Review: ADR-0002 Publish an OrderPlaced event to RabbitMQ through a transactional outbox

แหล่งที่ใช้: `docs/adr/0002-publish-order-placed-event-to-rabbitmq.md` และ `docs/scenario.md` เท่านั้น

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| QA1 Performance | ลูกค้า | กดสั่งซื้อ (`PlaceOrder`) | ช่วง 11.11 peak **200 orders/s**, ทุกระบบปกติ | ตัดบัตร, จัดส่ง, commit `orders` + `outbox` ใน transaction เดียว แล้วตอบลูกค้า | p95 `PlaceOrder` **≤ 500 ms** (วันนี้ approximately 300 ms → เหลือ budget approximately 200 ms) |
| QA2 Availability | Operator / ความล้มเหลว | MailFast, `loyaltyservice` หรือ RabbitMQ ล่ม 10 นาที | peak 200 orders/s | order ยังสำเร็จ; event ค้างใน queue หรือ `outbox` แล้วส่งต่อเมื่อระบบกลับมา | **0** order ที่ fail เพราะ email/loyalty/analytics; **0** event สูญหาย |
| QA3 Correctness | RabbitMQ / relay | ส่ง `OrderPlaced` ซ้ำ (redelivery หรือ relay crash ก่อน mark sent) | peak 200 orders/s | `loyaltyservice` ให้แต้มครั้งเดียว | double points = **0**; แต้มเข้า **≤ 60 s** หลัง order (ทุก order ไม่ใช่แค่ p95) |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Risk | ตัดบัตร (`chargeCard`) **ก่อน** เขียน order DB; ถ้า transaction ล้ม มีแค่ "log + alert" (Risks: *Card charged but the order transaction fails*) | ลูกค้าโดนตัดเงินแต่ไม่มี order, ไม่มีอีเมล, ไม่มีแต้ม; การแก้ด้วยมือไม่ scale ที่ 200 orders/s |
| Risk | Relay เป็น goroutine ใน `checkoutservice` ซึ่งมีหลาย replica แต่ ADR ไม่ระบุการ lock แถว (เช่น `SELECT … FOR UPDATE SKIP LOCKED`) หรือ leader election | ทุก replica publish แถวเดียวกันซ้ำ → duplicate ทุก event; loyalty กันได้ แต่ email ส่งซ้ำ (MailFast idempotency ยังเป็น Open question) |
| Risk | "single active consumer" ถูกเสนอเป็นวิธีคุม 50 emails/s แต่ MailFast p95 1.5 s / p99 8 s → ต้องมี in-flight approximately 50 × 1.5 = **75+ calls** พร้อมกันจึงจะได้ 50/s (Little's law) | ถ้า consumer ตัวเดียวทำงานแบบ sequential จะได้แค่ < 1 email/s; backlog ไม่มีวันหมด |
| Risk | Order DB ใหม่อยู่บน critical path ของ checkout (Consequences: *new single point of failure*) | เพิ่มทางให้ order fail ใหม่ ทั้งที่ scenario ห้ามแค่กรณี email/loyalty/analytics; แลกมาเพื่อกัน event หาย |
| Risk | Event มี `email`, `address` และถูก fanout ไปทุก queue รวม `analytics` | ข้อมูลส่วนบุคคลกระจายไปยัง consumer ที่ไม่ต้องใช้; ทุก consumer ใหม่ในปีหน้าก็ได้ไปด้วย |
| Sensitivity point | ความยาวของ peak 200 orders/s (ไม่มีใน scenario) | backlog = 150 emails/s × ระยะเวลา; peak 1 ชม. = 540,000 emails ใช้ approximately 3 ชม. กว่าจะหมด; retry ของ 0.5 % failures กิน quota 50/s ด้วย |
| Sensitivity point | relay polling interval + batch size + publish timeout | เป็นส่วนหนึ่งของ budget 60 s ของ loyalty โดยตรง; ADR ไม่ได้แบ่ง budget ให้แต่ละขั้น |
| Trade-off point | Transactional outbox | reliability ของ event ↑ (ไม่หายแม้ broker ล่ม) แต่ availability ของ checkout ↓, latency ของ `PlaceOrder` ↑, cost/operability ↓ (2 stateful components) |
| Trade-off point | Rate limit 50/s แบบ global ที่ email consumer | ทำตาม MailFast ได้ (reliability) แต่ email ช้าเป็นนาทีถึงชั่วโมงช่วง peak (usability) |
| Non-risk | Queue แยกต่อ consumer (`email`, `loyalty`, `analytics`) | backlog ของ email ไม่ทำให้ loyalty ช้า → QA3 เป็นไปได้ |
| Non-risk | Exactly once = at-least-once + `processed_orders` unique constraint ใน transaction เดียวกับการเพิ่มแต้ม | ถูกต้องและรองรับ duplicate ทั้งจาก broker และ relay |
| Non-risk | เลือก RabbitMQ แทน Kafka | ทีม 2 คนเคยใช้, 200 msg/s ไม่ต้องการ throughput ของ Kafka, มี DLQ/prefetch ในตัว |

## 3. Missing or weak

- **Options not considered:**
  - **RabbitMQ Streams** (replay ได้ใน broker เดิม) สำหรับ analytics/recommendations ซึ่ง ADR ใช้เป็นข้อเสียของ option 3
  - **Direct publish + publisher confirms** โดยมี outbox เฉพาะตอน broker ล่ม (ไม่ต้องเขียน DB ทุก order)
  - เขียน order สถานะ `PENDING` **ก่อน** `chargeCard` แล้ว update เป็น `PLACED` + outbox
- **Consequences not stated:** ADR ไม่ได้บอกว่าจะเกิดอะไรถ้า `loyaltyservice` ล่มนานกว่า 1 นาที (requirement จะพังแน่นอน). Validation เขียนว่า "≤ 1 min after `loyaltyservice` recovers" ซึ่งเป็นการ **นิยาม requirement ใหม่** ไม่ใช่ตามที่ scenario กำหนด ("within 1 minute of the order"). ADR ยังไม่ได้บอกด้วยว่า `emptyUserCart` อยู่ก่อนหรือหลัง transaction และจะทำอย่างไรถ้ามันล้ม
- **Unsupported claims:**
  - "Expected to stay well under 500 ms": ไม่มีตัวเลขของ PostgreSQL write ที่ 200 tx/s มารองรับ
  - "No event is lost": จริงเฉพาะเมื่อ DB มี durable replica; ADR ยังไม่ได้ตอบเรื่อง availability target (Open question)
  - "shared token bucket": ไม่ระบุว่าจะเก็บ state ไว้ที่ไหน ซึ่งอาจเป็น component ที่สามที่ต้องดูแล

## 4. Verdict: **Accept with changes**

1. **ปิดช่องว่างระหว่างการตัดบัตรกับ DB:** เขียน order `PENDING` ก่อน `chargeCard` หรือกำหนด refund/compensation flow แบบอัตโนมัติ แทนการ "log + alert"
2. **ระบุ concurrency ของ relay และ email consumer:** relay ต้องใช้ `SKIP LOCKED` หรือ leader election; email consumer ต้องมี approximately 75+ in-flight calls ภายใต้ global limiter 50/s และระบุว่า limiter เก็บ state ไว้ที่ไหน
3. **แบ่ง latency budget 60 s ของ loyalty** (relay interval + broker + processing) ให้ชัด และแก้ Validation ให้วัดจากเวลาที่เกิด order; ระบุพฤติกรรมเมื่อ `loyaltyservice` ล่มนานกว่า 1 นาที
