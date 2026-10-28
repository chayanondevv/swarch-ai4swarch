Agent: AI assistant using Copilot SDK in VS Code; Model: ไม่ทราบ

# ADR-0002: แยกงานหลังสั่งซื้อด้วยเหตุการณ์ OrderPlaced

- **Status:** Proposed
- **Date:** 2026-10-28
- **Deciders:** ยังไม่ระบุ

## Context and problem

ข้อความ requirement R-12 กำหนดให้หลังวางคำสั่งซื้อ ระบบต้องส่งอีเมลยืนยันผ่าน MailFast ซึ่งเป็นผู้ให้บริการภายนอกและเข้ามาแทน logic จำลองของ `emailservice`, มอบ loyalty points ผ่าน `loyaltyservice` ที่ต้องสร้างใหม่ และบันทึก analytics event สำหรับทีมการตลาด นอกจากนี้มีแนวโน้มว่าจะเพิ่มงานหลังสั่งซื้อ เช่น SMS, fraud scoring และการอัปเดต recommendations ในปีหน้า

ต้องเลือกวิธีประสานงานงานเหล่านี้โดยไม่ทำให้การเพิ่มงานในอนาคตต้องแก้ไขส่วนการวางคำสั่งซื้อทุกครั้ง Requirement ไม่ได้ระบุเทคโนโลยี message broker, ข้อกำหนดเวลาในการทำงานให้เสร็จ, ระดับการรับประกันการส่ง หรือข้อกำหนดว่าการวางคำสั่งซื้อจะต้องรอผลจากงานเหล่านี้หรือไม่

## Decision drivers

- รองรับการเพิ่มผู้รับเหตุการณ์หลังสั่งซื้อได้ง่าย (modifiability)
- ลดการผูกการวางคำสั่งซื้อกับ MailFast และบริการปลายทางอื่น (availability, modifiability)
- ป้องกันงานที่ล้มเหลวหรือการลองใหม่ทำให้เกิดผลซ้ำโดยไม่ตั้งใจ (reliability)
- จำกัดความซับซ้อนและต้นทุนของโครงสร้างพื้นฐาน (cost, operability)

## Considered options

1. ให้ส่วนที่ยืนยันคำสั่งซื้อเรียก MailFast, `loyaltyservice` และ analytics โดยตรง
2. เผยแพร่เหตุการณ์ `OrderPlaced` แล้วให้ผู้รับแต่ละรายดำเนินงานของตนเอง
3. ใช้ post-order orchestrator หนึ่งตัวเรียกบริการแต่ละรายและควบคุมลำดับการทำงาน

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. เรียกโดยตรง | โครงสร้างเข้าใจง่ายและไม่ต้องเพิ่ม event infrastructure (cost, operability); เห็นผลการเรียกแต่ละปลายทางในเส้นทางเดียว (observability) | การเพิ่ม SMS หรือ fraud scoring ทำให้ต้องแก้ส่วนยืนยันคำสั่งซื้อ (modifiability); ความล้มเหลวหรือความช้าของผู้ให้บริการอาจกระทบเวลาตอบสนองและ availability ของการสั่งซื้อ |
| 2. เหตุการณ์ `OrderPlaced` | เพิ่ม consumer ใหม่ได้โดยไม่ต้องเพิ่มการเรียกปลายทางใน publisher (modifiability); แยกความล้มเหลวและการปรับขนาดของแต่ละงานออกจากกันได้ (availability, scalability) | ต้องดูแลระบบส่งเหตุการณ์และการติดตามงาน (cost, operability); การประมวลผลอาจล่าช้าหรือเกิดซ้ำ จึงต้องกำหนดการลองใหม่และ idempotency (reliability) |
| 3. Orchestrator | มีจุดเดียวควบคุมลำดับ การลองใหม่ และการติดตามงาน (operability, reliability); เหมาะเมื่อมี workflow หรือเงื่อนไขระหว่างขั้นตอน (modifiability) | orchestrator กลายเป็นส่วนกลางที่ต้องแก้เมื่อเพิ่มงาน และอาจเป็นจุดพึ่งพาร่วม (modifiability, availability); เพิ่ม deployment/runtime component และต้นทุนการดูแล (cost) |

## Decision

เสนอให้เผยแพร่เหตุการณ์ `OrderPlaced` หลังจากคำสั่งซื้อได้รับการยืนยัน แล้วให้ผู้รับแยกกันดำเนินงานที่กำหนด:

- ผู้รับด้านอีเมลเชื่อมต่อ MailFast แทน logic จำลองของ `emailservice`
- `loyaltyservice` รับเหตุการณ์เพื่อมอบ loyalty points
- ผู้รับด้าน analytics บันทึกเหตุการณ์สำหรับทีมการตลาด

แนวทางนี้แยกงานปลายทางออกจาก publisher และรองรับการเพิ่ม SMS, fraud scoring หรือการอัปเดต recommendations ด้วย consumer เพิ่มเติม โดยไม่ต้องเพิ่มการเรียกเหล่านั้นในเส้นทางยืนยันคำสั่งซื้อ (modifiability, availability) ควรออกแบบ consumer ให้รองรับการประมวลผลซ้ำอย่างปลอดภัยและกำหนดวิธีติดตาม/ลองส่งเหตุการณ์ที่ล้มเหลว (reliability, operability) รายละเอียดกลไกส่งเหตุการณ์และการรับประกันการบันทึกเหตุการณ์พร้อมคำสั่งซื้อยังต้องเลือกหลังยืนยันข้อจำกัดของระบบ

## Consequences

- **Positive:** การเพิ่มงานหลังสั่งซื้อทำได้โดยเพิ่ม consumer แทนการขยายการเชื่อมต่อโดยตรงใน publisher (modifiability).
- **Positive:** ความล้มเหลวของ MailFast หรือผู้รับรายหนึ่งไม่จำเป็นต้องขัดขวางงานของผู้รับรายอื่น หากกลไกส่งเหตุการณ์และการ retry รองรับการแยกผู้รับ (availability, reliability).
- **Negative:** ผลของอีเมล, loyalty points และ analytics อาจไม่เกิดขึ้นพร้อมกับการยืนยันคำสั่งซื้อ (consistency, performance); ระยะเวลาที่ยอมรับได้ยังไม่ระบุ.
- **Negative:** ต้องมีการดูแลระบบ event delivery, retry, dead-letter handling และ monitoring (cost, operability).
- **Risks and mitigations:** เหตุการณ์อาจสูญหายระหว่างการยืนยันคำสั่งซื้อกับการเผยแพร่; ประเมิน transactional outbox หรือกลไกเทียบเท่าเมื่อเลือกเทคโนโลยี (reliability). การส่งซ้ำอาจให้ loyalty points ซ้ำหรือสร้างผลซ้ำที่ปลายทาง; ใช้ event identifier และ idempotent processing โดยยืนยันความสามารถกับแต่ละปลายทางก่อนเปิดใช้ (correctness).

## Validation

- ทดสอบตั้งแต่ยืนยันคำสั่งซื้อจนถึงผู้รับทั้งสาม โดยตรวจว่ามีการส่งอีเมลผ่าน MailFast, มอบ loyalty points และบันทึก analytics event ตามที่กำหนด
- ทดสอบการ retry และการรับเหตุการณ์ซ้ำ โดยยืนยันว่า loyalty points และผลข้างเคียงอื่นไม่เกิดซ้ำโดยไม่ตั้งใจ
- จำลอง MailFast หรือ consumer รายหนึ่งล้มเหลว แล้วตรวจว่าผู้รับรายอื่นยังดำเนินงานได้และเหตุการณ์ที่ล้มเหลวติดตามต่อได้
- วัดความล่าช้าการส่งถึงผู้รับและอัตราความล้มเหลว จากนั้นตกลงเกณฑ์ยอมรับร่วมกับเจ้าของ requirement เนื่องจาก requirement ไม่ได้ระบุตัวเลขไว้

## Open questions

- ใช้ event broker หรือกลไกส่งเหตุการณ์ใด และมีโครงสร้างพื้นฐานที่ต้องนำกลับมาใช้หรือไม่?
- การวางคำสั่งซื้อจะสำเร็จได้หรือไม่เมื่อ MailFast, `loyaltyservice` หรือ analytics ยังทำงานไม่สำเร็จ?
- ต้องรับประกันการส่งแบบใด และยอมรับความล่าช้าสูงสุดเท่าใดสำหรับอีเมล, loyalty points และ analytics?
- ใครเป็นเจ้าของ event schema, retention, การลองส่งซ้ำ และ dead-letter handling?
- MailFast มี API, authentication, rate limits และเงื่อนไขการจัดการข้อมูลใดบ้าง?
- loyalty points ต้องจัดการคำสั่งซื้อที่ถูกยกเลิก คืนเงิน หรือประมวลผลซ้ำอย่างไร?
- analytics event ต้องมีข้อมูลใด และมีข้อกำหนดด้าน privacy หรือ retention หรือไม่?
- ทีมใดเป็นผู้ตัดสินใจและดูแลการเปลี่ยนแปลงนี้?
