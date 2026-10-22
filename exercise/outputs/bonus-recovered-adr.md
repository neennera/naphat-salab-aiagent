Agent: Antigravity / Model: Gemini

# ADR-0003: สื่อสารระหว่างไมโครเซอร์วิสแบบ Synchronous ผ่าน gRPC โดยใช้ demo.proto ร่วมกัน

- **Status:** Reconstructed
- **Date:** 2026-10-21
- **Deciders:** ทีมผู้ออกแบบ Online Boutique (Reconstructed)

## Context and problem

ระบบ Online Boutique ถูกออกแบบเป็นระบบไมโครเซอร์วิสที่มีเซอร์วิสย่อยจำนวนมากทำงานร่วมกัน (เช่น `frontend`, `checkoutservice`, `cartservice`, `paymentservice`, `shippingservice` เป็นต้น) แต่ละเซอร์วิสจำเป็นต้องแลกเปลี่ยนข้อมูลและประสานงานกันในการทำธุรกรรม เช่น การดึงรายการสินค้า การคำนวณค่าจัดส่ง และการสั่งซื้อสินค้า ปัญหาที่ต้องตัดสินใจคือ สถาปัตยกรรมการสื่อสารระหว่างเซอร์วิส (Inter-service Communication) และการบริหารจัดการสัญญาอินเทอร์เฟซ (Interface Contract) ควรใช้รูปแบบและเทคโนโลยีใดเพื่อให้ตอบโจทย์ระบบ

## Decision drivers

1. **ความง่ายในการเรียนรู้และทำความเข้าใจ (Simplicity & Learnability):** ผู้พัฒนาหรือผู้ศึกษาตัวอย่างไมโครเซอร์วิสสามารถเห็นภาพรวมสัญญาข้อมูลของทั้งระบบได้จากที่เดียว
2. **ความเข้ากันได้และการรองรับหลายภาษา (Polyglot Interoperability):** เซอร์วิสต่าง ๆ สามารถพัฒนาด้วยภาษาที่แตกต่างกันได้ (Go, Python, Java ฯลฯ) โดยมี Type Safety และการสร้างโค้ด (Code Generation) อัตโนมัติ
3. **ประสิทธิภาพและความเร็ว (Performance & Low Latency):** การแลกเปลี่ยนข้อมูลและ serialization มีขนาดกะทัดรัดและรวดเร็ว
4. **ความสะดวกในการติดตั้งและทดสอบ (Deployability & Testability):** ไม่ต้องการ infrastructure เสริมที่ซับซ้อน เช่น Message Broker ภายนอกสำหรับการรันเดโมพื้นฐาน

## Considered options

1. **Synchronous gRPC ร่วมกับไฟล์นิยามรวมศูนย์ `demo.proto` เพียงไฟล์เดียว**
2. **Synchronous REST/JSON over HTTP โดยแยกเอกสาร OpenAPI/Swagger เป็นรายเซอร์วิส**
3. **Asynchronous Event-Driven ผ่าน Message Broker (เช่น Google Cloud Pub/Sub หรือ RabbitMQ)**

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| **1. Synchronous gRPC + Shared `demo.proto`** | - สัญญา API มี Type safety ชัดเจน เจนเนอเรตโค้ด stub ได้หลายภาษา<br>- Payload แบบ Protobuf binary มีประสิทธิภาพสูงและ latency ต่ำ<br>- เห็นภาพรวม RPC ทั้งหมดของระบบในไฟล์เดียว | - เกิด Schema Coupling ทุกเซอร์วิสต้องพึ่งพา proto ชุดเดียวกัน<br>- การเรียกแบบ Synchronous ก่อให้เกิด Cascading Latency และเสี่ยงต่อ Blocking |
| **2. REST/JSON over HTTP + Per-service Spec** | - เข้ากันได้กับเครื่องมือเว็บทั่วไปและ Debug ผ่าน curl/browser ง่าย<br>- แต่ละทีมแยกสเปก API เป็นอิสระจากกันได้ (Decoupled contracts) | - ประสิทธิภาพต่ำกว่า gRPC จากการ parse JSON text ขนาดใหญ่<br>- ขาดการบังคับ Type safety ข้ามภาษาอย่างเข้มงวด ต้องพึ่ง schema tooling เพิ่มเติม |
| **3. Asynchronous Event-Driven ผ่าน Message Broker** | - Loose coupling สูงมาก ทั้งด้านเวลา (Temporal) และสถานะ<br>- รองรับโหลดกระชากและเพิ่ม Availability ของระบบได้ดี | - สถาปัตยกรรมซับซ้อนขึ้น ต้องจัดการ Eventual Consistency และ Dead-letter queue<br>- ต้องติดตั้งและบำรุงรักษา Message Broker เพิ่มเติม ทำให้การรันเดโมยุ่งยาก |

## Decision

เลือก **ตัวเลือกที่ 1: Synchronous gRPC ร่วมกับไฟล์นิยามรวมศูนย์ `demo.proto`**

เนื่องจาก Online Boutique มีวัตถุประสงค์หลักเพื่อเป็นตัวอย่างสาธิต (Reference Architecture Demo) บน Cloud-native / Kubernetes การใช้ไฟล์ `demo.proto` เพียงไฟล์เดียวช่วยลดภาระในการจัดการ repository หลายแห่ง ทำให้ผู้ศึกษาสามารถเห็นภาพรวมของ API ทั้งหมดได้ทันที อีกทั้ง gRPC ยังเป็นเทคโนโลยีหลักในระบบนิเวศ Cloud Native Computing Foundation (CNCF) ที่รองรับ Polyglot services และมี OpenTelemetry plugin สำหรับ Distributed Tracing ในตัว

## Consequences

- **Positive:** ได้ประสิทธิภาพการสื่อสารสูง ขนาด payload เล็ก และมี Type safety ข้ามภาษาอย่างสมบูรณ์ (Quality attribute: Performance, Interoperability)
- **Positive:** ความง่ายในการจัดการ Interface Contract จากจุดเดียว ลด overhead ในการดูแล repository สเปกหลายแห่ง (Quality attribute: Maintainability ในบริบทโครงการเดโม)
- **Negative:** เกิด Temporal Coupling และ Cascading Failure เสี่ยงที่การล่มของ downstream service จะส่งผลกระทบต่อ checkout flow ทั้งหมด (Quality attribute: Availability, Reliability)
- **Negative:** เกิด Monolithic Contract Coupling หากมีการเปลี่ยนแปลง schema ใน `demo.proto` อาจส่งผลกระทบหรือต้อง recompile protobuf ให้กับทุกเซอร์วิส (Quality attribute: Modifiability, Team Autonomy)
- **Risks and mitigations:**
  - *ความเสี่ยง:* การเชื่อมต่อแบบซิงโครนัสค้างหรือ timeout เมื่อ downstream ตอบสนองช้า
  - *การบรรเทา:* กำหนด Timeout และ Context cancellation ไว้ล่วงหน้า (เช่น มีการใช้ `context.WithTimeout` 3 วินาที ในการเชื่อมต่อ gRPC client)

## Validation

- ตรวจสอบผ่านการรัน Integration Test / Load Test และสังเกตอัตราความสำเร็จของการทำธุรกรรม (Transaction Success Rate)
- ตรวจสอบการส่ง Trace Context ผ่าน OpenTelemetry (`otelgrpc.NewServerHandler()`, `otelgrpc.NewClientHandler()`) เพื่อดู Distributed Traces และ Latency ของแต่ละ RPC call

---

## Claim check

| # | Claim | Type | Evidence |
|---|---|---|---|
| 1 | เซอร์วิสทุกตัวนิยาม RPC interface และโครงสร้างข้อมูลร่วมกันในไฟล์ `demo.proto` เพียงไฟล์เดียว | **Evidenced** | `system/online-boutique/protos/demo.proto` (Line 1–262) |
| 2 | `frontend` สื่อสารกับเซอร์วิสปลายทาง (เช่น `checkoutservice`, `cartservice`) ผ่าน gRPC Client Connection | **Evidenced** | `system/online-boutique/src/frontend/main.go` (symbols: `mustConnGRPC`, `svc.checkoutSvcConn`) |
| 3 | `checkoutservice` ให้บริการ RPC `PlaceOrder` และเรียกบริการอื่น ๆ (เช่น payment, shipping) แบบ synchronous gRPC | **Evidenced** | `system/online-boutique/src/checkoutservice/main.go` (symbols: `PlaceOrder`, `pb.RegisterCheckoutServiceServer`) |
| 4 | มีการตั้งค่า Timeout ในการสร้างการเชื่อมต่อ gRPC ไว้ที่ 3 วินาที | **Evidenced** | `system/online-boutique/src/checkoutservice/main.go` (symbol: `mustConnGRPC` L212), `system/online-boutique/src/frontend/main.go` |
| 5 | ผู้พัฒนาเลือกรวมไฟล์ proto เพราะต้องการให้ผู้เรียนรู้เข้าใจระบบเดโมได้ง่ายในจุดเดียว | **Inferred** | ไม่ปรากฏในโค้ดหรือคอมเมนต์ เป็นการอนุมานจากลักษณะโครงการ demo ของ Google Cloud |
| 6 | ทีมผู้ออกแบบพิจารณาเปรียบเทียบ gRPC กับ REST และ Message Queue ก่อนตัดสินใจเลือก | **Inferred** | ไม่พบเอกสารการเปรียบเทียบในหลักฐาน เป็นการอนุมานตามโครงสร้างมาตรฐานของ ADR |
| 7 | gRPC มีประสิทธิภาพสูงกว่าและ payload กะทัดรัดกว่า REST/JSON | **Assumption** | ความรู้ทั่วไปในอุตสาหกรรมซอฟต์แวร์ (Industry knowledge เกี่ยวกับ Protobuf binary serialization) |
| 8 | การสื่อสารแบบซิงโครนัสทำให้เกิดความเสี่ยงเรื่อง Cascading failure และ Latency สะสม | **Assumption** | หลักการทางสถาปัตยกรรมซอฟต์แวร์ทั่วไปเรื่อง Microservices Coupling และ Distributed Systems |
