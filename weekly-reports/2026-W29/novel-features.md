# Đề Xuất Tính Năng Mới — Hướng Nghiên Cứu LVTN

**Mục tiêu tài liệu:** Liệt kê các tính năng mới, có giá trị nghiên cứu, phù hợp với LVTN 1 người 2 tháng,  
có thể viết thành báo cáo khoa học / hội nghị (RIVF, KSE, ACIIDS, v.v.)

---

## Tổng Quan — Ma Trận Lựa Chọn

| # | Tên Feature | Độ mới (1-5) | Khả năng làm (2 tháng) | Tiềm năng paper | Ưu tiên |
|---|-------------|--------------|------------------------|-----------------|---------|
| F1 | Legal Knowledge Graph + Visual | ⭐⭐⭐⭐ | ✅ Cao | ✅ Mạnh | 🔴 Làm |
| F2 | Grounded Citation Generation | ⭐⭐⭐⭐⭐ | ✅ Cao | ✅ Rất mạnh | 🔴 Làm |
| F3 | Structure-Aware Legal Chunking | ⭐⭐⭐⭐ | ✅ Cao | ✅ Mạnh | 🔴 Làm |
| F4 | Temporal-Aware Legal QA | ⭐⭐⭐⭐⭐ | 🟡 Trung bình | ✅ Rất mạnh | 🟡 Cân nhắc |
| F5 | Legal Contradiction Detection | ⭐⭐⭐⭐⭐ | 🔴 Thấp | ✅ Rất mạnh | 🟠 Tương lai |
| F6 | Smart Legal Change Alerts | ⭐⭐⭐ | ✅ Cao | 🟡 Trung bình | 🟢 Bonus |

---

## F1: Legal Knowledge Graph + Visualization (Đồ Thị Tri Thức Pháp Luật)

### Vấn đề hiện tại

AI linking pipeline (PhoBERT) đã trích xuất được các quan hệ giữa các văn bản (sửa đổi, bãi bỏ, hướng dẫn, viện dẫn) và lưu vào MongoDB. **Nhưng dữ liệu này bị "chôn vùi"** — người dùng không thể thấy bức tranh tổng thể: một luật có bao nhiêu nghị định hướng dẫn? Một nghị định đã bị sửa đổi bao nhiêu lần? Luật nào "trung tâm" nhất trong corpus?

### Tính mới trong industry

Các hệ thống pháp luật hiện có (thuvienphapluat.vn, vbpl.vn) chỉ hiển thị danh sách văn bản liên quan dạng text. **Chưa có hệ thống nào tại Việt Nam** xây dựng đồ thị tri thức pháp luật traversable với visual exploration.

Quốc tế: EUR-Lex (EU) có graph nhưng tiếng Anh, không áp dụng cấu trúc pháp luật Việt Nam.

### Thiết kế kỹ thuật

```
MongoDB (laws collection) 
    │
    ▼ extract edges từ ai_linking_result
Graph Model:
    Node: { law_id, title, type, issueDate, ministry }
    Edge: { source, target, relation_type, confidence }
    
    relation_type: [amends, repeals, guides, references, supplements]
    
    ▼ Store graph
Neo4j (hoặc MongoDB với adjacency list) 
    
    ▼ Serve via API
GET /api/law-graph/:law_id?depth=2
    → { nodes: [...], edges: [...] }
    
    ▼ Visualize
Frontend: D3.js Force-Directed Graph hoặc Cytoscape.js
    - Node size = số luật tham chiếu (in-degree)
    - Edge color = loại quan hệ
    - Click node → mở modal xem văn bản
    - Filter theo: ngày ban hành, bộ ngành, loại quan hệ
```

### Tính năng phân tích graph

```python
# Network analysis metrics
- PageRank → luật "quan trọng" nhất (nhiều luật khác tham chiếu)
- Weakly connected components → nhóm luật liên quan nhau
- In-degree centrality → "hub laws" (Bộ luật Dân sự, Hình sự, ...)
- Dead-end detection → văn bản bị bãi bỏ nhưng vẫn được tham chiếu
```

### Hướng viết paper

> **Tiêu đề gợi ý:** "Construction and Visualization of Vietnamese Legal Knowledge Graph from NLP-Extracted Relations"
>
> **Contribution:**
> 1. Pipeline tự động xây dựng knowledge graph từ PhoBERT-extracted relations
> 2. Graph analysis metrics cho Vietnamese legal corpus  
> 3. Interactive visualization UI với real-world use cases
>
> **Venue:** RIVF 2026, KSE 2026, hoặc ACIIDS 2027

### Estimated implementation

- Backend: 3 ngày (graph storage, API, PageRank)
- Frontend: 3 ngày (D3.js graph component)
- Analysis: 1 ngày
- **Total: 1 tuần**

---

## F2: Grounded Citation Generation — Chatbot Trích Dẫn Điều Khoản Cụ Thể

### Vấn đề hiện tại

Chatbot hiện tại trả lời câu hỏi pháp luật nhưng **không biết đang trích dẫn điều nào, khoản nào**. Người dùng không thể verify câu trả lời — "AI nói vậy nhưng có đúng không?". Đây là vấn đề nghiêm trọng trong domain pháp luật nơi **độ tin cậy là tối quan trọng**.

Ví dụ hiện tại:
```
User: Mức phạt vi phạm hành chính về giao thông khi vượt đèn đỏ là bao nhiêu?
Bot: Theo quy định hiện hành, mức phạt là từ 4 đến 6 triệu đồng...
     (Không biết đang nói về nghị định nào, điều nào)
```

Sau khi implement F2:
```
User: Mức phạt vi phạm hành chính về giao thông khi vượt đèn đỏ là bao nhiêu?
Bot: Theo **Điều 6, Khoản 5, Điểm a, Nghị định 100/2019/NĐ-CP** (sửa đổi bổ sung 
     bởi Nghị định 123/2021/NĐ-CP):
     
     > "Phạt tiền từ 4.000.000 đồng đến 6.000.000 đồng đối với người điều khiển 
     > xe ô tô và các loại xe tương tự xe ô tô vi phạm... không chấp hành hiệu 
     > lệnh của đèn tín hiệu giao thông."
     
     📌 [Xem Điều 6 – Nghị định 100/2019] [Xem bản gốc PDF]
```

### Tính mới trong industry

**"Grounded Generation"** là một trong những hướng nghiên cứu nóng nhất trong NLP 2024-2025 (RAG faithfulness, attribution). Áp dụng vào **pháp luật tiếng Việt là hoàn toàn mới** — chưa có paper tiếng Việt nào về citation-grounded legal QA.

### Thiết kế kỹ thuật

**Bước 1: Structure-Aware Chunking (kết hợp với F3)**
```python
# Chunk theo cấu trúc pháp lý thay vì character count
class LegalChunker:
    def chunk(self, law_text: str, law_id: str) -> List[Chunk]:
        # Parse: Chương → Mục → Điều → Khoản → Điểm
        # Mỗi chunk = 1 Điều (Article), với metadata:
        return Chunk(
            text=article_text,
            metadata={
                "law_id": law_id,
                "article_number": "Điều 6",
                "chapter": "Chương II",
                "law_title": "Nghị định 100/2019/NĐ-CP",
                "effective_date": "2020-01-01"
            }
        )
```

**Bước 2: Structured Output từ LLM**
```python
# Thay vì để LLM trả lời tự do, force output JSON
CITATION_PROMPT = """
Dựa vào các văn bản pháp luật sau, trả lời câu hỏi.
Bắt buộc output JSON format:
{
  "answer": "câu trả lời tổng hợp",
  "citations": [
    {
      "law_id": "NĐ-100-2019",
      "article": "Điều 6",
      "clause": "Khoản 5",
      "point": "Điểm a",
      "quote": "đoạn trích nguyên văn...",
      "relevance_score": 0.95
    }
  ],
  "confidence": "high|medium|low"
}
"""
```

**Bước 3: Frontend Highlight**
```tsx
// Component hiển thị câu trả lời có citation
<LegalAnswer>
  <AnswerText>{answer}</AnswerText>
  <CitationList>
    {citations.map(c => (
      <CitationCard 
        onClick={() => openLawModal(c.law_id, c.article)}
        highlight={c.quote}
      />
    ))}
  </CitationList>
</LegalAnswer>
```

### Metric đánh giá

```
1. Citation Precision: % trích dẫn trỏ đúng điều khoản thực sự tồn tại
2. Answer Faithfulness: câu trả lời có nhất quán với quote không (dùng NLI)
3. Citation Recall: % câu hỏi có ít nhất 1 citation được trả về
```

### Hướng viết paper

> **Tiêu đề gợi ý:** "CiteLaw-Vi: Citation-Grounded Question Answering for Vietnamese Legal Documents"
>
> **Contribution:**
> 1. Structure-aware chunking pipeline cho văn bản pháp luật Việt Nam
> 2. Citation-grounded generation với structured JSON output
> 3. Faithfulness evaluation framework cho Vietnamese legal QA
> 4. Dataset: 200+ câu hỏi/đáp pháp luật với gold-standard citations
>
> **Venue:** ACL-SRW 2026, EMNLP Findings, hoặc RIVF 2026

### Estimated implementation

- Chunker: 2 ngày
- LLM prompting + JSON output: 2 ngày
- Frontend citation UI: 2 ngày
- Evaluation + dataset: 2 ngày
- **Total: ~1.5 tuần**

---

## F3: Structure-Aware Legal Document Chunking

### Vấn đề hiện tại

ChromaDB hiện chunk văn bản theo character count (mặc định LangChain ~1000 chars). **Văn bản pháp luật có cấu trúc rất đặc thù:**

```
Chương I — Quy định chung
  Điều 1. Phạm vi điều chỉnh
    Khoản 1. Nghị định này quy định...
    Khoản 2. Nghị định này không áp dụng...
  Điều 2. Đối tượng áp dụng
    Khoản 1. ...
Chương II — Các hành vi vi phạm
  Điều 3. ...
```

Khi chunk theo character count, một chunk có thể cắt giữa Khoản 1 và Khoản 2 của cùng một Điều, **phá vỡ ngữ nghĩa**.

### Thiết kế kỹ thuật

```python
import re

DIEU_PATTERN = r"Điều\s+\d+[a-z]?\."
KHOAN_PATTERN = r"^\d+\.\s"
DIEM_PATTERN = r"^[a-z]\)\s"

class VietnameseLegalChunker:
    """
    Chunk theo Điều (Article level).
    Mỗi Điều = 1 chunk độc lập.
    Metadata: law_id, article_num, chapter, section
    """
    
    def parse_law(self, text: str) -> List[Article]:
        articles = re.split(DIEU_PATTERN, text)
        return [
            Article(
                number=self._extract_number(articles[i]),
                text=articles[i+1],
                chapter=self._find_chapter(i),
                clauses=self._parse_clauses(articles[i+1])
            )
            for i in range(len(articles))
        ]
    
    def chunk(self, articles: List[Article]) -> List[Document]:
        chunks = []
        for article in articles:
            # Nếu Điều quá dài (>2000 tokens), tách theo Khoản
            if len(article.text) > 2000:
                for clause in article.clauses:
                    chunks.append(Document(
                        page_content=clause.text,
                        metadata={
                            "article": article.number,
                            "clause": clause.number,
                            "law_id": self.law_id,
                            "granularity": "clause"
                        }
                    ))
            else:
                chunks.append(Document(
                    page_content=article.text,
                    metadata={
                        "article": article.number,
                        "law_id": self.law_id,
                        "granularity": "article"
                    }
                ))
        return chunks
```

### Tại sao novel

Existing chunking strategies trong LangChain (RecursiveCharacterTextSplitter, TokenTextSplitter) không hiểu cấu trúc pháp lý Việt Nam. **Đây là domain-specific contribution** có thể reuse cho bất kỳ hệ thống pháp luật Việt Nam nào.

### Hướng viết paper

Kết hợp F2 + F3 → một paper mạnh hơn về **"Structure-Aware Grounded RAG for Vietnamese Legal QA"**.

### Estimated implementation

- **Total: 2 ngày** (pure Python, không cần ML)

---

## F4: Temporal-Aware Legal QA (Hỏi Đáp Theo Thời Gian)

### Vấn đề hiện tại

Pháp luật Việt Nam **thay đổi rất thường xuyên**. Ví dụ, Nghị định 100/2019/NĐ-CP về xử phạt giao thông đã được sửa đổi bởi Nghị định 123/2021/NĐ-CP. Chatbot hiện tại không biết điều này — nó có thể trả lời với thông tin cũ.

**Câu hỏi có temporal context:**
- "Mức phạt uống rượu lái xe **trước năm 2021** là bao nhiêu?"
- "Luật **hiện hành** về lao động tối thiểu quy định gì?"
- "Nghị định nào **đang có hiệu lực** quy định về BHXH?"

### Thiết kế kỹ thuật

```python
# Schema mở rộng cho law document
{
    "law_id": "ND-100-2019",
    "title": "...",
    "effectiveDate": "2020-01-01",      # ngày có hiệu lực
    "expiryDate": None,                   # None = đang hiệu lực
    "amendedBy": ["ND-123-2021"],        # bị sửa đổi bởi
    "replacedBy": None,                   # bị thay thế bởi
    "status": "active|amended|repealed"
}

# Temporal-aware retriever
class TemporalRetriever:
    def retrieve(self, query: str, as_of_date: date = None) -> List[Document]:
        if as_of_date is None:
            as_of_date = date.today()
        
        # Filter ChromaDB: chỉ lấy docs có hiệu lực tại thời điểm as_of_date
        return self.vectorstore.similarity_search(
            query,
            filter={
                "effectiveDate": {"$lte": as_of_date.isoformat()},
                "$or": [
                    {"expiryDate": None},
                    {"expiryDate": {"$gte": as_of_date.isoformat()}}
                ]
            }
        )
```

**NLP Temporal Expression Extraction:**
```python
# Extract temporal intent từ câu hỏi
temporal_patterns = {
    "current": r"hiện (tại|nay|hành)|đang (có hiệu lực|áp dụng)",
    "before": r"trước (năm|tháng|ngày)\s+(\d+)",
    "after": r"sau (năm|tháng|ngày)\s+(\d+)",
    "specific": r"(năm|tháng)\s+(\d{4}|\d{1,2}/\d{4})"
}
```

### Tính mới trong industry

**Temporal reasoning trong legal QA là open research problem.** Không có hệ thống Việt Nam nào làm điều này. Quốc tế có một số paper về temporal legal reasoning nhưng không có tiếng Việt.

### Estimated implementation

- Schema extension + migration: 1 ngày
- Temporal retriever: 2 ngày
- NLP temporal extraction: 2 ngày
- **Total: ~1 tuần**

### Hướng viết paper

> **Tiêu đề gợi ý:** "Temporal-Aware Retrieval-Augmented Generation for Vietnamese Legal Question Answering"
>
> **Contribution:**
> 1. Temporal legal document schema cho VBPL Việt Nam
> 2. Time-conditioned retrieval pipeline
> 3. Temporal expression extraction cho tiếng Việt
> 4. Benchmark dataset: time-sensitive Vietnamese legal questions

---

## F5: Legal Contradiction Detection (Phát Hiện Mâu Thuẫn Văn Bản Pháp Luật)

### Tính mới và tầm quan trọng

Đây là **vấn đề thực tế nghiêm trọng** trong hệ thống pháp luật Việt Nam: nhiều nghị định hướng dẫn nhiều luật khác nhau, dẫn đến mâu thuẫn. Hiện không có công cụ tự động nào phát hiện điều này.

### Thiết kế

```python
# Dùng NLI (Natural Language Inference) để so sánh 2 điều khoản
# Labels: ENTAILMENT | NEUTRAL | CONTRADICTION

from transformers import pipeline

nli_pipeline = pipeline(
    "zero-shot-classification",
    model="joeddav/xlm-roberta-large-xnli"  # multilingual NLI
)

def check_contradiction(clause_a: str, clause_b: str) -> dict:
    result = nli_pipeline(
        sequences=clause_a,
        candidate_labels=["entailment", "neutral", "contradiction"],
        hypothesis_template="Điều khoản này: " + clause_b
    )
    return {
        "relation": result["labels"][0],
        "confidence": result["scores"][0]
    }
```

### Estimated implementation

- NLI model integration: 2 ngày
- Comparison pipeline cho related laws: 2 ngày
- Frontend: show flagged contradictions: 1 ngày
- **Total: ~1 tuần** (nhưng accuracy cần validate kỹ)

### Lưu ý

Độ phức tạp cao hơn F1-F3. Khuyến nghị làm sau khi F1, F2, F3 hoàn thành.

---

## F6: Smart Legal Change Alert (Cảnh Báo Thay Đổi Pháp Luật Cá Nhân Hóa)

### Mô tả

Khi crawler lấy về văn bản mới, hệ thống tự động:
1. Phân tích văn bản mới sửa đổi / bãi bỏ luật nào
2. Tìm users đã bookmark / xem luật đó
3. Gửi notification: "Văn bản bạn quan tâm (Nghị định 100/2019) vừa được sửa đổi bởi Nghị định 56/2026. Thay đổi chính: [tóm tắt tự động]"

### Kỹ thuật

```python
# Trigger: sau khi crawler save new law
async def on_new_law_crawled(new_law: Law):
    # 1. Dùng AI linking để tìm luật bị ảnh hưởng
    affected_laws = await ai_linking_service.find_affected(new_law)
    
    # 2. Tìm users quan tâm
    for affected_law in affected_laws:
        interested_users = await user_service.find_by_bookmarked_law(affected_law.id)
        
        # 3. Generate summary of changes
        change_summary = await llm.summarize_changes(
            old_law=affected_law, 
            new_law=new_law
        )
        
        # 4. Send notifications
        for user in interested_users:
            await notification_service.send(user, change_summary)
            await email_service.send(user, change_summary)
```

### Estimated implementation

- **Total: 2 ngày** (nếu email đã implement)

---

## Đề Xuất Roadmap Mới (Tích Hợp Novel Features)

```
W1  [16-22/7]  ✅ Phân tích + README + Reports
W2  [23-29/7]  🔴 Security (bcrypt) + Bugs (pdf, dotenv, chromadb script)
W3  [30/7-5/8] 🔵 F3: Structure-Aware Chunking + Vietnamese Embeddings (BGE-M3)
W4  [6-12/8]   🔵 F2: Grounded Citation Generation (LLM + Frontend)
W5  [13-19/8]  🔵 F1: Legal Knowledge Graph (backend graph + D3 viz)
W6  [20-26/8]  🔵 F4: Temporal-Aware QA (schema + retriever)
W7  [27/8-2/9] 🟢 Email notifications + F6: Smart Alerts (nếu kịp)
W8  [3-9/9]    🟢 Testing + Evaluation Dataset + Final Report
```

### Paper Outline Gợi Ý

> **Tiêu đề:** "Enhancing Vietnamese Legal Information Retrieval: Structure-Aware Chunking, Citation-Grounded Generation, and Knowledge Graph Visualization"
>
> **Abstract:** We extend an existing Vietnamese legal document system with three novel components: (1) a domain-specific chunker that respects Vietnamese legal document hierarchy (Chương/Điều/Khoản), (2) a citation-grounded RAG pipeline that forces the LLM to output structured citations for verification, and (3) an interactive knowledge graph derived from NLP-extracted cross-document relations. Experiments on a 500-document Vietnamese legal corpus show improvements in answer faithfulness (+X%), retrieval precision (+Y%), and user trust scores (+Z%).
>
> **Sections:** Introduction → Related Work (RAG, Legal NLP, KG) → System Overview → F3 Chunker → F2 Grounded Generation → F1 Knowledge Graph → Experiments → Conclusion
>
> **Target venues:** RIVF 2026 (Hội nghị Quốc gia KH&CN Thông tin - Truyền thông), KSE 2026, hoặc PACLIC 2026
