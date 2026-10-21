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

| ผู้เรียก → ผู้ถูกเรียก                        | หลักฐานที่ AI อ้าง                                                   | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ                                                                                                                 |
| --------------------------------------------- | -------------------------------------------------------------------- | ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------ |
| frontend → cartservice                        | `frontend.yaml` (CART_SERVICE_ADDR="cartservice:7070")               | ✅                             | มีการเซ็ตค่า env ชี้ไปที่ cartservice จริง                                                                                     |
| cartservice → redis-cart                      | `cartservice.yaml` (REDIS_ADDR="redis-cart:6379")                    | ✅                             | มีการนิยาม Deployment redis-cart และเซ็ต REDIS_ADDR จริง                                                                       |
| checkoutservice → paymentservice              | `checkoutservice.yaml` (PAYMENT_SERVICE_ADDR="paymentservice:50051") | ✅                             | มีการกำหนดค่า env นี้จริง                                                                                                      |
| recommendationservice → productcatalogservice | `recommendationservice.yaml` (PRODUCT_CATALOG_SERVICE_ADDR)          | ✅                             | มีการส่งค่า env ไปยัง productcatalogservice:3550 จริง                                                                          |
| frontend → shoppingassistantservice           | `frontend.yaml` (SHOPPING_ASSISTANT_SERVICE_ADDR)                    | ❌                             | พบค่า env ใน frontend แต่ไม่มี manifest ของ shoppingassistantservice ในโฟลเดอร์ manifests เพราะถูก comment ไว้ใน kustomization |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย):

ในข้อสุดท้าย พบว่า frontend มีการตั้งค่า env ชี้ไปที่ shoppingassistantservice แต่ระบบไม่ได้ deploy service (เพราะถูก comment ไว้ใน kustomization.yaml) ทำให้ถ้าดูแต่ env จะเข้าใจผิดว่ามี service นี้ และในไฟล์ manifest ก็ยังไม่เห็น Database ของ service อื่นๆ เช่น productcatalogservice

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด

ไฟล์ .proto เป็นเพียงการนิยาม Interface ของ API Contract และ Data Schema ว่ามี method อะไรบ้าง แต่ไม่ได้บอก Runtime Call Graph ว่า service ไหนเป็นคนเรียก service ไหนและไม่บอก infrastructure หรือ Data Store เช่น Redis ทำให้ใน Run A ไม่สามารถสร้างเส้นเชื่อมโยง dependency ระหว่าง container ได้จนกว่าจะได้อ่าน Kubernetes manifests ใน Run B

2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ

การดูเฉพาะ environment variables ใน Kubernetes manifests อาจทำให้เข้าใจผิดว่ามี service ทุกตัวทำงานอยู่จริง แต่จริง ๆ แล้ว service นั้นไม่ได้ถูก deploy อยู่ในระบบ หรือการไม่รู้ว่า dependencies ตัวไหนเป็น critical path (เช่น adservice หรือ recommendationservice ที่ถ้าล่ม หน้าเว็บยังอาจทำงานต่อไปได้)
