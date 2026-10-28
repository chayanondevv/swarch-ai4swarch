**ภาษา:** ภาษาไทย | [English](02-write-adr.en.md)

# งานที่ 2: เขียน ADR สำหรับความต้องการใหม่ (ประมาณ 20 นาที)

> **เชื่อมโยงกับการบรรยาย:** T3 (GPT-4 แบบ zero-shot ให้ผลที่สอดคล้องกัน แต่ *ครบถ้วนน้อยกว่ามนุษย์* ส่วน few-shot ช่วยลดช่องว่างได้มาก), AICH3 (บริบท), สัปดาห์ที่ 10 (message broker)
> **คำถาม:** ADR ที่ AI เขียนดีขึ้นแค่ไหน เมื่อให้ตัวอย่างและบริบทจริงแก่ AI

อ่าน [สถานการณ์ (ภาษาไทย)](../docs/scenario.th.md) ก่อน: ความต้องการ **R-12** ว่าด้วยสิ่งที่เกิดขึ้นหลังจากลูกค้าสั่งซื้อสินค้า

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่าง โดยแทนที่ `<paste R-12 here>` ด้วยข้อความ R-12 ภาษาอังกฤษที่อยู่ในกรอบอ้างอิงของ scenario

**Run a: ไม่มีตัวอย่าง (zero-shot)** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
language: Thai
```

**Run b: มีตัวอย่าง + บริบท** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
language: Thai
```

> บรรทัด `language: Thai` ทำให้ agent เขียน ADR เป็นภาษาไทย (หัวข้อของ template และชื่อทางเทคนิคยังคงเป็นภาษาอังกฤษ) หากต้องการภาษาอังกฤษให้ลบบรรทัดนี้ **ใช้ภาษาเดียวกันทั้ง Run a และ Run b** เพื่อให้เปรียบเทียบกันได้

> ใน Run a ให้ตรวจว่า agent ไม่ได้อ่านไฟล์อื่นนอกจาก template จริง ๆ หาก agent เปิด `docs/scenario.md` ให้บันทึกไว้ใน prompt log เพราะจะทำให้การเปรียบเทียบเปลี่ยนไป

## เปรียบเทียบ a กับ b

| เกณฑ์ | Run a | Run b |
|---|---|---|
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast) | ไม่ใช้เลย | ใช้ครบถ้วน |
| ทางเลือกที่พิจารณา (ระบุ) | **3 ทางเลือก (เชิงแนวคิดกว้างๆ):**<br>1. ให้ส่วนที่ยืนยันคำสั่งซื้อเรียก MailFast, `loyaltyservice` และ analytics โดยตรง<br>2. เผยแพร่เหตุการณ์ `OrderPlaced` แล้วให้ผู้รับแต่ละรายดำเนินงานของตนเอง<br>3. ใช้ post-order orchestrator หนึ่งตัวเรียกบริการแต่ละรายและควบคุมลำดับการทำงาน | **3 ทางเลือก (เชิงเทคนิคและการประยุกต์ใช้):**<br>1. เรียก MailFast, `loyaltyservice` และ analytics service แบบ synchronous จาก `checkoutservice` ในลำดับเดียวกับรูปแบบที่มีอยู่<br>2. ใช้ RabbitMQ เป็น message bus และให้ `checkoutservice` publish event `order.placed` หลังจากยืนยันคำสั่งซื้อแล้ว โดย consumer เป็นงานแยกต่อกัน<br>3. ใช้ workflow engine ที่มี state tracking เพื่อจัดการ retry และ compensating actions ของแต่ละ post-order step |
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร | **จัดการเฉพาะ Idempotency แบบกว้างๆ / ไม่ได้จัดการ Rate limit:**<br>ระบุในหลักการว่าต้องใช้ event identifier และ idempotent processing เพื่อป้องกันการแจกแต้มซ้ำ แต่ไม่ได้กล่าวถึงกลไกกำกับ rate limit ของ MailFast | **จัดการทั้งสองประเด็นอย่างเป็นรูปธรรม:**<br>• **Loyalty points:** ใช้ `loyaltyworker` โดยกำหนดให้ใช้ `order_id` เป็น Idempotency Key เพื่อการันตีว่าคำสั่งซื้อเดียวกันได้รับแต้มเพียงครั้งเดียวแม้เกิดการ retry<br>• **MailFast Rate Limit:** ออกแบบ `emailworker` ให้ใช้งาน Rate Limiter ควบคุมการส่งไม่ให้เกิน 50 emails/s |
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) | **ไม่มี**<br>เนื้อหาอ้างอิงเฉพาะข้อความใน requirement R-12 และหลักการออกแบบทั่วไป ไม่มีการระบุชื่อเครื่องมือ ตัวเลข หรือบริบททีมเพิ่มเติม | **มี (เป็นการเติมบริบททางเทคนิคเพื่อความสมบูรณ์ของ ADR):**<br>• เลือกระบุเครื่องมือเฉพาะคือ **RabbitMQ**<br>• สมมุติบริบททีมงาน (ทีมมี 4 คน, 2 คนมีประสบการณ์ RabbitMQ)<br>• อ้างอิงไฟล์และโค้ดระบบเดิม (`checkoutservice.PlaceOrder` ใน `main.go`)<br>• กำหนดชื่อเฉพาะให้ worker แยกย่อย (`emailworker`, `loyaltyworker`, `analyticsworker`) |

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| # | สิ่งที่เปลี่ยนจากร่างของ AI | เหตุผล |
|---|---|---|
| 1 | ตัดสินใจในประเด็น Open Questions ให้เบ็ดเสร็จ | แปลงคำถามเปิดให้กลายเป็นข้อตัดสินใจทางเทคนิคที่เด็ดขาดและพร้อมนำไป Implement ช่วยป้องกันปัญหา Message Loss และ Double Processing |
| 2 | ปรับแก้ส่วนที่ AI สมมุติบริบทขึ้นเอง | ให้เอกสารมุ่งเน้นที่ข้อกำหนดทางเทคนิคและข้อจำกัดของระบบ (System Constraints) ที่แท้จริง |
| 3 | เพิ่มโครงสร้าง Rate Limiter | ป้องกัน MailFast ปฏิเสธการเชื่อมต่อหรือส่ง HTTP 429 Too Many Requests กลับมาช่วง Peak Load 200 orders/s |
| 4 | เปลี่ยน Deciders เป็นชื่อตัวเอง | แสดง Ownership ในการตัดสินใจทางสถาปัตยกรรม และพร้อมรับรองเอกสารก่อนนำเสนอคณะกรรมการรีวิว
