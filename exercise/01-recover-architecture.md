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
| checkoutservice → paymentservice | `checkoutservice.yaml` (`PAYMENT_SERVICE_ADDR`) | ✅ | `src/checkoutservice/main.go:370` เรียก `Charge` ผ่าน gRPC จริง |
| checkoutservice → emailservice | `checkoutservice.yaml` (`EMAIL_SERVICE_ADDR`) | ✅ | `main.go:380` เรียก `SendOrderConfirmation` แต่ถ้าเรียกไม่สำเร็จก็แค่ log warning แล้วคำสั่งซื้อยังสำเร็จ (`main.go:273-277`) |
| frontend → adservice | `frontend.yaml` (`AD_SERVICE_ADDR`) | ✅ | `src/frontend/rpc.go:123` เรียก `GetAds` |
| cartservice → redis-cart | `cartservice.yaml` (`REDIS_ADDR`) | ✅ | `src/cartservice/src/Startup.cs:29-40` ใช้ `RedisCartStore` เมื่อตั้ง `REDIS_ADDR` ถ้าไม่ตั้งจะเก็บใน memory แทน (`:53`) และเปลี่ยนไปใช้ Spanner/AlloyDB ได้ |
| recommendationservice → productcatalogservice | `recommendationservice.yaml` (`PRODUCT_CATALOG_SERVICE_ADDR`) | ✅ | `src/recommendationservice/recommendation_server.py:131-136` สร้าง gRPC stub แล้วเรียก `ListProducts` (`:73`) |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย):
  - ไม่พบ dependency ที่ไม่มีจริง: ทั้ง 17 เส้นมีโค้ดเรียกจริง
  - **ขาด frontend → shoppingassistantservice (HTTP):** AI ไม่ใส่ในแผนภาพเพราะไม่มี manifest แต่ frontend บังคับต้องมี env นี้ (`mustMapEnv`, `src/frontend/main.go:139`) และส่ง HTTP POST ไปจริง (`src/frontend/handlers.go:464`) เมื่อเปิด `ENABLE_ASSISTANT` ตัว service เป็น Python Flask เรียก **Gemini** และใช้ **AlloyDB vector store** (`src/shoppingassistantservice/shoppingassistantservice.py:22-52`) จึงขาดทั้ง container และระบบภายนอก
  - **ไม่มีภาษาของแต่ละ service:** manifest ไม่บอก แต่ในโค้ดใช้หลายภาษา: Go (frontend, checkout, shipping, productcatalog), C# (cart), Node.js (currency, payment), Python (email, recommendation), Java (ad)
  - **container ที่น่าสงสัย:** `loadgenerator` เป็นเครื่องมือทดสอบและไม่อยู่ใน `kustomization.yaml` จึงไม่ควรนับเป็นส่วนของระบบ
  - **สิ่งที่ AI ทำถูก:** ไม่ใส่ระบบจ่ายเงินหรือ email ภายนอก เพราะ payment แค่สุ่ม UUID (`src/paymentservice/charge.js:85`) และ email ทำงานแบบ dummy เท่านั้น (`src/emailservice/email_server.py:124`)

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด
   - จาก `.proto` อย่างเดียว agent ได้แค่รายชื่อ 9 service กับ RPC ที่แต่ละตัวเปิดให้เรียก แต่ไม่มี dependency เลย เพราะ proto บอกแค่ว่า service *ให้บริการอะไร* ไม่ได้บอกว่า *ใครเรียกใคร* frontend และ redis-cart ก็ไม่ปรากฏ เพราะไม่ได้ให้บริการผ่าน gRPC พอเพิ่ม manifest ก็เห็นตัวแปร `*_ADDR` ซึ่งบอก dependency ได้ 17 เส้น แต่ยังไม่รู้ภาษาที่ใช้ และพลาด service ที่ไม่มี manifest
2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ
   - การที่ไม่มี shoppingassistantservice ในแผนภาพ เพราะเป็นทางเดียวที่ระบบส่งข้อมูลออกไปบริการ AI ภายนอก (Gemini) และใช้ฐานข้อมูล AlloyDB บน cloud คนใหม่จะเข้าใจว่าทุกอย่างอยู่ใน cluster ไม่พึ่งพา cloud ภายนอก แล้วอาจพลาดเรื่องค่าใช้จ่าย ความเป็นส่วนตัวของข้อมูล และจุดที่ระบบอาจล่มได้
