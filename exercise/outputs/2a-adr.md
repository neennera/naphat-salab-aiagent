<!-- Agent: Antigravity / Model: Gemini 3.8 Flash -->

# ADR-0002: Use an asynchronous message broker for post-order processing

- **Status:** Proposed
- **Date:** 2026-10-21
- **Deciders:** Software Architecture Team

## Context and problem

หลังจากที่คำสั่งซื้อเสร็จสมบูรณ์ ระบบ Online Boutique มีความต้องการทางธุรกิจใหม่ (R-12) ที่ต้องดำเนินการหลังการสั่งซื้อ (post-order actions) ดังนี้:
1. ส่งอีเมลยืนยันคำสั่งซื้อผ่านผู้ให้บริการภายนอกรายใหม่ คือ MailFast เพื่อแทนที่ mock logic เดิมใน `emailservice`
2. มอบแต้มสะสม (loyalty points) ให้แก่ลูกค้าผ่านบริการใหม่ที่จะสร้างขึ้น คือ `loyaltyservice`
3. บันทึกข้อมูล analytics event ว่า "order placed" เพื่อส่งให้ทีมการตลาด

นอกจากนี้ ทีมการตลาดยังมีแผนจะเพิ่มการทำงานหลังการสั่งซื้อเพิ่มเติมอีกในปีหน้า เช่น การส่ง SMS, การตรวจจับและคำนวณคะแนนการทุจริต (fraud scoring) และการอัปเดตระบบแนะนำสินค้า (recommendations update)

ปัญหาทางสถาปัตยกรรมคือ เราจะจัดการและเชื่อมต่อการทำงานเหล่านี้หลังการสั่งซื้ออย่างไร เพื่อไม่ให้กระทบต่อความเร็วในการตอบสนองของการสั่งซื้อ และสามารถรองรับการเพิ่มฟังก์ชันใหม่ในอนาคตได้อย่างยืดหยุ่น

## Decision drivers

1. **Modifiability / Extensibility:** สามารถเพิ่มงานหรือบริการใหม่หลังการสั่งซื้อในปีหน้าได้ง่าย โดยไม่ต้องแก้ไขโค้ดการสั่งซื้อหลัก
2. **Availability:** ความล้มเหลวหรือความล่าช้าของบริการปลายทาง (เช่น บริการภายนอก MailFast หรือ analytics) ต้องไม่ทำให้กระบวนการสั่งซื้อหลักล้มเหลว
3. **Performance (Latency):** เวลาในการตอบสนอง (latency) ของผู้ใช้งานในการสั่งซื้อไม่ควรถูกหน่วงจากงานที่ไม่จำเป็นต้องทำทันทีใน synchronous flow
4. **Reliability:** งานหลังการสั่งซื้อทุกงานต้องได้รับการประมวลผลอย่างมั่นใจ ไม่ตกหล่น

## Considered options

1. **Option 1: Synchronous orchestration via gRPC/HTTP calls from `checkoutservice`**  
   ให้ `checkoutservice` ทำการเรียกไปยัง MailFast (ผ่าน `emailservice`), `loyaltyservice`, และระบบ analytics ตามลำดับทีละตัวแบบ synchronous
2. **Option 2: Asynchronous event-driven architecture using a message broker (Publish-Subscribe)**  
   เมื่อสั่งซื้อสำเร็จ `checkoutservice` จะเผยแพร่ (publish) เหตุการณ์ `OrderPlaced` ไปยัง Message Broker ส่วนบริการอื่น ๆ (เช่น `emailservice`, `loyaltyservice`, analytics worker) จะเข้ามาสมัครรับข้อมูล (subscribe) และทำงานแยกจากกันอย่างเป็นอิสระ
3. **Option 3: In-process asynchronous background worker with database polling**  
   บันทึกสถานะงานลงในฐานข้อมูลหลัก แล้วสร้าง background worker ทำการ poll งานไปประมวลผล

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| **Option 1: Synchronous orchestration** | - สถาปัตยกรรมเรียบง่าย ไม่ต้องเพิ่มโครงสร้างพื้นฐานใหม่<br>- ติดตามผลลัพธ์และจัดการ error ได้ทันทีในจุดเดียว | - ความพร้อมใช้งานต่ำ (low availability): หาก MailFast ล่ม อาจกระทบต่อ checkout<br>- Latency สูงขึ้นตามจำนวนบริการที่ต้องเรียกต่อกัน<br>- ความสามารถในการปรับเปลี่ยนต่ำ (low modifiability): เพิ่มงานใหม่ต้องแก้โค้ด `checkoutservice` ทุกครั้ง |
| **Option 2: Asynchronous event-driven via Message Broker** | - ความสามารถในการปรับเปลี่ยนสูง (high modifiability): รองรับงานใหม่ในปีหน้าได้ทันทีโดยเพิ่ม subscriber ใหม่<br>- ความพร้อมใช้งานสูง (high availability): ความล่าช้าหรือการล่มของ subscriber ไม่กระทบขั้นตอน checkout<br>- Latency ต่ำ: checkout ตอบกลับผู้ใช้ได้ทันทีหลัง publish event | - ต้องติดตั้งและดูแล Message Broker เพิ่มเติม (เพิ่มความซับซ้อนในการจัดการระบบ)<br>- การันตี consistency เป็นแบบ eventual consistency |
| **Option 3: In-process worker with DB polling** | - ไม่ต้องติดตั้ง message broker ตัวใหม่ ใช้ฐานข้อมูลเดิม | - Database polling อาจสร้างภาระ (load) แก่ฐานข้อมูล<br>- Scalability ต่ำ และขยายไปรองรับหลายบริการได้ยากกว่า broker |

## Decision

เลือก **Option 2: Asynchronous event-driven architecture using a message broker (Publish-Subscribe)**

**เหตุผล:**  
ความต้องการระบุชัดเจนว่าจะมีงานเพิ่มเติมเข้ามาอีกหลายอย่างในปีหน้า (SMS, fraud scoring, recommendations update) สถาปัตยกรรมแบบ Event-Driven ผ่าน Message Broker ช่วยแยกการทำงาน (decouple) ระหว่างการสั่งซื้อหลักกับงานประมวลผลย่อย ทำให้ระบบ checkout ตอบสนองได้รวดเร็วและมีความพร้อมใช้งานสูง ไม่ถูกบล็อกจากความล่าช้าของบริการภายนอกอย่าง MailFast

## Consequences

- **Positive:**
  - เพิ่ม Modifiability: การเพิ่มงานใหม่ในปีหน้าสามารถทำได้โดยการเพิ่ม subscriber เข้ามาฟัง event `OrderPlaced` โดยไม่ต้องแตะต้อง `checkoutservice`
  - เพิ่ม Performance (Latency): กระบวนการ checkout ไม่ต้องรอการทำงานของ MailFast หรือ `loyaltyservice` ทำให้ตอบสนองผู้ใช้ได้รวดเร็ว
  - เพิ่ม Availability: หาก MailFast หรือ analytics ขัดข้อง ชั่วคราว กระบวนการ checkout ยังคงสำเร็จได้ตามปกติ และ message สามารถรอประมวลผลซ้ำในคิวได้
- **Negative:**
  - ลด Modifiability / Operability ในระดับระบบ: เพิ่ม operational overhead จากการต้องดูแลรักษาโครงสร้างพื้นฐานของ Message Broker
  - เพิ่มความซับซ้อนในการทดสอบและติดตามการทำงาน (Observability / Traceability) ข้าม service แบบ asynchronous
- **Risks and mitigations:**
  - *Risk:* ข้อความอาจสูญหายหาก Message Broker ล่ม  
    *Mitigation:* เปิดใช้งาน message persistence และตั้งค่า acknowledgement
  - *Risk:* ปัญหาข้อความถูกประมวลผลซ้ำ (at-least-once delivery) โดยเฉพาะการแจกแต้ม loyalty  
    *Mitigation:* ออกแบบ subscriber ให้รองรับ idempotency (เช่น ตรวจสอบ order ID ก่อนให้แต้ม)

## Validation

- ดำเนินการทดสอบโหลด (load test) เพื่อวัด latency ของกระบวนการสั่งซื้อว่าไม่เพิ่มขึ้นเมื่อมี subscriber เพิ่มขึ้น
- ทดสอบการจำลองกรณี MailFast ล่ม เพื่อยืนยันว่าการสั่งซื้อของลูกค้ายังสำเร็จได้ 100%
- ตรวจสอบอัตราการสูญหายของข้อความ (message loss rate) ใน Message Broker ต้องเป็น 0%

## Open questions

*(เนื่องจากข้อกำหนดไม่ได้ระบุตัวเลข ข้อมูลทีม หรือข้อจำกัดของระบบ จึงบันทึกเป็นคำถามเปิด)*
1. อัตราปริมาณงาน (throughput / peak load) และ SLA latency สูงสุดที่ยอมรับได้ของกระบวนการสั่งซื้อคือเท่าใด?
2. ข้อจำกัดและ rate limit ของ API ผู้ให้บริการภายนอก MailFast เป็นเท่าใด?
3. นโยบายการให้แต้มสะสมของ `loyaltyservice` มีกรอบเวลาที่ต้องได้รับ (SLA) และข้อกำหนดเรื่องความถูกต้องอย่างไร?
4. ทีมพัฒนามีความเชี่ยวชาญเทคโนโลยี Message Broker ตัวใด (เช่น RabbitMQ, Apache Kafka, NATS, Google Cloud Pub/Sub) และมีข้อจำกัดด้าน infrastructure อย่างไร?
