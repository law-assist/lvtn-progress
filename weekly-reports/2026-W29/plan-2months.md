# Kế Hoạch 2 Tháng — Phát Triển Độc Lập (1 Người)

**Sinh viên:** Nguyễn Nhật Trần  
**Thời gian:** 16/07/2026 – 15/09/2026 (8 tuần)  
**Hướng chính:** Tiếp nối + mở rộng nghiên cứu → bài báo khoa học

---

## Bối Cảnh Quan Trọng — Insight Từ Codebase

Sau khi đọc kỹ codebase, phát hiện 2 điểm thuận lợi lớn:

**1. MongoDB đã lưu cây nội dung có cấu trúc pháp lý:**
```
laws.content = { type: "phan/chuong/muc/dieu/khoan/diem", value: "...", children: [...] }
```
→ **F3 (Structure-Aware Chunking) không cần parse lại** — chỉ cần query MongoDB theo type="dieu", lấy từng Điều làm chunk riêng.

**2. AI Linking đã ghi quan hệ hai chiều vào MongoDB:**
```
laws.references = [{ targetId, relationType, confidence, position }]
```
→ **F1 (Knowledge Graph) không cần extract lại** — chỉ cần đọc `references` field và build graph API.

---

## Tổng Quan Mục Tiêu

| Nhóm | Nội dung | Output |
|------|----------|--------|
| **Fix nền tảng** | Security + bugs + setup | Hệ thống stable, reproducible |
| **Research Features** | F1+F2+F3+F4 (novel) | Contribution cho paper |
| **Đánh giá** | Evaluation dataset + metrics | Basis cho paper experiments |
| **Deliverable** | Report LVTN + paper draft | Nộp thầy |

---

## Tuần 1: 16/07 – 22/07 — Phân Tích & Lập Kế Hoạch ✅ DONE

**Đã hoàn thành:**
- [x] Đọc toàn bộ codebase 4 repo, lập danh sách gaps
- [x] Tạo README.md Vietnamese cho 4 repo
- [x] Tạo PR lên từng repo
- [x] Architecture diagram (draw.io)
- [x] Kế hoạch + novel features proposal

---

## Tuần 2: 23/07 – 29/07 — Fix Nền Tảng (Security + Bugs)

### Ưu tiên: Phải làm trước khi implement feature mới

#### 2.1 — Backend: SHA-256 → bcrypt [P0 Security]

**File:** `datcuong-backend/src/modules/user/user.service.ts`  
Hiện tại dùng `crypto.createHash('sha256')` không có salt.

```bash
npm install bcrypt @types/bcrypt
```

```typescript
// Thay:
const hash = crypto.createHash('sha256').update(password).digest('hex');
// Bằng:
const hash = await bcrypt.hash(password, 12);
// Và verify:
const valid = await bcrypt.compare(password, storedHash);
```

Viết migration script để flag user cũ cần đổi mật khẩu (không thể migrate tự động vì SHA-256 một chiều).

**Time:** 1 ngày

#### 2.2 — Chatbot: ChromaDB Build Script [P1]

**File:** Tạo mới `datcuong-ai-chatbot/scripts/build_vectorstore.py`

Hiện không có script build ChromaDB — người mới không thể chạy được. Script cần:
- Kết nối MongoDB, lấy toàn bộ `laws`
- Dùng `data_processing.py` đã có để flatten content
- Chunk mặc định (chunk_size=3000) → dùng cấu trúc Điều (xem tuần 3)
- Lưu vào `./database/` với embedding đã chọn

**Time:** 1 ngày

#### 2.3 — AI Law Linking: Fix dotenv path [P2]

**File:** `datcuong-ai-law-linking/src/services/*.py`  
Thay `dotenv_path="../.env"` bằng path tuyệt đối:
```python
import os
from dotenv import load_dotenv
load_dotenv(dotenv_path=os.path.join(os.path.dirname(__file__), '../../.env'))
```

**Time:** 0.5 ngày

#### 2.4 — Frontend: Fix PDF Export [P2]

**File:** `datcuong-frontend/src/` — tìm jsPDF component  
Văn bản PDF bị lỗi định dạng khi có ký tự tiếng Việt dài.

Hướng xử lý: dùng `@react-pdf/renderer` thay jsPDF+html2canvas để render server-side — tránh vấn đề font rendering của browser.

**Time:** 1.5 ngày

---

## Tuần 3: 30/07 – 05/08 — F3: Structure-Aware Chunking + BGE-M3

### Mục tiêu: Cải thiện nền tảng RAG trước khi làm citation

#### 3.1 — F3: Article-Level Chunking từ MongoDB Tree

**Insight:** Backend đã lưu cây nội dung với `type` field:
```json
{ "type": "dieu", "value": "Điều 6. Xử phạt vi phạm giao thông...", "children": [...] }
```

Thay vì flatten toàn bộ văn bản rồi chunk theo character, **query trực tiếp các node type="dieu"** từ MongoDB:

```python
# datcuong-ai-chatbot/utils/legal_chunker.py

def extract_article_chunks(law_doc: dict) -> List[Document]:
    """
    Duyệt cây MongoDB, extract từng Điều thành 1 Document.
    Metadata đầy đủ: law_id, article_number, chapter, law_title, numberDoc
    """
    chunks = []
    
    def traverse(node, context):
        if node.get("type") == "dieu":
            # Collect toàn bộ text của Điều này (recursive)
            article_text = flatten_subtree(node)
            chunks.append(Document(
                page_content=article_text,
                metadata={
                    "law_id": law_doc["_id"],
                    "law_title": law_doc["name"],
                    "numberDoc": law_doc.get("numberDoc", ""),
                    "department": law_doc.get("department", ""),
                    "article_header": node["value"][:100],  # "Điều 6. Xử phạt..."
                    "chapter": context.get("chapter", ""),
                    "section": context.get("section", ""),
                    "effective_date": law_doc.get("dateApproved", ""),
                }
            ))
        # ... recurse with updated context
    
    traverse(law_doc["content"], {})
    return chunks
```

**Kết quả:** Mỗi Điều = 1 chunk riêng với metadata đầy đủ → cơ sở cho F2 citations.

**Time:** 2 ngày

#### 3.2 — Thay Embedding Model: all-MiniLM-L6-v2 → BGE-M3

**Vấn đề:** `all-MiniLM-L6-v2` train tiếng Anh, kém với tiếng Việt.

```python
# Thay:
from langchain_huggingface import HuggingFaceEmbeddings
embeddings = HuggingFaceEmbeddings(model_name="all-MiniLM-L6-v2")

# Bằng:
from langchain_ollama import OllamaEmbeddings
embeddings = OllamaEmbeddings(model="bge-m3")
# ollama pull bge-m3  (~570MB, multilingual SOTA)
```

Rebuild ChromaDB với embedding mới + structure-aware chunks.

**Time:** 1 ngày (+ 1 ngày rebuild + test)

#### 3.3 — Evaluation nhỏ: So sánh trước/sau

Test 20 câu hỏi pháp luật thực tế, so sánh:
- `all-MiniLM + character-chunking` (baseline)
- `BGE-M3 + article-chunking` (new)

Metric: manual relevance score (0/1) cho top-3 retrieved chunks.

**Time:** 1 ngày

**Deliverable W3:** PR chatbot với structure-aware chunker + BGE-M3 + eval results

---

## Tuần 4: 06/08 – 12/08 — F2: Grounded Citation Generation

### Mục tiêu: Chatbot trả lời có trích dẫn điều khoản cụ thể

Đây là **contribution chính cho paper**.

#### 4.1 — Backend: Structured JSON Output từ LLM

**File:** `datcuong-ai-chatbot/components/generate.py` (hoặc tương đương)

Thay vì để LLM trả lời tự do, thêm prompt instruction để output JSON có citation:

```python
CITATION_PROMPT = ChatPromptTemplate.from_template("""
Bạn là luật sư chuyên nghiệp. Dựa vào các điều khoản pháp luật dưới đây, hãy trả lời câu hỏi.

**Bắt buộc:** Output JSON theo đúng format sau:
{{
  "answer": "câu trả lời tổng hợp bằng tiếng Việt",
  "citations": [
    {{
      "law_id": "id văn bản",
      "numberDoc": "100/2019/NĐ-CP",
      "article_header": "Điều 6. Xử phạt người điều khiển xe ô tô...",
      "relevant_text": "đoạn trích liên quan nhất, nguyên văn",
      "relevance": "giải thích tại sao điều này liên quan"
    }}
  ],
  "confidence": "high|medium|low"
}}

Văn bản pháp luật tham khảo:
{context}

Câu hỏi: {question}
""")
```

```python
# Parse JSON output từ LLM (dùng Pydantic)
class Citation(BaseModel):
    law_id: str
    numberDoc: str
    article_header: str
    relevant_text: str
    relevance: str

class CitationResponse(BaseModel):
    answer: str
    citations: List[Citation]
    confidence: str
```

**Time:** 2 ngày

#### 4.2 — Cập nhật API Response Schema

Thay response `{ context, answer }` bằng:
```json
{
  "answer": "Theo Điều 6 Nghị định 100/2019...",
  "citations": [
    {
      "numberDoc": "100/2019/NĐ-CP",
      "article_header": "Điều 6. Xử phạt người điều khiển xe ô tô...",
      "relevant_text": "Phạt tiền từ 4.000.000 đồng đến 6.000.000 đồng...",
      "relevance": "Quy định mức phạt cụ thể cho hành vi vượt đèn đỏ"
    }
  ],
  "confidence": "high"
}
```

**Time:** 0.5 ngày

#### 4.3 — Frontend: Citation Card Component

Thêm component hiển thị citation bên dưới câu trả lời chatbot:

```tsx
// CitationCard.tsx
<div className="citation-card" onClick={() => openLawModal(citation.law_id)}>
  <span className="law-badge">{citation.numberDoc}</span>
  <p className="article-header">{citation.article_header}</p>
  <blockquote className="relevant-text">"{citation.relevant_text}"</blockquote>
  <span className="relevance-note">{citation.relevance}</span>
  <button>Xem văn bản gốc →</button>
</div>
```

**Time:** 2 ngày

#### 4.4 — Evaluation: Citation Precision

Với 20 câu hỏi test từ tuần 3, check thêm:
- Citation Precision: % citations trỏ đúng điều khoản tồn tại trong DB
- Citation Faithfulness: câu trả lời có nhất quán với `relevant_text` không?

**Time:** 1 ngày

**Deliverable W4:** PR chatbot (structured output) + PR frontend (citation UI) + eval results

---

## Tuần 5: 13/08 – 19/08 — F1: Legal Knowledge Graph

### Mục tiêu: Visualization đồ thị quan hệ văn bản pháp luật

**Insight quan trọng:** AI Law Linking đã ghi quan hệ vào MongoDB (`laws.references`). Chỉ cần:
1. Expose graph API từ dữ liệu này
2. Thêm D3.js visualization

#### 5.1 — Backend: Graph API

**File:** `datcuong-backend/src/modules/law/` — thêm controller mới

```typescript
// GET /api/laws/graph/:id?depth=2
// Trả về subgraph xung quanh văn bản id, depth levels
interface LawNode {
  id: string;
  title: string;
  numberDoc: string;
  type: string;  // Luật/Nghị định/Thông tư...
  department: string;
  status: 'active' | 'amended' | 'repealed';
  importance: number;  // PageRank score
}

interface LawEdge {
  source: string;
  target: string;
  relationType: 'amends' | 'repeals' | 'guides' | 'references' | 'supplements';
  confidence: number;
}

// GET /api/laws/graph/stats
// Trả về top-10 luật quan trọng nhất (PageRank), centrality scores
```

**PageRank đơn giản:**
```typescript
// Tính in-degree (số luật khác tham chiếu đến) → dùng làm importance proxy
const importance = await LawModel.aggregate([
  { $unwind: "$references" },
  { $group: { _id: "$references.targetId", count: { $sum: 1 } } },
  { $sort: { count: -1 } }
]);
```

**Time:** 2 ngày

#### 5.2 — Frontend: D3.js Force-Directed Graph

```tsx
// components/LawGraph.tsx
// Dùng d3-force để render graph
// - Node size ~ importance (in-degree)
// - Edge color ~ relation type
// - Click node → sidebar hiện chi tiết văn bản
// - Double-click → expand thêm 1 hop
// - Filter panel: loại quan hệ, bộ ngành, năm ban hành
```

Thư viện: `react-force-graph` (wrapper D3 cho React) — nhẹ hơn dùng D3 thô.

**Time:** 3 ngày

**Deliverable W5:** PR backend (graph API) + PR frontend (graph viz page)

---

## Tuần 6: 20/08 – 26/08 — F4: Temporal-Aware QA + Email Notifications

#### 6.1 — F4: Temporal-Aware Retrieval

**Vấn đề:** Chatbot không biết ngày hiệu lực của văn bản — có thể trả lời với luật đã hết hiệu lực.

**Bổ sung metadata ChromaDB:**
```python
# Thêm vào legal_chunker.py metadata:
"effective_date": law_doc.get("dateApproved"),  # đã có trong MongoDB
"expiry_date": None,  # cần thêm field này vào laws schema
"status": law_doc.get("status", "active"),  # active/amended/repealed
```

**Time-filtered retriever:**
```python
class TemporalRetriever:
    def retrieve(self, query: str, as_of: str = None) -> List[Document]:
        filter_dict = {"status": "active"}
        if as_of:
            filter_dict["effective_date"] = {"$lte": as_of}
        return self.vectorstore.similarity_search(query, filter=filter_dict, k=5)
```

**NLP: Detect temporal intent trong câu hỏi:**
```python
TEMPORAL_PATTERNS = {
    "before": r"trước (năm|tháng)\s+(\d{4})",
    "specific": r"(năm|tháng)\s+(\d{4})",
    "current": r"hiện (tại|nay|hành)|đang có hiệu lực",
}
def extract_temporal_constraint(query: str) -> Optional[date]: ...
```

**Time:** 2 ngày

#### 6.2 — Email Notifications

```bash
npm install nodemailer @nestjs-modules/mailer @types/nodemailer
```

```typescript
// datcuong-backend/src/modules/mail/mail.module.ts
// Templates: welcome, password-reset, request-update, law-change-alert
// ENV: SMTP_HOST, SMTP_PORT, SMTP_USER, SMTP_PASS
```

Trigger email khi: đăng ký mới, luật mới được crawl (thông báo lawyers).

**Time:** 2 ngày

#### 6.3 — F6 Bonus: Smart Change Alert

Kết hợp email + AI linking: khi crawl luật mới, check `references` → tìm users có bookmark luật bị ảnh hưởng → gửi notification tóm tắt thay đổi.

**Time:** 1 ngày (nếu kịp)

**Deliverable W6:** PR chatbot (temporal), PR backend (email + smart alert)

---

## Tuần 7: 27/08 – 02/09 — Evaluation Dataset + Admin Polish

### Mục tiêu chính: Xây dựng evaluation dataset cho paper

#### 7.1 — Tạo Evaluation Dataset (quan trọng nhất cho paper)

Xây dựng **Vietnamese Legal QA Benchmark (VLegalQA-50)**:

```
50 câu hỏi pháp luật thực tế, mỗi câu có:
- Câu hỏi tiếng Việt
- Ground-truth: điều khoản nào là đáp án đúng (law_id + article_number)
- Ground-truth answer: câu trả lời đúng

Phân loại:
- 20 câu về giao thông (phạt vi phạm, điều kiện lái xe)
- 15 câu về lao động (lương, hợp đồng, BHXH)  
- 15 câu về dân sự/hình sự (hợp đồng, bồi thường)
```

Dùng dataset này để đo:
- Retrieval precision@3: % có ground-truth article trong top-3
- Answer faithfulness: NLI score giữa answer và cited text
- Citation accuracy: % citations trỏ đúng điều khoản

**Time:** 2 ngày

#### 7.2 — Chạy Experiments

| Configuration | Retrieval P@3 | Citation Acc. |
|--------------|---------------|---------------|
| Baseline: all-MiniLM + char-chunk + no citation | ? | N/A |
| Ours: BGE-M3 + article-chunk + citation | ? | ? |
| Ours + temporal filter | ? | ? |

**Time:** 1 ngày

#### 7.3 — Admin UI Polish

Hoàn thiện các admin page còn stub:
- User management: ban/unban, role assignment
- Law management CRUD trong admin panel
- Request management dashboard

**Time:** 2 ngày

---

## Tuần 8: 03/09 – 09/09 — Paper Draft + Final

#### 8.1 — Paper Draft

**Outline bài báo:**

```
Title: "Structure-Aware Chunking and Citation-Grounded Generation 
        for Vietnamese Legal Question Answering"

1. Introduction
   - Problem: Vietnamese legal QA lacks structured retrieval + citation
   - Contribution: F3 + F2 + F1 + evaluation

2. Related Work
   - RAG for legal QA (CUAD, LexGLUE, etc.)
   - Vietnamese NLP (PhoBERT, VLSP)
   - Citation-grounded generation

3. System Overview
   - 4-service architecture
   - Data pipeline (MongoDB tree structure)

4. Structure-Aware Chunking (F3)
   - Vietnamese legal document hierarchy
   - Article-level chunking algorithm
   - Comparison with character-based chunking

5. Citation-Grounded RAG (F2)
   - Structured JSON prompting
   - Citation validation pipeline
   - Frontend integration

6. Legal Knowledge Graph (F1)
   - Graph construction from AI-extracted relations
   - PageRank importance scoring
   - Visualization use cases

7. Experiments
   - VLegalQA-50 dataset
   - Baseline vs. proposed system
   - Ablation study: each component's contribution

8. Conclusion & Future Work
```

**Target venue:** RIVF 2026 (deadline ~Oct 2026), KSE 2026, hoặc ACIIDS 2027

**Time:** 3 ngày (draft)

#### 8.2 — Final Integration Test

- End-to-end test toàn bộ hệ thống sau tất cả changes
- Docker compose test: tất cả 4 service chạy cùng lúc
- Demo video 5 phút cho GVHD

**Time:** 2 ngày

---

## Timeline Tổng Hợp

```
         Security  Structure   Citation   Graph      Temporal  Eval   Paper
         & Bugs    Chunk+BGE  Generation  KG         + Email   + Admin Draft
W1 ✅ ─────────────────────────────────────────────────────────────────────
W2    ████████
W3              ████████
W4                         ████████
W5                                    ████████
W6                                              ████████
W7                                                        ████████
W8                                                                  ████████
```

---

## Deliverables Cuối Kỳ

| Deliverable | Mô tả |
|-------------|-------|
| 4 PRs merged | bcrypt, structure-chunker, citation, graph |
| VLegalQA-50 | Dataset 50 câu hỏi + ground truth (có thể public) |
| Paper draft | 8-10 trang, RIVF/KSE format |
| Demo video | 5 phút demo full system |
| Báo cáo LVTN | Chương mới: contributions + experiments |

---

## Ma Trận Rủi Ro

| Rủi ro | Khả năng | Fallback |
|--------|----------|----------|
| BGE-M3 không đủ RAM local | Trung bình | `paraphrase-multilingual-mpnet-base-v2` (HuggingFace, nhẹ hơn) |
| LLM (Llama3) không follow JSON instruction | Cao | Parse với regex fallback; thử llama3.1 |
| D3.js graph quá phức tạp | Trung bình | Dùng `react-force-graph` library thay raw D3 |
| 50 câu hỏi dataset không đủ | Thấp | Giảm xuống 30 câu với 3 domains |
| Paper không kịp RIVF 2026 | Trung bình | Target KSE 2026 (deadline thường tháng 7-8 năm sau) |
| Merge PR bị block (không có write permission) | Thấp | Dùng fork, demo từ personal fork |
