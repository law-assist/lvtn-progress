# Báo Cáo Tuần 1 — Hệ Thống Tra Cứu Văn Bản Pháp Luật

**Sinh viên:** Nguyễn Nhật Trần  
**GVHD:** ThS. Lê Đình Thuận & ThS. Võ Thanh Hùng  
**Tuần:** 16/07/2026 – 22/07/2026 (Tuần 1 / 12)  
**Giai đoạn:** Tiếp nối đề tài nhóm, phát triển độc lập (1 người, 3 tháng)

---

## 1. Bức Tranh Tổng Thể Hệ Thống

### 1.1 Mô tả tổng quan

Hệ thống **Law Assistant** gồm 4 service độc lập giao tiếp qua HTTP, đóng gói bằng Docker, triển khai qua GitHub Actions CI/CD.

```
┌─────────────────────────────────────────────────────────────────┐
│                     LAW ASSISTANT SYSTEM                        │
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │   Frontend   │───▶│   Backend    │───▶│    MongoDB       │  │
│  │  Next.js 14  │    │  NestJS 10   │    │  law_linking DB  │  │
│  │  Port 29000  │    │  Port 29001  │    │  (laws, users,   │  │
│  │  App Router  │    │  REST API    │    │   requests)      │  │
│  └──────┬───────┘    └──────┬───────┘    └──────────────────┘  │
│         │                  │                                    │
│         │      ┌───────────┘                                    │
│         │      │  ┌──────────────────┐                         │
│         │      └─▶│  AI Law Linking  │                         │
│         │         │  FastAPI/Python  │                         │
│         │         │  Port 28000      │                         │
│         │         │  PhoBERT NER+STS │                         │
│         │         └──────────────────┘                         │
│         │                                                       │
│         │      ┌──────────────────────────────────┐            │
│         └─────▶│         AI Chatbot               │            │
│                │  FastAPI/Python - Port 28080      │            │
│                │  LangChain + LangGraph            │            │
│                │  ChromaDB (vector store)          │            │
│                │  Ollama Llama3 (local LLM)        │            │
│                └──────────────────────────────────┘            │
└─────────────────────────────────────────────────────────────────┘
```

> Xem sơ đồ chi tiết: `architecture.drawio` / `architecture.drawio.png`

### 1.2 Luồng dữ liệu chính

| Luồng | Mô tả |
|-------|-------|
| **Crawler** | Backend → Puppeteer+Cheerio → thuvienphapluat.vn → MongoDB |
| **AI Linking** | Backend → AI Law Linking (PhoBERT NER/STS) → ghi quan hệ vào `laws.references` |
| **Chatbot** | Frontend (SSE) → AI Chatbot → ChromaDB + Ollama Llama3 → câu trả lời |
| **CI/CD** | GitHub Actions → Docker Hub → `kanghcmut/lvtn-backend-app`, `kanghcmut/lvtn-frontend-app` |

### 1.3 Tech stack

| Layer | Công nghệ |
|-------|-----------|
| Frontend | Next.js 14 App Router, Ant Design, SWR, next-auth, SSE |
| Backend | NestJS 10, Mongoose, Puppeteer, JWT (1h access / 7d refresh), AutoMapper |
| Database | MongoDB (`law_linking`), soft delete qua `isDeleted` |
| AI Linking | FastAPI, PhoBERT (NER 13 nhãn, Classification 6 nhãn, STS cosine ≥ 0.7) |
| AI Chatbot | FastAPI, LangChain, LangGraph StateGraph, ChromaDB, Ollama Llama3 |
| DevOps | Docker (node:24-alpine, nvidia/cuda:12.5.0), GitHub Actions, Docker Hub |

---

## 2. Đánh Giá Hiện Trạng

### 2.1 Những gì đã hoàn thành (từ nhóm trước)

- [x] Crawler tự động từ thuvienphapluat.vn — lưu văn bản dưới dạng cây nội dung có thứ bậc
- [x] REST API đầy đủ (Auth, User, Law, Crawler, Request) với Swagger tại `/api`
- [x] Phân quyền 3 tầng: User / Lawyer / Admin
- [x] Frontend route groups: `(auth)`, `(main)` → `(public)` / `(user)` / `(lawyer)` / `(admin)`
- [x] Xem văn bản + modal cross-reference (jQuery highlight + split-pane)
- [x] Export PDF (jsPDF + html2canvas)
- [x] Thông báo realtime qua SSE
- [x] Pipeline AI liên kết văn bản (PhoBERT NER → Classification → STS)
- [x] RAG Chatbot (LangGraph 4-step: translate → analyze → retrieve → generate)
- [x] CI/CD qua GitHub Actions, đa kiến trúc (amd64 + arm64)

---

### 2.2 Gaps — Những vấn đề cần giải quyết

#### G1 — Bảo mật: SHA-256 → bcrypt 🔴

**File:** `datcuong-backend/src/modules/user/user.service.ts`

```typescript
// Hiện tại (sai):
const hash = crypto.createHash('sha256').update(password).digest('hex');

// SHA-256 không có salt → cùng password luôn ra cùng hash
// → dễ bị tấn công bằng rainbow table
```

SHA-256 là thuật toán hash nhanh, không có salt, không được thiết kế cho password. Bcrypt có salt tự động + cost factor (chậm có chủ ý → brute-force tốn thời gian hơn).

**Cần làm:** Cài `bcrypt`, thay toàn bộ chỗ hash/verify password. Do SHA-256 một chiều, không migrate được — cần force user đổi mật khẩu hoặc reset DB dev.

---

#### G2 — Chatbot: Embedding model sai ngôn ngữ 🔴

**File:** `datcuong-ai-chatbot/api/api.py` + `datcuong-ai-chatbot/utils/data_processing.py`

```python
# Hiện tại:
embeddings = HuggingFaceEmbeddings(model_name="all-MiniLM-L6-v2")
```

`all-MiniLM-L6-v2` được train trên tiếng Anh. Khi embed câu tiếng Việt *"xử phạt vi phạm giao thông"*, model không hiểu ngữ nghĩa đúng → ChromaDB trả về kết quả không liên quan → chatbot trả lời sai.

**Cần làm:** Đổi sang `bge-m3` qua Ollama (đa ngôn ngữ, bao gồm tiếng Việt). Sau khi đổi phải rebuild toàn bộ ChromaDB.

---

#### G3 — Chatbot: LLM router là stub 🟡

**File:** `datcuong-ai-chatbot/components/router.py`

```python
# Hiện tại — luôn trả về "legislation" bất kể câu hỏi là gì:
def router(state):
    return "legislation"
```

Mọi câu hỏi đều đi vào cùng 1 collection ChromaDB, kể cả câu hỏi không liên quan pháp luật ("xin chào", "thời tiết hôm nay thế nào"). LLM phải xử lý context không liên quan → lãng phí, có thể trả lời sai.

**Cần làm:** Phân loại câu hỏi trước khi retrieve — ít nhất phân biệt `pháp luật` vs `không liên quan` để trả lời từ chối đúng cách.

---

#### G4 — Chatbot: Reranker chưa implement 🟡

**File:** `datcuong-ai-chatbot/components/document_ranking.py`

```python
# File hoàn toàn rỗng — không có nội dung
```

ChromaDB trả về top-5 kết quả theo cosine similarity, nhưng không có bước rerank. Vấn đề: cosine similarity đo vector gần nhau về mặt toán học, nhưng không đảm bảo document nào thực sự *trả lời* được câu hỏi. Reranker (cross-encoder) đọc cặp (câu hỏi, document) để chấm điểm chính xác hơn.

**Cần làm:** Implement `document_ranking.py` với cross-encoder model, rerank top-5 → giữ top-3.

---

#### G5 — Backend: Email notification chưa có 🟡

**File:** không tồn tại — chưa có module nào

Luận văn đề cập tính năng gửi email (mục 8.3 Hướng phát triển) nhưng code không có. Hiện tại không gửi email khi: đăng ký tài khoản, đặt lại mật khẩu, luật sư phản hồi yêu cầu tư vấn.

**Cần làm:** Tạo `EmailModule` trong NestJS dùng `nodemailer`, template cho các sự kiện chính.

---

#### G6 — Frontend: Upload tài liệu dùng mockapi.io 🟡

**File:** `datcuong-frontend/src/` — component upload tài liệu của lawyer/admin

Upload tài liệu (PDF đính kèm) đang gọi đến `mockapi.io` — một service mock API bên ngoài dùng để test. Dữ liệu upload không thực sự được lưu về hệ thống.

**Cần làm:** Thay bằng `multer` trên NestJS backend — lưu file vào server hoặc object storage. Đơn giản nhất: `MulterModule` với `diskStorage`, lưu vào `/uploads/`.

---

#### G7 — Frontend: Export PDF lỗi định dạng 🟠

**File:** `datcuong-frontend/src/` — component gọi jsPDF + html2canvas

Văn bản dài hoặc có bảng bị vỡ layout khi export. Nguyên nhân: `html2canvas` chụp ảnh DOM → jsPDF chèn ảnh vào PDF, không phải render text thực. Font tiếng Việt, line break, page break không được xử lý đúng.

**Cần làm:** Thay bằng `@react-pdf/renderer` — render PDF server-side với text thực, hỗ trợ tiếng Việt tốt hơn.

---

#### G8 — Frontend + Backend: Admin management stub 🟠

**File:** `datcuong-frontend/src/app/(admin)/` — nhiều trang chưa kết nối API

Các trang admin (quản lý user, quản lý văn bản, thống kê) đã có UI skeleton nhưng chưa gọi đúng API backend, hoặc API backend chưa có endpoint tương ứng.

**Cần làm:** Rà soát từng trang admin, map với Swagger backend (`http://localhost:29001/api`), kết nối các endpoint còn thiếu.

---

#### G9 — Chatbot: Không có script build ChromaDB 🟠

**File:** `datcuong-ai-chatbot/` — không có script build sẵn

README chatbot hướng dẫn người dùng tự viết script để đưa dữ liệu từ MongoDB vào ChromaDB. Nếu không có sẵn script, người mới setup sẽ không biết cách build vector store → service không chạy được.

**Cần làm:** Tạo `datcuong-ai-chatbot/scripts/build_vectorstore.py` — kéo `laws` từ MongoDB, embed, lưu vào ChromaDB.

---

#### G10 — AI Law Linking: dotenv path tương đối 🟠

**File:** `datcuong-ai-law-linking/src/services/` — các file Python load `.env`

```python
# Hiện tại:
load_dotenv(dotenv_path="../.env")
# Path tương đối với working directory khi chạy script
# Chạy từ thư mục khác → không tìm thấy .env → crash
```

**Cần làm:**
```python
import os
from pathlib import Path
load_dotenv(Path(__file__).parent.parent.parent / ".env")
```

---

#### G11 — Chatbot: Không xử lý lỗi khi ChromaDB chưa build 🟠

**File:** `datcuong-ai-chatbot/api/api.py`

Nếu thư mục `./database/all-MiniLM-L6-v2/3000_300/` không tồn tại, service crash ngay khi khởi động với traceback không rõ ràng. Không có check sớm hay thông báo hữu ích cho người dùng.

**Cần làm:** Thêm startup check — nếu ChromaDB chưa có, log hướng dẫn rõ ràng thay vì crash.

---

#### G12 — AI Law Linking: False match khi tên văn bản trùng nhau 🟠

**File:** `datcuong-ai-law-linking/src/services/ner_matching_service.py` (hoặc tương đương)

Pipeline tìm văn bản tham chiếu bằng cách so khớp `name` và `numberDoc`. Nếu trong DB có 2 văn bản cùng tên viết tắt nhưng khác số hiệu, hoặc văn bản được tham chiếu chưa được crawl vào DB, pipeline có thể match sai hoặc bỏ sót.

**Cần làm:** Thêm fallback — khi NER trích xuất được số hiệu đầy đủ (e.g. `100/2019/NĐ-CP`), ưu tiên exact match số hiệu thay vì chỉ match tên.

---

#### G13 — Chatbot: `mongo_handler.py` comment out datetime conversion 🟢

**File:** `datcuong-ai-chatbot/utils/mongo_handler.py`

```python
# Các trường này bị comment out:
# doc["dateApproved"] = str(doc["dateApproved"])
# doc["createdAt"] = str(doc["createdAt"])
```

MongoDB trả về datetime object, nếu dùng trực tiếp trong JSON response sẽ lỗi serialization.

**Cần làm:** Bỏ comment — convert datetime → string trước khi trả về.

---

#### G14 — Backend: Không có unit test 🟢

**File:** `datcuong-backend/src/` — thư mục `test/` gần như trống

Jest và Supertest đã được cài (`package.json`) nhưng không có test case nào được viết. Không có coverage cho các module quan trọng như auth, law, crawler.

**Cần làm:** Viết test tối thiểu cho: auth (login, register, token refresh), law (search, get by id), crawler (parse output format).

---

## 3. Phát Hiện Từ Codebase

Khi đọc kỹ code, phát hiện 2 điểm nhóm trước đã làm nhưng chưa khai thác:

**A. MongoDB đã lưu cây nội dung có cấu trúc pháp lý:**
```
laws.content = { type: "phan|chuong|muc|dieu|khoan|diem", value: "...", children: [...] }
```
File `datcuong-ai-chatbot/utils/data_processing.py` hiện **flatten toàn bộ cây** thành 1 chuỗi rồi chunk theo character count — mất hoàn toàn thông tin cấu trúc (Điều nào, Khoản mấy).

**B. AI Law Linking đã ghi quan hệ hai chiều vào MongoDB:**
```
laws.references = [{ targetId, relationType, confidence, position }]
```
Dữ liệu này nằm trong DB nhưng không có API nào expose ra, không có UI nào hiển thị.

---

## 4. Công Việc Tuần Này (Tuần 1)

| # | Công việc | Trạng thái |
|---|-----------|------------|
| 1 | Đọc toàn bộ codebase 4 repo, lập danh sách bugs/gaps | ✅ Done |
| 2 | Tạo README.md tiếng Việt cho cả 4 repo | ✅ Done |
| 3 | Tạo PR lên từng repo (qua fork — chỉ có pull permission) | ✅ Done |
| 4 | Lập kế hoạch 3 tháng chi tiết | ✅ Done |
| 5 | Tạo thư mục lvtn-progress, skeleton tracking | ✅ Done |

---

## 5. Đề Xuất Tính Năng Mới

### F1: Legal Knowledge Graph + Visualization

**Vấn đề hiện tại:**

AI Law Linking đã extract và lưu quan hệ giữa các văn bản vào `laws.references` trong MongoDB, nhưng không có cách nào để xem. Người dùng không biết: Nghị định này bị sửa đổi mấy lần? Luật nào được nhiều văn bản khác tham chiếu đến nhất?

**Ví dụ cụ thể:** Giả sử DB có:

```
Bộ luật Hình sự 2015  ←── được tham chiếu bởi NĐ 144/2021
Bộ luật Hình sự 2015  ←── được tham chiếu bởi NĐ 100/2019
NĐ 100/2019           ←── được sửa đổi bởi NĐ 123/2021
```

Hiện tại 3 quan hệ này nằm trong `laws.references` không ai thấy. Sau F1, mở trang `/graph` trên frontend → thấy ngay BLHS 2015 là node to nhất (nhiều văn bản trỏ vào), click NĐ 100 → thấy nó đã bị sửa đổi bởi NĐ 123/2021.

**Cần làm:**
- Backend: 1 endpoint `GET /api/laws/graph/:id?depth=2` đọc `laws.references` → trả về `{nodes, edges}`
- Backend: tính in-degree mỗi node → dùng làm `importance` (node to = nhiều văn bản khác tham chiếu)
- Frontend: `react-force-graph` render graph, click node → mở sidebar chi tiết văn bản

**File liên quan:** `datcuong-backend/src/modules/law/law.controller.ts`, `law.service.ts`, `datcuong-frontend/src/app/(main)/`

---

### F2: Chatbot trả lời có trích dẫn điều khoản cụ thể (Grounded Citation)

**Vấn đề hiện tại:**

**File:** `datcuong-ai-chatbot/api/api.py` — response schema hiện là:
```json
{ "context": [...], "answer": "Theo quy định hiện hành..." }
```

Chatbot trả lời nhưng không biết đang dựa vào Điều nào, Nghị định nào. Người dùng không thể kiểm tra lại.

**Ví dụ cụ thể:**

Câu hỏi: *"Uống rượu lái xe máy bị phạt bao nhiêu tiền?"*

Hiện tại:
```
Bot: Theo quy định, mức phạt từ 2–3 triệu đồng...
     (không biết từ NĐ nào, Điều nào)
```

Sau F2:
```
Bot: Theo Điều 6, Khoản 4 Nghị định 100/2019/NĐ-CP, mức phạt là 2–3 triệu 
     đồng và tước GPLX 10–12 tháng.

┌──────────────────────────────────────────────────────┐
│ NĐ 100/2019/NĐ-CP — Điều 6, Khoản 4                 │
│ "Phạt tiền từ 2.000.000 đồng đến 3.000.000 đồng đối │
│  với người điều khiển xe mô tô... trong máu hoặc     │
│  hơi thở có nồng độ cồn vượt quá 80 miligam..."      │
│                                                      │
│  [Xem toàn bộ Điều 6 →]                              │
└──────────────────────────────────────────────────────┘
```

**Cần làm:**
- Chatbot (`datcuong-ai-chatbot/api/api.py`): dùng `LangChain .with_structured_output()` để force LLM output JSON có `citations`
- Chatbot (`datcuong-ai-chatbot/utils/data_processing.py`): thêm `article_number`, `chapter` vào metadata ChromaDB (dùng cây MongoDB đã có)
- Frontend: thêm `CitationCard` component bên dưới câu trả lời, click → mở modal cross-reference đã có sẵn đúng điều đó

**Lý do kết hợp được với F3:** F3 tạo ra chunks đã có metadata `article_number` → F2 dùng metadata đó để fill vào citation. Không làm F3 trước thì F2 không có đủ thông tin để cite đúng đến Điều level.

---

### F3: Chunk văn bản theo Điều thay vì theo số ký tự (Structure-Aware Chunking)

**Vấn đề hiện tại:**

**File:** `datcuong-ai-chatbot/utils/data_processing.py`

```python
# Hiện tại: flatten toàn bộ cây thành 1 chuỗi
def content_processing(content):
    # duyệt đệ quy, nối tất cả value thành string
    ...
# Sau đó LangChain tự chunk theo character count (3000 chars)
```

Văn bản pháp luật có cấu trúc: `Chương → Mục → Điều → Khoản → Điểm`. Chunk theo ký tự có thể cắt giữa Khoản 1 và Khoản 2 của cùng 1 Điều — phá vỡ ngữ nghĩa.

**Ví dụ cụ thể:**

```
Điều 6. Xử phạt người điều khiển xe mô tô...
  Khoản 1. Phạt cảnh cáo hoặc phạt tiền từ 100.000 đến 200.000...
  Khoản 2. Phạt tiền từ 200.000 đến 400.000...
  Khoản 3. Phạt tiền từ 400.000 đến 600.000...   ← chunk bị cắt ở đây
  Khoản 4. Phạt tiền từ 2.000.000 đến 3.000.000... ← nằm ở chunk tiếp theo
```

Người hỏi về mức phạt nồng độ cồn (Khoản 4) có thể bị trả về Khoản 1/2/3 vì chúng cùng Điều 6 nhưng nằm trong chunk trước.

**Cần làm:**

Thay vì flatten → chunk by character, duyệt cây MongoDB và **mỗi node `type="dieu"` = 1 Document riêng** với metadata:
```python
Document(
    page_content=<toàn bộ text của Điều đó>,
    metadata={
        "article_header": "Điều 6. Xử phạt...",
        "chapter": "Chương II",
        "law_title": "NĐ 100/2019/NĐ-CP",
        "numberDoc": "100/2019/NĐ-CP",
        "law_id": "...",
    }
)
```

**File cần sửa:** `datcuong-ai-chatbot/utils/data_processing.py` — thêm hàm `extract_article_chunks()` thay thế `content_processing()`

---

### F4: Chatbot hỏi đáp theo thời gian hiệu lực văn bản (Temporal-Aware QA)

**Vấn đề hiện tại:**

Chatbot không biết ngày hiệu lực của văn bản. Pháp luật Việt Nam thay đổi thường xuyên — nhiều nghị định cũ bị thay thế bởi nghị định mới.

**Ví dụ cụ thể:**

```
NĐ 46/2016: có hiệu lực 01/08/2016 → hết hiệu lực 31/12/2019
NĐ 100/2019: có hiệu lực 01/01/2020 → đang hiệu lực (sửa đổi bởi NĐ 123/2021)
```

Câu hỏi: *"Uống rượu lái xe năm 2018 bị phạt bao nhiêu?"*

Hiện tại chatbot có thể trả lời dựa trên NĐ 100/2019 — **sai**, vì NĐ đó chưa tồn tại năm 2018. Năm 2018 phải áp dụng NĐ 46/2016.

Tình huống thực tế cần điều này: luật sư tra cứu để tranh tụng vụ án cũ, kiểm tra hợp đồng ký năm X có hợp lệ theo luật lúc đó không.

**Sau F4:**

Câu hỏi: *"Năm 2018 uống rượu lái xe máy bị phạt bao nhiêu?"*

```
1. Hệ thống phát hiện "năm 2018" → filter ChromaDB: 
   chỉ lấy văn bản có effectiveDate ≤ 2018-12-31 
   và expiryDate ≥ 2018-12-31 (hoặc chưa hết hiệu lực)

2. NĐ 100/2019 bị loại (chưa có năm 2018)
   NĐ 46/2016 được giữ lại

3. Bot: Năm 2018, áp dụng NĐ 46/2016 (hiệu lực 2016–2019).
   Mức phạt: 1–2 triệu đồng, tước GPLX 1–3 tháng.
   
   ⚠️ Văn bản này đã hết hiệu lực. Mức phạt hiện hành (NĐ 100/2019) cao hơn.

┌──────────────────────────────────────┐
│ NĐ 46/2016 — Điều 5, Khoản 3        │
│ Hiệu lực: 01/08/2016 – 31/12/2019   │  ← badge "Hết hiệu lực"
│ "Phạt tiền từ 1.000.000 đồng..."    │
└──────────────────────────────────────┘
```

**Cần làm:**
- Thêm `effectiveDate`, `expiryDate`, `status` vào metadata ChromaDB khi build vector store
- Backend: thêm 2 field này vào `laws` schema nếu chưa có (crawler đã crawl `dateApproved` — cần thêm `expiryDate`)
- Chatbot (`datcuong-ai-chatbot/api/api.py`): thêm temporal extraction từ câu hỏi → filter khi query ChromaDB
- Frontend: thêm badge "Hết hiệu lực" / "Đang hiệu lực" trên citation card

**Trường hợp khó cần xử lý riêng:** Một văn bản có thể chỉ bị sửa đổi **một số điều**, không phải toàn bộ — cần track hiệu lực ở mức Điều, không chỉ ở mức văn bản. Đây là phần phức tạp nhất, có thể để scope của research paper.

---

## 6. Kế Hoạch 3 Tháng (Tổng Quan)

| Tuần | Thời gian | Chủ đề |
|------|-----------|--------|
| W1 | 16–22/7 | Phân tích, README, kế hoạch ← **tuần này** |
| W2 | 23–29/7 | G1 bcrypt + G10 dotenv + G9 ChromaDB script + G7 PDF fix |
| W3 | 30/7–5/8 | F3 Structure-Aware Chunking + G2 BGE-M3 embedding |
| W4 | 6–12/8 | F2 Citation Generation (LLM + API) |
| W5 | 13–19/8 | F2 Citation Generation (Frontend UI) |
| W6 | 20–26/8 | F1 Legal Knowledge Graph (Backend API + PageRank) |
| W7 | 27/8–2/9 | F1 Legal Knowledge Graph (Frontend D3 viz) |
| W8 | 3–9/9 | F4 Temporal-Aware QA |
| W9 | 10–16/9 | G5 Email + G3 Router + G4 Reranker |
| W10 | 17–23/9 | G6 Upload + G8 Admin + G11 Error handling |
| W11 | 24–30/9 | Evaluation dataset (VLegalQA-50) + chạy experiments |
| W12 | 1–7/10 | Paper draft + báo cáo LVTN + final demo |

> Xem chi tiết từng tuần: `plan-2months.md`

---

## 7. Câu Hỏi Cho GVHD

1. Trong F2 (Citation Generation) và F4 (Temporal QA), thầy nghĩ hướng nào phù hợp hơn với thực tiễn pháp luật Việt Nam để tập trung sâu?

2. Thầy muốn em target RIVF 2026 (deadline ~tháng 8) hay KSE/ACIIDS năm sau?

3. Để đánh giá khách quan cần xây VLegalQA-50 (50 câu hỏi pháp luật + ground-truth citation). Thầy có thể hỗ trợ review tính chính xác của ground-truth không?

4. Fix lỗi bảo mật (G1: SHA-256 → bcrypt) có cần đưa vào chương "Cải tiến" của báo cáo LVTN không?

5. Thầy có tài nguyên GPU để thử fine-tune thêm không, hay nên giữ BGE-M3 pre-trained?

---

## 8. Ghi Chú

- Repo gốc tại org `law-assist` trên GitHub — `trannhatt` chỉ có quyền `pull` trên 4 repo code, tất cả thay đổi submit qua PR từ fork cá nhân.
- `lvtn-progress` repo: `trannhatt` có quyền admin — push trực tiếp được.
- 4 README PR đã tạo: `datcuong-backend` PR#1, `datcuong-frontend` PR#7, `datcuong-ai-law-linking` PR#2, `datcuong-ai-chatbot` PR#1.
- Môi trường local: macOS, cần Docker Desktop + Ollama để chạy toàn bộ hệ thống.
