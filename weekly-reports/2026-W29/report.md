# Báo Cáo Tuần 1 — Hệ Thống Tra Cứu Văn Bản Pháp Luật

**Sinh viên:** Nguyễn Nhật Trần  
**GVHD:** ThS. Lê Đình Thuận & ThS. Võ Thanh Hùng  
**Tuần:** 16/07/2026 – 22/07/2026 (Tuần 1 / 8)  
**Giai đoạn:** Tiếp nối đề tài nhóm, phát triển độc lập (1 người, 2 tháng)

---

## 1. Bức Tranh Tổng Thể Hệ Thống (Big Picture)

### 1.1 Mô tả tổng quan

Hệ thống **Law Assistant** là nền tảng tra cứu và hỗ trợ pháp luật thông minh, gồm 4 service độc lập giao tiếp qua HTTP, được đóng gói bằng Docker và triển khai tự động qua GitHub Actions CI/CD.

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

> Xem sơ đồ chi tiết tại: `architecture.drawio` (mở bằng draw.io hoặc diagrams.net)

### 1.2 Luồng dữ liệu chính

| Luồng | Mô tả |
|-------|-------|
| **Crawler** | Backend → Puppeteer+Cheerio → thuvienphapluat.vn → MongoDB |
| **Liên kết văn bản** | Backend → AI Law Linking (PhoBERT NER/STS) → kết quả liên kết |
| **Chatbot** | Frontend (SSE) → AI Chatbot → ChromaDB + Ollama → câu trả lời |
| **CI/CD** | GitHub Actions → Docker Hub → `kanghcmut/lvtn-backend-app`, `kanghcmut/lvtn-frontend-app` |

### 1.3 Tech stack tóm tắt

| Layer | Công nghệ |
|-------|-----------|
| Frontend | Next.js 14 App Router, Ant Design, SWR, next-auth, SSE |
| Backend | NestJS 10, Mongoose, Puppeteer, JWT (1h/7d), AutoMapper |
| Database | MongoDB (`law_linking`), soft delete qua `isDeleted` |
| AI Linking | FastAPI, PhoBERT (NER 13 nhãn, Classification 6 nhãn, STS cosine) |
| AI Chatbot | FastAPI, LangChain, LangGraph, ChromaDB, Ollama Llama3 |
| DevOps | Docker (node:24-alpine, nvidia/cuda:12.5.0), GitHub Actions, Docker Hub |

---

## 2. Đánh Giá Hiện Trạng (Current State Assessment)

### 2.1 Những gì đã hoàn thành (từ nhóm trước)

- [x] Crawler tự động từ thuvienphapluat.vn (Puppeteer + Cheerio)
- [x] REST API đầy đủ (Auth, User, Law, Crawler, Request) với Swagger
- [x] Phân quyền 3 tầng: User / Lawyer / Admin
- [x] Frontend với route groups: `(auth)`, `(main)`, `(public)`, `(user)`, `(lawyer)`, `(admin)`
- [x] Xem văn bản + modal cross-reference (jQuery highlight + split-pane)
- [x] Export PDF (jsPDF + html2canvas)
- [x] Thông báo realtime qua SSE
- [x] Pipeline AI liên kết văn bản (PhoBERT NER → Classification → STS)
- [x] RAG Chatbot (LangGraph 4-step: translate → analyze → retrieve → generate)
- [x] CI/CD qua GitHub Actions, đa kiến trúc (amd64 + arm64)

### 2.2 Khoảng cách giữa luận văn và code thực tế (Gaps)

| # | Vấn đề | Mức độ |
|---|--------|--------|
| G1 | **Bảo mật nghiêm trọng**: Password hash bằng SHA-256 (không salt), phải dùng bcrypt | 🔴 Critical |
| G2 | **Embedding sai ngôn ngữ**: Chatbot dùng `all-MiniLM-L6-v2` (tiếng Anh) cho văn bản tiếng Việt | 🔴 High |
| G3 | LLM router trong LangGraph là stub — chỉ pass-through, không routing thực sự | 🟡 Medium |
| G4 | `document_ranking` trong chatbot luôn trống (chưa implement reranker) | 🟡 Medium |
| G5 | Email notification chưa hiện thực | 🟡 Medium |
| G6 | Upload tài liệu dùng mockapi.io (không phải storage thực) | 🟡 Medium |
| G7 | Tải xuống văn bản PDF còn lỗi định dạng | 🟠 Low-Medium |
| G8 | Admin management stub — nhiều tính năng quản trị chưa kết nối | 🟠 Low-Medium |
| G9 | ChromaDB build script chưa có — phải tự build vector store thủ công | 🟠 Low-Medium |
| G10 | `dotenv_path="../.env"` trong AI linking — path phụ thuộc working directory | 🟠 Low |

---

## 3. Công Việc Tuần Này (Tuần 1)

| # | Công việc | Trạng thái |
|---|-----------|------------|
| 1 | Đọc toàn bộ codebase 4 repo, lập danh sách bugs/gaps | ✅ Done |
| 2 | Tạo README.md Vietnamese cho cả 4 repo | ✅ Done |
| 3 | Tạo PR lên từng repo (qua fork vì chỉ có pull permission) | ✅ Done |
| 4 | Lập kế hoạch 2 tháng chi tiết | ✅ Done (tài liệu này) |
| 5 | Tạo thư mục lvtn-progress, skeleton tracking | ✅ Done |

---

## 4. Phát Hiện Quan Trọng Từ Codebase (Thuận Lợi Cho Nghiên Cứu)

Khi đọc kỹ code, phát hiện 2 điểm mà nhóm trước đã làm rất tốt nhưng **chưa khai thác hết tiềm năng**:

**A. MongoDB đã lưu cây nội dung có cấu trúc pháp lý đầy đủ:**
```
laws.content = { type: "phan|chuong|muc|dieu|khoan|diem", value: "...", children: [...] }
```
Crawler của backend đã phân tích cú pháp văn bản thành cây có thứ bậc. Chatbot hiện tại **flatten toàn bộ** thành 1 chuỗi rồi chunk theo character — mất hoàn toàn cấu trúc.

→ **Cơ hội:** Chunk theo từng "Điều" (Article-level) với metadata đầy đủ → nền tảng cho citation chính xác.

**B. AI Law Linking đã ghi quan hệ vào MongoDB:**
```
laws.references = [{ targetId, relationType, confidence, position }]
```
Pipeline PhoBERT đã extract và lưu các quan hệ pháp lý (sửa đổi, bãi bỏ, hướng dẫn...) dưới dạng structured data.

→ **Cơ hội:** Build knowledge graph visualization từ dữ liệu này mà không cần extract lại — chỉ cần API + D3.js.

---

## 5. Đề Xuất Hướng Nghiên Cứu Mới (Xứng Tầm LVTN + Paper)

Dựa trên phân tích trên, đề xuất 4 tính năng nghiên cứu mới chưa có trong hệ thống hiện tại và **chưa có trong industry Việt Nam**:

### F1: Legal Knowledge Graph + Visualization
**Vấn đề:** Quan hệ giữa các văn bản đang "chôn vùi" trong DB. Không ai thấy được: "Nghị định này bị sửa đổi bao nhiêu lần? Luật nào là trung tâm nhất?"

**Đề xuất:** Expose graph API từ `laws.references` + D3.js force-directed graph trên frontend.

**Paper angle:** Network analysis cho Vietnamese legal corpus (PageRank, centrality, hub laws)

### F2: Grounded Citation Generation ⭐ (contribution chính)
**Vấn đề:** Chatbot trả lời không biết mình đang trích từ Điều nào, Nghị định nào → người dùng không thể verify.

**Đề xuất:** Force LLM output JSON với `citations: [{numberDoc, article_header, relevant_text}]` → frontend hiển thị citation card + link đến văn bản gốc.

**Paper angle:** Citation-grounded generation cho Vietnamese legal QA — chưa có paper nào

### F3: Structure-Aware Article-Level Chunking ⭐ (contribution chính)
**Vấn đề:** ChromaDB hiện chunk theo character count, phá vỡ cấu trúc Điều/Khoản.

**Đề xuất:** Dùng cây MongoDB đã có, chunk theo Điều → mỗi chunk = 1 Điều hoàn chỉnh với metadata (chapter, article_number, law_title).

**Paper angle:** Domain-specific chunking cho legal RAG (kết hợp với F2)

### F4: Temporal-Aware Legal QA
**Vấn đề:** "Mức phạt trước 2021 là bao nhiêu?" — chatbot không biết temporal context.

**Đề xuất:** Thêm effective_date filter vào ChromaDB retrieval + detect temporal intent từ câu hỏi.

**Paper angle:** Temporal reasoning trong Vietnamese legal QA

> Xem thiết kế kỹ thuật chi tiết: `novel-features.md`

---

## 6. Kế Hoạch 2 Tháng (Tổng Quan)

| Tuần | Chủ đề | Contributions |
|------|--------|---------------|
| W1 (16-22/7) | Phân tích, README, kế hoạch | ← **tuần này** |
| W2 (23-29/7) | Security (bcrypt) + bug fixes | Nền tảng stable |
| W3 (30/7-5/8) | **F3** Structure-Aware Chunking + BGE-M3 | Improved retrieval |
| W4 (6-12/8) | **F2** Citation-Grounded Generation | Novel contribution |
| W5 (13-19/8) | **F1** Legal Knowledge Graph | Novel contribution |
| W6 (20-26/8) | **F4** Temporal-Aware QA + Email | Novel contribution |
| W7 (27/8-2/9) | Evaluation dataset (VLegalQA-50) + admin | Paper experiments |
| W8 (3-9/9) | Paper draft + final integration test | Deliverable |

> Xem chi tiết từng tuần tại: `plan-2months.md`

---

## 7. Câu Hỏi Cho GVHD

1. **Hướng nghiên cứu chính:** Trong F2 (Citation Generation) và F4 (Temporal QA), thầy nghĩ hướng nào có giá trị nghiên cứu cao hơn và phù hợp hơn với thực tiễn pháp luật Việt Nam?

2. **Venue paper:** Thầy muốn em target RIVF 2026 (tháng 11, deadline ~tháng 8) hay một venue quốc tế như ACIIDS 2027?

3. **Dataset:** Để đánh giá khách quan, em cần xây VLegalQA-50 (50 câu hỏi + ground-truth citations). Thầy có thể hỗ trợ review tính chính xác của ground-truth không?

4. **Phạm vi LVTN:** Fix lỗi bảo mật (SHA-256 → bcrypt) có cần đưa vào chương "Cải tiến" của báo cáo không?

5. **Embedding:** Thầy có tài nguyên GPU để fine-tune PhoBERT thêm cho embedding không, hay nên giữ BGE-M3 pre-trained?

---

## 8. Ghi Chú

- Repo gốc tại org `law-assist` trên GitHub, tôi (`trannhatt`) chỉ có quyền `pull` nên tất cả thay đổi được submit qua PR từ fork cá nhân.
- Tất cả 4 README PR đã được tạo: datcuong-backend (PR#1), datcuong-frontend (PR#7), datcuong-ai-law-linking (PR#2), datcuong-ai-chatbot (PR#1).
- Môi trường local: macOS, cần Docker Desktop và Ollama để chạy toàn bộ hệ thống.
