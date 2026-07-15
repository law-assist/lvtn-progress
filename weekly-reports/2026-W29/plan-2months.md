# Kế Hoạch 3 Tháng — Phát Triển Độc Lập

**Sinh viên:** Nguyễn Nhật Trần  
**Thời gian:** 16/07/2026 – 07/10/2026 (12 tuần)

---

## Timeline Tổng Quan

```
W1  ✅ Phân tích + README + Reports
W2     G1 bcrypt + G9 build script + G10 dotenv + G7 PDF fix
W3     F3 Article-level chunking + G2 BGE-M3 embedding
W4     F2 Citation (AI + BE)
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

**BE:** `datcuong-backend/src/modules/user/user.service.ts`

```bash
npm install bcrypt @types/bcrypt
```

```typescript
// Xoá:
import * as crypto from 'crypto';
const hash = crypto.createHash('sha256').update(password).digest('hex');
const isMatch = hash === storedHash;

// Thêm:
import * as bcrypt from 'bcrypt';
const hash = await bcrypt.hash(password, 12);
const isMatch = await bcrypt.compare(password, storedHash);
```

Vì SHA-256 một chiều, không migrate được → thêm field `requirePasswordReset: boolean` vào User schema, set `true` cho toàn bộ user cũ, force đổi mật khẩu khi login.

**FE:** `datcuong-frontend/src/` — check response `requirePasswordReset: true` → redirect sang trang `/reset-password`.

> Ref: https://docs.nestjs.com/security/encryption-and-hashing

---

### G9 — ChromaDB Build Script

**AI:** tạo mới `datcuong-ai-chatbot/scripts/build_vectorstore.py`

```python
import sys, os
sys.path.append(os.path.join(os.path.dirname(__file__), '..'))

from utils.mongo_handler import get_legislation_by_query
from utils.data_processing import extract_article_chunks   # viết ở W3
from langchain_chroma import Chroma
from langchain_ollama import OllamaEmbeddings

def build():
    laws = get_legislation_by_query({})["data"]
    docs = []
    for law in laws:
        docs.extend(extract_article_chunks(law))

    embeddings = OllamaEmbeddings(model="bge-m3")
    store = Chroma(
        persist_directory="./database/bge-m3/article-level",
        collection_name="legislation",
        embedding_function=embeddings,
    )
    store.add_documents(docs)
    print(f"Built {len(docs)} chunks from {len(laws)} laws")

if __name__ == "__main__":
    build()
```

> Ref: https://python.langchain.com/docs/integrations/vectorstores/chroma

---

### G10 — dotenv path

**AI:** `datcuong-ai-law-linking/src/services/*.py`

```python
# Xoá:
load_dotenv(dotenv_path="../.env")

# Thêm:
from pathlib import Path
load_dotenv(Path(__file__).parent.parent.parent / ".env")
```

> Ref: https://saurabh-kumar.com/python-dotenv

---

### G7 — PDF Export fix

**FE:** `datcuong-frontend/src/`

```bash
npm install @react-pdf/renderer
```

Thay component dùng `jsPDF + html2canvas` bằng `@react-pdf/renderer` — render text thực, không chụp ảnh DOM, hỗ trợ font tiếng Việt đúng cách.

```tsx
import { Document, Page, Text, StyleSheet } from '@react-pdf/renderer';

export const LawPdfDocument = ({ law }) => (
  <Document>
    <Page style={styles.page}>
      <Text style={styles.title}>{law.name}</Text>
      <Text>{law.content_flat}</Text>
    </Page>
  </Document>
);
```

> Ref: https://react-pdf.org/components

---

## Tuần 3: 30/07–05/08 — F3: Article-Level Chunking + BGE-M3

### F3 — Chunk theo Điều

**AI:** `datcuong-ai-chatbot/utils/data_processing.py`

```python
from langchain_core.documents import Document

def extract_article_chunks(law_doc: dict) -> list[Document]:
    """
    Duyệt cây MongoDB (type: phan/chuong/muc/dieu/khoan/diem).
    Mỗi node type='dieu' → 1 Document riêng.
    """
    chunks = []
    ctx = {"chapter": "", "section": ""}

    def flatten(node: dict) -> str:
        parts = [node.get("value", "")]
        for child in node.get("children", []):
            parts.append(flatten(child))
        return " ".join(filter(None, parts))

    def traverse(node: dict):
        t = node.get("type", "")
        if t == "chuong":
            ctx["chapter"] = node.get("value", "")[:80]
        elif t == "muc":
            ctx["section"] = node.get("value", "")[:80]
        elif t == "dieu":
            chunks.append(Document(
                page_content=flatten(node),
                metadata={
                    "law_id":         str(law_doc.get("_id", "")),
                    "law_title":      law_doc.get("name", ""),
                    "numberDoc":      law_doc.get("numberDoc", ""),
                    "department":     law_doc.get("department", ""),
                    "article_header": node.get("value", "")[:120],
                    "chapter":        ctx["chapter"],
                    "section":        ctx["section"],
                    "effective_date": str(law_doc.get("dateApproved", "")),
                    "status":         law_doc.get("status", "active"),
                }
            ))
            return   # không duyệt sâu hơn vào Khoản/Điểm
        for child in node.get("children", []):
            traverse(child)

    traverse(law_doc.get("content", {}))
    return chunks
```

> Ref: https://python.langchain.com/docs/concepts/document_loaders

---

### G2 — Đổi Embedding sang BGE-M3

**AI:** `datcuong-ai-chatbot/api/api.py`

```python
# Xoá:
from langchain_huggingface import HuggingFaceEmbeddings
embeddings = HuggingFaceEmbeddings(model_name="all-MiniLM-L6-v2")
PERSIST_DIR = "./database/all-MiniLM-L6-v2/3000_300"

# Thêm:
from langchain_ollama import OllamaEmbeddings
embeddings = OllamaEmbeddings(model="bge-m3")   # ollama pull bge-m3
PERSIST_DIR = "./database/bge-m3/article-level"
```

Sau khi đổi: chạy `python scripts/build_vectorstore.py` để rebuild ChromaDB.

> Ref: https://ollama.com/library/bge-m3 | https://python.langchain.com/docs/integrations/text_embedding/ollama

**Đánh giá W3:** test 20 câu hỏi, ghi kết quả P@3 vào `deliverables/eval-w3.md`.

---

## Tuần 4–5: 06–19/08 — F2: Grounded Citation Generation

### F2 — AI (W4)

**AI:** `datcuong-ai-chatbot/api/api.py` + node `generate`

**Bước 1 — Schema:**
```python
from pydantic import BaseModel

class Citation(BaseModel):
    law_id: str
    numberDoc: str
    article_header: str   # "Điều 6. Xử phạt người điều khiển xe mô tô..."
    relevant_text: str    # đoạn trích nguyên văn

class CitationResponse(BaseModel):
    answer: str
    citations: list[Citation]
    confidence: str       # "high" | "medium" | "low"
```

**Bước 2 — Structured output:**
```python
from langchain_ollama import ChatOllama
from langchain_core.prompts import ChatPromptTemplate

llm = ChatOllama(model="llama3", temperature=0)
structured_llm = llm.with_structured_output(CitationResponse)

PROMPT = ChatPromptTemplate.from_template("""
Bạn là luật sư. Trả lời câu hỏi dựa vào các điều khoản bên dưới.
Với mỗi điểm trong câu trả lời, cite đúng điều khoản tương ứng.

Điều khoản:
{context}

Câu hỏi: {question}
""")

chain = PROMPT | structured_llm
result: CitationResponse = chain.invoke({"context": ctx_str, "question": q})
```

**Bước 3 — Cập nhật node `generate` trong LangGraph** trả về `CitationResponse` thay vì string thuần.

**Bước 4 — DTO mới** `datcuong-ai-chatbot/utils/dto.py`:
```python
class QuestionAnswerResponse(BaseModel):
    answer: str
    citations: list[Citation]
    confidence: str
```

> Ref: https://python.langchain.com/docs/how_to/structured_output | https://python.langchain.com/docs/concepts/chat_models/#structured-output

---

### F2 — Backend (W4)

**BE:** `datcuong-backend/src/modules/law/law.controller.ts`

```typescript
// GET /api/laws/:id/article?header=Điều+6
@Get(':id/article')
async getArticle(
  @Param('id') id: string,
  @Query('header') header: string,
) {
  return this.lawService.findArticleByHeader(id, header);
}
```

**BE:** `datcuong-backend/src/modules/law/law.service.ts`

```typescript
async findArticleByHeader(lawId: string, header: string) {
  const law = await this.lawModel.findById(lawId).select('content name numberDoc');
  return this.searchTreeByHeader(law.content, header);
}

private searchTreeByHeader(node: any, header: string): any {
  if (node?.value?.startsWith(header.slice(0, 20))) return node;
  for (const child of node?.children ?? []) {
    const found = this.searchTreeByHeader(child, header);
    if (found) return found;
  }
  return null;
}
```

> Ref: https://docs.nestjs.com/controllers

---

### F2 — Frontend (W5)

**FE:** tạo `datcuong-frontend/src/components/CitationCard.tsx`

```tsx
interface CitationCardProps {
  numberDoc: string;
  articleHeader: string;
  relevantText: string;
  lawId: string;
}

export function CitationCard({ numberDoc, articleHeader, relevantText, lawId }: CitationCardProps) {
  return (
    <div className="border-l-4 border-blue-400 pl-3 my-2 bg-blue-50 rounded-r">
      <span className="text-xs font-bold text-blue-700">{numberDoc}</span>
      <p className="text-sm font-medium mt-1">{articleHeader}</p>
      <blockquote className="text-xs text-gray-600 italic mt-1">
        "{relevantText}"
      </blockquote>
      <button
        className="text-xs text-blue-500 underline mt-1"
        onClick={() => openLawModal(lawId, articleHeader)}
      >
        Xem văn bản gốc →
      </button>
    </div>
  );
}
```

**FE:** `datcuong-frontend/src/app/(main)/.../chatbot/` — cập nhật type response, render `<CitationCard />` bên dưới mỗi tin nhắn của bot.

> Ref: https://ant.design/components/card | https://swr.vercel.app

---

## Tuần 6–7: 20/08–02/09 — F1: Legal Knowledge Graph

### F1 — Backend (W6)

**BE:** `datcuong-backend/src/modules/law/law.service.ts`

```typescript
async buildSubgraph(rootId: string, depth: number) {
  const nodes = new Map<string, object>();
  const edges: object[] = [];

  const traverse = async (id: string, d: number) => {
    if (d === 0 || nodes.has(id)) return;
    const law = await this.lawModel
      .findById(id)
      .select('name numberDoc type department references');
    if (!law) return;

    nodes.set(id, {
      id, title: law.name, numberDoc: law.numberDoc,
      type: law.type, department: law.department,
    });
    for (const ref of law.references ?? []) {
      edges.push({
        source: id, target: String(ref.targetId),
        relationType: ref.relationType, confidence: ref.confidence,
      });
      await traverse(String(ref.targetId), d - 1);
    }
  };

  await traverse(rootId, depth);
  return { nodes: [...nodes.values()], edges };
}

async getTopLawsByInDegree(limit: number) {
  return this.lawModel.aggregate([
    { $unwind: { path: '$references', preserveNullAndEmptyArrays: false } },
    { $group: { _id: '$references.targetId', inDegree: { $sum: 1 } } },
    { $sort: { inDegree: -1 } },
    { $limit: limit },
    { $lookup: { from: 'laws', localField: '_id', foreignField: '_id', as: 'law' } },
    { $unwind: '$law' },
    { $project: { title: '$law.name', numberDoc: '$law.numberDoc', inDegree: 1 } },
  ]);
}
```

**BE:** `datcuong-backend/src/modules/law/law.controller.ts`

```typescript
@Get('graph/stats/top')
getTopLaws() { return this.lawService.getTopLawsByInDegree(10); }

@Get('graph/:id')
getLawGraph(@Param('id') id: string, @Query('depth') depth = '2') {
  return this.lawService.buildSubgraph(id, Number(depth));
}
```

> Ref: https://www.mongodb.com/docs/manual/reference/operator/aggregation/group | https://docs.nestjs.com/techniques/mongodb

---

### F1 — Frontend (W7)

```bash
npm install react-force-graph
```

**FE:** tạo `datcuong-frontend/src/app/(main)/(public)/law-graph/page.tsx`

```tsx
'use client';
import dynamic from 'next/dynamic';
const ForceGraph2D = dynamic(
  () => import('react-force-graph').then(m => m.ForceGraph2D),
  { ssr: false }
);

const EDGE_COLORS = {
  amends: '#f59e0b', repeals: '#ef4444',
  guides: '#3b82f6', references: '#9ca3af',
};

export default function LawGraphPage() {
  const [graphData, setGraphData] = useState({ nodes: [], links: [] });
  const [selected, setSelected] = useState(null);

  useEffect(() => {
    fetch('/api/laws/graph/stats/top')
      .then(r => r.json())
      .then(top => {
        // fetch subgraph cho từng node top, merge
        // transform edges → links (react-force-graph dùng "links")
      });
  }, []);

  return (
    <div className="flex h-screen">
      <ForceGraph2D
        graphData={graphData}
        nodeLabel="title"
        nodeVal={n => Math.log((n.inDegree ?? 1) + 1) * 4}
        linkColor={l => EDGE_COLORS[l.relationType] ?? '#9ca3af'}
        onNodeClick={n => setSelected(n)}
      />
      {selected && (
        <aside className="w-80 p-4 border-l bg-white overflow-y-auto">
          <h2 className="font-bold">{selected.title}</h2>
          <p className="text-sm text-gray-500">{selected.numberDoc}</p>
          <button onClick={() => router.push(`/laws/${selected.id}`)}>
            Xem văn bản →
          </button>
        </aside>
      )}
    </div>
  );
}
```

> Ref: https://github.com/vasturiano/react-force-graph | https://nextjs.org/docs/app/building-your-application/optimizing/lazy-loading

---

## Tuần 8: 03–09/09 — F4: Temporal-Aware QA

### F4 — Backend

**BE:** `datcuong-backend/src/modules/law/schemas/law.schema.ts` — thêm 2 field:

```typescript
expiryDate: { type: Date, default: null },
status: { type: String, enum: ['active', 'amended', 'repealed'], default: 'active' },
```

Crawler cập nhật `status` khi AI linking ghi `relationType === 'repeals'` → set `status = 'repealed'` cho văn bản bị bãi bỏ.

> Ref: https://mongoosejs.com/docs/schematypes.html

---

### F4 — AI

**AI:** tạo `datcuong-ai-chatbot/utils/temporal_extractor.py`

```python
import re
from datetime import date

PATTERNS = [
    (r"năm (\d{4})",                lambda m: date(int(m.group(1)), 12, 31)),
    (r"trước (?:năm )?(\d{4})",     lambda m: date(int(m.group(1)) - 1, 12, 31)),
    (r"tháng (\d{1,2})[/\-](\d{4})",lambda m: date(int(m.group(2)), int(m.group(1)), 28)),
    (r"hiện tại|hiện nay|hiện hành|đang có hiệu lực", lambda m: date.today()),
]

def extract_as_of_date(query: str) -> date | None:
    for pattern, fn in PATTERNS:
        m = re.search(pattern, query, re.IGNORECASE)
        if m:
            return fn(m)
    return None  # None → dùng today, chỉ lấy active docs
```

**AI:** `datcuong-ai-chatbot/api/api.py` — node `retrieve`:

```python
from utils.temporal_extractor import extract_as_of_date

def retrieve(state: GraphState) -> GraphState:
    query = state["structured_query"]
    as_of = extract_as_of_date(state["raw_question"]) or date.today()

    where = {
        "$and": [
            {"effective_date": {"$lte": as_of.isoformat()}},
            {"$or": [{"status": "active"}, {"status": "amended"}]},
        ]
    }
    docs = vectorstore.similarity_search(query, k=5, filter=where)
    return {**state, "context": docs, "as_of_date": as_of.isoformat()}
```

> Ref: https://docs.trychroma.com/guides#filtering-by-metadata | https://docs.python.org/3/library/re.html

---

### F4 — Frontend

**FE:** `datcuong-frontend/src/components/CitationCard.tsx` — thêm status badge:

```tsx
<span className={`text-xs px-2 py-0.5 rounded-full ${
  law.status === 'active' ? 'bg-green-100 text-green-700' : 'bg-red-100 text-red-700'
}`}>
  {law.status === 'active' ? '✓ Đang hiệu lực' : '⚠ Hết hiệu lực'}
</span>
```

**FE:** chatbot area — hiển thị note nhỏ bên dưới nếu có temporal filter:
```
📅 Trả lời theo văn bản có hiệu lực tại: 31/12/2018
```

---

## Tuần 9: 10–16/09 — Email + Router + Reranker

### G5 — Email

**BE:**
```bash
npm install @nestjs-modules/mailer nodemailer handlebars
```

Tạo `datcuong-backend/src/modules/mail/mail.module.ts`:

```typescript
MailerModule.forRootAsync({
  useFactory: (config: ConfigService) => ({
    transport: {
      host: config.get('SMTP_HOST'),
      port: Number(config.get('SMTP_PORT')),
      secure: false,
      auth: { user: config.get('SMTP_USER'), pass: config.get('SMTP_PASS') },
    },
    defaults: { from: '"Law Assistant" <noreply@lawassist.vn>' },
    template: {
      dir: join(__dirname, 'templates'),
      adapter: new HandlebarsAdapter(),
    },
  }),
  inject: [ConfigService],
})
```

Templates cần có: `welcome.hbs`, `password-reset.hbs`, `request-update.hbs`

Trigger trong `user.service.ts` sau `save()`:
```typescript
await this.mailService.sendMail({
  to: user.email,
  subject: 'Chào mừng đến Law Assistant',
  template: 'welcome',
  context: { name: user.name },
});
```

> Ref: https://nest-modules.github.io/mailer | https://nodemailer.com/about

---

### G3 — LLM Router

**AI:** `datcuong-ai-chatbot/components/router.py`

```python
from langchain_ollama import ChatOllama
from langchain_core.prompts import PromptTemplate

llm = ChatOllama(model="llama3", temperature=0)

CLASSIFY_PROMPT = PromptTemplate.from_template("""
Phân loại câu hỏi sau thành 1 trong 3: legal / general / unknown
- legal: liên quan pháp luật, luật, nghị định, quy định, xử phạt
- general: câu chào hỏi, thời tiết, không liên quan pháp luật
- unknown: không rõ

Câu hỏi: {question}
Chỉ trả lời đúng 1 từ (legal / general / unknown):
""")

def router(state: GraphState) -> str:
    result = (CLASSIFY_PROMPT | llm).invoke({"question": state["raw_question"]})
    intent = result.content.strip().lower()
    return "direct_answer" if intent == "general" else "retrieve"
```

Cập nhật LangGraph: thêm node `direct_answer`, conditional edge từ router.

> Ref: https://langchain-ai.github.io/langgraph/how-tos/branching | https://python.langchain.com/docs/how_to/routing

---

### G4 — Reranker

**AI:** `datcuong-ai-chatbot/components/document_ranking.py`

```python
from sentence_transformers import CrossEncoder

_reranker = None

def get_reranker():
    global _reranker
    if _reranker is None:
        _reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")
    return _reranker

def document_ranking(state: GraphState) -> GraphState:
    query = state["structured_query"]
    docs  = state["context"]

    scores = get_reranker().predict([(query, d.page_content) for d in docs])
    ranked = [doc for _, doc in sorted(zip(scores, docs), reverse=True)]
    return {**state, "context": ranked[:3]}
```

> Ref: https://www.sbert.net/docs/cross_encoder/usage/usage.html | https://huggingface.co/cross-encoder/ms-marco-MiniLM-L-6-v2

---

## Tuần 10: 17–23/09 — Upload + Admin

### G6 — File Upload

**BE:**
```bash
npm install @nestjs/platform-express multer @types/multer
```

Tạo `datcuong-backend/src/modules/upload/upload.controller.ts`:

```typescript
@Post()
@UseInterceptors(FileInterceptor('file', {
  storage: diskStorage({
    destination: './uploads',
    filename: (_, file, cb) => cb(null, `${Date.now()}-${file.originalname}`),
  }),
  limits: { fileSize: 10 * 1024 * 1024 },
  fileFilter: (_, file, cb) => cb(null, file.mimetype === 'application/pdf'),
}))
async upload(@UploadedFile() file: Express.Multer.File) {
  return { url: `/uploads/${file.filename}` };
}
```

Thêm `ServeStaticModule` để serve `/uploads/` folder.

**FE:** tìm component dùng `mockapi.io` → thay bằng:
```typescript
const formData = new FormData();
formData.append('file', selectedFile);
const { url } = await axios.post('/api/upload', formData);
```

> Ref: https://docs.nestjs.com/techniques/file-upload | https://docs.nestjs.com/recipes/serve-static

---

### G8 — Admin UI

**FE:** `datcuong-frontend/src/app/(admin)/` — rà soát từng trang:

| Trang | API cần kết nối | Status cần check |
|-------|----------------|-----------------|
| `users/` | `GET /api/users`, `PATCH /api/users/:id`, `DELETE /api/users/:id` | ? |
| `laws/` | `GET /api/laws`, `DELETE /api/laws/:id` | ? |
| `requests/` | `GET /api/requests`, `PATCH /api/requests/:id/status` | ? |

Dùng Swagger (`http://localhost:29001/api`) để xác nhận endpoint nào đã có, cái nào cần thêm vào backend.

> Ref: https://ant.design/components/table (dùng cho data tables trong admin)

---

## Tuần 11: 24–30/09 — Evaluation

### VLegalQA-50

Dataset file `deliverables/VLegalQA-50.json`:

```json
[
  {
    "id": "Q001",
    "question": "Uống rượu bia khi lái xe máy bị phạt bao nhiêu?",
    "ground_truth_law": "100/2019/NĐ-CP",
    "ground_truth_article": "Điều 6, Khoản 4",
    "ground_truth_answer": "2–3 triệu đồng, tước GPLX 10–12 tháng",
    "category": "giao_thong",
    "temporal": false
  },
  {
    "id": "Q041",
    "question": "Năm 2018 uống rượu lái xe máy bị phạt bao nhiêu?",
    "ground_truth_law": "46/2016/NĐ-CP",
    "ground_truth_article": "Điều 5, Khoản 3",
    "ground_truth_answer": "1–2 triệu đồng, tước GPLX 1–3 tháng",
    "category": "giao_thong",
    "temporal": true,
    "as_of": "2018-12-31"
  }
]
```

Phân loại: 20 giao thông · 15 lao động · 15 dân sự/hình sự · 10 temporal

### Experiments

| Config | Retrieval P@3 | Citation Acc. | Temporal Acc. |
|--------|---------------|---------------|---------------|
| Baseline: all-MiniLM + char-chunk | — | N/A | N/A |
| + BGE-M3 + Article-chunk (F3) | — | N/A | N/A |
| + Citation F2 | — | — | N/A |
| + Temporal F4 | — | — | — |

---

## Tuần 12: 01–07/10 — Paper + Final

### Paper

**Title:** *Structure-Aware Chunking and Citation-Grounded Generation for Vietnamese Legal QA*

```
1. Introduction
2. Related Work  
   - RAG for legal QA: CUAD, LexGLUE, LegalBench
   - Vietnamese NLP: PhoBERT (VinAI), VLSP shared tasks
   - Citation-grounded generation
3. System Architecture
4. F3: Structure-Aware Article-Level Chunking
5. F2: Citation-Grounded Generation
6. F1: Legal Knowledge Graph
7. F4: Temporal-Aware Retrieval
8. Experiments (VLegalQA-50)
9. Conclusion
```

Target venue: RIVF 2026 · KSE 2026 · ACIIDS 2027

### Final checklist

- [ ] `docker-compose up` chạy ổn tất cả 4 service
- [ ] PR: bcrypt, F3+BGE-M3, F2, F1, F4 merged
- [ ] `deliverables/VLegalQA-50.json` public
- [ ] Paper draft ≥ 8 trang
- [ ] Báo cáo LVTN: chương mới Cải tiến + Kết quả thực nghiệm

---

## Rủi Ro

| Rủi ro | Fallback |
|--------|----------|
| BGE-M3 không đủ RAM | `paraphrase-multilingual-mpnet-base-v2` — HuggingFace, nhẹ hơn |
| Llama3 không follow structured output | Thử `llama3.1`; hoặc thêm JSON regex fallback parser |
| react-force-graph render chậm graph lớn | Giới hạn depth=1, max 50 nodes khi hiển thị |
| 50 câu dataset không đủ | Giảm xuống 30 câu, 3 categories |
| PR không merge được (thiếu quyền) | Demo từ fork cá nhân `trannhatt` |
