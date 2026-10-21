<!-- Agent: Antigravity / Model: Gemini 3.8 Flash -->

# ADR-0002: Use RabbitMQ for asynchronous post-order processing

- **Status:** Proposed
- **Date:** 2026-10-21
- **Deciders:** Software Architecture Team & ณภัทร เสรีรักษ์ (6632061321)

## Context and problem

ระบบ Online Boutique ในปัจจุบันประมวลผลคำสั่งซื้อผ่าน `checkoutservice.PlaceOrder` โดยเรียก `paymentservice`, `shippingservice` และ `emailservice` แบบ synchronous ตามลำดับผ่าน gRPC (`system/online-boutique/src/checkoutservice/main.go` บรรทัดที่ 252–277)

ความต้องการทางธุรกิจใหม่ (R-12) กำหนดว่าหลังจากคำสั่งซื้อสำเร็จ ระบบจะต้อง:

1. ส่งอีเมลยืนยันคำสั่งซื้อผ่าน MailFast ซึ่งเป็น external email provider รายใหม่แทนที่ mock logic เดิม
2. มอบแต้มสะสม (loyalty points) ผ่าน `loyaltyservice` ที่จะพัฒนาขึ้นใหม่ โดยต้องได้รับแต้มภายใน 1 นาที และมอบให้เพียงครั้งเดียวเท่านั้น (exactly-once)
3. บันทึกข้อมูล analytics event "order placed" ให้ทีมการตลาด
4. รองรับการเพิ่มงานใหม่ในปีหน้า เช่น การส่ง SMS, การตรวจจับทุจริต (fraud scoring) และการอัปเดตระบบแนะนำสินค้า

**ข้อเท็จจริงและข้อจำกัดสำคัญจาก Scenario:**

- **Checkout latency:** ค่า p95 ของ `PlaceOrder` ปัจจุบันอยู่ที่ประมาณ 300 ms และมีข้อกำหนดเข้มงวดว่าต้องไม่เกิน **500 ms**
- **ปริมาณงานช่วงพีก:** สูงถึง **200 orders/s** ในช่วงเทศกาล 11.11
- **ข้อจำกัดของ MailFast:** latency สูงมาก (p95 อยู่ที่ **1.5 s**, p99 อยู่ที่ **8 s**), มีอัตราความล้มเหลวประมาณ **0.5%**, และจำกัดอัตราการเรียก (rate limit) ไว้ที่ **50 emails/s**
- **Availability:** คำสั่งซื้อต้องไม่ล้มเหลวแม้ว่า email, loyalty หรือ analytics จะล่ม
- **ข้อจำกัดด้านทีมและโครงสร้างพื้นฐาน:** ทีมพัฒนา 4 คนถนัด Go และ gRPC โดยมี 2 คนเคยใช้งาน **RabbitMQ** มาก่อน; ระบบทั้งหมดรันบน Kubernetes และบริษัทมีนโยบายเลือกใช้ส่วนประกอบแบบ **open-source ที่ติดตั้งเอง (self-hosted)** มากกว่า cloud-managed service

หากประมวลผลงานเหล่านี้แบบ synchronous ต่อไป เวลา latency รวมจะพุ่งเกิน 1.8 วินาที (ละเมิด SLA 500 ms) และช่วงพีก 200 orders/s จะเกินขีดจำกัด 50 emails/s ของ MailFast ทันที

## Decision drivers

1. **Performance (Latency & Throughput):** p95 ของ `PlaceOrder` ต้องอยู่ต่ำกว่า 500 ms ภายใต้โหลดช่วงพีก 200 orders/s
2. **Availability:** การล่มหรือตอบสนองช้าของบริการภายนอก (MailFast) หรือบริการรอง ต้องไม่กระทบต่อความสำเร็จของการสั่งซื้อ
3. **Rate Limiting & Buffering:** ต้องมีกลไกหน่วง/จำกัดอัตราการส่งอีเมลไปยัง MailFast ไม่ให้เกิน 50 emails/s ในช่วงพีก
4. **Reliability & Idempotency:** ต้องการันตีการส่งมอบงาน และแจกแต้มสะสมได้ภายใน 1 นาทีอย่างถูกต้องเพียงครั้งเดียว (no double points)
5. **Modifiability / Extensibility:** สามารถเพิ่ม subscriber ใหม่ (SMS, fraud scoring) ในปีหน้าได้ง่าย โดยไม่ต้องแก้โค้ด `checkoutservice`
6. **Operability & Team Familiarity:** ต้องเป็น open-source รันบน Kubernetes ได้เอง และสอดคล้องกับทักษะของทีม 4 คน (Go/gRPC, ประสบการณ์ RabbitMQ 2 คน)

## Considered options

1. **Option 1: Synchronous gRPC calls from `checkoutservice` with goroutines**  
   ให้ `checkoutservice` เรียก gRPC หรือยิง background goroutine ใน memory ไปยัง services ต่างๆ
2. **Option 2: Self-hosted RabbitMQ on Kubernetes (Topic/Fanout Exchange)**  
   ติดตั้ง RabbitMQ cluster บน Kubernetes ให้ `checkoutservice` ทำหน้าที่ publish เหตุการณ์ `OrderPlaced` เข้า exchange แล้วแยกคิวสำหรับแต่ละ consumer (`emailservice`, `loyaltyservice`, analytics worker)
3. **Option 3: Managed Cloud Pub/Sub or Apache Kafka**  
   ใช้ Google Cloud Pub/Sub หรือติดตั้ง Apache Kafka cluster สำหรับ event streaming

### Pros and cons of the options

| Option                                           | Pros                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Cons                                                                                                                                                                                                                                                                |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Option 1: Synchronous / in-memory goroutines** | - ไม่ต้องติดตั้ง infrastructure เพิ่มเติม<br>- โค้ดพัฒนาต่อยอดได้ทันที                                                                                                                                                                                                                                                                                                                                                                                                                | - **ไม่ผ่านเกณฑ์ Performance:** หากเรียก synchronous จะใช้เวลา > 1.8 s (เกิน 500 ms)<br>- **ทำลาย Availability:** Goroutine ใน memory จะสูญหายเมื่อ pod restart ระหว่างช่วงพีก<br>- ไม่สามารถควบคุม rate limit 50/s ของ MailFast ได้เมื่อโหลดพุ่งถึง 200/s          |
| **Option 2: Self-hosted RabbitMQ on Kubernetes** | - **ตอบโจทย์ Latency:** `checkoutservice` ใช้เวลาเพียง ~2-5 ms ในการ publish message แล้วตอบกลับผู้ใช้ได้ทันที (p95 รวม ~305 ms)<br>- **Rate Limiting & Buffering สมบูรณ์แบบ:** คิวของ `emailservice` สามารถบัฟเฟอร์โหลด 200/s และควบคุม worker ให้ดึงไปยิง MailFast ไม่เกิน 50/s<br>- **ตรงกับทีมและ Ops:** ทีมมี 2 คนเคยใช้ RabbitMQ, รองรับ Go อย่างดี, เป็น open-source รันบน k8s ได้ตามนโยบาย<br>- **Modifiability สูง:** รองรับงานปีหน้าได้ทันทีเพียง bind คิวใหม่เข้า exchange | - ต้องดูแล StatefulSet / RabbitMQ Operator บน Kubernetes<br>- ต้องจัดการ message persistence และ idempotency ในฝั่ง consumer                                                                                                                                        |
| **Option 3: Apache Kafka / Cloud Pub/Sub**       | - Kafka รองรับ throughput มหาศาลและ replay event ได้<br>- Cloud Pub/Sub ไม่ต้องดูแล server เอง                                                                                                                                                                                                                                                                                                                                                                                        | - ขัดแย้งกับข้อจำกัดด้านนโยบายบริษัท (ปฏิเสธ cloud-managed service)<br>- Kafka มี operational overhead สูงเกินจำเป็นสำหรับปริมาณงาน 200 orders/s และไม่มีคนในทีมมีประสบการณ์ Kafka<br>- ไม่เหมาะกับการควบคุม rate limit ระดับ 50/s เท่ากับ queue model ของ RabbitMQ |

## Decision

เลือก **Option 2: Self-hosted RabbitMQ on Kubernetes (Topic/Fanout Exchange)**

**รายละเอียดการออกแบบ:**

1. **Event Publishing:** เมื่อ `checkoutservice.PlaceOrder` ตัดเงินและบันทึกคำสั่งซื้อสำเร็จ จะ publish ข้อความ `OrderPlaced` ไปยัง RabbitMQ Exchange แบบ Fanout/Topic จากนั้นคืน response `PlaceOrderResponse` ให้ลูกค้าทันที ทำให้ latency อยู่ในระดับ ~300 ms (ต่ำกว่าขีดจำกัด 500 ms อย่างสบาย)
2. **MailFast Rate Limiting:** สร้างคิว `order.email` สำหรับ `emailservice` โดยตั้งค่า worker prefetch count และ concurrency ให้ยิงไปยัง MailFast รวมกันไม่เกิน **50 requests/s** คิวจะทำหน้าที่ buffer คำสั่งซื้อส่วนเกินในช่วงพีก 200 orders/s ไว้ได้อย่างปลอดภัย
3. **Loyalty Exactly-Once:** สร้างคิว `order.loyalty` ให้ `loyaltyservice` ประมวลผล โดยออกแบบ consumer ให้เป็น idempotent โดยนำ `order_id` ไปบันทึกลง database/Redis พร้อม unique constraint เพื่อป้องกันการแจกแต้มซ้ำ และประมวลผลทันทีเพื่อให้ได้รับแต้มภายใน 1 นาที
4. **Extensibility:** ระบบ analytics และฟังก์ชันใหม่ในปีหน้า (SMS, fraud scoring) สามารถสร้างคิวใหม่มา bind กับ exchange ได้ทันทีโดยไม่ต้องแก้ไขโค้ด `checkoutservice`
5. **Dead-Letter Exchange (DLX / DLQ):** กำหนด Dead-Letter Queue (DLQ) สำหรับเก็บข้อความที่ retry ครบ 3–5 ครั้งแล้วยังไม่สำเร็จ เพื่อไม่ให้คิวหลักติดขัดจาก poison message หาก MailFast ล่มต่อเนื่อง พร้อมส่ง alert แจ้งเตือนทีม SRE
6. **Infrastructure & Quorum Queues:** กำหนดให้คิวทั้งหมดใน RabbitMQ เป็นแบบ **Quorum Queues (ใช้ Raft consensus)** แทน Classic Mirrored Queues เพื่อความทนทานของข้อมูลสูงสุดบน Kubernetes และป้องกันข้อความสูญหายเมื่อโหนดล่ม
7. **Auto-scaling สำหรับ Loyalty SLA:** ติดตั้ง KEDA หรือ Kubernetes HPA ให้ทำ Auto-scaling ขยาย pod ของ `loyaltyservice` ตามจำนวนข้อความที่ค้างในคิว เพื่อรับประกัน SLA ว่าลูกค้าจะได้รับแต้มสะสมภายใน 1 นาที แม้ในช่วงพีก 200 orders/s

## Consequences

- **Positive:**
  - **Performance (Latency):** ปลดภาระงาน post-order ออกจาก critical path ทำให้ p95 latency ของ `PlaceOrder` คงอยู่ที่ ~300–310 ms ซึ่งผ่านเป้าหมาย ≤ 500 ms อย่างสมบูรณ์
  - **Availability:** หาก MailFast ล่ม ( failure rate 0.5%) หรือ `loyaltyservice` ขัดข้อง คำสั่งซื้อของลูกค้ายังคงสำเร็จ 100% ข้อความจะรออยู่ในคิวเพื่อ retry
  - **Rate Limit & Scalability:** สามารถรองรับโหลดพีก 200 orders/s ในเทศกาล 11.11 ได้โดย MailFast ไม่ปฏิเสธคำขอด้วย error 429
  - **Modifiability:** ทีมการตลาดสามารถเพิ่ม SMS, fraud scoring, หรือ recommendation updates ในปีหน้าได้อิสระ
- **Negative:**
  - **Operability:** ต้องติดตั้งและดูแลรักษา RabbitMQ บน Kubernetes (แนะนำให้ใช้ RabbitMQ Cluster Kubernetes Operator) พร้อม persistent volume
  - **Consistency:** สถานะของระบบหลังการสั่งซื้อเป็นแบบ Eventual Consistency (ผู้ใช้อาจได้รับอีเมลหรือแต้มช้าลงเล็กน้อยในช่วงพีก 11.11)
- **Risks and mitigations:**
  - _Risk:_ ข้อความสูญหายหากโหนด RabbitMQ ขัดข้อง  
    _Mitigation:_ กำหนดคิวเป็น durable และ publish message แบบ persistent พร้อมตั้งค่า RabbitMQ clustering แบบ Quorum Queues
  - _Risk:_ การส่งข้อความซ้ำจากการ retry ส่งผลให้ลูกค้าได้แต้มสะสมซ้ำ  
    _Mitigation:_ ใช้ `order_id` เป็น Idempotency Key ใน `loyaltyservice` ตรวจสอบก่อนเพิ่มคะแนน

## Validation

- **Load Testing:** ทำการทดสอบระบบที่ 200 orders/s ต่อเนื่อง 10 นาที โดยใช้เครื่องมือจำลองโหลด (Locust) เพื่อยืนยันว่า p95 latency ของ `PlaceOrder` ไม่เกิน 500 ms
- **MailFast Throttling Test:** ตรวจสอบ metric การเรียก MailFast ว่าไม่เกิน 50 requests/s ตลอดช่วงทดสอบ
- **Fault Injection:** จำลองสถานการณ์ MailFast ล่ม 5 นาที ยืนยันว่าการสั่งซื้อสำเร็จ 100% และเมื่อ MailFast กลับมา ข้อความในคิวถูกส่งต่อจนครบโดยไม่มีข้อความสูญหาย
- **Idempotency Verification:** ทดสอบยิงข้อความ `OrderPlaced` ซ้ำในคิว loyalty ยืนยันว่าไม่มีการให้แต้มซ้ำ
