# AI-Chatbot-Learning


**Get List of installed**
```
ollama list
```

Go to URL: cd /opt/enterprise-chatbot
Activate env : source rag-env/bin/activate

Qdrant Vector Database
/opt/enterprise-chatbot/docker/qdrant/docker-compose.yml

5. Install the embedding model
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

**Create the PDF ingestion script**
Now we will connect all three components:
- PyMuPDF extracts PDF text.
- Ollama generates embeddings.
- Qdrant stores the vectors and document information.


  Install the PDF processing environment
  




