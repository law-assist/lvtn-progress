# Kế Hoạch 3 Tháng — Phát Triển Độc Lập

**Sinh viên:** Nguyễn Nhật Trần  
**Thời gian:** 16/07/2026 – 07/10/2026 (12 tuần)

Mỗi task đều ghi rõ file nào cần đụng vào (BE/FE/AI), và link tham chiếu.

---

## Timeline Tổng Quan

```
W1  ✅ Phân tích + README + Reports
W2     G1 bcrypt + G9 build script + G10 dotenv + G7 PDF fix
W3     F3 Article-level chunking + G2 BGE-M3 embedding
W4     F2 Citation (AI side)
W5     F2 Citation (Frontend)
W6     F1 Knowledge Graph (Backend)
W7     F1 Knowledge Graph (Frontend)
W8     F4 Temporal-Aware QA
W9     G5 Email + G3 Router + G4 Reranker
W10    G6 Upload + G8 Admin
W11    Evaluation dataset VLegalQA-50
W12    Paper draft + báo cáo LVTN + final
```

---

## Tuần 2: 23–29/07 — Fix Nền Tảng

### G1 — SHA-256 → bcrypt
- **BE:** `datcuong-backend/src/modules/user/user.service.ts` — thay `crypto.createHash('sha256')` bằng `bcrypt.hash()`
- **FE:** thêm trang reset password nếu user cũ bị flag
- **Ref:** https://docs.nestjs.com/security/encryption-and-hashing

### G9 — ChromaDB build script
- **AI:** tạo `datcuong-ai-chatbot/scripts/build_vectorstore.py` — kéo laws từ MongoDB, embed, lưu ChromaDB
- **Ref:** https://python.langchain.com/docs/integrations/vectorstores/chroma

### G10 — dotenv path
- **AI:** `datcuong-ai-law-linking/src/services/*.py` — thay `dotenv_path="../.env"` bằng `Path(__file__).parent.parent / ".env"`
- **Ref:** https://pypi.org/project/python-dotenv

### G7 — PDF export
- **FE:** `datcuong-frontend/src/` — thay jsPDF+html2canvas bằng `@react-pdf/renderer`
- **Ref:** https://react-pdf.org

---

## Tuần 3: 30/07–05/08 — F3: Structure-Aware Chunking + BGE-M3

### F3 — Chunk theo Điều thay vì ký tự
- **AI:** `datcuong-ai-chatbot/utils/data_processing.py` — thêm `extract_article_chunks()` duyệt cây MongoDB (`type="dieu"`) thay vì flatten toàn bộ rồi chunk by character
- Mỗi chunk = 1 Điều, metadata: `article_header`, `chapter`, `numberDoc`, `law_id`
- **Ref:** https://python.langchain.com/docs/how_to/document_loader_custom

### G2 — Đổi embedding all-MiniLM-L6-v2 → BGE-M3
- **AI:** `datcuong-ai-chatbot/api/api.py` — thay `HuggingFaceEmbeddings(model_name="all-MiniLM-L6-v2")` bằng `OllamaEmbeddings(model="bge-m3")`
- Sau khi đổi: chạy build script để rebuild ChromaDB
- **Ref:** https://ollama.com/library/bge-m3 | https://python.langchain.com/docs/integrations/text_embedding/ollama

**Đánh giá W3:** test 20 câu hỏi, so sánh P@3 (% có đáp án trong top-3) trước/sau

---

## Tuần 4–5: 06–19/08 — F2: Citation Generation

### F2 — AI (W4)
- **AI:** `datcuong-ai-chatbot/api/api.py` + node `generate` trong LangGraph
- Dùng `llm.with_structured_output(CitationResponse)` để force output JSON có `citations[]`
- Response schema mới: `{ answer, citations: [{numberDoc, article_header, relevant_text, law_id}], confidence }`
- **Ref:** https://python.langchain.com/docs/how_to/structured_output

### F2 — Backend (W4)
- **BE:** `datcuong-backend/src/modules/law/law.controller.ts` — thêm `GET /api/laws/:id/article?header=Điều+6` để FE mở đúng điều từ citation
- **Ref:** NestJS controller docs đã có trong Swagger `/api`

### F2 — Frontend (W5)
- **FE:** tạo `datcuong-frontend/src/components/CitationCard.tsx` — hiển thị badge NĐ + trích đoạn + nút "Xem văn bản gốc" (dùng lại modal cross-reference đã có)
- **FE:** `datcuong-frontend/src/app/(main)/.../chatbot/` — render `<CitationCard />` bên dưới câu trả lời
- **Ref:** https://ant.design/components/card (AntD Card component)

---

## Tuần 6–7: 20/08–02/09 — F1: Legal Knowledge Graph

### F1 — Backend (W6)
- **BE:** `datcuong-backend/src/modules/law/law.service.ts` — `buildSubgraph(id, depth)` đọc `laws.references` → `{ nodes, edges }`
- **BE:** thêm `GET /api/laws/graph/:id?depth=2` và `GET /api/laws/graph/stats/top` (top 10 luật by in-degree)
- **Ref:** MongoDB aggregation `$unwind` + `$group`: https://www.mongodb.com/docs/manual/aggregation

### F1 — Frontend (W7)
- **FE:** tạo `datcuong-frontend/src/app/(main)/(public)/law-graph/page.tsx` dùng `react-force-graph`
- Node size ~ in-degree; edge color ~ loại quan hệ; click node → sidebar chi tiết
- **Ref:** https://github.com/vasturiano/react-force-graph

---

## Tuần 8: 03–09/09 — F4: Temporal-Aware QA

### F4 — Backend
- **BE:** `datcuong-backend/src/modules/law/` — thêm `expiryDate`, `status` vào Law schema
- Crawler cập nhật `status = 'repealed'` khi AI linking ghi relation `relationType = 'repeals'`
- **Ref:** Mongoose schema: https://mongoosejs.com/docs/schematypes.html

### F4 — AI
- **AI:** tạo `datcuong-ai-chatbot/utils/temporal_extractor.py` — regex detect "năm 2018", "trước 2022", "hiện hành" → trả về `as_of: date`
- **AI:** `datcuong-ai-chatbot/api/api.py` node `retrieve` — thêm ChromaDB `where_filter` lọc theo `effective_date ≤ as_of` và `status`
- **Ref:** ChromaDB where filter: https://docs.trychroma.com/guides#filtering-by-metadata

### F4 — Frontend
- **FE:** citation card thêm badge "✓ Đang hiệu lực" / "⚠ Hết hiệu lực"
- **FE:** chatbot area hiển thị "Trả lời theo văn bản hiệu lực tại: 31/12/2018" nếu có temporal filter

---

## Tuần 9: 10–16/09 — Email + Router + Reranker

### G5 — Email
- **BE:** tạo `datcuong-backend/src/modules/mail/` dùng `@nestjs-modules/mailer` + nodemailer
- Templates: welcome, password-reset, request-update
- **Ref:** https://nest-modules.github.io/mailer

### G3 — LLM Router
- **AI:** `datcuong-ai-chatbot/components/router.py` — thay stub bằng LLM classify "legal / general / unknown"
- Nếu "general" → skip RAG, trả lời trực tiếp
- **Ref:** https://python.langchain.com/docs/how_to/routing

### G4 — Reranker
- **AI:** `datcuong-ai-chatbot/components/document_ranking.py` — implement dùng `CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")`, rerank top-5 → giữ top-3
- **Ref:** https://www.sbert.net/docs/cross_encoder/usage/usage.html

---

## Tuần 10: 17–23/09 — Upload + Admin

### G6 — File Upload
- **BE:** `datcuong-backend/src/modules/upload/` — `MulterModule` với diskStorage, endpoint `POST /api/upload`, giới hạn 10MB PDF
- **FE:** thay `mockapi.io` call bằng `axios.post('/api/upload', formData)`
- **Ref:** https://docs.nestjs.com/techniques/file-upload

### G8 — Admin UI
- **FE:** `datcuong-frontend/src/app/(admin)/` — rà soát từng trang, map với Swagger, kết nối API còn thiếu
- **BE:** bổ sung endpoint admin còn thiếu (ban user, delete law, manage requests)

---

## Tuần 11: 24–30/09 — Evaluation

### VLegalQA-50
Xây dataset 50 câu hỏi pháp luật + ground truth (law_id + article + answer):
- 20 câu giao thông, 15 câu lao động, 15 câu dân sự/hình sự
- 10 câu temporal (hỏi luật tại thời điểm cụ thể)

Chạy experiments, fill bảng:

| Config | P@3 | Citation Acc. | Temporal Acc. |
|--------|-----|---------------|---------------|
| Baseline (all-MiniLM + char-chunk) | - | - | - |
| + BGE-M3 + Article-chunk (F3) | - | - | - |
| + Citation F2 | - | - | - |
| + Temporal F4 | - | - | - |

---

## Tuần 12: 01–07/10 — Paper + Final

### Paper
- **Title:** *Structure-Aware Chunking and Citation-Grounded Generation for Vietnamese Legal QA*
- **Sections:** Intro → Related Work → System → F3 → F2 → F1 → F4 → Experiments → Conclusion
- **Venue:** RIVF 2026 / KSE 2026 / ACIIDS 2027

### Final checklist
- [ ] Tất cả 4 service chạy ổn với `docker-compose up`
- [ ] Tất cả PR merged (bcrypt, F2, F3, F1, F4)
- [ ] VLegalQA-50 lên GitHub/HuggingFace
- [ ] Paper draft ~8 trang
- [ ] Báo cáo LVTN chương mới: Cải tiến + Kết quả thực nghiệm
