# Private AI (LibreChat on AWS Bedrock) — Native Setup (no Docker)

A private chat + document AI for your team. Everything runs inside **your own AWS
account**: Claude answers via **Bedrock**, documents are embedded via **Amazon Titan**
(also Bedrock), and files are stored in **your S3 bucket**. No consumer AI service
(ChatGPT, Claude.ai) is ever contacted, and nothing is used for training.

This project is the **full LibreChat source**, run natively with Node + local services
(no Docker).

```
Staff browser ──► LibreChat (Node)  ──► Bedrock  (Claude chat + Titan embeddings)
                       │
                       ├─► MongoDB            (chat history, users)      — local :27017
                       ├─► Postgres+pgvector  (document search / RAG)     — local :5432
                       ├─► rag_api (Python)   (embedding service)         — local :8000
                       └─► S3 bucket          (uploaded files)            — your AWS
```

---

## Part A — AWS setup (one time)

### 1. Account + guardrails
1. Create a **business AWS account**; enable **MFA** on root, then stop using root.
2. **AWS Budgets** → monthly budget + email alert.

### 2. Enable the models in Bedrock (us-east-1)
Bedrock console → **Model access** → request:
- **Anthropic Claude** models you want to use
- **Amazon Titan Text Embeddings V2**  ← required for document search

### 3. Create a private S3 bucket
S3 → create bucket (e.g. `virtualheight-librechat-files`), **Block all public access ON**,
enable default encryption + versioning.

### 4. Create the IAM user
IAM → user `librechat-app` → attach the inline policy in [`iam-policy.json`](iam-policy.json)
(replace the bucket name) → create an **access key**. Put it in `.env`.

---

## Part B — Local services

### MongoDB (chat history, users)  — already installed here
Start it (as your user, data stored in `./data-node`):
```bash
mongod --dbpath ./data-node --port 27017 --bind_ip 127.0.0.1
```
Keep it running in its own terminal (or set up a systemd service on EC2).

### Postgres + pgvector (document search / RAG)
Required only for **uploading documents and asking about them**. Chat works without it.
```bash
sudo apt-get update
sudo apt-get install -y postgresql postgresql-contrib
# install pgvector:
sudo apt-get install -y postgresql-16-pgvector    # match your postgres version
sudo -u postgres psql -c "CREATE USER myuser WITH PASSWORD 'change_me_strong_db_password';"
sudo -u postgres psql -c "CREATE DATABASE mydatabase OWNER myuser;"
sudo -u postgres psql -d mydatabase -c "CREATE EXTENSION IF NOT EXISTS vector;"
```
Set the same password in `.env` (`POSTGRES_PASSWORD`).

### rag_api (Python embedding service → Bedrock Titan)
This is the privacy-critical piece: it embeds documents through **Bedrock**, not OpenAI.
```bash
cd ~                       # outside this project folder
git clone https://github.com/danny-avila/rag_api.git
cd rag_api
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
# it reads the SAME AWS creds + EMBEDDINGS_* settings; point it at this project's .env
# or export: EMBEDDINGS_PROVIDER=bedrock, EMBEDDINGS_MODEL=amazon.titan-embed-text-v2:0,
#            AWS_DEFAULT_REGION, AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, DB_HOST=127.0.0.1
uvicorn main:app --host 127.0.0.1 --port 8000
```

---

## Part C — Run LibreChat

### 1. Fill in `.env`
Replace every `<<< FILL IN`: the IAM access key (3 places — Bedrock chat, embeddings, S3),
`AWS_BUCKET_NAME`, `POSTGRES_PASSWORD`, and (on a server) `DOMAIN_CLIENT`/`DOMAIN_SERVER`.
Encryption secrets are already generated.

### 2. Install + build (one time)
```bash
npm ci
npm run build:packages
npm run frontend        # builds the client UI
```

### 3. Start the backend
```bash
npm run backend         # serves on http://localhost:3080
```
For development with auto-reload use `npm run backend:dev`.

### 4. Create your first login
`ALLOW_REGISTRATION=false` closes sign-ups. For the **first admin**: temporarily set
`ALLOW_REGISTRATION=true`, start the backend, register in the browser, then set it back to
`false` and restart. Add teammates the same way.

---

## Part D — Deploy on EC2 (native)
- Ubuntu 22.04, t3.large, 30+ GB disk.
- **Security group:** SSH (22) from your office IP only; HTTPS (443) for staff. Do **not**
  expose 3080 / 27017 / 5432 / 8000.
- Install Node 20+, MongoDB, Postgres+pgvector as above.
- Run MongoDB, Postgres, rag_api, and LibreChat as **systemd services** so they survive reboots
  (ask me and I'll generate the unit files).
- Put **Caddy** or **nginx** on 443 to terminate TLS and proxy to `127.0.0.1:3080`.

---

## Part E — Verify & lock down
- **Model invocation logging: leave OFF** in Bedrock (default) — otherwise prompts get copied to S3.
- **Backups:** snapshot the box / back up `data-node/` + Postgres; enable S3 versioning.
- **Test:** log in, chat (confirm Claude answers), then upload a dummy document and ask about it
  (confirms the Titan embedding path, not OpenAI).

### Why nothing leaks
- `ENDPOINTS=bedrock` → chat only to Bedrock.
- `EMBEDDINGS_PROVIDER=bedrock` → document indexing only to Bedrock (Titan).
- `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` blank → those providers can't be used by accident.
