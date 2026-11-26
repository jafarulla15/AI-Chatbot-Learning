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
│   └── export_database_schema.py
│
├── prompts/
│   └── sql_prompt.py
│
├── scripts/
│   └── inspect_schema.py
│
└── db_chat.py
```

Create /opt/enterprise-chatbot/database/export_database_schema.py:

```


import os
import json
import pyodbc

from datetime import datetime, timezone
from dotenv import load_dotenv


# ============================================================
# CONFIGURATION
# ============================================================

ENV_FILE = "/opt/enterprise-chatbot/.env"

OUTPUT_FILE = "/opt/enterprise-chatbot/earn_database_schema.json"

load_dotenv(ENV_FILE)

SQL_SERVER = os.getenv("SQL_SERVER", "127.0.0.1")
SQL_PORT = os.getenv("SQL_PORT", "1433")
SQL_DATABASE = os.getenv("SQL_DATABASE", "Earn")
SQL_USERNAME = os.getenv("SQL_USERNAME")
SQL_PASSWORD = os.getenv("SQL_PASSWORD")

ODBC_DRIVER = os.getenv(
    "ODBC_DRIVER",
    "ODBC Driver 18 for SQL Server"
)


# ============================================================
# DATABASE CONNECTION
# ============================================================

def get_connection():

    if not SQL_USERNAME or not SQL_PASSWORD:
        raise ValueError(
            "SQL_USERNAME or SQL_PASSWORD missing in .env"
        )

    connection_string = (
        f"DRIVER={{{ODBC_DRIVER}}};"
        f"SERVER={SQL_SERVER},{SQL_PORT};"
        f"DATABASE={SQL_DATABASE};"
        f"UID={SQL_USERNAME};"
        f"PWD={SQL_PASSWORD};"
        "Encrypt=yes;"
        "TrustServerCertificate=yes;"
        "Connection Timeout=30;"
    )

    return pyodbc.connect(connection_string)


# ============================================================
# GENERIC QUERY EXECUTION
# ============================================================

def fetch_all(cursor, query, params=None):

    cursor.execute(query, params or [])

    columns = [
        column[0]
        for column in cursor.description
    ]

    rows = cursor.fetchall()

    return [
        dict(zip(columns, row))
        for row in rows
    ]


def fetch_one(cursor, query, params=None):

    cursor.execute(query, params or [])

    row = cursor.fetchone()

    if row is None:
        return None

    columns = [
        column[0]
        for column in cursor.description
    ]

    return dict(zip(columns, row))


# ============================================================
# DATABASE INFORMATION
# ============================================================

def get_database_info(cursor):

    query = """
        SELECT
            DB_NAME() AS database_name,
            CAST(SERVERPROPERTY('ServerName') AS NVARCHAR(256))
                AS server_name,
            CAST(SERVERPROPERTY('ProductVersion') AS NVARCHAR(100))
                AS sql_server_version,
            CAST(SERVERPROPERTY('Edition') AS NVARCHAR(256))
                AS sql_server_edition,
            compatibility_level,
            collation_name,
            create_date
        FROM sys.databases
        WHERE name = DB_NAME();
    """

    return fetch_one(cursor, query)


# ============================================================
# SCHEMAS
# ============================================================

def get_schemas(cursor):

    query = """
        SELECT
            schema_id,
            name AS schema_name
        FROM sys.schemas
        WHERE name NOT IN (
            'sys',
            'INFORMATION_SCHEMA'
        )
        ORDER BY name;
    """

    return fetch_all(cursor, query)


# ============================================================
# TABLES AND VIEWS
# ============================================================

def get_tables_and_views(cursor):

    query = """
        SELECT
            s.name AS schema_name,
            o.name AS object_name,
            o.object_id,
            o.type AS object_type,
            o.type_desc AS object_type_desc,

            CAST(ep.value AS NVARCHAR(MAX))
                AS description,

            CASE
                WHEN o.type = 'V'
                THEN OBJECT_DEFINITION(o.object_id)
                ELSE NULL
            END AS view_definition,

            CAST(NULL AS BIGINT)
                AS estimated_row_count

        FROM sys.objects o

        INNER JOIN sys.schemas s
            ON o.schema_id = s.schema_id

        LEFT JOIN sys.extended_properties ep
            ON ep.class = 1
            AND ep.major_id = o.object_id
            AND ep.minor_id = 0
            AND ep.name = 'MS_Description'

        WHERE
            o.type IN ('U', 'V')
            AND o.is_ms_shipped = 0

        ORDER BY
            s.name,
            o.name;
    """

    return fetch_all(cursor, query)


# ============================================================
# COLUMNS
# ============================================================

def get_columns(cursor):

    query = """
        SELECT
            s.name AS schema_name,
            o.name AS object_name,
            o.object_id,

            c.column_id,
            c.name AS column_name,

            ts.name AS type_schema,
            t.name AS data_type,

            c.max_length,
            c.precision,
            c.scale,

            c.is_nullable,
            c.is_identity,
            c.is_computed,

            dc.definition AS default_definition,
            cc.definition AS computed_definition,

            c.collation_name,

            CAST(ep.value AS NVARCHAR(MAX))
                AS description

        FROM sys.columns c

        INNER JOIN sys.objects o
            ON c.object_id = o.object_id

        INNER JOIN sys.schemas s
            ON o.schema_id = s.schema_id

        INNER JOIN sys.types t
            ON c.user_type_id = t.user_type_id

        INNER JOIN sys.schemas ts
            ON t.schema_id = ts.schema_id

        LEFT JOIN sys.default_constraints dc
            ON c.default_object_id = dc.object_id

        LEFT JOIN sys.computed_columns cc
            ON c.object_id = cc.object_id
            AND c.column_id = cc.column_id

        LEFT JOIN sys.extended_properties ep
            ON ep.class = 1
            AND ep.major_id = c.object_id
            AND ep.minor_id = c.column_id
            AND ep.name = 'MS_Description'

        WHERE
            o.type IN ('U', 'V')
            AND o.is_ms_shipped = 0

        ORDER BY
            s.name,
            o.name,
            c.column_id;
    """

    return fetch_all(cursor, query)


# ============================================================
# INDEXES AND PRIMARY KEYS
# ============================================================

def get_indexes(cursor):

    query = """
        SELECT
            s.name AS schema_name,
            o.name AS object_name,
            o.object_id,

            i.name AS index_name,
            i.index_id,
            i.type_desc AS index_type,

            i.is_primary_key,
            i.is_unique,
            i.is_unique_constraint,
            i.is_disabled,

            ic.key_ordinal,
            ic.is_included_column,
            ic.is_descending_key,

            c.name AS column_name

        FROM sys.indexes i

        INNER JOIN sys.objects o
            ON i.object_id = o.object_id

        INNER JOIN sys.schemas s
            ON o.schema_id = s.schema_id

        LEFT JOIN sys.index_columns ic
            ON i.object_id = ic.object_id
            AND i.index_id = ic.index_id

        LEFT JOIN sys.columns c
            ON ic.object_id = c.object_id
            AND ic.column_id = c.column_id

        WHERE
            o.type IN ('U', 'V')
            AND o.is_ms_shipped = 0
            AND i.index_id > 0
            AND i.is_hypothetical = 0

        ORDER BY
            s.name,
            o.name,
            i.index_id,
            ic.key_ordinal,
            ic.index_column_id;
    """

    return fetch_all(cursor, query)


# ============================================================
# FOREIGN KEYS
# ============================================================

def get_foreign_keys(cursor):

    query = """
        SELECT

            fk.object_id AS foreign_key_id,
            fk.name AS foreign_key_name,

            fk.parent_object_id,
            fk.referenced_object_id,

            ps.name AS schema_name,
            pt.name AS table_name,

            pc.name AS column_name,

            rs.name AS referenced_schema_name,
            rt.name AS referenced_table_name,

            rc.name AS referenced_column_name,

            fkc.constraint_column_id,

            fk.delete_referential_action_desc
                AS on_delete,

            fk.update_referential_action_desc
                AS on_update,

            fk.is_disabled,
            fk.is_not_trusted

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

        WHERE
            pt.is_ms_shipped = 0

        ORDER BY
            ps.name,
            pt.name,
            fk.name,
            fkc.constraint_column_id;
    """

    return fetch_all(cursor, query)


# ============================================================
# CHECK CONSTRAINTS
# ============================================================

def get_check_constraints(cursor):

    query = """
        SELECT

            s.name AS schema_name,
            t.name AS table_name,

            cc.name AS constraint_name,
            cc.definition,

            cc.is_disabled,
            cc.is_not_trusted

        FROM sys.check_constraints cc

        INNER JOIN sys.tables t
            ON cc.parent_object_id = t.object_id

        INNER JOIN sys.schemas s
            ON t.schema_id = s.schema_id

        WHERE
            t.is_ms_shipped = 0

        ORDER BY
            s.name,
            t.name,
            cc.name;
    """

    return fetch_all(cursor, query)


# ============================================================
# DEFAULT CONSTRAINTS
# ============================================================

def get_default_constraints(cursor):

    query = """
        SELECT

            s.name AS schema_name,
            t.name AS table_name,

            c.name AS column_name,

            dc.name AS constraint_name,
            dc.definition

        FROM sys.default_constraints dc

        INNER JOIN sys.tables t
            ON dc.parent_object_id = t.object_id

        INNER JOIN sys.schemas s
            ON t.schema_id = s.schema_id

        INNER JOIN sys.columns c
            ON dc.parent_object_id = c.object_id
            AND dc.parent_column_id = c.column_id

        WHERE
            t.is_ms_shipped = 0

        ORDER BY
            s.name,
            t.name,
            c.column_id;
    """

    return fetch_all(cursor, query)


# ============================================================
# TABLE TRIGGERS
# ============================================================

def get_triggers(cursor):

    query = """
        SELECT

            s.name AS schema_name,
            t.name AS table_name,

            tr.name AS trigger_name,

            tr.is_disabled,
            tr.is_instead_of_trigger,

            OBJECT_DEFINITION(tr.object_id)
                AS trigger_definition

        FROM sys.triggers tr

        INNER JOIN sys.tables t
            ON tr.parent_id = t.object_id

        INNER JOIN sys.schemas s
            ON t.schema_id = s.schema_id

        WHERE
            tr.parent_class = 1
            AND tr.is_ms_shipped = 0

        ORDER BY
            s.name,
            t.name,
            tr.name;
    """

    return fetch_all(cursor, query)


# ============================================================
# ORGANIZE COLUMNS
# ============================================================

def organize_columns(columns):

    result = {}

    for col in columns:

        object_id = col["object_id"]

        item = {
            "column_id": col["column_id"],
            "column_name": col["column_name"],
            "data_type": col["data_type"],
            "type_schema": col["type_schema"],
            "max_length": col["max_length"],
            "precision": col["precision"],
            "scale": col["scale"],
            "is_nullable": bool(col["is_nullable"]),
            "is_identity": bool(col["is_identity"]),
            "is_computed": bool(col["is_computed"]),
            "default_definition": col["default_definition"],
            "computed_definition": col["computed_definition"],
            "collation_name": col["collation_name"],
            "description": col["description"]
        }

        result.setdefault(object_id, []).append(item)

    return result


# ============================================================
# ORGANIZE INDEXES
# ============================================================

def organize_indexes(indexes):

    result = {}

    for idx in indexes:

        object_id = idx["object_id"]
        index_id = idx["index_id"]

        if object_id not in result:
            result[object_id] = {}

        if index_id not in result[object_id]:

            result[object_id][index_id] = {
                "index_name": idx["index_name"],
                "index_type": idx["index_type"],
                "is_primary_key": bool(idx["is_primary_key"]),
                "is_unique": bool(idx["is_unique"]),
                "is_unique_constraint": bool(
                    idx["is_unique_constraint"]
                ),
                "is_disabled": bool(idx["is_disabled"]),
                "key_columns": [],
                "included_columns": []
            }

        if idx["column_name"]:

            column_info = {
                "column_name": idx["column_name"],
                "is_descending": bool(
                    idx["is_descending_key"]
                )
            }

            if idx["is_included_column"]:
                result[object_id][index_id][
                    "included_columns"
                ].append(column_info)

            else:
                result[object_id][index_id][
                    "key_columns"
                ].append(column_info)

    final_result = {}

    for object_id, indexes_by_id in result.items():

        final_result[object_id] = []

        for index in indexes_by_id.values():
            final_result[object_id].append(index)

    return final_result


# ============================================================
# ORGANIZE FOREIGN KEYS
# ============================================================

def organize_foreign_keys(foreign_keys):

    result = {}

    for fk in foreign_keys:

        fk_id = fk["foreign_key_id"]

        if fk_id not in result:

            result[fk_id] = {
                "foreign_key_name": fk["foreign_key_name"],
                "schema_name": fk["schema_name"],
                "table_name": fk["table_name"],

                "referenced_schema_name":
                    fk["referenced_schema_name"],

                "referenced_table_name":
                    fk["referenced_table_name"],

                "on_delete": fk["on_delete"],
                "on_update": fk["on_update"],

                "is_disabled": bool(fk["is_disabled"]),
                "is_not_trusted": bool(fk["is_not_trusted"]),

                "column_mappings": []
            }

        result[fk_id]["column_mappings"].append({

            "column_order": fk["constraint_column_id"],

            "column_name": fk["column_name"],

            "referenced_column_name":
                fk["referenced_column_name"]

        })

    return list(result.values())


# ============================================================
# ORGANIZE CHECK CONSTRAINTS
# ============================================================

def organize_checks(checks):

    result = {}

    for item in checks:

        key = (
            item["schema_name"],
            item["table_name"]
        )

        result.setdefault(key, []).append({

            "constraint_name": item["constraint_name"],
            "definition": item["definition"],
            "is_disabled": bool(item["is_disabled"]),
            "is_not_trusted": bool(item["is_not_trusted"])

        })

    return result


# ============================================================
# ORGANIZE DEFAULT CONSTRAINTS
# ============================================================

def organize_defaults(defaults):

    result = {}

    for item in defaults:

        key = (
            item["schema_name"],
            item["table_name"]
        )

        result.setdefault(key, []).append({

            "column_name": item["column_name"],
            "constraint_name": item["constraint_name"],
            "definition": item["definition"]

        })

    return result


# ============================================================
# ORGANIZE TRIGGERS
# ============================================================

def organize_triggers(triggers):

    result = {}

    for item in triggers:

        key = (
            item["schema_name"],
            item["table_name"]
        )

        result.setdefault(key, []).append({

            "trigger_name": item["trigger_name"],
            "is_disabled": bool(item["is_disabled"]),

            "is_instead_of_trigger": bool(
                item["is_instead_of_trigger"]
            ),

            "trigger_definition": item["trigger_definition"]

        })

    return result


# ============================================================
# MAIN EXPORT FUNCTION
# ============================================================

def export_database_schema():

    connection = None

    try:

        print("Connecting to SQL Server...")

        connection = get_connection()

        cursor = connection.cursor()

        print("Reading database information...")
        database_info = get_database_info(cursor)

        print("Reading schemas...")
        schemas = get_schemas(cursor)

        print("Reading tables and views...")
        objects = get_tables_and_views(cursor)

        print("Reading columns...")
        columns = get_columns(cursor)

        print("Reading indexes and primary keys...")
        indexes = get_indexes(cursor)

        print("Reading foreign keys...")
        foreign_keys = get_foreign_keys(cursor)

        print("Reading check constraints...")
        checks = get_check_constraints(cursor)

        print("Reading default constraints...")
        defaults = get_default_constraints(cursor)

        print("Reading triggers...")
        triggers = get_triggers(cursor)

        # ---------------------------------------------
        # Organize metadata
        # ---------------------------------------------

        columns_map = organize_columns(columns)

        indexes_map = organize_indexes(indexes)

        foreign_keys_list = organize_foreign_keys(
            foreign_keys
        )

        checks_map = organize_checks(checks)

        defaults_map = organize_defaults(defaults)

        triggers_map = organize_triggers(triggers)

        # ---------------------------------------------
        # Initialize schema structure
        # ---------------------------------------------

        schema_data = {}

        for schema in schemas:

            schema_name = schema["schema_name"]

            schema_data[schema_name] = {
                "tables": {},
                "views": {}
            }

        # ---------------------------------------------
        # Add tables and views
        # ---------------------------------------------

        for obj in objects:

            schema_name = obj["schema_name"]
            object_name = obj["object_name"]
            object_id = obj["object_id"]

            if schema_name not in schema_data:
                schema_data[schema_name] = {
                    "tables": {},
                    "views": {}
                }

            is_view = obj["object_type"] == "V"

            object_data = {

                "object_id": object_id,

                "object_type": obj["object_type_desc"],

                "description": obj["description"],

                "estimated_row_count":
                    obj["estimated_row_count"],

                "columns": columns_map.get(
                    object_id, []
                ),

                "indexes": indexes_map.get(
                    object_id, []
                ),

                "primary_key_columns": [],

                "foreign_keys": [],

                "check_constraints": [],

                "default_constraints": [],

                "triggers": []
            }

            if is_view:

                object_data["view_definition"] = (
                    obj["view_definition"]
                )

                schema_data[schema_name][
                    "views"
                ][object_name] = object_data

            else:

                object_data["foreign_keys"] = [
                    fk for fk in foreign_keys_list
                    if fk["schema_name"] == schema_name
                    and fk["table_name"] == object_name
                ]

                object_data["check_constraints"] = (
                    checks_map.get(
                        (schema_name, object_name), []
                    )
                )

                object_data["default_constraints"] = (
                    defaults_map.get(
                        (schema_name, object_name), []
                    )
                )

                object_data["triggers"] = (
                    triggers_map.get(
                        (schema_name, object_name), []
                    )
                )

                # Extract primary key columns
                for index in object_data["indexes"]:

                    if index["is_primary_key"]:

                        object_data[
                            "primary_key_columns"
                        ] = [
                            item["column_name"]
                            for item in index["key_columns"]
                        ]

                schema_data[schema_name][
                    "tables"
                ][object_name] = object_data

        # ---------------------------------------------
        # Final JSON structure
        # ---------------------------------------------

        output = {

            "export_metadata": {

                "exported_at": datetime.now(
                    timezone.utc
                ).isoformat(),

                "database_name": SQL_DATABASE,

                "exported_by": SQL_USERNAME,

                "export_version": "1.0"

            },

            "database": database_info,

            "schemas": schema_data

        }

        # ---------------------------------------------
        # Save JSON
        # ---------------------------------------------

        os.makedirs(
            os.path.dirname(OUTPUT_FILE),
            exist_ok=True
        )

        with open(
            OUTPUT_FILE,
            "w",
            encoding="utf-8"
        ) as file:

            json.dump(
                output,
                file,
                indent=4,
                ensure_ascii=False,
                default=str
            )

        # ---------------------------------------------
        # Summary
        # ---------------------------------------------

        total_tables = 0
        total_views = 0

        for schema_name, schema in schema_data.items():

            table_count = len(schema["tables"])
            view_count = len(schema["views"])

            total_tables += table_count
            total_views += view_count

            print(
                f"{schema_name}: "
                f"{table_count} tables, "
                f"{view_count} views"
            )

        print("\n====================================")
        print("DATABASE SCHEMA EXPORT SUCCESSFUL")
        print("====================================")

        print(f"Database: {SQL_DATABASE}")
        print(f"Total schemas: {len(schema_data)}")
        print(f"Total tables: {total_tables}")
        print(f"Total views: {total_views}")
        print(f"Total foreign keys: {len(foreign_keys_list)}")
        print(f"Output file: {OUTPUT_FILE}")

    except Exception as e:

        print("\nSCHEMA EXPORT FAILED")
        print(str(e))

        raise

    finally:

        if connection:
            connection.close()


# ============================================================
# ENTRY POINT
# ============================================================

if __name__ == "__main__":

    export_database_schema()



```

3. Check your .env file
Make sure /opt/enterprise-chatbot/.env contains:

```

SQL_SERVER=127.0.0.1
SQL_PORT=1433
SQL_DATABASE=Earn
SQL_USERNAME=llm_reader
SQL_PASSWORD=YOUR_DATABASE_PASSWORD

ODBC_DRIVER=ODBC Driver 18 for SQL Server
```

4. Install dependencies and execute

```
cd /opt/enterprise-chatbot

source venv/bin/activate

pip install pyodbc python-dotenv

python database/export_database_schema.py
```

5. Verify the exported JSON

ls -lh /opt/enterprise-chatbot/earn_database_schema.json


============================================================ 
6. Important: How we will use this JSON with Ollama

2. Prepare the project structure
```
/opt/enterprise-chatbot/
│
├── .env
├── earn_database_schema.json
│
├── database/
│   └── export_database_schema.py
│
├── text_to_sql/
│   └── text_to_sql.py
│
└── venv/

```

3. Install the required Python packages
pip install pyodbc requests python-dotenv sqlglot

4. Configure the .env file
nano /opt/enterprise-chatbot/.env

```

SQL_SERVER=127.0.0.1
SQL_PORT=1433
SQL_DATABASE=Earn
SQL_USERNAME=llm_reader
SQL_PASSWORD=EarnLLM@2026Strong!

ODBC_DRIVER=ODBC Driver 18 for SQL Server


# OLLAMA
OLLAMA_URL=http://127.0.0.1:11434
OLLAMA_MODEL=gpt-oss:20b

# TEXT TO SQL
SCHEMA_FILE=/opt/enterprise-chatbot/earn_database_schema.json
MAX_SCHEMA_TABLES=8
MAX_RESULT_ROWS=200
SQL_QUERY_TIMEOUT=60
OLLAMA_TIMEOUT=600

# Optional second LLM call to explain SQL results
EXPLAIN_RESULTS=false


```

6. Create the complete Text-to-SQL Python application
nano /opt/enterprise-chatbot/text_to_sql/text_to_sql.py

```
import os
import re
import json
import requests
import pyodbc
import sqlglot

from dotenv import load_dotenv
from sqlglot import exp
from decimal import Decimal
from datetime import date, datetime


# ============================================================
# CONFIGURATION
# ============================================================

load_dotenv("/opt/enterprise-chatbot/.env")

SQL_SERVER = os.getenv("SQL_SERVER", "127.0.0.1")
SQL_PORT = os.getenv("SQL_PORT", "1433")
SQL_DATABASE = os.getenv("SQL_DATABASE", "Earn")
SQL_USERNAME = os.getenv("SQL_USERNAME")
SQL_PASSWORD = os.getenv("SQL_PASSWORD")

ODBC_DRIVER = os.getenv(
    "ODBC_DRIVER",
    "ODBC Driver 18 for SQL Server"
)

OLLAMA_URL = os.getenv(
    "OLLAMA_URL",
    "http://127.0.0.1:11434"
).rstrip("/")

OLLAMA_MODEL = os.getenv("OLLAMA_MODEL", "gpt-oss:20b")

SCHEMA_FILE = os.getenv(
    "SCHEMA_FILE",
    "/opt/enterprise-chatbot/earn_database_schema.json"
)

MAX_SCHEMA_TABLES = int(os.getenv("MAX_SCHEMA_TABLES", "8"))
MAX_RESULT_ROWS = int(os.getenv("MAX_RESULT_ROWS", "200"))
SQL_QUERY_TIMEOUT = int(os.getenv("SQL_QUERY_TIMEOUT", "60"))
OLLAMA_TIMEOUT = int(os.getenv("OLLAMA_TIMEOUT", "600"))

EXPLAIN_RESULTS = (
    os.getenv("EXPLAIN_RESULTS", "false").lower() == "true"
)


# ============================================================
# LOAD DATABASE SCHEMA JSON
# ============================================================

def load_database_schema():

    if not os.path.exists(SCHEMA_FILE):
        raise FileNotFoundError(
            f"Schema JSON not found: {SCHEMA_FILE}"
        )

    with open(SCHEMA_FILE, "r", encoding="utf-8") as file:
        schema = json.load(file)

    if "schemas" not in schema:
        raise ValueError(
            "Invalid schema JSON: 'schemas' property missing"
        )

    return schema


# ============================================================
# BUILD TABLE AND VIEW CATALOG
# ============================================================

def build_object_catalog(schema_json):

    catalog = {}

    for schema_name, schema_data in schema_json["schemas"].items():

        for object_type in ("tables", "views"):

            for object_name, object_data in schema_data.get(
                object_type, {}
            ).items():

                key = (schema_name.lower(), object_name.lower())

                catalog[key] = {
                    "schema_name": schema_name,
                    "object_name": object_name,
                    "object_type": object_type,
                    "object_data": object_data
                }

    return catalog


# ============================================================
# SELECT RELEVANT TABLES
# ============================================================

STOP_WORDS = {
    "the", "is", "are", "was", "were", "how", "many",
    "what", "which", "who", "where", "when", "show",
    "give", "me", "list", "all", "total", "count",
    "number", "of", "in", "on", "at", "from", "to",
    "and", "or", "with", "for", "by", "a", "an",
    "registered", "registration", "please", "find"
}


def tokenize(text):

    words = re.findall(r"[a-zA-Z0-9_]+", text.lower())

    return {
        word for word in words
        if len(word) > 1 and word not in STOP_WORDS
    }


def get_object_search_text(item):

    data = item["object_data"]

    parts = [
        item["schema_name"],
        item["object_name"],
        item["object_type"]
    ]

    if data.get("description"):
        parts.append(str(data["description"]))

    for column in data.get("columns", []):

        parts.append(str(column.get("column_name", "")))
        parts.append(str(column.get("data_type", "")))

        if column.get("description"):
            parts.append(str(column["description"]))

    return " ".join(parts).lower()


def select_relevant_objects(question, catalog):

    question_tokens = tokenize(question)

    scores = []

    for key, item in catalog.items():

        search_text = get_object_search_text(item)
        object_tokens = tokenize(search_text)

        score = len(question_tokens.intersection(object_tokens))

        # Boost exact table-name matches.
        table_name = item["object_name"].lower()

        if table_name in question.lower():
            score += 5

        # Boost schema-name matches.
        schema_name = item["schema_name"].lower()

        if schema_name in question.lower():
            score += 2

        scores.append((score, key, item))

    scores.sort(key=lambda x: x[0], reverse=True)

    selected = [
        item for score, key, item in scores
        if score > 0
    ][:MAX_SCHEMA_TABLES]

    # If no useful keyword match, provide a small fallback set.
    if not selected:
        selected = [
            item for score, key, item in scores[:MAX_SCHEMA_TABLES]
        ]

    # Add one-hop FK-related objects to improve join awareness.
    selected_keys = {
        (item["schema_name"].lower(), item["object_name"].lower())
        for item in selected
    }

    related_items = []

    for item in list(selected):

        for fk in item["object_data"].get("foreign_keys", []):

            related_key = (
                fk["referenced_schema_name"].lower(),
                fk["referenced_table_name"].lower()
            )

            if related_key in catalog and related_key not in selected_keys:

                related_items.append(catalog[related_key])
                selected_keys.add(related_key)

    remaining_slots = max(
        0, MAX_SCHEMA_TABLES - len(selected)
    )

    selected.extend(related_items[:remaining_slots])

    return selected


# ============================================================
# CREATE COMPACT SCHEMA CONTEXT FOR OLLAMA
# ============================================================

def build_schema_context(selected_objects):

    context = []

    for item in selected_objects:

        data = item["object_data"]

        object_info = {
            "schema": item["schema_name"],
            "name": item["object_name"],
            "type": item["object_type"],
            "columns": [],
            "primary_key_columns": data.get(
                "primary_key_columns", []
            ),
            "foreign_keys": []
        }

        for column in data.get("columns", []):

            object_info["columns"].append({

                "name": column["column_name"],
                "type": column["data_type"],
                "nullable": column["is_nullable"],
                "identity": column["is_identity"],
                "description": column.get("description")

            })

        for fk in data.get("foreign_keys", []):

            object_info["foreign_keys"].append({

                "name": fk["foreign_key_name"],

                "columns": fk["column_mappings"],

                "references": (
                    f'{fk["referenced_schema_name"]}.'
                    f'{fk["referenced_table_name"]}'
                )

            })

        context.append(object_info)

    return json.dumps(
        context,
        ensure_ascii=False,
        indent=2
    )


# ============================================================
# OLLAMA: GENERATE SQL
# ============================================================

def generate_sql(question, schema_context):

    system_prompt = """
You are an expert Microsoft SQL Server T-SQL developer.

Your task is to generate a SQL query using ONLY the
database schema supplied by the application.

STRICT RULES:

1. Generate exactly one read-only SELECT statement.
2. Never generate INSERT, UPDATE, DELETE, DROP, ALTER,
   CREATE, TRUNCATE, EXEC, or stored procedure calls.
3. Do not generate multiple SQL statements.
4. Use only the supplied tables and views.
5. Always use schema-qualified table names.
   Example: [ben].[Neet]
6. Use only columns that exist in the supplied schema.
7. Use the supplied foreign keys when joining tables.
8. Do not invent columns, tables, relationships or values.
9. For count questions, use COUNT_BIG(*) or COUNT(*)
   as appropriate.
10. For a list or detail question, return useful columns,
    not SELECT *.
11. Use valid Microsoft SQL Server T-SQL syntax.
12. Do not include Markdown code fences.

Return ONLY a valid JSON object with this structure:

{
  "sql": "SELECT ...",
  "explanation": "Brief explanation of the query"
}

If the supplied schema is insufficient to answer the question,
return:

{
  "sql": "",
  "explanation": "The supplied database schema is insufficient..."
}
"""

    user_prompt = f"""
DATABASE SCHEMA:

{schema_context}

USER QUESTION:

{question}

Generate the SQL Server query.
"""

    payload = {

        "model": OLLAMA_MODEL,

        "stream": False,

        "format": "json",

        "options": {
            "temperature": 0,
            "num_predict": 400
        },

        "messages": [
            {
                "role": "system",
                "content": system_prompt
            },
            {
                "role": "user",
                "content": user_prompt
            }
        ]
    }

    response = requests.post(
        f"{OLLAMA_URL}/api/chat",
        json=payload,
        timeout=OLLAMA_TIMEOUT
    )

    response.raise_for_status()

    result = response.json()

    content = result["message"]["content"]

    try:
        generated = json.loads(content)

    except json.JSONDecodeError:
        raise ValueError(
            "Ollama did not return valid JSON:\n" + content
        )

    sql = generated.get("sql", "").strip()
    explanation = generated.get("explanation", "")

    if not sql:
        raise ValueError(
            explanation or "Ollama could not generate SQL."
        )

    return sql, explanation


# ============================================================
# REMOVE MARKDOWN FENCES
# ============================================================

def clean_sql(sql):

    sql = sql.strip()

    sql = re.sub(
        r"^```(?:sql)?\s*",
        "",
        sql,
        flags=re.IGNORECASE
    )

    sql = re.sub(
        r"\s*```$",
        "",
        sql
    )

    return sql.strip()


# ============================================================
# VALIDATE SQL
# ============================================================

def validate_sql(sql, catalog):

    sql = clean_sql(sql)

    # Reject SQL comments and suspicious batch separators.
    if "--" in sql or "/*" in sql or "*/" in sql:
        raise ValueError(
            "SQL comments are not allowed."
        )

    if re.search(r"(?im)^\s*GO\s*$", sql):
        raise ValueError(
            "SQL batch separators are not allowed."
        )

    # Parse exactly one T-SQL statement.
    statements = sqlglot.parse(
        sql,
        read="tsql"
    )

    if len(statements) != 1 or statements[0] is None:
        raise ValueError(
            "Only one SQL statement is allowed."
        )

    tree = statements[0]

    # Only SELECT queries are accepted.
    if not isinstance(tree, exp.Select):
        raise ValueError(
            "Only SELECT queries are allowed."
        )

    # Explicitly reject unsafe constructs.
    forbidden_nodes = (
        exp.Insert,
        exp.Update,
        exp.Delete,
        exp.Create,
        exp.Drop,
        exp.Alter,
        exp.Command,
        exp.Into
    )

    for node in tree.walk():

        if isinstance(node, forbidden_nodes):
            raise ValueError(
                f"Unsafe SQL operation detected: {type(node).__name__}"
            )

    # Identify CTE aliases, so references to CTEs
    # are not mistaken for physical database tables.
    cte_names = {
        cte.alias_or_name.lower()
        for cte in tree.find_all(exp.CTE)
    }

    # Validate physical table/view references.
    allowed_objects = set(catalog.keys())

    for table in tree.find_all(exp.Table):

        table_name = table.name.lower()
        schema_name = (table.db or "").lower()

        # CTE references are not physical tables.
        if not schema_name and table_name in cte_names:
            continue

        if not schema_name:
            raise ValueError(
                f"Schema qualification required for table: {table_name}"
            )

        key = (schema_name, table_name)

        if key not in allowed_objects:
            raise ValueError(
                "Table/view not found in exported schema: "
                f"{schema_name}.{table_name}"
            )

    # Use SQL Server syntax when returning the validated query.
    return tree.sql(dialect="tsql")


# ============================================================
# SQL SERVER CONNECTION
# ============================================================

def get_sql_connection():

    if not SQL_USERNAME or not SQL_PASSWORD:
        raise ValueError(
            "SQL_USERNAME or SQL_PASSWORD is missing in .env"
        )

    connection_string = (
        f"DRIVER={{{ODBC_DRIVER}}};"
        f"SERVER={SQL_SERVER},{SQL_PORT};"
        f"DATABASE={SQL_DATABASE};"
        f"UID={SQL_USERNAME};"
        f"PWD={SQL_PASSWORD};"
        "Encrypt=yes;"
        "TrustServerCertificate=yes;"
        "Connection Timeout=30;"
    )

    return pyodbc.connect(
        connection_string,
        autocommit=True,
        timeout=30
    )


# ============================================================
# CONVERT SQL VALUES TO JSON-SAFE VALUES
# ============================================================

def make_json_safe(value):

    if isinstance(value, Decimal):
        return float(value)

    if isinstance(value, (datetime, date)):
        return value.isoformat()

    if isinstance(value, bytes):
        return value.hex()

    return value


# ============================================================
# EXECUTE SQL QUERY
# ============================================================


def execute_sql(sql):

    connection = None

    try:
        connection = get_sql_connection()

        # Set query execution timeout on the connection.
        connection.timeout = SQL_QUERY_TIMEOUT

        cursor = connection.cursor()

        # Limit returned rows.
        cursor.execute(
            f"SET ROWCOUNT {MAX_RESULT_ROWS};"
        )

        # Avoid waiting indefinitely for locked records.
        cursor.execute(
            "SET LOCK_TIMEOUT 5000;"
        )

        print("\nExecuting SQL against SQL Server...")

        cursor.execute(sql)

        if cursor.description is None:
            raise ValueError(
                "The SQL statement did not return a result set."
            )

        columns = [
            column[0]
            for column in cursor.description
        ]

        rows = []

        for row in cursor.fetchall():

            rows.append({
                columns[i]: make_json_safe(value)
                for i, value in enumerate(row)
            })

        return {
            "columns": columns,
            "rows": rows,
            "row_count": len(rows),
            "max_rows": MAX_RESULT_ROWS
        }

    finally:

        if connection:
            connection.close()


# ============================================================
# OPTIONAL: OLLAMA EXPLAINS THE DATABASE RESULTS
# ============================================================

def explain_results(question, sql, result):

    prompt = f"""
The user asked:

{question}

SQL executed:

{sql}

SQL Server returned:

{json.dumps(result, ensure_ascii=False, default=str)}

Explain the result clearly and accurately.

Rules:
- Use only the returned data.
- Do not invent additional statistics.
- If no records were returned, say so.
- If the result is truncated, mention the limit.
- Keep the answer concise.
"""

    payload = {

        "model": OLLAMA_MODEL,

        "stream": False,

        "options": {
            "temperature": 0,
            "num_predict": 250
        },

        "messages": [
            {
                "role": "user",
                "content": prompt
            }
        ]
    }

    response = requests.post(
        f"{OLLAMA_URL}/api/chat",
        json=payload,
        timeout=OLLAMA_TIMEOUT
    )

    response.raise_for_status()

    return response.json()["message"]["content"]


# ============================================================
# MAIN TEXT-TO-SQL WORKFLOW
# ============================================================

def ask_database(question):

    print("\n========================================")
    print("EARN DATABASE - TEXT TO SQL")
    print("========================================")

    # 1. Load schema.
    print("\n[1] Loading database schema...")

    schema_json = load_database_schema()

    # 2. Build table catalog.
    print("[2] Building table catalog...")

    catalog = build_object_catalog(schema_json)

    print(f"    Available objects: {len(catalog)}")

    # 3. Select relevant tables.
    print("[3] Selecting relevant tables...")

    selected_objects = select_relevant_objects(
        question,
        catalog
    )

    print(
        "    Selected:",
        [
            f'{x["schema_name"]}.{x["object_name"]}'
            for x in selected_objects
        ]
    )

    if not selected_objects:
        raise ValueError(
            "No relevant tables or views were found."
        )

    # 4. Build compact schema context.
    print("[4] Preparing schema context...")

    schema_context = build_schema_context(
        selected_objects
    )

    # 5. Ask Ollama to generate SQL.
    print("[5] Sending question to Ollama...")

    generated_sql, sql_explanation = generate_sql(
        question,
        schema_context
    )

    print("\nGenerated SQL:")
    print("----------------------------------------")
    print(generated_sql)
    print("----------------------------------------")

    print("\nSQL explanation:")
    print(sql_explanation)

    # 6. Validate SQL.
    print("\n[6] Validating SQL...")

    validated_sql = validate_sql(
        generated_sql,
        catalog
    )

    # 7. Execute against SQL Server.
    print("[7] Executing validated SQL...")

    result = execute_sql(validated_sql)

    # 8. Display results.
    print("\n========================================")
    print("SQL SERVER RESULTS")
    print("========================================")

    print(f"Returned rows: {result['row_count']}")

    if result["row_count"] == 0:
        print("\nNo records found.")

    else:

        print("\nColumns:")
        print(" | ".join(result["columns"]))

        print("\nRecords:")

        for index, row in enumerate(result["rows"], start=1):

            print(f"\nRecord {index}")

            for column, value in row.items():
                print(f"  {column}: {value}")

    # 9. Optional natural-language explanation.
    if EXPLAIN_RESULTS:

        print("\n[8] Asking Ollama to explain the results...")

        answer = explain_results(
            question,
            validated_sql,
            result
        )

        print("\n========================================")
        print("NATURAL LANGUAGE ANSWER")
        print("========================================")

        print(answer)

    if result["row_count"] >= MAX_RESULT_ROWS:

        print(
            "\nNote: The result may be limited to "
            f"{MAX_RESULT_ROWS} rows."
        )

    return {
        "question": question,
        "sql": validated_sql,
        "sql_explanation": sql_explanation,
        "result": result
    }


# ============================================================
# INTERACTIVE TERMINAL
# ============================================================

def main():

    print("\nEARN DATABASE CHAT")
    print("Type 'exit' to quit.")

    while True:

        question = input(
            "\nAsk your database: "
        ).strip()

        if question.lower() in ("exit", "quit"):
            print("Goodbye!")
            break

        if not question:
            continue

        try:

            ask_database(question)

        except requests.exceptions.ConnectionError:

            print(
                "\nERROR: Cannot connect to Ollama. "
                "Check that Ollama is running."
            )

        except requests.exceptions.Timeout:

            print(
                "\nERROR: Ollama request timed out. "
                "Check model performance and OLLAMA_TIMEOUT."
            )

        except pyodbc.Error as e:

            print("\nSQL SERVER ERROR:")
            print(str(e))

        except Exception as e:

            print("\nAPPLICATION ERROR:")
            print(str(e))


if __name__ == "__main__":
    main()

```






