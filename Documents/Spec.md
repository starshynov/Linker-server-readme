# Linker
Project name **Linker**  

### Current Doc Version  
**v1.0 (November 2025)**  updated by Oleksandr Starshynov

---

## 1. Introduction  

### Brief description  
**Linker** is an intelligent system for processing, vectorizing, and standardizing text fragments.  
The application automates repetitive work with large volumes of text, combining classical keyword search with semantic vector-based search.  

### Project goal  
To create a tool that frees a person from repetitive text-processing tasks,  
allowing them to focus on analysis, identifying relationships, and defining new model training rules.  
Linker is not about saving time — it’s about building a *thinking partner* for researchers.  

### Product history  
The project began as a personal tool to support the author’s hobby — the study of Dutch history, specifically the city of Alkmaar.  
The first version focuses on manual editing and analysis of topic-related fragments.  
Future releases will introduce automatic topic detection, visual link graphs, and a feedback-based training loop for the model.

---

## 2. Overall Architecture  

### Architectural approach  
**Modular Monolithic Architecture** — an application with clearly defined layers: UI, API, Model, and Storage.  
The components are logically separated but operate within one context, providing simplicity, transparency, and reproducibility.  

### Core principles  
- Separation of concerns between UI / API / Model / Storage  
- Integration of two data types: relational (Supabase) and vector (Qdrant)  
- Full local model execution (SentenceTransformer)  
- Minimal dependencies, easy local setup  
- Ready for future modularization or scaling  

### Components  

#### Frontend (TypeScript)
An interface for creating, editing, and analyzing text fragments.  

**Features:**  
- Create fragments and assign topics  
- Edit and group fragments by `topic`  
- Visualize hierarchy via `id` / `parentId`  
- Search by content or semantic similarity  
- Direct REST API communication  
- Local JSON fragment storage  

#### Backend (FastAPI + Python)
The backend receives text, splits it into subfragments, vectorizes them, and coordinates data between databases.  

**Pipeline:**  
1. Receives a text fragment  
2. Splits it into semantic subfragments  
3. Assigns `id` and `parentId`  
4. Saves the original text in Supabase  
5. Generates embeddings using SentenceTransformer  
6. Inserts embeddings into Qdrant  
7. Returns search results to the frontend  

**Main endpoints:**  
- `POST /fragment` — add a fragment  
- `GET /fragment/{id}` — get a fragment  
- `GET /search/text` — search by text content  
- `GET /search/vector` — semantic similarity search  

#### AI Model Layer
Uses a local `SentenceTransformer (all-MiniLM-L6-v2)` model fine-tuned on custom triplets.  

**Highlights:**  
- Local path: `models/custom-embeddings-v1`  
- Fine-tuned using triplets *(Anchor / Positive / Negative)*  
- Domain-specific optimization (Dutch history / Alkmaar)  
- Consistent embeddings across sessions  
- Retraining possible without breaking compatibility  

#### Data Layer
- **Supabase (PostgreSQL):** stores original texts, metadata (`id`, `parentId`, `topic`, timestamps`)  
- **Qdrant (Vector DB):** stores embeddings and performs nearest-neighbor vector search  

**Data Flow:**  
`Frontend → FastAPI → Supabase + Qdrant → Frontend`

---

## Code Structure (Server)

The server side of **Linker** is implemented as a well-structured FastAPI application.  
Each module is responsible for a specific part of the data processing pipeline — from text ingestion to embedding generation and storage.

At the core lies **`main.py`**, which serves as the FastAPI entry point.  
It defines all HTTP routes, initializes dependencies, and launches the application.

Configuration parameters — such as Supabase and Qdrant connection URLs, API keys, model paths, and training parameters — are stored in **`config.py`**.

The **`models.py`** file defines Pydantic schemas used for data validation and API payloads.  
These include entities like `Fragment`, `SearchRequest`, and `SearchResult`.

Incoming text data is processed through **`text_splitter.py`**, which divides long texts into smaller, meaningful fragments,  
and **`process_fragment.py`**, which normalizes, cleans, and assigns `id` and `parentId` values to each fragment.

The embedding pipeline is managed by **`embeddings.py`** and **`embeddings_generator.py`**.  
The first module loads the local SentenceTransformer model and handles individual embedding generation,  
while the second can generate embeddings in batches for larger datasets or reprocessing tasks.

For vector search operations, **`qdrant_service.py`** connects to the Qdrant database,  
handling collection management, data upserts, and nearest-neighbor search queries.  
In parallel, **`supabase_client.py`** manages communication with the Supabase PostgreSQL database,  
storing text fragments, topics, and metadata.

The model training process is defined in **`train_embeddings.py`**, which fine-tunes the SentenceTransformer model on triplet data  
(Anchor / Positive / Negative) to adapt it to project-specific semantics.  
Training datasets are stored in the **`TrainingData/`** directory,  
while trained model versions are saved locally under **`models/custom-embeddings-v1/`**.

Logs of training sessions and metrics (such as loss, epoch, and sample count) are recorded in **`training_logs.csv`**.  
All required Python dependencies are listed in **`requirements.txt`**,  
and backend setup instructions are documented in **`README.md`**.

To start the backend locally, run:

```
# from the Linker_server directory
pip install -r requirements.txt
python -m uvicorn main:app --reload
```
## 3. Technology Stack  

- **Frontend:** TypeScript (no frameworks), Live Server  
- **Backend:** Python + FastAPI  
- **Databases:** Supabase (PostgreSQL), Qdrant (Vector Search)  
- **AI:** SentenceTransformer (all-MiniLM-L6-v2), local fine-tuned model  
- **Deployment:** Local run; Docker planned  
- **Version control:** GitHub
- 
---

## 4. Users and Roles  

### Current  
- **Researcher** — create, edit, and explore text fragments  

### Planned roles  
- **Admin** — manage models and databases  
- **Editor** — clean and organize text  
- **Analyst** — analyze fragment relationships and trends  

---

## 5. Integration and Communication  

### Data exchange  
REST API communication between the frontend and FastAPI.  
Asynchronous coordination between Supabase and Qdrant within the backend.  

### Versioning  
- **API:** `/api/v1`  
- **Model:** `models/custom-embeddings-v1`  
- **Qdrant:** collections use versioned prefixes for safe upgrades.  

---

## 6. Data and Storage  

### Data model  
Each fragment includes:  
- `id` — unique identifier  
- `parentId` — reference to parent fragment  
- `text` — content  
- `topic` — manually assigned theme  
- `embedding` — vector representation  
- `timestamp` — creation date  

### Storage  
- **Supabase:** stores text and metadata  
- **Qdrant:** embeddings, indexes, semantic search  
- **Model folder:** local model versions and checkpoints  

### Search modes  
- **Strict text match** — exact keyword match  
- **Like search** — partial content match  
- **Vector search** — semantic similarity search  

---

## 7. Security and PII  

- All processing runs locally  
- No external data transmission  
- Planned Supabase Auth integration  
- Isolated user collections and sessions  
- Optional AES-256-GCM encryption (future)  

---

## 8. Logging and Monitoring  

- FastAPI (uvicorn) logs  
- Local error and latency logs  
- Prometheus integration planned for latency/throughput tracking  

---

## 9. DevOps and CI/CD  

- **Repository:** GitHub
- **Frontend:** open `index.html` using Live Server  
- **Backend:** run `python -m uvicorn main:app --reload`  
- **Planned:** Docker Compose and GitHub Actions for automatic build and deploy  

---

## 10. Testing and Quality  

- **Unit tests:** pytest (API endpoints)  
- **Manual testing:** frontend and integration  
- **Future:** tests for embedding consistency and semantic accuracy  

---

## 11. Monitoring and Incident Response  

- **Metrics:** response time, number of stored fragments, search latency  
- **Future:** alert system for model or index failure  

---

## 12. Roadmap and Future Development  

- Docker containerization  
- Supabase Auth integration  
- Automatic topic classification  
- Visual graph of fragment relationships  
- Per-user data collections  
- Offline-first mode  
- Model feedback learning loop  

