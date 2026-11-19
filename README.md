# AI-Chatbot-Learning

2. Technology selection
We will use the following components.
Component              ---	Technology
LLM                    ---		Ollama + Qwen3
PDF                    ---	 extraction	PyMuPDF
OCR for scanned PDFs   ---		Tesseract
Embedding model        ---		BGE-M3
Vector database        ---		Qdrant
Metadata / history     ---		SQL Server

Why BGE-M3? It supports multilingual embeddings, including English and many other languages, making it a useful candidate for your English/Bangla project documents.



**Get List of installed**
```
ollama list
```

Step 3. Prepare the project directory

/
├── opt/
│   └── enterprise-chatbot/
│       ├── backend/
│       ├── frontend/
│       ├── docker/
│       └── scripts/
│
└── srv/
    └── llm/
        ├── models/
        ├── logs/
        ├── backups/
        └── documents/

/srv/llm/knowledge-base/
│
├── incoming/
│   └── Upload PDFs here
│
├── processed/
│   └── Successfully indexed documents
│
├── failed/
│   └── Failed PDF processing
│
├── archive/
│   └── Original PDF backups
│
└── scripts/
    └── Ingestion scripts

    
We will use:
- /opt/enterprise-chatbot for application code.
- /srv/llm/models for downloaded LLM models.
- /srv/llm/documents for documents used in future RAG implementation.
- /srv/llm/backups for backups.



Go to URL: cd /opt/enterprise-chatbot
Activate env : source rag-env/bin/activate

**4. Install Qdrant Vector Database**
/opt/enterprise-chatbot/docker/qdrant/docker-compose.yml

**Docker File:**
```
services:
  qdrant:
    image: qdrant/qdrant:latest
    container_name: enterprise-qdrant

    restart: unless-stopped

    ports:
      - "127.0.0.1:6333:6333"

    volumes:
      - /srv/llm/qdrant:/qdrant/storage

    environment:
      QDRANT__SERVICE__API_KEY: "${QDRANT_API_KEY}"

    security_opt:
      - no-new-privileges:true
```

**Create .env and give content at below:**
QDRANT_API_KEY=YOUR_GENERATED_SECRET
**key creation command:** openssl rand -hex 32


**5. Install the embedding model**
The embedding model is different from your Qwen3 LLM.
- Qwen3 generates answers.
- BGE-M3 converts documents and questions into vectors.
- Qdrant searches those vectors to find relevant document content.
Download BGE-M3:
ollama pull bge-m3
Verify:
ollama list

For BGE-M3, we will configure Qdrant with 1024 dimensions.

Our initial processing tools:
**Library	Purpose**
PyMuPDF   ---	Extract PDF text
Requests  ---	Communicate with Ollama
Qdrant   ---	 Client	Insert and search vectors
Python-dotenv   ---		Manage configuration
tqdm   ---		Show ingestion progress

**For scanned PDFs, install OCR (en/bn):**
sudo apt install -y tesseract-ocr tesseract-ocr-eng
sudo apt install -y tesseract-ocr-ben
**Check:**
tesseract --list-langs

**Generate embeddings and save in Qdrant**
Chunk + Vector + Document ID + Page Number

**8. Create the PDF ingestion script**
Now we will connect all three components:
- PyMuPDF extracts PDF text.
- Ollama generates embeddings.
- Qdrant stores the vectors and document information.


  Install the PDF processing environment
  




