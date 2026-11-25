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
  



**9. Verify that your PDF documents are stored in Qdrant**
```
curl http://127.0.0.1:6333/collections/earn_knowledge \
-H "api-key: YOUR_GENERATED_SECRET"
```




**10. Test document retrieval using a real question**








**11. Connect retrieved knowledge with Qwen3**

    

##Start RAG Answer:
**Create the final RAG answer-generation script**

cd /opt/enterprise-chatbot
source rag-env/bin/activate
nano rag_chat.py

**Execute the chatbot**

source /opt/enterprise-chatbot/rag-env/bin/activate
python /opt/enterprise-chatbot/rag_chat.py







**Disable thinking mode if you're using Qwen3**

```
payload = {
    "model": "qwen3:8b",
    "messages": messages,
    "think": False,
    "stream": False,
    "keep_alive": "30m",
    "options": {
        "temperature": 0.2,
        "num_ctx": 4096,
        "num_predict": 256,
        "num_thread": 16
    }
}
```








**Test Ollama independently of your RAG application:**
This is important because we need to identify whether the delay comes from the model or your Python RAG pipeline.
Run:
```
curl http://localhost:11434/api/chat \
-H "Content-Type: application/json" \
-d '{
  "model": "qwen3:8b",
  "messages": [
    {
      "role": "user",
      "content": "Explain an enterprise project management system in 5 sentences."
    }
  ],
  "think": false,
  "stream": false,
  "keep_alive": "30m",
  "options": {
    "num_ctx": 2048,
    "num_predict": 200,
    "num_thread": 16
  }
}' | jq '{
  answer: .message.content,
  load_seconds: (.load_duration / 1000000000),
  prompt_tokens: .prompt_eval_count,
  output_tokens: .eval_count,
  generation_seconds: (.eval_duration / 1000000000)
}'
```









# SQL Server database to your local LLM (Ollama)

**Step 1 – Connect Python to SQL Server**
```
pip install pyodbc requests python-dotenv sqlglot
```
Install Microsoft ODBC Driver 18 for SQL Server if it is not already installed.
Create a .env file:

```
DB_SERVER=192.168.1.100
DB_NAME=EARN_DB
DB_USER=chatbot_readonly
DB_PASSWORD=your_password

OLLAMA_URL=http://127.0.0.1:11434
OLLAMA_MODEL=your-model-name
```

First, connect to SQL Server:

sqlcmd -S localhost -U sa -C -d Earn

3. Install Python dependencies
Activate your existing Python virtual environment and install

pip install pyodbc requests python-dotenv sqlglot

4. Configure database connection
   nano /opt/enterprise-chatbot/.env

   ```
   OLLAMA_URL=http://127.0.0.1:11434

SQL_SERVER=127.0.0.1
SQL_PORT=1433
SQL_DATABASE=Earn
SQL_USERNAME=llm_reader
SQL_PASSWORD=YourStrongPassword123!
```


5. Test Python connection to SQL Server
Create test_sql.py:

```

import os
import pyodbc
from dotenv import load_dotenv

load_dotenv("/opt/enterprise-chatbot/.env")

conn_str = (
    "DRIVER={ODBC Driver 18 for SQL Server};"
    f"SERVER={os.getenv('SQL_SERVER')},{os.getenv('SQL_PORT')};"
    f"DATABASE={os.getenv('SQL_DATABASE')};"
    f"UID={os.getenv('SQL_USERNAME')};"
    f"PWD={os.getenv('SQL_PASSWORD')};"
    "Encrypt=yes;"
    "TrustServerCertificate=yes;"
)

conn = pyodbc.connect(conn_str)

cursor = conn.cursor()

cursor.execute("""
    SELECT TOP 10 *
    FROM [ben].[Neet]
""")

columns = [column[0] for column in cursor.description]

for row in cursor.fetchall():
    print(dict(zip(columns, row)))

conn.close()

```



2. Create a dedicated Python project structure

```
enterprise-chatbot/
│
├── .env
│
├── database/
│   ├── __init__.py
│   ├── connection.py
│   └── schema_retrieval.py
│
├── prompts/
│   └── sql_prompt.py
│
├── scripts/
│   └── inspect_schema.py
│
└── db_chat.py
```

Create database/schema_retrieval.py:

```

import json
from database.connection import get_connection


def get_database_schema(allowed_schemas=None):

    if allowed_schemas is None:
        allowed_schemas = ["ben"]

    conn = get_connection()
    cursor = conn.cursor()

    try:
        # --------------------------------------
        # 1. RETRIEVE TABLES AND COLUMN DETAILS
        # --------------------------------------

        placeholders = ",".join("?" for _ in allowed_schemas)

        column_query = f"""
            SELECT
                s.name AS schema_name,
                o.name AS table_name,
                o.type_desc AS object_type,
                c.name AS column_name,
                t.name AS data_type,
                c.max_length,
                c.precision,
                c.scale,
                c.is_nullable,
                c.is_identity,
                CASE
                    WHEN pk.column_id IS NOT NULL THEN 1
                    ELSE 0
                END AS is_primary_key

            FROM sys.objects o

            INNER JOIN sys.schemas s
                ON o.schema_id = s.schema_id

            INNER JOIN sys.columns c
                ON o.object_id = c.object_id

            INNER JOIN sys.types t
                ON c.user_type_id = t.user_type_id

            LEFT JOIN (
                SELECT
                    ic.object_id,
                    ic.column_id
                FROM sys.indexes i
                INNER JOIN sys.index_columns ic
                    ON i.object_id = ic.object_id
                    AND i.index_id = ic.index_id
                WHERE i.is_primary_key = 1
                  AND ic.key_ordinal > 0
            ) pk
                ON c.object_id = pk.object_id
                AND c.column_id = pk.column_id

            WHERE o.type IN ('U', 'V')
              AND s.name IN ({placeholders})

            ORDER BY
                s.name,
                o.name,
                c.column_id;
        """

        cursor.execute(column_query, allowed_schemas)

        schema = {}

        for row in cursor.fetchall():

            table_name = f"{row.schema_name}.{row.table_name}"

            if table_name not in schema:
                schema[table_name] = {
                    "object_type": row.object_type,
                    "columns": [],
                    "foreign_keys": []
                }

            schema[table_name]["columns"].append({
                "name": row.column_name,
                "data_type": row.data_type,
                "max_length": row.max_length,
                "precision": row.precision,
                "scale": row.scale,
                "nullable": bool(row.is_nullable),
                "identity": bool(row.is_identity),
                "primary_key": bool(row.is_primary_key)
            })

        # --------------------------------------
        # 2. RETRIEVE FOREIGN KEY RELATIONSHIPS
        # --------------------------------------

        fk_query = f"""
            SELECT
                ps.name AS parent_schema,
                pt.name AS parent_table,
                pc.name AS parent_column,

                rs.name AS referenced_schema,
                rt.name AS referenced_table,
                rc.name AS referenced_column,

                fk.name AS foreign_key_name

            FROM sys.foreign_keys fk

            INNER JOIN sys.foreign_key_columns fkc
                ON fk.object_id = fkc.constraint_object_id

            INNER JOIN sys.tables pt
                ON fkc.parent_object_id = pt.object_id

            INNER JOIN sys.schemas ps
                ON pt.schema_id = ps.schema_id

            INNER JOIN sys.columns pc
                ON fkc.parent_object_id = pc.object_id
                AND fkc.parent_column_id = pc.column_id

            INNER JOIN sys.tables rt
                ON fkc.referenced_object_id = rt.object_id

            INNER JOIN sys.schemas rs
                ON rt.schema_id = rs.schema_id

            INNER JOIN sys.columns rc
                ON fkc.referenced_object_id = rc.object_id
                AND fkc.referenced_column_id = rc.column_id

            WHERE ps.name IN ({placeholders})
              AND rs.name IN ({placeholders})

            ORDER BY pt.name, fk.name;
        """

        cursor.execute(
            fk_query,
            allowed_schemas + allowed_schemas
        )

        for row in cursor.fetchall():

            parent = f"{row.parent_schema}.{row.parent_table}"

            if parent not in schema:
                continue

            schema[parent]["foreign_keys"].append({
                "constraint": row.foreign_key_name,
                "column": row.parent_column,
                "references": (
                    f"{row.referenced_schema}."
                    f"{row.referenced_table}."
                    f"{row.referenced_column}"
                )
            })

        return schema

    finally:
        cursor.close()
        conn.close()


# --------------------------------------
# 3. EXPORT SCHEMA AS JSON
# --------------------------------------

def save_schema_to_file(
    output_file="/opt/enterprise-chatbot/schema.json"
):

    schema = get_database_schema()

    with open(output_file, "w", encoding="utf-8") as file:

        json.dump(
            schema,
            file,
            indent=4,
            ensure_ascii=False
        )

    print(f"Schema saved successfully: {output_file}")

    return schema


```












