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

| ผู้เรียก → ผู้ถูกเรียก | หลักฐานที่ AI อ้าง | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ |
|---|---|---|---|
| **Shopper → frontend** | `frontend.yaml` (`frontend-external` LoadBalancer Service, port `http:80`) | ✅ | ยืนยันแล้ว พบการสร้าง Service แบบ LoadBalancer เพื่อเปิดรับ HTTP Traffic จากผู้ใช้งานภายนอกพอร์ต 80 ส่งต่อเข้า Container พอร์ต 8080 |
| **frontend → productcatalogservice** | `frontend.yaml` (`PRODUCT_CATALOG_SERVICE_ADDR`); `demo.proto` (`ProductCatalogService`) | ✅ | ยืนยันแล้ว มีการกำหนด Environment Variable ชี้ไปยังปลายทาง `productcatalogservice:3550` เพื่อเรียกดูและค้นหารายการสินค้า |
| **frontend → currencyservice** | `frontend.yaml` (`CURRENCY_SERVICE_ADDR`) ; `demo.proto` (`CurrencyService`) | ✅ | ยืนยันแล้ว มีการกำหนด Environment Variable ชี้ไปยังปลายทาง `currencyservice:7000` สำหรับคำนวณและแปลงสกุลเงิน |
| **frontend → cartservice** | `frontend.yaml` (`CART_SERVICE_ADDR`) ; `demo.proto` (`CartService`)| ✅ | ยืนยันแล้ว มีการกำหนด Environment Variable ชี้ไปยังปลายทาง `cartservice:7070` สำหรับจัดการเพิ่ม/ดึง/ลบสินค้าในตะกร้า |
| **frontend → recommendationservice** | `frontend.yaml` (`RECOMMENDATION_SERVICE_ADDR`); `demo.proto` (`RecommendationService`) | ✅ | ยืนยันแล้ว มีการกำหนด Environment Variable ชี้ไปยังปลายทาง `recommendationservice:8080` เพื่อดึงรายการสินค้าแนะนำ |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย): …
* **`checkoutservice → OpenTelemetry Collector` (`COLLECTOR_SERVICE_ADDR`)**
  * **ประเด็น:** ในซอร์สโค้ดจริง (`main.go`) มีการเชื่อมต่อไปยัง OpenTelemetry Collector ผ่าน gRPC เมื่อเปิดโหมด Tracing (`ENABLE_TRACING=1`) แต่ AI **ไม่พบความสัมพันธ์นี้** จากการอ่านเฉพาะ Manifests และ `.proto`
* **`checkoutservice → GCP Profiler`**
  * **ประเด็น:** ซอร์สโค้ดมีการเรียกใช้งาน Google Cloud Profiler (`profiler.Start`) เมื่อเปิดโหมด Profiling (`ENABLE_PROFILER=1`) ซึ่งไม่ปรากฏในความสัมพันธ์ที่ AI สกัดออกมาได้
* **`cartservice → redis-cart` (ในระดับ Interface Contract)**
  * **ประเด็น:** แม้ AI จะพบความสัมพันธ์นี้จากไฟล์ Manifest (`cartservice.yaml`) แต่ถ้าพิจารณาในมุมของ gRPC / Proto Interface จะพบว่าไม่มี API Contract นิยามการเชื่อมต่อนี้ไว้ เนื่องจากเป็น Internal State Storage Persistence Mechanism ภายในบริการเอง

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด

จากการทดลองเปรียบเทียบระหว่าง **Run A** (อ่านเฉพาะ `demo.proto`) และ **Run B** (อ่าน `demo.proto` + `kubernetes-manifests/`) AI Agent ไม่สามารถระบุข้อมูลสำคัญต่อไปนี้ได้เมื่อมีเพียงไฟล์ `.proto`:

- **ไม่รู้ความสัมพันธ์ขณะทำงานของบริการ (Runtime Dependencies):** 
   * *สาเหตุ:* ไฟล์ `.proto` ทำหน้าที่เป็นเพียง **Data Contract (ข้อตกลงอินเทอร์เฟซ)** ที่นิยามว่าแต่ละบริการมี RPC Methods อะไร และส่งข้อมูลรูปแบบใด แต่**ไม่ได้ระบุ โค้ดที่ทำการเรียก (Caller/Client Code)** ว่าบริการใดเป็นฝ่ายส่ง Request ไปหาบริการใด ทำให้ใน Run A Agent ไม่สามารถลากเส้นเชื่อมต่อ (Edges) ระหว่างบริการได้เลย ต่างจาก Run B ที่สกัดความสัมพันธ์ได้จาก Environment Variables เช่น `PRODUCT_CATALOG_SERVICE_ADDR` ในไฟล์ Manifests
- **ไม่รู้การมีอยู่ของ Frontend และ Entry Point ของระบบ:**
   * *สาเหตุ:* บริการ `frontend` ให้บริการผ่านโปรโตคอล HTTP (Web UI) จึงไม่มีการนิยาม RPC Methods ไว้ในไฟล์ `.proto` ทำให้ Run A ไม่พบส่วนติดต่อผู้ใช้งานเลย ในขณะที่ Run B ค้นพบ `frontend` และพอร์ตที่เปิดรับ Traffic จากภายนอกจากไฟล์ `frontend.yaml`
- **ไม่รู้ฐานข้อมูลและโครงสร้างพื้นฐานด้านข้อมูล (Data Stores & Infrastructure Containers):**
   * *สาเหตุ:* การเชื่อมต่อไปยังฐานข้อมูลอย่าง Redis (`redis-cart`) เป็นกลไกภายในของบริการ (Internal State Storage) ไม่ใช่ gRPC Interface ที่เปิดให้บริการอื่นเรียกใช้ จึงไม่อยู่ในไฟล์ `.proto`
- **ไม่รู้หน่วยในการปรับใช้งานจริง (Deployment Units & Container Mapping):**
   * *สาเหตุ:* ไฟล์ `.proto` ประกาศเพียง Logical Services แต่ไม่สามารถยืนยันได้ว่า 1 Service จะถูกรันแยกเป็น 1 Deployment / Container หรือไม่ รวมถึงไม่ทราบการตั้งค่า Network Ports จริงในระดับ Container

2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ

ข้อผิดพลาดที่เสี่ยงต่อการสร้างความเข้าใจผิดให้สมาชิกใหม่ในทีมมากที่สุด คือ **การระบุให้ `loadgenerator → frontend` เป็น Runtime Dependency หลักของระบบ**

**เหตุผลที่ทำให้เข้าใจผิดร้ายแรง:**
* **ความจริงในระบบ:** ไฟล์ `loadgenerator.yaml` ถูก **Comment (ปิดการใช้งาน)** ไว้ในไฟล์ `kustomization.yaml` ซึ่งหมายความว่า ในสภาพแวดล้อมการปรับใช้งานจริง (Production/Default Deployment) ตัว `loadgenerator` จะ**ไม่ถูกสร้างขึ้นมาทำงาน**
* **ผลกระทบต่อสมาชิกใหม่:** หากปล่อยให้แผนภาพแสดงเส้นเชื่อมต่อนี้ไว้ สมาชิกใหม่จะเข้าใจผิดว่า `loadgenerator` เป็นองค์ประกอบสำคัญ (Core Component) ของระบบ Online Boutique ที่ต้องมีอยู่เสมอ หรืออาจเข้าใจว่าเป็นระบบ Background Job ภายใน ทั้งที่ในความเป็นจริงเป็นเพียง **Testing Tool/Development Workload** สำหรับจำลองโหลดเข้าหน้าเว็บเท่านั้น