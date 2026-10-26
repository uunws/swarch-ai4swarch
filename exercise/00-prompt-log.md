**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชื่อ / รหัสนิสิต:** นันทพร พัวพันบุญ / 6631330021
**agent ที่ใช้:** Claude Code (VS Code extension)
**โมเดล:** Claude Opus 5.5 (`claude-opus-5-5`)
**แพ็กเกจที่ใช้:** Claude Pro

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code + Claude Opus 5.5 | `prompts/1-…` run A | `prompts/1-recover-architecture.md`, `system/online-boutique/protos/demo.proto` | ใช่ | 4 | ทำ 1-A, 1-B, 2-a, 2-b ต่อกันใน session เดียว ไม่ได้เปิดแชตใหม่ทุก run ตามที่ README กำหนด |
| 2 | 1-B | Claude Code + Claude Opus 5.5 | `prompts/1-…` run B | + `kubernetes-manifests/` ทุกไฟล์ (README.md, kustomization.yaml และไฟล์ .yaml ของทั้ง 11 deployment) | ใช่ | 4 | dependency ถูกทั้ง 17 เส้น แต่ขาด frontend → shoppingassistantservice |
| 3 | 2-a | Claude Code + Claude Opus 5.5 | `prompts/2-…` run a, context: none | `prompts/2-write-adr.md`, `docs/adr/template.md`, ข้อความ R-12 (agent ดึงเฉพาะส่วน R-12 จาก `docs/scenario.md` แทนการที่นิสิตวางเอง) | เกือบ: ไม่ได้อ่านส่วนอื่นของ scenario แต่ใน session เดียวกันเคยเห็น manifest จาก 1-B มาแล้ว | 4 | ADR ไม่มีตัวเลขจาก scenario |
| 4 | 2-b | Claude Code + Claude Opus 5.5 | `prompts/2-…` run b | + `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `system/online-boutique/src/checkoutservice/main.go` | ใช่ | 4 | |
| 5 | 3 | Claude Code + Claude Opus 5.5 | `prompts/3-…` adr: `docs/adr/0002-publish-order-placed-event-to-rabbitmq.md`, language: Thai | `prompts/3-review-adr.md`, `docs/adr/0002-publish-order-placed-event-to-rabbitmq.md`, `docs/scenario.md` (ตามที่ agent ระบุในไฟล์ผลลัพธ์) | ใช่ | 3 | ทำในแชตใหม่ |
| 6 | ตรวจ 1-B | Claude Code + Claude Opus 5.5 | ให้ agent ตรวจ dependency ของ 1-B กับซอร์สโค้ด | `src/frontend/` (main.go, rpc.go, handlers.go), `src/checkoutservice/main.go`, `src/cartservice/src/Startup.cs`, `src/recommendationservice/recommendation_server.py`, `src/paymentservice/charge.js`, `src/emailservice/email_server.py`, `src/shoppingassistantservice/`, `src/productcatalogservice/catalog_loader.go`, `src/shippingservice/`, `src/currencyservice/server.js`, `src/adservice/` | ใช่ (ใบงาน 01 อนุญาตให้ตรวจกับซอร์สโค้ดได้) | 4 | |
| 7 | bonus | Claude Code + Claude Opus 5.5 | `prompts/bonus-…` decision: sync gRPC + shared `demo.proto`, language: Thai | `prompts/bonus-recover-decision.md`, `docs/adr/template.md`, `protos/demo.proto`, `src/checkoutservice/main.go`, `src/frontend/main.go` (ตามที่ agent อ้างในไฟล์ผลลัพธ์) | ใช่ | 3 | ทำในแชตใหม่ |
