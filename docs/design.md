# Expense Tracker: Design Doc

โปรเจคฝึก Next.js แบบ fullstack สำหรับใช้เป็น portfolio สมัครงาน

## 1. เป้าหมาย

แอปบันทึกรายรับ-รายจ่ายส่วนตัว ผู้ใช้แต่ละคนเห็นเฉพาะข้อมูลของตัวเอง

## 2. ขอบเขตฟีเจอร์

### MVP (เวอร์ชันแรกที่ใช้งานได้จริง)
- สมัครสมาชิก / ล็อกอิน / ล็อกเอาท์
- เพิ่ม แก้ไข ลบ รายการรายรับ-รายจ่าย
- แต่ละรายการมี: จำนวนเงิน, ประเภท (รายรับ/รายจ่าย), หมวดหมู่, วันที่, โน้ต
- ดูรายการทั้งหมดของตัวเอง
- สรุปยอดรวมรายรับ / รายจ่าย / คงเหลือ ของเดือนนี้

### Phase 2 (ทำทีหลัง)
- กราฟสรุปรายเดือน, สรุปตามหมวดหมู่
- filter / search / pagination
- งบประมาณรายเดือนต่อหมวด
- export CSV
- แนบรูปสลิป

## 3. Tech Stack

| ส่วน | เลือกใช้ |
|---|---|
| Framework | Next.js (App Router) + TypeScript |
| Styling | Tailwind CSS + shadcn/ui |
| Database | PostgreSQL (Neon) |
| ORM | Prisma |
| Auth | Auth.js หรือ Better Auth |
| Validation | Zod |
| Deploy | Vercel |

## 4. Database Design

### ความสัมพันธ์

```
User (1) ────< (หลาย) Category
User (1) ────< (หลาย) Transaction
Category (1) ──< (หลาย) Transaction
```

### User
| ฟิลด์ | ชนิด | หมายเหตุ |
|---|---|---|
| id | string | primary key |
| email | string | unique |
| name | string | |
| createdAt | datetime | |

(ฟิลด์ที่เกี่ยวกับ auth เช่น password hash, session จะเพิ่มในด่าน 4)

### Category
| ฟิลด์ | ชนิด | หมายเหตุ |
|---|---|---|
| id | string | primary key |
| name | string | เช่น อาหาร, เดินทาง, เงินเดือน |
| userId | string | foreign key → User |
| monthlyBudget | decimal? | Phase 2: งบรายเดือน (เว้นว่าง = ไม่ตั้งงบ) |

### Transaction
| ฟิลด์ | ชนิด | หมายเหตุ |
|---|---|---|
| id | string | primary key |
| amount | decimal | จำนวนเงิน |
| type | enum | `INCOME` หรือ `EXPENSE` |
| date | datetime | วันที่เกิดรายการ |
| note | string? | ไม่บังคับ |
| userId | string | foreign key → User |
| categoryId | string | foreign key → Category |
| createdAt | datetime | เวลาที่สร้างระเบียน |

## 5. การตัดสินใจออกแบบและเหตุผล

1. **`amount` ใช้ decimal ไม่ใช้ float**: float มีปัญหาความแม่นยำ (`0.1 + 0.2 = 0.30000000000000004`) ข้อมูลการเงินต้องตรงเป๊ะ
2. **ใส่ `userId` ทั้งใน Category และ Transaction**: ทุก query ต้องกรองด้วย `where: { userId }` เพื่อไม่ให้ผู้ใช้เห็นข้อมูลของคนอื่น
3. **แยก Category เป็นตารางเอง**: หลีกเลี่ยงข้อมูลสะกดไม่ตรงกัน และแก้ชื่อหมวดที่เดียวได้ (normalization)
4. **`type` เป็น enum**: จำกัดค่าให้เป็น INCOME / EXPENSE ป้องกันพิมพ์ผิด
5. **นโยบายการลบข้อมูล (onDelete)**:
   - `User → Transaction/Category`: **Cascade** (ลบผู้ใช้ ข้อมูลของเขาลบตาม)
   - `Category → Transaction`: **Restrict** (ห้ามลบหมวดที่ยังมีรายการ ให้ UI ถามว่าจะย้ายรายการไปหมวดไหนก่อน เพื่อกันข้อมูลการเงินหายโดยไม่ตั้งใจ)
6. **งบประมาณ**: เริ่มจากฟิลด์ `monthlyBudget` ใน Category ถ้าต้องการงบต่างกันแต่ละเดือน ค่อยแยกเป็นตาราง `Budget (categoryId, month, amount)`

## 6. Routes

| URL | หน้า | ใครเข้าได้ |
|---|---|---|
| `/` | หน้าแรก | ทุกคน |
| `/login`, `/register` | เข้าสู่ระบบ / สมัคร | ทุกคน |
| `/dashboard` | สรุปยอดเดือนนี้ + รายการล่าสุด | ล็อกอินแล้ว |
| `/transactions` | รายการทั้งหมด + เพิ่ม/แก้/ลบ | ล็อกอินแล้ว |
| `/categories` | จัดการหมวดหมู่ | ล็อกอินแล้ว |

หน้าที่ต้องล็อกอินจะป้องกันด้วย middleware

## 7. Roadmap

1. Setup + Deploy ตั้งแต่วันแรก
2. ต่อ Database + แสดงข้อมูล
3. CRUD ด้วย Server Actions + Zod
4. Auth
5. ฟีเจอร์เสริม
6. ขัดเกลาเพื่อ Portfolio (responsive, README, test)
