# ตั้งค่าสำรองข้อมูลขึ้น Google Drive

ทำครั้งเดียว ใช้เวลาประมาณ 10 นาที

---

## ทำไมไม่ใช้ Service Account

วิธีที่คนมักลองก่อนคือสร้าง service account แล้วแชร์โฟลเดอร์ใน My Drive ให้ **วิธีนี้ใช้ไม่ได้แล้ว** จะเจอ error:

```
Service Accounts do not have storage quota.
Leverage shared drives, or use OAuth delegation instead.
```

Shared Drive ต้องมี Google Workspace แบบเสียเงิน ถ้าใช้ Gmail ธรรมดาจึงต้องใช้ **OAuth refresh token** ซึ่งจะอัปโหลดเข้า Drive ของคุณเองและใช้ quota ของคุณเอง

---

## ขั้นตอน

### 1. สร้างโปรเจกต์และเปิด Drive API

1. เข้า https://console.cloud.google.com/
2. สร้างโปรเจกต์ใหม่ (ตั้งชื่ออะไรก็ได้ เช่น `trip-splitter`)
3. ไปที่ **APIs & Services → Library** ค้นหา `Google Drive API` แล้วกด **Enable**

### 2. ตั้งค่า OAuth consent screen

1. ไปที่ **APIs & Services → OAuth consent screen**
2. User Type เลือก **External** → Create
3. กรอกชื่อแอปและอีเมลของคุณ ที่เหลือกด Save and Continue ผ่านไปได้
4. หน้า Scopes ไม่ต้องเพิ่มอะไร กด Save and Continue

> ### ⚠️ ขั้นตอนที่ห้ามลืม
>
> กลับมาหน้า OAuth consent screen แล้วกด **PUBLISH APP** ให้สถานะเป็น **In production**
>
> ถ้าปล่อยเป็น **Testing** → refresh token จะ**หมดอายุทุก 7 วัน** และการสำรองข้อมูลจะพังเงียบ ๆ
>
> แอปที่ใช้ส่วนตัว (ผู้ใช้ไม่เกิน 100 คน) **ไม่ต้อง**ผ่าน verification ของ Google กด Publish ได้เลย ผู้ใช้แค่คลิกข้ามหน้าจอเตือน "unverified app"

### 3. สร้าง OAuth Client ID

1. ไปที่ **APIs & Services → Credentials → Create Credentials → OAuth client ID**
2. Application type เลือก **Desktop app**
3. กด Create แล้วจะได้ **Client ID** กับ **Client secret** — เก็บไว้

### 4. ขอ refresh token

กด **Download JSON** ที่ client ที่เพิ่งสร้าง จะได้ไฟล์ `client_secret_xxxxx.json`

จากนั้นรันบนเครื่องตัวเอง (ไม่ใช่บน Streamlit Cloud) โดยวางไฟล์ JSON ไว้โฟลเดอร์เดียวกัน:

```bash
pip install requests
python get_gdrive_token.py client_secret_xxxxx.json
```

ถ้าไม่ใส่ชื่อไฟล์ สคริปต์จะหาไฟล์ `client_secret*.json` ในโฟลเดอร์ให้เอง

browser จะเปิดขึ้นมาให้กดอนุญาต — เจอหน้าจอ **"Google hasn't verified this app"**
ให้กด **Advanced → Go to ... (unsafe)** แล้วกด **Continue**

เสร็จแล้วสคริปต์จะพิมพ์ข้อความที่พร้อม copy ไปใช้ต่อ

> ไฟล์ต้องเป็นชนิด **Desktop app** (ข้างในขึ้นต้นด้วย `{"installed": ...}`)
> ถ้าเป็น `{"web": ...}` สคริปต์จะปฏิเสธ เพราะ flow นี้ต้องใช้ loopback redirect
> ซึ่ง client ชนิด web ต้องลงทะเบียน redirect URI ไว้ก่อน

### 5. (ถ้าต้องการ) กำหนดโฟลเดอร์ปลายทาง

สร้างโฟลเดอร์ใน Google Drive แล้วดู URL:

```
https://drive.google.com/drive/folders/1AbCdEfGhIjKlMnOp
                                        └── นี่คือ folder_id
```

### 6. ใส่ค่าใน Streamlit Secrets

**Streamlit Cloud** → Manage app → **Settings → Secrets** วางแบบนี้:

```toml
[gdrive]
client_id     = "xxxxx.apps.googleusercontent.com"
client_secret = "GOCSPX-xxxxx"
refresh_token = "1//xxxxx"
folder_id     = "1AbCdEfGhIjKlMnOp"   # เว้นว่างได้ = ลงที่ root
```

**รันในเครื่อง** ให้สร้างไฟล์ `.streamlit/secrets.toml` เนื้อหาเดียวกัน
และ**อย่า commit ไฟล์นี้ขึ้น git** — เพิ่ม `.streamlit/secrets.toml` ใน `.gitignore`

---

## การใช้งาน

ไปที่ **จัดการ → 💾 สำรองข้อมูล** (ต้องใส่ PIN ก่อน)

| ปุ่ม | ทำอะไร |
|---|---|
| ☁️ อัปโหลดขึ้น Drive ตอนนี้ | สร้างไฟล์ `trip_backup_YYYYMMDD_HHMMSS.json` บน Drive |
| 🔄 โหลดรายการไฟล์บน Drive | ดูไฟล์สำรองที่มีอยู่ ใหม่→เก่า |
| ♻️ กู้คืนจาก Drive | เลือกไฟล์แล้วเขียนทับข้อมูลปัจจุบัน |

---

## เรื่องความปลอดภัยที่ต้องรู้

ไฟล์สำรองมี **เลขบัญชีธนาคาร พร้อมเพย์ ข้อความแชทส่วนตัว และ pin_hash ของทุกคน** อยู่ในนั้น

- PIN 4 หลักมีความเป็นไปได้แค่ 10,000 แบบ ต่อให้แฮชด้วย PBKDF2 ก็ยัง brute-force ได้ ให้ถือว่าไฟล์นี้ = ข้อมูลอ่อนไหว
- แท็บสำรองข้อมูลถูกล็อกด้วย PIN แล้ว แต่ **สมาชิกทุกคนที่รู้ PIN ตัวเองก็เข้าได้** ถ้าต้องการให้เฉพาะเจ้าของระบบเข้าได้ ต้องเพิ่มแนวคิด "เจ้าของ" เข้าไปอีกชั้น
- โฟลเดอร์บน Drive **อย่าตั้งเป็น "ใครมีลิงก์ก็เข้าได้"**
- scope ที่ขอคือ `drive.file` ซึ่งเข้าถึงได้เฉพาะไฟล์ที่แอปนี้สร้างเอง มองไม่เห็นไฟล์อื่นใน Drive ของคุณ

---

## แก้ปัญหา

| อาการ | สาเหตุ / วิธีแก้ |
|---|---|
| `invalid_grant` หลังใช้ได้ 7 วัน | OAuth consent screen ยังเป็น Testing → กด Publish to Production แล้วขอ token ใหม่ |
| สคริปต์ไม่คืน `refresh_token` | เคยกดอนุญาตไปแล้ว ให้ถอนสิทธิ์ที่ https://myaccount.google.com/permissions แล้วรันใหม่ |
| `Service Accounts do not have storage quota` | ยังใช้ service account อยู่ ต้องเปลี่ยนมาใช้ OAuth ตามคู่มือนี้ |
| `403 insufficientPermissions` | scope ไม่ถูก ต้องเป็น `drive.file` และขอ token ใหม่ |
| `404` ตอนอัปโหลด | `folder_id` ผิด หรือโฟลเดอร์ถูกลบ |
| Token ตายเพราะไม่ได้ใช้ | refresh token ที่ไม่ถูกใช้ 6 เดือนจะหมดอายุ — สำรองข้อมูลเป็นระยะจะไม่เจอปัญหานี้ |
| `redirect_uri_mismatch` | ใช้ไฟล์ชนิด `web` อยู่ ต้องสร้าง client ใหม่เป็น **Desktop app** |
| สคริปต์ค้างที่ "รออนุญาต" | พอร์ต 8765 ถูกใช้อยู่ ปิดโปรแกรมที่ใช้พอร์ตนั้นแล้วรันใหม่ |
