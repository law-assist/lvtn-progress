# Hướng dẫn chạy Local — Law Assist Stack

## Tổng quan

| Service | Port | Tech | Status |
|---------|------|------|--------|
| Backend | 29001 | NestJS 10 | ✅ |
| Frontend | 3000 | Next.js 14 | ✅ |
| AI Chatbot | 28080 | FastAPI + LangGraph | ✅ |
| AI Law Linking | 8000 | FastAPI + PhoBERT | ⚠️ cần model weights |

---

## Yêu cầu

- Node 22+
- Python 3.12+
- MongoDB 7.0 (via Homebrew)
- Ollama (local LLM)

---

## 1. Cài dependencies

### MongoDB
```bash
brew tap mongodb/brew
brew trust mongodb/brew
brew install mongodb-community@7.0
brew services start mongodb/brew/mongodb-community@7.0
```

### Ollama + llama3
```bash
brew install ollama
brew services start ollama
ollama pull llama3   # ~4.7GB, chờ xong mới start AI Chatbot
```

### Backend (NestJS)
```bash
cd datcuong-backend
npm install --legacy-peer-deps
```

### Frontend (Next.js)
```bash
cd datcuong-frontend
npm install --legacy-peer-deps
```

### AI Chatbot (Python)
```bash
cd datcuong-ai-chatbot
pip install -r requirements.txt
```

### AI Law Linking (Python)
```bash
cd datcuong-ai-law-linking
pip install -r requirements.txt
```

---

## 2. Tạo file .env

### `datcuong-backend/.env`
```
MONGODB_URI=mongodb://localhost:27017/law_linking
JWT_SECRET=lvtn-local-dev-secret-2026
PORT=29001
AI_HOST=http://localhost:8000
NODE_ENV=development
PUPPETEER_EXECUTABLE_PATH=
CRAWLED=false
TZ=Asia/Ho_Chi_Minh
```

### `datcuong-frontend/.env.local`
```
NEXT_PUBLIC_API_HOST=http://localhost:29001
BACKEND_API_HOST=http://localhost:29001
NEXT_SERVER_API_HOST=http://localhost:29001
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=lvtn-local-nextauth-secret-2026
NODE_ENV=development
REMOTE_PROTOCOL=https
REMOTE_HOSTNAME=picsum.photos
REMOTE_PHOTO_MOCK_DATA=i.imgur.com
```

### `datcuong-ai-chatbot/.env`
```
MONGO_URI=mongodb://localhost:27017
MONGO_DBNAME=law_linking
LANGSMITH_TRACING=false
LANGSMITH_API_KEY=
LANGSMITH_PROJECT=law-chatbot
```

### `datcuong-ai-law-linking/.env`
```
MONGO_URI=mongodb://localhost:27017
MONGO_DBNAME=law_linking
```

---

## 3. Seed dữ liệu mẫu vào MongoDB

MongoDB mặc định trống. Dùng script dưới để import 2 văn bản luật mẫu từ file test:

```python
# Chạy từ thư mục gốc law-assist/
python3 - << 'EOF'
import json
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017")
laws = client["law_linking"]["laws"]

for path in [
    "datcuong-ai-law-linking/testing_ner.json",
    "datcuong-ai-law-linking/testing_similarity.json"
]:
    with open(path) as f:
        doc = json.load(f)
    doc.pop("_id", None)
    doc.setdefault("fields", "Pháp luật")
    doc.setdefault("isDeleted", False)
    laws.insert_one(doc)

print(f"Tổng laws: {laws.count_documents({})}")
client.close()
EOF
```

> **Lưu ý:** Để có đầy đủ dữ liệu thực tế, dùng crawler của backend (cần admin token).  
> Xem API: `GET http://localhost:29001/crawler/auto`

---

## 4. Build ChromaDB vector store (AI Chatbot)

ChromaDB cần được build từ MongoDB trước khi start AI Chatbot.

```bash
cd datcuong-ai-chatbot
python scripts/build_vectorstore.py
```

Output: `Done. Vector store has N chunks.`

> Mỗi khi thêm dữ liệu mới vào MongoDB, chạy lại script này để cập nhật vector store.

---

## 5. Start các service

Mở **4 terminal riêng**:

### Terminal 1 — Backend
```bash
cd datcuong-backend
npm run dev
# → Nest application successfully started (port 29001)
```

### Terminal 2 — Frontend
```bash
cd datcuong-frontend
npm run dev
# → Ready in Xs (port 3000)
```

### Terminal 3 — AI Chatbot
```bash
cd datcuong-ai-chatbot/api
python api.py
# → Uvicorn running on http://0.0.0.0:28080
```

### Terminal 4 — AI Law Linking (⚠️ cần model weights)
```bash
cd datcuong-ai-law-linking
python api.py
# → Uvicorn running on http://0.0.0.0:8000
```

> **AI Law Linking** yêu cầu model weights tại:
> - `models/ner_model/pytorch_model.bin` hoặc `model.safetensors`
> - `models/Classification_model/pytorch_model.bin`
> - `models/STS_model_V2/bert-sts-V2.pt`
>
> Các file này không commit vào git do kích thước lớn. Liên hệ tác giả để lấy weights.

---

## 6. Truy cập

| URL | Mô tả |
|-----|-------|
| http://localhost:3000 | Frontend (giao diện người dùng) |
| http://localhost:29001 | Backend API |
| http://localhost:28080/docs | AI Chatbot — Swagger UI |
| http://localhost:8000/docs | AI Law Linking — Swagger UI |

---

## 7. Kiểm tra hoạt động

```bash
# Backend đang chạy?
curl http://localhost:29001/auth/register \
  -X POST -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"Test1234!","fullName":"Test","role":"user"}'

# Chatbot đang chạy?
curl http://localhost:28080/agents/question-answering \
  -X POST -H "Content-Type: application/json" \
  -d '{"query":"Điều kiện để được trợ giúp pháp lý là gì?"}'
```

---

## Lỗi thường gặp

| Lỗi | Nguyên nhân | Fix |
|-----|-------------|-----|
| `mongod: command not found` | MongoDB chưa cài | `brew install mongodb-community@7.0` |
| `ollama pull` timeout | Mạng chậm | Chờ đợi, `ollama list` để kiểm tra |
| `No module named 'langchain.prompts'` | LangChain v1.x đổi import | Đã fix trong code: dùng `langchain_core.prompts` |
| `OSError: Error no file named model.safetensors` | Model weights thiếu | Cần copy thủ công vào `models/` |
| `CUDA error` trên Mac | Model hardcode `.to("cuda")` | Đã fix trong `api.py`: auto-detect device |
| ChromaDB trống | Chưa build vector store | Chạy `python scripts/build_vectorstore.py` |
| Frontend lỗi API | Backend chưa chạy | Đảm bảo Backend start trước |

---

## Cấu trúc dữ liệu MongoDB

Database `law_linking`, collection `laws`:

```json
{
  "_id": "ObjectId",
  "name": "Nghị định 14/2013/NĐ-CP ...",
  "category": "Nghị định",
  "department": "Chính phủ",
  "numberDoc": "14/2013/NĐ-CP",
  "fields": "Tư pháp",
  "content": {
    "header": [...],
    "description": [...],
    "mainContent": [
      {
        "type": "dieu",
        "value": "Điều 1. ...",
        "content": [
          { "type": "khoan", "value": "1. ...", "content": [...] }
        ]
      }
    ],
    "footer": [...]
  },
  "relationLaws": [],
  "isDeleted": false
}
```
