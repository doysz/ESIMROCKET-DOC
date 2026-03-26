# eSIMRocket Webhook Integration Guide

คู่มือฉบับนี้จัดทำขึ้นสำหรับทีมนักพัฒนา (Developers) ของพาร์ทเนอร์ เพื่อใช้ในการเชื่อมต่อระบบ Webhook ของ eSIMRocket เอกสารนี้สรุปข้อมูล **"การทำงานจริง 100%"** ของระบบ เพื่อให้สามารถนำไปพัฒนา (Integration) วางแผน State Machine และตรวจสอบความถูกต้อง (Cross-check) ได้อย่างแม่นยำ

---

## 1. ข้อมูลพื้นฐานและระบบความปลอดภัย (Security & Envelope)

ระบบ eSIMRocket จะส่ง Webhook แบบ **POST** Request ไปยัง Endpoint ที่พาร์ทเนอร์ตั้งค่าไว้ในระบบ

### 1.1 โครงสร้าง Payload พื้นฐาน (Envelope)
ทุก Webhook จะมีโครงสร้างหลักเหมือนกันดังนี้:
```json
{
  "event": "esim.activated",
  "eventType": "SMDP_EVENT",
  "timestamp": 1760000000,
  "data": { ... }
}
```
* **`event`**: ชื่อ Event หลักที่ควรใช้ในการทำ Routing และประมวลผลฝั่งพาร์ทเนอร์ (เช่น `esim.activated`, `esim.depleted`)
* **`eventType`**: กลุ่มของ Event เพื่อความสะดวกในการจัดหมวดหมู่ (เช่น `SMDP_EVENT`, `ORDER_STATUS`)
* **`timestamp`**: เวลาที่เกิด Event (Unix Timestamp ระดับวินาที)
* **`data`**: ข้อมูลรายละเอียดของ Event นั้นๆ (จะแตกต่างกันไปตามประเภทของ `event`)

### 1.2 Security Headers
เพื่อความปลอดภัย ระบบจะแนบ Headers เหล่านี้ไปกับทุก Request:
- `Content-Type: application/json`
- `User-Agent: eSIMRocket-Webhook/1.0`
- `X-Webhook-Access-Code`: รหัส Access Code ของพาร์ทเนอร์ (หากตั้งค่าไว้ใน Portal)
- `X-Webhook-Signature`: HMAC SHA256 Signature สำหรับตรวจสอบความน่าเชื่อถือของข้อมูล (หากตั้งค่า Webhook Secret ไว้)
  * **รูปแบบที่ส่งไป**: `sha256=<hex_digest>`
  * **วิธีการตรวจสอบฝั่งพาร์ทเนอร์**: นำ **Raw Request Body** ทั้งก้อน (ห้ามแปลงเป็น Object ก่อน) มาคำนวณ HMAC-SHA256 ด้วย Webhook Secret ของคุณ แล้วนำไปเทียบกับค่าใน Header หากตรงกันแสดงว่าข้อมูลมาจากระบบเราจริงและไม่ได้ถูกแทรกแซงระหว่างทาง

---

## 2. นิยามสถานะของ eSIM (Status Glossary)

เพื่อให้ฝั่งพาร์ทเนอร์สามารถนำไปสร้าง UI/UX และวาง State Machine (เปิด/ปิดปุ่มต่างๆ) ได้อย่างถูกต้อง **กรุณายึดค่าจากฟิลด์ `data.esimStatus` ใน Payload เป็นแหล่งอ้างอิงหลัก** (Source of Truth สำหรับสถานะ eSIM)

| `data.esimStatus` | ความหมายทางธุรกิจ | การนำไปใช้ฝั่งพาร์ทเนอร์ (UX / Flow) | Action ที่ระบบอนุญาตให้ทำได้ (UI ควรเปิด) |
|---|---|---|---|
| **`GOT_RESOURCE`** | **เตรียมพร้อม:** โปรไฟล์ถูกจัดสรรและดาวน์โหลดลงเครื่องแล้ว แต่ยัง **ไม่เริ่มใช้งานดาต้า** (Data Usage = 0) | แสดงสถานะ "พร้อมใช้งาน" (Ready to use) หรือ "ติดตั้งแล้ว" ควรแสดงคำแนะนำการเปิด Data Roaming ให้ลูกค้า | Top-up, Cancel (ถ้ามีนโยบายคืนเงิน) |
| **`IN_USE`** | **กำลังใช้งาน:** เริ่มมีการใช้งานดาต้าแล้ว | แสดงสถานะ "กำลังใช้งาน" แสดงแถบการใช้งาน Data (Usage Bar) เปิดรับแจ้งเตือนเน็ตใกล้หมด | Top-up, Revoke |
| **`USED_UP`** | **โควต้าหมด:** ดาต้าหมดแล้ว แต่ซิมยังไม่หมดอายุ (วันยังเหลือ) | แสดงสถานะ "อินเทอร์เน็ตหมด" เสนอหน้าต่างให้ลูกค้าซื้อแพ็กเกจเสริม (Top-up) ทันที | Top-up, Revoke |
| **`USED_EXPIRED`** | **หมดอายุ (มีการใช้งาน):** หมดระยะเวลาของแพ็กเกจและเคยมีการใช้งานดาต้าไปแล้ว | แสดงสถานะ "หมดอายุ" แนะนำให้ลูกค้าซื้อ eSIM ใหม่ | (สถานะสิ้นสุด) |
| **`UNUSED_EXPIRED`** | **หมดอายุ (ยังไม่ใช้งาน):** หมดอายุโดยที่ยังไม่ได้เริ่มใช้ดาต้าเลยแม้แต่ไบต์เดียว | แสดงสถานะ "หมดอายุ (ไม่ได้ใช้งาน)" อาจใช้เข้า Flow คืนเงิน หรือร้องเรียนตามนโยบายของพาร์ทเนอร์ | (สถานะสิ้นสุด) |
| **`CANCEL`** | **ยกเลิก:** ยกเลิกการใช้งานและคืนเงินเรียบร้อยแล้ว | แสดงสถานะ "ยกเลิกแล้ว" ซ่อนออกจากรายการใช้งานหลัก | (สถานะสิ้นสุด) |
| **`REVOKED`** | **เพิกถอน:** ปิดการทำงานของซิมถาวร (ลบทิ้งจากระบบเครือข่าย) | แสดงสถานะ "เพิกถอน" ใช้สำหรับกรณีทุจริตหรือทำผิดเงื่อนไข | (สถานะสิ้นสุด) |

> **หมายเหตุสำหรับ Developer:** 
> 1. ฟิลด์ `data.smdpStatus` ที่แนบไปในบาง Payload มีไว้เพื่อใช้ในการ Debug ระดับเครือข่าย (เช่น ดูว่ากำลัง INSTALLATION หรือ RELEASED) ไม่ควรนำมาใช้เป็นตัวควบคุม State Machine หลัก ให้ใช้ `data.esimStatus` เท่านั้น
> 2. API ของ eSIMRocket (`/esim/topup`, `/esim/cancel` ฯลฯ) ไม่ได้มีการ Hard-block สถานะก่อนเรียกไปยัง Upstream เพื่อความยืดหยุ่นสูงสุด ดังนั้นการควบคุมว่า "สถานะไหนกดปุ่มอะไรได้บ้าง" ควรถูกบังคับ (Enforce) ที่ฝั่ง UI ของพาร์ทเนอร์ตามตารางด้านบน

---

## 3. Webhook Event Matrix (ตารางการทำงาน 100%)

ตารางนี้สรุปความสัมพันธ์แบบ "1 Event = 1 Case" อย่างชัดเจน เพื่อให้ทราบว่าเมื่อเกิดเหตุการณ์ใด พาร์ทเนอร์จะได้รับ `event` อะไร และ `data.esimStatus` จะถูกบังคับค่า (Force) เป็นอะไร (ระบบเรามีการทำ Normalization และ Force ค่าให้ตรงตาม State Machine เสมอ)

| เหตุการณ์ที่เกิดขึ้นจาก Upstream (Trigger) | Event ที่ส่งให้พาร์ทเนอร์ (`event`) | กลุ่ม (`eventType`) | ค่า `data.esimStatus` ที่ปรากฏใน Payload |
|---|---|---|---|
| **สั่งซื้อแพ็กเกจสำเร็จ** (ระบบได้รับคำสั่งซื้อ) | `order.created` | `ORDER_STATUS` | *(ไม่อ้างอิง ใช้สถานะ Order)* |
| **ระบบจัดสรร Profile eSIM เรียบร้อย** | `order.completed` | `ORDER_STATUS` | *(ไม่อ้างอิง ใช้สถานะ Order)* |
| **สั่งซื้อล้มเหลว / คืนเงิน** | `order.failed` / `order.refunded`| `ORDER_STATUS` | *(ไม่อ้างอิง ใช้สถานะ Order)* |
| **ดาวน์โหลด/ติดตั้ง eSIM** (แต่ยังไม่ใช้ดาต้า - Usage=0) | `esim.activated` | `SMDP_EVENT` | **`GOT_RESOURCE`** (ถูกคำนวณจากกฎ Unused Window) |
| **เริ่มใช้งาน Data ครั้งแรก** (Network จับสัญญาณ) | `esim.activated` | `SMDP_EVENT` | `IN_USE` |
| **ใช้งาน Data จนหมดโควต้า** | `esim.depleted` | `ESIM_STATUS` | **`USED_UP`** (Force ให้ 100%) |
| **หมดอายุ** (ถึงวัน Expires) | `esim.expired` | `ESIM_STATUS` | `USED_EXPIRED` หรือ `UNUSED_EXPIRED` (แยกเคสให้) |
| **พาร์ทเนอร์สั่งยกเลิก eSIM สำเร็จ** (Cancel) | `esim.cancelled` | `ESIM_STATUS` | **`CANCEL`** (Force ให้ 100%) |
| **พาร์ทเนอร์สั่งเพิกถอน eSIM ถาวร** (Revoke) | `esim.revoked` | `ESIM_STATUS` | **`REVOKED`** (Force ให้ 100%) |
| **แจ้งเตือนเน็ตใกล้หมด** (50%, 80%, 90%) | `esim.low_data_warning`| `DATA_USAGE` | *(อิงตามสถานะล่าสุด ไม่เปลี่ยน State หลัก)* |
| **แจ้งเตือนใกล้วันหมดอายุ** (1 วัน) | `esim.expiry_warning` | `VALIDITY_USAGE`| *(อิงตามสถานะล่าสุด ไม่เปลี่ยน State หลัก)* |
| **อัปเดตปริมาณ Data ปกติ** (Usage Sync) | `esim.usage.update` | `DATA_USAGE` | *(อิงตามสถานะล่าสุด ไม่เปลี่ยน State หลัก)* |
| **การเติมเงินสำเร็จ** (Top-up Completed) | `topup.completed` | `ESIM_STATUS` | *(Payload สำหรับ Top-up โดยเฉพาะ)* |
| **การเติมเงินอยู่ระหว่างดำเนินการ** (Processing) | `topup.status` | `ESIM_STATUS` | *(Payload สำหรับ Top-up โดยเฉพาะ)* |
| **ระบบหลังบ้านมีการเปลี่ยนราคาสินค้า** | `package.catalogue.updated`| `PACKAGE_CATALOGUE_UPDATED` | - |

---

## 4. Payload Reference (โครงสร้างข้อมูลของแต่ละ Event)

ข้อมูลใน `data` จะถูกแปลง Key ให้อยู่ในรูปแบบ **camelCase** เสมอ เพื่อให้ง่ายต่อการนำไปใช้ในภาษา Programming สมัยใหม่

### 4.1 กลุ่ม Order (`order.completed`)
ใช้สำหรับตรวจสอบผลการสั่งซื้อและ **รับค่า ICCID** เพื่อนำไปแสดงผล/ผูกกับบัญชีลูกค้า

```json
{
  "event": "order.completed",
  "eventType": "ORDER_STATUS",
  "timestamp": 1760000001,
  "data": {
    "orderId": "1e857301-b308-42b7-aad7-4adf6f0afbdf",
    "orderNumber": "ORD-1772089207457-RCB94",
    "orderNo": "B26022607000005", // เลขอ้างอิง Upstream
    "partnerReferenceId": "ORDER-10004", // เลขอ้างอิงของพาร์ทเนอร์เอง (สำคัญมาก)
    "orderStatus": "GOT_RESOURCE",
    "status": "COMPLETED",
    "packageCode": "PQ2I5PFIT",
    "quantity": 1,
    "iccids": ["89852240810732710975"] // Array ของ ICCID นำไปใช้ดึง QR Code ต่อไป
  }
}
```

### 4.2 กลุ่มสถานะหลักของ eSIM (`esim.activated` / `esim.depleted` / `esim.expired` / `esim.cancelled`)
ตัวอย่างนี้ใช้สำหรับ Event `esim.activated` (สำหรับ Event อื่นๆ โครงสร้างจะคล้ายกัน แต่เปลี่ยนที่ชื่อ `event` และ `esimStatus`)

```json
{
  "event": "esim.activated", 
  "eventType": "SMDP_EVENT", 
  "timestamp": 1760000002,
  "data": {
    "iccid": "89852240810732710975",
    "orderNumber": "ORD-1772089207457-RCB94",
    "partnerReferenceId": "ORDER-10004",
    "packageCode": "PQ2I5PFIT",
    "packageName": "Thailand 100MB 7Days",
    "esimStatus": "IN_USE", // ฟิลด์ที่ใช้คุม UI State (เป็น GOT_RESOURCE ได้ถ้ายังไม่ใช้งาน)
    "smdpStatus": "ENABLED", // สถานะระดับเน็ตเวิร์ก (เก็บไว้เป็น Log)
    "activatedAt": "2026-02-26T07:01:39+0000",
    "expiredTime": "2026-04-03T07:01:39+0000",
    "totalVolume": 5473566720, // หน่วยเป็น Bytes
    "orderUsage": 278032921, // หน่วยเป็น Bytes (ใช้งานไปแล้ว)
    "remain": 5195533799 // หน่วยเป็น Bytes (เหลือให้ใช้)
  }
}
```
### 4.3 กลุ่มการใช้งาน / แจ้งเตือน (`esim.usage.update` / `esim.low_data_warning`)
ใช้สำหรับการอัปเดตตัวเลขหลอด Data Usage หรือส่ง Push Notification แจ้งเตือนผู้ใช้

```json
{
  "event": "esim.usage.update", // หรือ esim.low_data_warning
  "eventType": "DATA_USAGE",
  "timestamp": 1760000009,
  "data": {
    "iccid": "89852240810732710975",
    "orderNumber": "ORD-1772089207457-RCB94",
    "totalVolume": 5368709120, // (Bytes)
    "orderUsage": 3221225472, // (Bytes)
    "remain": 2147483648, // (Bytes)
    "remainThreshold": 0.4 // สัดส่วนดาต้าที่เหลือ (0.4 = เหลือ 40%)
  }
}
```

### 4.4 กลุ่มการเติมเงิน (`topup.completed`)
ใช้สำหรับอัปเดตปริมาณ Data และวันหมดอายุ **หลังจากเติมเงินสำเร็จ** (มีโครงสร้างซ้อน Object สำหรับข้อมูล Top-up โดยเฉพาะ)

```json
{
  "event": "topup.completed",
  "eventType": "ESIM_STATUS",
  "timestamp": 1760000012,
  "data": {
    "orderNumber": "TOPUP-TEST-0001",
    "iccid": "89852240810734216906",
    "packageCode": "TOPUP_JC045",
    "amount": 1.58, // ต้นทุนที่ตัดจากบัญชีพาร์ทเนอร์
    "partnerReferenceId": "TOPUP-TEST-REF-0001",
    "status": "COMPLETED",
    "data": { 
      // ข้อมูล Snapshot อัปเดตล่าสุดของซิมเบอร์นี้ (นำไปบวกเพิ่ม/อัปเดต UI ทันที)
      "iccid": "89852240810734216906",
      "expiredTime": "2026-03-20T21:37:45.575Z", // วันหมดอายุใหม่
      "totalVolume": 1178599424, // Data รวมทั้งหมด (Bytes)
      "totalDuration": 7, // จำนวนวันใช้งาน (ถ้ามี)
      "orderUsage": 0,
      "topupEsimTranNo": "26022617020013"
    }
  }
}
```

### 4.5 กลุ่มข้อมูลแคตตาล็อกอัปเดต (`package.catalogue.updated`)
ส่งเมื่อระบบเรามีการเปลี่ยนราคา หรือแก้ไข Package ใดๆ ให้พาร์ทเนอร์นำไปอัปเดตในระบบตัวเอง

```json
{
  "event": "package.catalogue.updated",
  "eventType": "PACKAGE_CATALOGUE_UPDATED",
  "timestamp": 1760000014,
  "data": {
    "obj": {
      "changes": [
        {
          "changeType": "UPDATED", // หรือ CREATED / DEACTIVATED
          "packageCode": "PQ2I5PFIT",
          "before": { "price": 1000 },
          "after": { "price": 1200 }
        }
      ]
    }
  }
}
```

---

## 5. คู่มือการทดสอบ (Testing Guide)

เรามีเครื่องมือให้ทีม Dev จำลองยิง Webhook ข้อมูลจริงไปยังระบบของคุณ เพื่อทดสอบ Parsing และ Routing ได้ง่ายๆ

### 5.1 ทดสอบด้วย Postman Collection (แนะนำ)
เพื่อความสะดวกในการส่งครบทุกเคส ดาวน์โหลดไฟล์ Postman Collection ได้ที่ Repository นี้:
👉 `docs/webhook-matrix.postman_collection.json` 
1. นำเข้า (Import) สู่ Postman ของคุณ
2. ตั้งค่าตัวแปร (Variables): `baseUrl`, `apiKey`, `apiSecret`
3. กดยิงทดสอบ (Send) ทีละเคสได้เลย ระบบจะสร้าง Webhook ปลอมที่มี Signature ถูกต้องยิงไปหาเซิร์ฟเวอร์ของคุณ

### 5.2 ทดสอบด้วย cURL
คุณสามารถสั่งให้ระบบเรายิง Webhook ได้ผ่าน cURL command:

```bash
curl -X POST "https://api.esimrocket.com/api/partner/webhook" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "x-api-secret: YOUR_API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "event": "esim.activated",
    "testData": {
      "iccid": "89852240810732710975",
      "orderNumber": "ORD-TEST-0001",
      "esimStatus": "IN_USE"
    }
  }'
```
*คุณสามารถเปลี่ยน `event` และ `testData` ได้ตามโครงสร้างในข้อ 4*

---

## 6. ข้อควรระวังและ Best Practices ในการ Integrate

เพื่อป้องกันบั๊กในระดับ Production กรุณาปฏิบัติตามคำแนะนำต่อไปนี้:

1. **Idempotency (รับมือการยิงซ้ำ)**
   ระบบของเรามีกลไกรับประกันการส่ง (Delivery Guarantee) โดยจะทำ Retry สูงสุด 3 ครั้ง (ด้วย Exponential Backoff: 1s, 2s, 5s) หากระบบของคุณตอบกลับช้า หรือ Error การประมวลผลฝั่งคุณต้องรองรับการทำงานซ้ำ **แนะนำให้ใช้ Key ป้องกันการซ้ำซ้อนโดยผสม: `event` + `timestamp` + `iccid` (หรือ `orderNumber`)**
2. **การตอบกลับอย่างรวดเร็ว (Acknowledge First)**
   เมื่อได้รับ Webhook ควรตรวจสอบ Signature แล้วตอบกลับ `HTTP 200 OK` ภายในไม่เกิน 10 วินาที ก่อนจะนำข้อมูลไปประมวลผลต่อ (เช่น เซฟลงฐานข้อมูล หรือเรียก External Services) เพื่อป้องกัน Timeout
3. **แหล่งข้อมูลอ้างอิงที่ถูกต้อง (Single Source of Truth)**
   เราทำการ Normalize ความซับซ้อนของโครงข่าย (Upstream) มาให้หมดแล้ว โปรดใช้ `event` ควบคู่กับ `data.esimStatus` เป็นตัวขับเคลื่อน UI และ State Machine ของคุณ **(ไม่แนะนำให้พยายามแกะ Logic จากฟิลด์ `smdpStatus` เอง)**
4. **หน่วยของ Data Usage คือ ไบต์ (Bytes) เสมอ**
   โปรดนำค่า `totalVolume`, `orderUsage`, และ `remain` ไปหาร 1024 ตามระดับ (KB, MB, GB) เพื่อแสดงผลให้ลูกค้าดู
5. **การตั้งค่า Webhook Secret**
   อย่าลืมตั้งค่า Webhook Secret ใน Partner Portal และเขียนโค้ดตรวจสอบ `X-Webhook-Signature` ในทุกรีเควส เพื่อป้องกันการยิงข้อมูลปลอมเข้าสู่ระบบของคุณ
