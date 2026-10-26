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
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast) | ไม่ใช้เลย เพราะไม่มีข้อมูล จึงย้ายทั้งหมดไปไว้ใน Open questions | ใช้ครบ: p95 ≤ 500 ms (ปัจจุบันประมาณ 300 ms), 200 orders/s, MailFast p95 1.5 s / p99 8 s / 0.5 % / 50 emails/s, loyalty ภายใน 1 นาที, ทีม 4 คนถนัด Go และมี 2 คนเคยใช้ RabbitMQ แล้วคำนวณ backlog ว่า email จะค้างเพิ่ม 9,000 ฉบับต่อนาทีของ peak |
| ทางเลือกที่พิจารณา (ระบุ) | 1) เรียกแบบ synchronous 2) async ใน process 3) event + message broker (ไม่ระบุชื่อ broker) 4) workflow engine → เลือกข้อ 3 | 1) ต่อ synchronous chain เดิม 2) goroutine แบบ fire-and-forget ใน checkout 3) RabbitMQ 4) Kafka → เลือก RabbitMQ เพราะทีมเคยใช้ |
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร | ทั่วไป: consumer แบบ idempotent ใช้ order ID ป้องกันซ้ำ, retry + backoff + DLQ ส่วน rate limit แค่บอกว่า "ขึ้นกับ limit ของ MailFast" | ชัดเจน: at-least-once + ตาราง `processed_orders` ที่ `order_id` เป็น unique ใน transaction เดียวกับการให้แต้ม, แยก queue ของ loyalty ไม่ให้รอ email, จำกัด 50/s แบบ global ทุก replica, ชี้ว่า checkout ไม่มี DB จึงทำ outbox ไม่ได้ (`main.go:66-86`) |
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) | แทบไม่มี เพราะไม่ระบุตัวเลขหรือเครื่องมือเลย แต่ ADR จึงกว้างเกินจนนำไปใช้จริงไม่ได้ | มี: publish timeout "≤ 50 ms", RabbitMQ "quorum queues 3 nodes", การ propagate trace ทาง message header และ schema แบบ protobuf ไม่มีใน scenario ทั้งหมด ส่วน Kafka ที่ว่า "ไม่มีใครในทีมเคยใช้" เป็นการตีความจาก "สองคนเคยใช้ RabbitMQ" |

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| # | สิ่งที่เปลี่ยนจากร่างของ AI | เหตุผล |
|---|---|---|
| 1 | ตัดค่า publish timeout "≤ 50 ms" ออก ให้ตั้งค่า timeout, polling interval และ batch size ของ outbox relay จากผล load test แทน (Decision ข้อ 2, Validation) | ตัวเลข 50 ms AI แต่งขึ้นเอง ไม่มีใน scenario ถ้าใส่ไว้ คนอ่านจะเข้าใจว่าเป็นข้อกำหนดที่มีที่มา |
| 2 | ตัด "replicated across 3 RabbitMQ nodes" ออก ให้กำหนดจำนวน node จาก load test และ availability target แล้วเพิ่ม availability target เป็นคำถามใน Open questions (Decision ข้อ 3) | จำนวน node ไม่มีใน scenario และมีผลกับทั้งค่าใช้จ่ายและ availability จึงควรตัดสินจากข้อมูล ไม่ใช่ค่าที่ AI เดา |
| 3 | เพิ่ม order database (PostgreSQL แบบ self-hosted) + transactional outbox ใน `checkoutservice` และ outbox relay ส่ง event ไป RabbitMQ แทนการ publish ตรงจาก `PlaceOrder` พร้อมอัปเดตชื่อ ADR, Consequences, Risks, Validation และตัดคำถามเรื่องนี้ออกจาก Open questions | AI ทิ้งไว้เป็นคำถามว่าถ้า RabbitMQ ล่มจะยอมให้ event หายไหม เลือกไม่ยอม เพราะ event ที่หายแปลว่าลูกค้าไม่ได้แต้มเลย ผิดข้อกำหนด "ได้แต้มครั้งเดียวเท่านั้น" และ checkout ไม่มี state (`main.go:66-86`) จึงต้องเพิ่ม DB แลกกับการต้องดูแล component เพิ่มอีกตัว และ DB จะอยู่บน critical path ของ checkout |
| 4 | ใส่ชื่อตัวเองเป็น Deciders และลบบรรทัด "Agent: …" | ADR ต้องมีผู้รับผิดชอบเป็นสถาปนิก ไม่ใช่ AI |
