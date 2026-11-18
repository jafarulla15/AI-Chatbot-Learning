# AI-Chatbot-Learning


**Get List of installed**
```
ollama list
```

Go to URL: cd /opt/enterprise-chatbot
Activate env : source rag-env/bin/activate


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







