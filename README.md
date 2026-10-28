# RAG Document Chatbot (Node.js)

A simple **RAG (Retrieval-Augmented Generation)** application that answers questions from an uploaded PDF — your own personal "ask my document" AI, running on Google's free Gemini API.

Upload a PDF → ask a question → get an answer generated **only** from that document (no hallucinated outside knowledge). If the answer isn't in the document, it says so instead of making something up.

---

## Features

- Upload a PDF and extract its text (handles multi-page PDFs)
- Automatic text chunking with overlap (so no relevant sentence gets cut in half)
- Free embedding generation via Google AI Studio (Gemini Embedding API)
- Simple in-memory/on-disk vector search (cosine similarity) — no external vector database needed
- **Embeddings are cached per document.** Re-uploading the same PDF costs zero extra API calls.
- Only the top 2–3 most relevant chunks are sent to the AI per question (keeps token usage low)
- Clear "Answer not found in the document" response when the document doesn't contain the answer
- Full error handling (bad file type, empty PDF, missing API key, API downtime, bad input, etc.)
- Clean, modular folder structure (routes / controllers / services / utils / middleware / config)

---

## Tech Stack

- **Node.js** + **Express.js**
- **Google AI Studio (Gemini API)** (free tier) — for both embeddings and text generation
  - Embeddings: `gemini-embedding-001`
  - Generation: `gemini-2.5-flash`
- **pdfjs-dist** — Mozilla's actively-maintained PDF text extraction engine
  (Note: this project deliberately avoids the commonly-tutorialized `pdf-parse` package,
  which bundles a very old, unmaintained PDF engine that fails on some valid PDFs.)
- **Multer** — file upload handling
- Custom lightweight vector store (JSON file cache + cosine similarity) — no paid vector DB required
- **dotenv** for environment variables

No Python anywhere in this project.

---

## Project Structure

```
rag-chatbot/
├── server.js                       # Entry point
├── src/
│   ├── app.js                      # Express app setup
│   ├── config/
│   │   └── config.js                # Loads & validates .env
│   ├── routes/
│   │   └── index.js                 # /api/health, /api/upload, /api/ask
│   ├── controllers/
│   │   ├── documentController.js    # Upload logic
│   │   └── questionController.js    # Ask logic
│   ├── services/
│   │   ├── pdfService.js            # PDF text extraction
│   │   ├── chunkService.js          # Text chunking
│   │   ├── embeddingService.js      # Calls Google embedding API
│   │   ├── vectorStore.js           # Caching + similarity search
│   │   └── llmService.js            # Calls Google Gemini for the answer
│   ├── middleware/
│   │   ├── upload.js                 # Multer config + PDF-only validation
│   │   └── errorHandler.js           # Central error formatting
│   └── utils/
│       ├── AppError.js
│       ├── fileHash.js               # Used as the "document ID"
│       └── logger.js
├── uploads/                          # Uploaded PDFs land here
├── data/embeddings/                  # Cached embeddings (one JSON file per document)
├── public/
│   └── test.html                     # Minimal browser test page (upload + ask, no build step)
├── .env.example
└── .gitignore
```

---

## Installation

### 1. Prerequisites
- [Node.js](https://nodejs.org/) v18 or newer installed
- A free Google account

### 2. Get your free Google AI Studio API key (you need to do this — I can't get it for you)
1. Go to https://aistudio.google.com/apikey
2. Sign in with your Google account.
3. Click **"Create API key"**.
4. Choose "Create API key in new project" if you don't have a Google Cloud project yet.
5. Copy the key that's generated.

### 3. Set up the project
```bash
# unzip the project, then:
cd rag-chatbot
npm install
```

### 4. Configure environment variables
```bash
cp .env.example .env
```
Open `.env` and paste your key:
```
GOOGLE_API_KEY=your_actual_key_here
```

### 5. Run it
```bash
npm start
```
You should see:
```
[INFO] RAG Document Chatbot running on http://localhost:5000
```

### 6. Try the built-in test page
Open **http://localhost:5000/test.html** in your browser — a minimal page to upload a PDF and ask questions without needing curl.

For auto-restart on file changes during development:
```bash
npm run dev
```

---

## Environment Variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `PORT` | No | `5000` | Port the server runs on |
| `GOOGLE_API_KEY` | **Yes** | — | Your free Google AI Studio API key. App will start without it but upload/ask will fail with a clear error. |
| `EMBEDDING_MODEL` | No | `gemini-embedding-001` | Free-tier Google model used to convert text into vectors |
| `LLM_MODEL` | No | `gemini-2.5-flash` | Free-tier Google model used to generate answers |
| `CHUNK_SIZE_WORDS` | No | `350` | Words per text chunk |
| `CHUNK_OVERLAP_WORDS` | No | `50` | Overlapping words between consecutive chunks |
| `TOP_K_CHUNKS` | No | `3` | How many chunks are retrieved per question |

---

## API Endpoints

### `GET /api/health`
Checks the server is alive and whether required env vars are set.
```bash
curl http://localhost:5000/api/health
```

### `POST /api/upload`
Upload a PDF. Form-data field name must be `file`.
```bash
curl -X POST http://localhost:5000/api/upload -F "file=@/path/to/document.pdf"
```
Response:
```json
{
  "success": true,
  "message": "Document processed and embeddings cached successfully.",
  "docId": "a1b2c3d4e5f6...",
  "fileName": "document.pdf",
  "numPages": 3,
  "numChunks": 5
}
```
Save the `docId` — you'll need it to ask questions. Uploading the exact same file again returns instantly from cache and makes **zero** API calls.

### `POST /api/ask`
```bash
curl -X POST http://localhost:5000/api/ask \
  -H "Content-Type: application/json" \
  -d '{"docId": "a1b2c3d4e5f6...", "question": "What is the notice period?"}'
```
Response:
```json
{
  "success": true,
  "question": "What is the notice period?",
  "answer": "...",
  "sources": [
    { "excerpt": 1, "relevanceScore": 0.842, "preview": "..." }
  ]
}
```

---

## Where Your Free API Quota Is Used (so you know what costs what)

1. **Uploading a PDF** → 1 embedding API call for the whole document (all chunks sent together via Google's `batchEmbedContents`). This is the only "bulk" step, and it's **skipped entirely** if you re-upload a file you've already processed (checked via content hash).
2. **Asking a question** → 1 tiny embedding call (just for your question) + 1 generation call (only the top 2–3 chunks are sent, not the whole document). This happens every time you ask, by design.

That's it — no other calls are made anywhere in the app.

---

## Things to Know / Current Limitations

- Google AI Studio's free tier has daily/per-minute rate limits that can change; if you hit a `429` error, the app surfaces Google's exact error message so you know what happened.
- Scanned PDFs (images of text, no real text layer) aren't supported — there's no OCR here. You'll get a clear "empty PDF" error in that case.
- The vector store is plain JSON files on disk — perfect for personal/single-user use. It is not built for thousands of documents or concurrent users.
- If Google renames or deprecates a model, swap `EMBEDDING_MODEL` / `LLM_MODEL` in `.env` — no code changes needed.

## Future Improvements

- Add a simple frontend (upload form + chat UI)
- Support multiple documents per session / a document library view
- Add OCR support for scanned PDFs
- Swap the JSON file store for a real embedded vector DB (e.g. LanceDB) if document count grows
- Add streaming responses for faster perceived answer time
- Add authentication if this is ever exposed beyond localhost

---

## What You Need to Do (I can't do this part for you)

1. Create a free Google AI Studio API key (see Installation step 2 above).
2. Paste that key into your own `.env` file as `GOOGLE_API_KEY`.
3. Run `npm install` and `npm start` on your own machine.

That's the only manual step — everything else in this project is ready to run as-is.
# RagChatbot
#   R a g C h a t b o t  
 