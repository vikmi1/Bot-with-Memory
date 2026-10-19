**English** · [Deutsch](README.de.md) · [Русский](README.ru.md)

# Telegram Bot with Memory: Conversation + RAG over Documents

The bot keeps a coherent conversation (short-term memory) and answers questions about uploaded PDF, DOCX and TXT files
(long-term memory based on RAG and ChromaDB). Each user's data is isolated.

The bot's interface is in Russian; the messages below are translated for readability.

## Features

- Conversation that takes history into account: the latest messages + a short summary of the older part of the conversation.
- Upload PDF, DOCX, TXT → text extraction → overlapping chunks → OpenAI embeddings → ChromaDB.
- Answers about documents are based only on the retrieved fragments, with a "Sources in documents" block.
- Document management: list, choose the active document, delete with confirmation.
- Documents and the index survive a restart (short-term memory is cleared).

## Installation

Python 3.11 or newer is required.

```bash
python -m venv .venv
source .venv/bin/activate      # macOS/Linux
.venv\Scripts\activate         # Windows PowerShell
pip install -r requirements.txt
cp .env.example .env           # in Windows PowerShell: Copy-Item .env.example .env
```

## Configuring .env

Open `.env` and fill in the two required values:

| Variable | What it is |
|---|---|
| `BOT_TOKEN` | bot token from [@BotFather](https://t.me/BotFather) |
| `OPENAI_API_KEY` | key from [platform.openai.com](https://platform.openai.com/api-keys) |

The other variables have sensible defaults:

| Variable | Default | Purpose |
|---|---|---|
| `CHAT_MODEL` | `gpt-5-mini` | model for answers and summaries |
| `EMBED_MODEL` | `text-embedding-3-small` | embedding model |
| `CHROMA_PATH` | `./data/chroma` | ChromaDB folder |
| `UPLOADS_PATH` | `./data/uploads` | folder with the original files (`{user_id}/`) |
| `SHORT_MEMORY_MESSAGES` | `10` | how many recent messages are sent to the model |
| `SHORT_MEMORY_SUMMARY_AFTER` | `20` | after how many messages the history is collapsed into a summary (`0` — disabled) |
| `CHUNK_SIZE` / `CHUNK_OVERLAP` | `500` / `75` | chunk size and overlap, in characters |
| `RAG_TOP_K` | `5` | how many fragments are passed to the model |
| `MAX_DOCUMENT_SIZE_MB` | `20` | maximum file size |
| `RAG_MAX_DISTANCE` | `0.7` | relevance threshold (cosine distance; lower is stricter) |
| `MAX_ANSWER_TOKENS` | `4000` | answer token limit (for reasoning models it includes the "thinking") |
| `LOG_LEVEL` | `INFO` | logging level |

Paths in `.env` are relative to the project folder. The `.env` file is not committed to git.

## Running

```bash
python bot.py
```

The log shows the line "Бот запущен: @bot_name" (bot started). Stop it with Ctrl+C.
The bot works only in private chats.

## Tests

```bash
pytest
```

## Bot commands

| Command | What it does |
|---|---|
| `/start` | greeting and a short help |
| `/help` | all commands and an explanation of the memory |
| `/documents` | list of documents: number, file, date, number of chunks, short id |
| `/use_document 2` | search only in document #2 |
| `/all_documents` | search all documents again |
| `/delete_document 2` | delete document #2 (with "Delete" / "Cancel" confirmation buttons) |
| `/clear_chat` | clear the conversation history and its summary (documents stay) |

### Sample session

```
You:  /start
You:  [send company.pdf]
Bot:  File received. Extracting text and building the index…
Bot:  Done: company.pdf has been added to long-term memory. Chunks created: 24. You can ask questions now.
You:  Do you do web development?
Bot:  Yes, ...
      Sources in documents: company.pdf, fragments 2 and 4
You:  And how long does it take?   ← "it" is clear from the conversation history
Bot:  I couldn't find confirmation of this in the uploaded documents.
You:  /clear_chat
Bot:  The current conversation history has been cleared
```

## How it works

```
bot.py                     Telegram handlers (aiogram), startup, logging
config.py                  reading and validating .env
services/
  document_loader.py       safe file name, text extraction (TXT/PDF/DOCX)
  chunker.py               chunk_text(): chunks by paragraphs/sentences with overlap
  embeddings.py            OpenAI embeddings in batches
  vector_store.py          ChromaDB: long_memory collection, cosine similarity
  registry.py              SQLite data/app.db: documents and user_settings tables
  documents.py             "file → index" pipeline and document deletion
  short_memory.py          history and summary per user_id (in-memory deque)
  rag.py                   retrieve_context(), prompt assembly, model call, summarization
tests/                     pytest
data/uploads/{user_id}/    original files
data/chroma/               vector database
data/app.db                document registry
```

### Short-term memory

- The history is stored in a `deque` per `user_id`; the user's message and the assistant's answer count as one message each.
- The latest `SHORT_MEMORY_MESSAGES` messages are sent to the model.
- So that summarization can kick in (the threshold of 20 is larger than the window of 10), the `deque` holds `max(SHORT_MEMORY_MESSAGES, SHORT_MEMORY_SUMMARY_AFTER + 2)` messages; on overflow the oldest one is evicted (FIFO).
- When there are more than `SHORT_MEMORY_SUMMARY_AFTER` messages, the model writes a summary (facts, goals, decisions; no secrets) and the last 6 messages stay in the history. The summary is added to the system prompt. If summarization fails, the history stays as it is and the attempt is repeated later.

### Long-term memory and data isolation

- All chunks are stored in one `long_memory` collection, but **every search, count and deletion** includes a `user_id` filter (see `_where()` in `services/vector_store.py`); when a document is selected, `doc_id` is added.
- The document registry and deletion also check `user_id`, so another user's document can neither be found nor deleted, even by substituting its `doc_id`.
- The search drops results beyond the `RAG_MAX_DISTANCE` threshold and removes near-duplicate chunks. If nothing relevant is found, the model is told so and answers "I couldn't find confirmation of this in the uploaded documents."
- A short follow-up question ("And how long does it take?") is searched together with the user's previous question.
- `/use_document` strictly limits the search to the selected document: if nothing is found there, the bot does not fall back to other documents. The selection is stored in SQLite and survives a restart; it is reset by `/all_documents` or by deleting the document.

## Security and privacy

- The file name is stripped of paths and special characters; the file is saved as `data/uploads/{user_id}/{doc_id}_{name}`.
- File extension and size are checked; no user data is ever passed to a shell.
- The user sees only clear messages; tracebacks and error details go to the log.
- The logs contain no document contents or message texts; the bot token and the OpenAI key are masked (`***`).
- The texts of your documents and questions are sent to OpenAI — keep this in mind when uploading confidential data.

## Common problems

- **"Не задана переменная окружения BOT_TOKEN/OPENAI_API_KEY"** (environment variable not set) — fill in `.env`; the placeholder values from `.env.example` will not work.
- **PDF without text** — it is a scan; the bot does not do OCR, upload a PDF with a text layer.
- **Changing `EMBED_MODEL` after indexing** — models have different vector dimensions, so old documents will no longer be found. Delete them with `/delete_document` (or clear the `data/chroma` folder together with `data/app.db`) and upload them again.
- **The model answers too slowly** — `gpt-5-mini` reasons before answering; for speed, set a less "thinking" model in `CHAT_MODEL`.
