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
| 1 | #17 เกิด N+1 calls ต่อสินค้าในตะกร้า | Evidenced | Evidenced ✅ | `src/checkoutservice/main.go:343-355` ในลูป `for i, item := range items` เรียก `GetProduct` และ `convertCurrency` ทีละชิ้นจริง |
| 2 | #22 `convertCurrency` ไม่ส่ง deadline/trace context ต่อ | Evidenced | Evidenced ✅ | `main.go:360` ใช้ `context.TODO()` แทน `ctx` ที่รับมา จึงไม่ส่ง deadline และ trace ของ request ต่อ |
| 3 | #23 ไม่มี retry หรือ circuit breaker | Evidenced (ในขอบเขตไฟล์ที่อ่าน) | Inferred ⚠️ | `main.go:214-216` ไม่มี retry option ใน `grpc.NewClient` จริง แต่ "หาไม่เจอในโค้ด" ไม่ได้แปลว่า "ระบบไม่มี" `kubernetes-manifests/frontend.yaml` มี annotation `sidecar.istio.io/rewriteAppHTTPProbers` แปลว่าอาจรันกับ Istio ซึ่งตั้ง retry ได้นอกโค้ด |
| 4 | #8 service เขียนด้วยหลายภาษา จึงต้องการ contract แบบ language-neutral | Inferred | Inferred ✅ (ถูกต้องตามหลักฐานที่ให้) | หลักฐานที่ให้มีแค่ Go 2 ไฟล์ จึงติดป้าย Inferred ได้ถูกต้อง แต่ถ้าเปิดซอร์สอื่นจะเห็นว่าใช้หลายภาษาจริง: C# (`src/cartservice/src/Startup.cs`), Node.js (`src/paymentservice/charge.js`), Python (`src/emailservice/email_server.py`), Java (`src/adservice/.../AdService.java`) ส่วนข้อที่ว่า *เพราะใช้หลายภาษาจึงเลือก gRPC* ยังเป็นการอนุมาน |
| 5 | #9 ใช้ไฟล์เดียวเพื่อให้เห็น contract ทั้งระบบในที่เดียว เหมาะกับ demo | Inferred | Inferred ✅ แต่ premise คลาดเคลื่อน | ส่วนที่ว่าเป็น demo มีหลักฐานบางส่วน: `protos/demo.proto:19` ใช้ `go_package = ".../microservices-demo/..."` แต่ "ไฟล์เดียวที่แชร์กัน" ไม่จริงทั้งหมด เพราะมีสำเนาแยกใน `src/paymentservice/proto/demo.proto`, `src/currencyservice/proto/demo.proto`, `src/adservice/src/main/proto/demo.proto` เนื้อหาเหมือนกัน ต่างแค่ไม่มี `option go_package` จึงเป็น contract เดียวที่ copy ไปไว้ในแต่ละ service ซึ่งมีโอกาสแตกต่างกันในอนาคต |

**นับ:** จากข้อกล่าวอ้างทั้งหมด agent ระบุว่า *Evidenced* กี่ข้อ **17** (ไม่รวม #5 ที่ AI ระบุว่า Evidenced ครึ่งหนึ่ง) ตามที่นิสิตตรวจ **15** (ลดข้อ #23 และ #27 เป็น Inferred เพราะเป็นการ "หาไม่เจอในไฟล์ 3 ไฟล์" ไม่ใช่หลักฐานว่าไม่มีจริง: ข้อ #27 ว่าไม่มี load test แต่ใน repo มี `src/loadgenerator/locustfile.py` และ `kubernetes-manifests/loadgenerator.yaml`)

## สะท้อนคิด

1. agent เคยระบุการเดาว่าเป็น *Evidenced* หรือไม่ ยกตัวอย่าง หรืออธิบายว่าตรวจสอบอย่างไรว่าไม่มี
   - ไม่ได้เดามั่ว แต่มี 2 ข้อที่ติดป้ายแรงเกินไป คือ #23 "ไม่มี retry/circuit breaker" และ #27 "ไม่มี load test" ทั้งสองข้อคือ "หาไม่เจอใน 3 ไฟล์" ซึ่งไม่ใช่หลักฐานว่าระบบไม่มีจริง ใน repo มี `loadgenerator` และ manifest มี annotation ของ Istio อยู่ AI เขียนว่า "ในขอบเขตไฟล์ที่อ่าน" ไว้ก็ซื่อสัตย์ดี แต่คนที่อ่านแค่คอลัมน์ Type จะเห็นคำว่า Evidenced ข้อ Evidenced อื่นที่ตรวจ (#17, #22) ตรงกับโค้ดจริง
2. หากนักพัฒนาใหม่อ่านเฉพาะ ADR ที่กู้คืนมานี้ เขาอาจเข้าใจผิดในเรื่องใด
   - อาจเข้าใจว่าทุก service import `demo.proto` ไฟล์เดียวกัน แก้ที่เดียวแล้วจะมีผลทั้งระบบ แต่จริง ๆ payment, currency และ ad มีสำเนาของตัวเอง ถ้าแก้แค่ `protos/demo.proto` สำเนาเหล่านั้นจะไม่เปลี่ยนตาม นอกจากนี้อาจเข้าใจว่าเหตุผลในหัวข้อ Decision (เหมาะกับ demo, ต้องได้ผลทันที) เป็นของผู้ออกแบบเดิม ทั้งที่ส่วนใหญ่ AI อนุมานเอาเอง
3. AI กู้คืนอะไรจากโค้ดได้บ้าง และอะไรที่มีเพียงสถาปนิกผู้ออกแบบเดิมเท่านั้นที่รู้ ยกตัวอย่างหนึ่งตัวอย่างจาก ADR ของนิสิต
   - AI กู้ได้ดีในส่วนที่ "เป็นอย่างไร": ลำดับการเรียก service, protocol, การใช้ insecure credentials, deadline ที่ไม่ถูกส่งต่อ, N+1 calls แต่ "ทำไม" กู้ไม่ได้ เช่น ข้อ #10 "เลือก synchronous เพราะ order ต้องได้ผลทันที" หรือข้อ #12 ที่ว่าทางเลือก REST หรือ async เคยถูกพิจารณา ในโค้ดไม่มีข้อมูลนี้เลย มีแค่ผู้ออกแบบเดิมที่รู้ว่าเคยเทียบทางเลือกอะไร และให้น้ำหนักกับอะไร
