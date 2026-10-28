AI agent: AI assistant using Copilot SDK in VS Code

# ADR-0002: Use an asynchronous post-order workflow with RabbitMQ

- **Status:** Proposed
- **Date:** 2026-10-28
- **Deciders:** Course teaching team

## Context and problem

ตามข้อกำหนด R-12 หลังจาก Online Boutique รับคำสั่งซื้อแล้ว จะต้อง (1) ส่งอีเมลยืนยันคำสั่งซื้อผ่าน MailFast ซึ่งเป็นผู้ให้บริการอีเมลใหม่แทน mock `emailservice` เดิม, (2) มอบ loyalty points ผ่าน `loyaltyservice` ที่ต้องสร้างขึ้นใหม่, และ (3) บันทึกเหตุการณ์ analytics ชื่อ "order placed" ให้ทีมการตลาด จากเอกสารสถานการณ์ [docs/scenario.md](../../docs/scenario.md) และการเรียก `checkoutservice.PlaceOrder` ใน [system/online-boutique/src/checkoutservice/main.go](../../system/online-boutique/src/checkoutservice/main.go) เราทราบว่า `PlaceOrder` ทำงานแบบเรียงลำดับใน critical path: เรียก `paymentservice` → `shippingservice` → `emailservice` ผ่าน gRPC แบบ synchronous และหาก email call ล้ม จะมีเพียง log warning แล้วให้คำสั่งซื้อยังดำเนินต่อไป

ความท้าทายใหม่คือ ตัวอย่างงานหลังคำสั่งซื้อที่ต้องเพิ่มเข้ามา 3 อย่างนี้มีลักษณะต่างกัน: MailFast มี p95 1.5 s, p99 8 s, อัตราความล้มประมาณ 0.5%, และมี rate limit 50 emails/s; loyalty points ต้องได้รับภายใน 1 นาที และต้องให้ได้อย่างเดียว (exactly once); analytics event ต้องถูกบันทึกให้ marketing team โดยไม่ให้คำสั่งซื้อล้มลงเมื่อ service เหล่านี้ขัดข้อง

นอกจากนี้ ด้วยข้อจำกัดที่ระบุไว้ในสถานการณ์: p95 ของ `PlaceOrder` ต้องไม่เกิน 500 ms (ปัจจุบันประมาณ 300 ms), peak load 200 orders/s ในช่วง 11.11, และการวางคำสั่งซื้อต้องไม่ล้มเพราะ email, loyalty หรือ analytics ขัดข้อง ทีมมีนักพัฒนาจำนวน 4 คน โดย 2 คนเคยใช้งาน RabbitMQ มาก่อน และระบบทำงานบน Kubernetes โดยชอบใช้ open-source/self-hosted มากกว่า cloud-only managed service

## Decision drivers

1. **Performance:** ต้องรักษา p95 ของ `PlaceOrder` ไม่เกิน 500 ms แม้จะมีงานหลังคำสั่งซื้อเพิ่มขึ้น
2. **Availability:** คำสั่งซื้อต้องไม่ล้มเพราะ MailFast, `loyaltyservice` หรือ analytics service ขัดข้อง
3. **Correctness:** loyalty points ต้องมอบภายใน 1 นาทีและไม่ให้มอบซ้ำ
4. **Modifiability:** คาดว่าจะมีงานเพิ่มเติมหลังคำสั่งซื้อในปีหน้า เช่น SMS, fraud scoring, recommendations update อย่างน้อยต้องไม่ต้องแก้ไข checkout หลักแบบซ้ำซ้อน
5. **Operational fit:** ใช้เทคโนโลยีที่เข้ากับ Kubernetes, open-source/self-hosted, และสอดคล้องกับความ熟ของทีม

## Considered options

1. เรียก MailFast, `loyaltyservice` และ analytics service แบบ synchronous จาก `checkoutservice` ในลำดับเดียวกับรูปแบบที่มีอยู่
2. ใช้ RabbitMQ เป็น message bus และให้ `checkoutservice` publish event `order.placed` หลังจากยืนยันคำสั่งซื้อแล้ว โดย consumer เป็นงานแยกต่อกัน
3. ใช้ workflow engine ที่มี state tracking เพื่อจัดการ retry และ compensating actions ของแต่ละ post-order step

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous calls in checkout | ง่ายสุด; ใช้ pattern ที่ทีมคุ้นเคยกับ Go/gRPC; ไม่ต้องเพิ่ม broker หรือ queue | `PlaceOrder` จะต้องรอ MailFast และบริการอื่น ๆ ทำงานเสร็จ; p95 1.5 s และ p99 8 s ของ MailFast อาจทำให้ latency เกิน 500 ms; หาก loyalty หรือ analytics ล้ม จะทำให้ระบบรอหรือมีผลต่อ order success; ไม่สะดวกเมื่อมีงานเพิ่มขึ้นในอนาคต |
| 2. RabbitMQ event-driven workflow | แยก critical path ของ checkout จากความล่าช้าและความล้มเหลวของ downstream services; รองรับ retry, rate limiting และ consumer scaling แยกกัน; เหมาะกับ Kubernetes และ self-hosted; ทีมมีประสบการณ์ใช้ RabbitMQ อยู่ 2 คน | ต้องจัดการ broker, durable queues, retry policy, dead-letter queue และ monitoring เพิ่ม; ต้องทำ idempotency สำหรับ consumer เพื่อไม่ให้ loyalty points หรือ analytics event ซ้ำ |
| 3. Workflow engine | ให้ state tracking และ retry/compensation ใน one place และอาจทำให้การ monitor ดูสะดวก | เพิ่มแพลตฟอร์มและความซับซ้อนที่ทีมต้องดูแล; ไม่พบ requirement ว่า workflow ต้องมีเงื่อนไขหรือ compensating logic ซับซ้อนมาก; หัวข้อ cost/operational overhead ยังไม่มีข้อมูลชัดเจน |

## Decision

เลือกใช้แบบ asynchronous event-driven ผ่าน RabbitMQ โดยให้ `checkoutservice` ส่งเหตุการณ์ `order.placed` ไปยังคิวที่มีความทนทานหลังจากคำสั่งซื้อได้รับการยืนยันและก่อนตอบกลับ customer โดยไม่รอ downstream services เสร็จสิ้น

ระบบจะมี worker แยกกัน 3 ตัว:

- `emailworker`: ส่งอีเมลผ่าน MailFast ด้วย retry/backoff และใช้ rate limiter เพื่อหลีกเลี่ยงการพุ่งเกิน 50 emails/s
- `loyaltyworker`: เรียก `loyaltyservice` เพื่อมอบ points โดยใช้ `order_id` เป็น idempotency key เพื่อให้แน่ใจว่าคำสั่งซื้อเดียวกันจะได้รับ points เพียงครั้งเดียว
- `analyticsworker`: ส่ง "order placed" event ให้ทีมการตลาด

รูปแบบนี้ทำให้ critical path ของ `PlaceOrder` สั้นลงและลดความเสี่ยงที่คำสั่งซื้อจะล้มเพราะ service ภายนอกทำงานไม่เสร็จ ทั้งยังเปิดทางให้เพิ่มงานใหม่ในอนาคต เช่น SMS หรือ recommendations update ได้โดยการเพิ่ม consumer ใหม่ ไม่ต้องแก้ checkout หลักซ้ำอีก

## Consequences

- **Positive:** `PlaceOrder` สามารถรักษา latency ในกรอบ p95 ≤ 500 ms เพราะ task ที่มี latency สูงและมีความล้มได้ถูกย้ายออกจาก critical path (quality attribute: performance)
- **Positive:** หาก MailFast, `loyaltyservice` หรือ analytics service ล้ม คำสั่งซื้อยังสามารถเสร็จสิ้นได้ (quality attribute: availability)
- **Positive:** การเพิ่ม consumer ใหม่สำหรับ future workflow จะไม่ทำให้ `checkoutservice` ผูกติดกับ logic ของ MailFast/loyalty/analytics มากขึ้น (quality attribute: modifiability)
- **Positive:** RabbitMQ และคิวที่ใช้ self-hosted บน Kubernetes สอดคล้องกับข้อจำกัดด้าน open-source และประสบการณ์ของทีม (quality attribute: operability, cost)
- **Negative:** การประมวลผลแบบ asynchronous ทำให้การส่งอีเมลหรือมอบ points ไม่ได้เสร็จก่อนที่ checkout ตอบกลับลูกค้า (quality attribute: consistency, user experience)
- **Negative:** ต้องมีการจัดการ operational overhead เพิ่มเติม เช่น durable queue, retry, dead-letter queue และ monitoring (quality attribute: operability)
- **Risks and mitigations:** ความเสี่ยงจากข้อความซ้ำหรือ duplicate delivery จะลดลงด้วย idempotency key, durable messages และ consumer deduplication; ความเสี่ยงจาก backpressure ในช่วง peak load จะลดลงด้วย consumer scaling และ rate limiting สำหรับ MailFast; ความเสี่ยงจาก message loss จะลดลงด้วย publish confirm, durable queue และ outbox pattern ที่ต้องบันทึกงานก่อน publish

## Validation

1. ทำ load test ที่ 200 orders/s และยืนยันว่า p95 ของ `PlaceOrder` ยังไม่เกิน 500 ms เมื่อมี downstream workers ทำงานอยู่
2. จำลอง MailFast ที่ช้าและล้ม (p95 1.5 s, p99 8 s, fail rate 0.5%) เพื่อยืนยันว่า checkout ยังสำเร็จและ message ถูก retry อย่างต่อเนื่อง
3. ทดสอบ `loyaltyworker` เมื่อมี duplicate delivery หรือ retry ซ้ำหลายครั้ง เพื่อยืนยันว่า points ถูกมอบได้เพียงครั้งเดียวและภายใน 1 นาที
4. ปิด `loyaltyservice` หรือ analytics service เป็นเวลาเพื่อยืนยันว่าคำสั่งซื้อยังไม่ล้ม และ queue backlog จะลดลงเมื่อ service กลับมา
5. ตรวจสอบ metric ที่สำคัญ เช่น queue depth, consumer lag, retry count, dead-letter count, email send latency และเวลาในการมอบ points เพื่อให้ทีมติดตามประสิทธิภาพจริงได้

## Open questions

- ควรใช้รูปแบบ outbox แบบใดเพื่อให้การ publish `order.placed` และการสร้าง order เกิดเป็น transactionเดียวกัน และยังไม่ทำให้ธุรกรรมสูญหาย
- `loyaltyservice` และ analytics event ต้องใช้ idempotency key แบบใด: `order_id`, `user_id + order_id`, หรือ event UUID
- ควรกำหนด retry policy และ dead-letter handling อย่างไร สำหรับ MailFast, loyalty และ analytics เพื่อให้ไม่เกิด queue overflow หรือการทำซ้ำไม่จำกัด
- ควรมี topic/queue แยกสำหรับแต่ละ downstream หรือใช้ topic เดียวสำหรับทุกงาน แล้วค่อย dispatch ไปยัง consumer ต่าง ๆ
- หากมีงานหลายอย่างเพิ่มขึ้นในปีหน้า เช่น SMS และ fraud scoring จะใช้ consumer แยกกันตามประเภทหรือรวมใน workflow group เดียว
