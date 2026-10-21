<!-- Agent: Antigravity / Model: Gemini 3.8 Flash -->

# ATAM Review: ADR-0002 Use RabbitMQ for asynchronous post-order processing

- **Reviewer:** Architecture Review Board (AI Chief Analyst)
- **Document Under Review:** `docs/adr/0002-use-rabbitmq-for-post-order-processing.md`
- **Context Evidence:** `docs/scenario.md`

---

## 1. Quality-attribute scenarios

1. **Scenario 1: Checkout Latency during Peak Traffic (Performance)**
   - *Source:* ลูกค้าสั่งซื้อสินค้าในช่วงเทศกาลโปรโมชัน 11.11
   - *Stimulus:* อัตราคำสั่งซื้อพุ่งสูงสุดถึง **200 orders/s**
   - *Environment:* ระบบทำงานในสภาวะโหลดสูงสุดบน Kubernetes cluster
   - *Response:* `checkoutservice` หักเงิน สำรองค่าจัดส่ง ล้างตะกร้าสินค้า และ publish เหตุการณ์ `OrderPlaced` ไปยัง RabbitMQ แล้วส่ง response กลับทันที
   - *Response Measure:* ค่า p95 latency ของ `PlaceOrder` คงอยู่ที่ **≤ 500 ms** (คาดการณ์ ~305 ms) โดยไม่ถูกหน่วงจากบริการภายนอก

2. **Scenario 2: MailFast Failure and Rate Limiting (Availability & Robustness)**
   - *Source:* ผู้ให้บริการภายนอก MailFast
   - *Stimulus:* MailFast เกิดความล่าช้า (p99 อยู่ที่ 8 วินาที), ล้มเหลว 0.5% ของคำขอ หรือเกิด downtime นาน 5 นาที ขณะที่คำสั่งซื้อเข้ามา 200 orders/s
   - *Environment:* การทำงานปกติและช่วงโหลดสูง
   - *Response:* คิว `order.email` ใน RabbitMQ ทำหน้าที่บัฟเฟอร์ข้อความ และควบคุม worker ส่งคำขอไม่เกิน **50 requests/s** ตาม rate limit; ข้อความที่ส่งไม่สำเร็จจะถูก retry และย้ายเข้า Dead-Letter Queue (DLQ) เมื่อครบกำหนด
   - *Response Measure:* คำสั่งซื้อของลูกค้าสำเร็จ 100% (ไม่มีการล้มเหลวเพราะ email) และ MailFast ไม่ปฏิเสธคำขอด้วย error 429

3. **Scenario 3: Timely and Exactly-Once Loyalty Awarding (Reliability & Timeliness)**
   - *Source:* คำสั่งซื้อที่ชำระเงินสำเร็จ
   - *Stimulus:* ลูกค้าทำรายการสั่งซื้อเรียบร้อย
   - *Environment:* ช่วงโหลดพีก 200 orders/s
   - *Response:* `loyaltyservice` ดึงข้อความจาก Quorum Queue ตรวจสอบ `order_id` เพื่อป้องกันการให้แต้มซ้ำ (idempotent) และขยาย pod อัตโนมัติด้วย KEDA ตามความยาวคิว
   - *Response Measure:* แต้มสะสมถูกบันทึกให้ลูกค้า**ภายใน 1 นาที** และได้รับ**เพียงครั้งเดียวเท่านั้น (0 double awards)**

---

## 2. Findings table

| Type | Finding | Why it matters |
|---|---|---|
| **Non-risk** | การแยก asynchronous processing ด้วย RabbitMQ Fanout/Topic Exchange ช่วยปลด MailFast ออกจาก synchronous checkout path | ทำให้ระบบ checkout บรรลุ SLA p95 ≤ 500 ms ได้อย่างแน่นอน และคิวสามารถ buffer โหลดพีก 200 orders/s ไม่ให้เกิน rate limit 50 req/s ของ MailFast |
| **Risk** | **Dual-Write / Crash-after-Payment Risk:** ใน `checkoutservice` หากหักเงินบัตรเครดิตสำเร็จ แต่ pod แครชหรือเกิด network partition ก่อนที่คำสั่ง publish สู่ RabbitMQ จะทำงาน จะทำให้ไม่มี message ส่งเข้า broker | ลูกค้าถูกหักเงินและได้รับสินค้า แต่ไม่เคยได้รับอีเมลยืนยัน ไม่ได้แต้ม loyalty และไม่มีข้อมูล analytics เกิดความไม่สอดคล้องของข้อมูลระดับวิกฤต |
| **Trade-off point** | **Eventual Consistency แลกกับ Performance & Availability:** การตอบรับคำสั่งซื้อทันทีแล้วส่งต่องานเข้าคิว | ปรับปรุง latency และความพร้อมใช้งานของ checkout ได้มหาศาล แต่แลกมาด้วยความล่าช้าในการส่งอีเมลยืนยันคำสั่งซื้อ (หากมีคิวสะสมช่วง 11.11 อีเมลอาจมาช้าหลายนาที) ซึ่งส่งผลต่อ User Experience |
| **Sensitivity point** | การตั้งค่า Concurrency และ Rate Limiting ของ `emailservice` worker | ระบบมีความอ่อนไหวสูงมากต่อจำนวน worker หาก scale pod ของ `emailservice` โดยไม่มี Distributed Rate Limiter รวม จะยิงเกิน 50 req/s และถูก MailFast block ทันที |
| **Non-risk** | การกำหนดใช้ **Quorum Queues (Raft)** และการเพิ่ม **Dead-Letter Queue (DLQ)** | ป้องกันข้อความสูญหายเมื่อ Kubernetes node ขัดข้อง และป้องกัน poison message วนซ้ำในคิวจนทำให้ระบบค้าง |

---

## 3. Missing or weak

1. **ขาดการออกแบบ Transactional Outbox Pattern หรือ Local Fallback:**  
   ADR กำหนดให้ publish ไป RabbitMQ ทันทีหลังตัดเงิน (`chargeCard`) ซึ่งเป็น dual-write ที่ไม่มี transaction ครอบคลุม หาก network หลุดช่วงเสี้ยววินาทีนี้ message จะหายไปถาวร
2. **ไม่ได้ระบุการคำนวณ Capacity และ Sizing ของ RabbitMQ Storage:**  
   หาก MailFast ล่ม 1 ชั่วโมงในช่วงพีก 200 orders/s คิวอีเมลจะสะสมข้อความถึง $200 \times 3600 = 720,000$ ข้อความ ADR ยังไม่ได้ระบุขนาด disk และ memory limit ที่ต้องรองรับ
3. **การควบคุม Rate Limit แบบรวม (Global Rate Limiting):**  
   ADR ระบุการตั้งค่า worker prefetch count แต่หาก `emailservice` ถูก deploy หลาย replicas บน Kubernetes การตั้งค่าในระดับ pod จะไม่สามารถการันตี Global Rate Limit ที่ 50 requests/s ได้

---

## 4. Verdict

**Accept with changes** (ยอมรับ แต่มีเงื่อนไขให้ปรับปรุงก่อนนำไป Implement จริง)

**3 การเปลี่ยนแปลงที่สำคัญที่สุดที่ต้องแก้ไข:**
1. **แก้ปัญหา Dual-Write:** เพิ่มกลไก Retry/Outbox หรือ fallback logging ลง persistent store ใน `checkoutservice` เพื่อการันตีว่าคำสั่งซื้อที่หักเงินแล้วจะต้องถูกส่งเข้า RabbitMQ เสมอ แม้ pod จะ restart
2. **ทำ Centralized Rate Limiter สำหรับ MailFast:** กำหนดให้ใช้ Token Bucket หรือ Redis Rate Limiter ร่วมกันระหว่าง worker ของ `emailservice` เพื่อให้มั่นใจว่าจะไม่ยิงเกิน 50 requests/s รวมทั้งระบบ
3. **กำหนดนโยบาย Disk Alert และ Queue Backlog Sizing:** วางแผนขอบเขตขนาดของ Quorum Queues และแจ้งเตือนทีม SRE เมื่อคิวสะสมเกินขีดจำกัดเพื่อป้องกัน RabbitMQ memory exhaustion
