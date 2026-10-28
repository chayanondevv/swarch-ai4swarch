**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชญานนท์ ทาธิจันทร์ / รหัสนิสิต:6870040521
**agent ที่ใช้:** เช่น GitHub Copilot (Agent mode), Gemini Code Assist, Gemini CLI, Claude Code, Codex, Cursor, Cline: …
**โมเดล:** …
**แพ็กเกจที่ใช้:** เช่น Copilot Student, Copilot Free, Gemini รุ่นฟรี: …

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | GitHub Copilot using Copilot SDK in VS Code; model not known. | `prompts/1-…` run A | • `AGENTS.md`<br>• `prompts/1-recover-architecture.md`<br>• `system/online-boutique/protos/demo.proto` | อยู่ในขอบเขต| 3 | ระบุได้ 9 Logical Services อย่างแม่นยำ |
| 2 | 1-B | GitHub Copilot SDK in VS Code; model: not exposed in this session. | `prompts/1-…` run B | • `prompts/1-recover-architecture.md`<br>• `system/online-boutique/protos/demo.proto`<br>• `system/online-boutique/kubernetes-manifests/README.md`<br>• `system/online-boutique/kubernetes-manifests/adservice.yaml`<br>• `system/online-boutique/kubernetes-manifests/cartservice.yaml`<br>• `system/online-boutique/kubernetes-manifests/checkoutservice.yaml`<br>• `system/online-boutique/kubernetes-manifests/currencyservice.yaml`<br>• `system/online-boutique/kubernetes-manifests/emailservice.yaml`<br>• `system/online-boutique/kubernetes-manifests/frontend.yaml`<br>• `system/online-boutique/kubernetes-manifests/kustomization.yaml`<br>• `system/online-boutique/kubernetes-manifests/loadgenerator.yaml`<br>• `system/online-boutique/kubernetes-manifests/paymentservice.yaml`<br>• `system/online-boutique/kubernetes-manifests/productcatalogservice.yaml`<br>• `system/online-boutique/kubernetes-manifests/recommendationservice.yaml`<br>• `system/online-boutique/kubernetes-manifests/shippingservice.yaml` | อยู่ในขอบเขต | 5 | • กู้คืนสถาปัตยกรรมได้สมบูรณ์ขึ้นมากเนื่องจากได้ข้อมูล Deployment Specs<br>• สามารถระบุ Container, Environment Variables, Ports และเพิ่มเติมบริการ Frontend รวมถึง LoadGenerator ที่ไม่มีใน `.proto` ได้ |
| 3 | 2-a | AI assistant using Copilot SDK in VS Code; Model: ไม่ทราบ | `prompts/2-…` run a | • `AGENTS.md`<br>• `2-write-adr.md`<br>• `template.md` | อยู่ในขอบเขต | 3 | เป็นเพียงข้อเสนอในเชิงทฤษฎี/แนวทางกว้าง ๆ (High-Level Conceptual Proposal) |
| 4 | 2-b | AI assistant using Copilot SDK in VS Code; Model: ไม่ทราบ | `prompts/2-…` run b | • `AGENTS.md`<br>• `prompts/2-write-adr.md`<br>• `docs/adr/template.md`<br>• `docs/scenario.md`<br>• `docs/adr/0001-use-redis-for-cart-storage.md`<br>• `system/online-boutique/src/checkoutservice/main.go` | อยู่ในขอบเขต | 5 | มีรายละเอียดบริบทจริงเชิงลึก (Concrete Context-Aware Details) |
| 5 | 3 | Copilot SDK in VS Code (AI assistant) | `prompts/3-…` | • `AGENTS.md`<br>• `prompts/3-review-adr.md`<br>• `docs/adr/0002-2b-adr.md`<br>• `docs/scenario.md` | อยู่ในขอบเขต | 5 | |
| 6 | | | | | | | |
| 7 | | | | | | | |