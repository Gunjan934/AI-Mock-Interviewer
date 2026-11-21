# Phase 6 — RAG-Based Knowledge Retrieval and Grounded AI Interviewing

## Status
**IN PROGRESS / PARTIALLY IMPLEMENTED (initial foundation added, verification pending).** RAG is disabled by default (`RAG_ENABLED=0`).

## Architecture
- **Independent module:** `app/rag/` (config, document loader, text cleaner, chunking, embedding service, persistent ChromaDB vector store, retriever, context builder, RAG service/exceptions).
- **Relational metadata:** `KnowledgeDocument` model + migration `f1a2b3c4d5e6_phase6_knowledge_documents.py` (user-owned, status/pages/chunks/embedding/errors). Vector embeddings stored in ChromaDB (not SQLAlchemy).
- **Services/routes/UI:** `app/services/knowledge_service.py`, `app/routes/knowledge.py` (`/knowledge/*`), templates `app/templates/knowledge/{index,upload,detail,search}.html`.
- **AI integration:** dedicated RAG prompts added to existing prompt registry; uses existing NVIDIA AI service/provider abstraction (`nvidia/nemotron-3.5-bolt-e4b` config preserved). Retrieved context treated as untrusted reference data.
- **Preservation:** Phase 1–5 components, `baseline_v1`, `ai_v1`, Phase 4 ML, interview agent/question-generation behavior, auth, and DB/migrations preserved (additive integration only).

## Implemented components
- Document ingestion pipeline: upload validation (type/size/secure filename), extraction (PDF/TXT/MD, DOCX best-effort), cleaning, paragraph-preferred chunking with overlap/metadata/stable IDs, status tracking (PENDING/PROCESSING/COMPLETED/FAILED), retry/delete with vector cleanup.
- Embedding service: lazy cached Sentence Transformers (`all-MiniLM-L6-v2` default), batch normalized embeddings, consistent model config.
- Vector store: persistent ChromaDB (`instance/vector_store`, collection `interview_knowledge`, cosine space), metadata upsert/query/delete, user isolation filters.
- Retrieval: query embedding, Top-K + similarity threshold, dedupe, source metadata, `NO_RELEVANT_CONTEXT` when below threshold.
- Grounded generation helpers: context builder with source attribution, RAG answer/question/feedback generation via existing AI service (JSON-structured prompts), insufficient-context handling.
- UI: Knowledge Base dashboard/upload/detail, search, and RAG playground (retrieval results and grounded response separated).

## Configuration
- `RAG_ENABLED=false` (default)
- `RAG_VECTOR_STORE_PATH=instance/vector_store`
- `RAG_EMBEDDING_MODEL=sentence-transformers/all-MiniLM-L6-v2`
- `RAG_TOP_K=5`, `RAG_CHUNK_SIZE=500`, `RAG_CHUNK_OVERLAP=75`
- `RAG_SIMILARITY_THRESHOLD=0.35` (cosine similarity interpretation documented in retriever)
- `RAG_MAX_CONTEXT_CHARS=6000`, `RAG_COLLECTION_NAME=interview_knowledge`, `RAG_DISTANCE_METRIC=cosine`

## Verification status (current)
- **Compile:** app/migrations/scripts compile successfully.
- **Migration:** `knowledge_documents` migration applied in local environment.
- **Dependencies:** `chromadb` / `sentence-transformers` installation and real embedding/vector-store/NVIDIA-grounded verification **pending in this environment**.
- **Tests:** Phase 6 test suite and full regression re-run **pending completion**.
- **End-to-end flows:** upload→extract→chunk→embed→store→retrieve→generate, persistence across restart, user isolation proof, deletion cleanup, and retrieval evaluation metrics **pending**.

## Security/privacy notes
- Per-user ownership and retrieval filtering enforced; private documents not shared/indexed globally.
- Secure filenames, extension/MIME/size validation, no file execution, safe deletion.
- Retrieved content treated as untrusted reference data (prompt-injection guard in prompts/context handling).
- No private resume auto-indexing into shared knowledge base; no automatic training on candidate data.

## Limitations (current)
- DOCX/OCR not fully supported unless dependencies/OCR implemented (PDF image-only extraction reported clearly).
- Chunking uses approximate token sizing (character-based heuristic).
- Relevance threshold/Top-K require empirical tuning per corpus.
- RAG-assisted integration into live question generation/evaluation/learning feedback is partially scaffolded and requires controlled enablement and end-to-end verification.
- Retrieval quality metrics require a labelled evaluation set (not fabricated).