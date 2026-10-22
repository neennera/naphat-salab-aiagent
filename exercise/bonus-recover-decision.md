**ภาษา:** ภาษาไทย | [English](bonus-recover-decision.en.md)

# โบนัส (ไม่บังคับ ทำที่บ้าน): กู้คืนการตัดสินใจที่ไม่ได้บันทึกไว้ (ประมาณ 15 นาที)

> **เชื่อมโยงกับการบรรยาย:** T5 (Dhar et al. สร้างการตัดสินใจจากโค้ดและแผนภาพเพื่อลด _การสูญหายของความรู้_ หรือ knowledge vaporization), AICH4
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

เลือกข้อกล่าวอ้าง**ห้าข้อ**จากตารางของ agent โดยเป็นข้อที่ agent ระบุว่า _Evidenced_ (มีหลักฐาน) อย่างน้อยสองข้อ และ _Inferred_ (อนุมานเอา) อย่างน้อยสองข้อ แล้วตรวจสอบแต่ละข้อด้วยตัวเอง

| #   | ข้อกล่าวอ้าง (ย่อ)                                                        | AI ระบุว่า | นิสิตระบุว่า | เหตุผล (path ของไฟล์หรือการให้เหตุผล)                                                     |
| --- | ------------------------------------------------------------------------- | ---------- | ------------ | ----------------------------------------------------------------------------------------- |
| 1   | ทุกเซอร์วิสนิยาม RPC และ data types ร่วมกันในไฟล์ `demo.proto` ไฟล์เดียว  | Evidenced  | Evidenced    | ปรากฏชัดเจนใน `system/online-boutique/protos/demo.proto` (Line 1–262)                     |
| 2   | `frontend` เชื่อมต่อไปยัง `checkoutservice` ผ่าน gRPC Client Connection   | Evidenced  | Evidenced    | มีโค้ดเรียกใช้งานใน `system/online-boutique/src/frontend/main.go` (`mustConnGRPC`)        |
| 3   | มีการกำหนด Timeout ในการสร้างการเชื่อมต่อ gRPC Client ไว้ที่ 3 วินาที     | Evidenced  | Evidenced    | มีโค้ด `time.Second*3` ใน `system/online-boutique/src/checkoutservice/main.go` (Line 212) |
| 4   | รวมไฟล์ proto เพื่อให้ผู้เรียนเข้าใจสถาปัตยกรรมเดโมได้ง่ายในจุดเดียว      | Inferred   | Inferred     | ในโค้ดไม่มีคอมเมนต์ระบุ เป็นการอนุมานจากเป้าหมายโครงการ demo ของ GCP                      |
| 5   | ทีมผู้ออกแบบเปรียบเทียบ gRPC กับ REST และ Message Queue ก่อนตัดสินใจเลือก | Inferred   | Inferred     | ในโค้ดไม่มีบันทึกการเปรียบเทียบทางเลือกอื่น AI อนุมานตามโครงสร้างมาตรฐาน ADR              |

**นับ:** จากข้อกล่าวอ้างทั้งหมด agent ระบุว่า _Evidenced_ **4** ข้อ ตามที่นิสิตตรวจ **4** ข้อ

## สะท้อนคิด

1. **agent เคยระบุการเดาว่าเป็น _Evidenced_ หรือไม่ ยกตัวอย่าง หรืออธิบายว่าตรวจสอบอย่างไรว่าไม่มี:**
   ไม่พบ การตรวจสอบทำได้โดยดูว่าข้อที่ระบุเป็น `Evidenced` ต้องมี path ของไฟล์และชื่อฟังก์ชันในโค้ดอ้างอิงชัดเจน หากเป็นเหตุผลเบื้องหลัง (Rationale) หรือทางเลือกที่ถูกปฏิเสธ AI จะจัดเป็น `Inferred` ทั้งหมด

2. **หากนักพัฒนาใหม่อ่านเฉพาะ ADR ที่กู้คืนมานี้ เขาอาจเข้าใจผิดในเรื่องใด:**
   อาจเข้าใจผิดว่าการรวม schema ทุกเซอร์วิสไว้ใน `demo.proto` ไฟล์เดียวเป็น Best Practice ในระดับ Production ทั้งที่จริงทำเพื่อความสะดวกในการ Demo บน Kubernetes เท่านั้น

3. **AI กู้คืนอะไรจากโค้ดได้บ้าง และอะไรที่มีเพียงสถาปนิกผู้ออกแบบเดิมเท่านั้นที่รู้ ยกตัวอย่างหนึ่งตัวอย่างจาก ADR ของนิสิต:**
   AI กู้คืนข้อเท็จจริงเชิงโครงสร้างได้ (What/How) เช่น มี RPC ใดบ้างและเรียกหากันอย่างไร แต่ไม่รู้บริบทการตัดสินใจเบื้องหลัง (Why) เช่น เหตุผลที่ยอมรับความเสี่ยงเรื่อง Cascading failure โดยไม่ใช้ Message Queue ตั้งแต่แรก
