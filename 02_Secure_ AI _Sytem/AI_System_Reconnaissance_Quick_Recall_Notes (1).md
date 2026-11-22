# AI System Reconnaissance — Quick Recall Notes

> **Goal:** Discover AI/ML services, identify the framework behind them, safely inspect exposed metadata, and map how the components connect.
>
> **Core flow:** Discover → Fingerprint → Enumerate → Map → Report.

---

## 1. What is AI Reconnaissance?

Traditional recon looks for web servers, SSH, and databases. AI recon looks for the extra services that build, store, serve, and monitor machine-learning systems.

Think of an AI deployment as a **factory**:
- **Model server** = machine that produces predictions.
- **MLflow / registry** = project records and model catalogue.
- **Vector database** = searchable memory for a RAG chatbot.
- **Jupyter** = engineer's workbench.
- **Object storage** = warehouse holding model files.
- **Prometheus** = factory monitoring dashboard.

Finding one service is useful. Finding how they trust and communicate with one another is the real attack-surface map.

## 2. AI Infrastructure Stack — What to Recognise

| Component | What it does | Common ports | Useful clues / endpoints |
|---|---|---:|---|
| NVIDIA Triton | Serves predictions from ML models | 8000 HTTP, 8001 gRPC, 8002 metrics | `/v2/health/ready`, `/v2/models` |
| TensorFlow Serving | Serves TensorFlow models | 8500 gRPC, 8501 HTTP | `/v1/models/<name>` |
| TorchServe | Serves PyTorch models | 8080, 8081 management, 8082 metrics | `/ping`, `/models` |
| Ollama | Runs local LLMs | 11434 | `/api/tags`, `/api/show` |
| vLLM | High-throughput LLM serving | 8000 | `/v1/models` |
| MLflow | Tracks experiments and model versions | 5000 | `/api/2.0/mlflow/` |
| Kubeflow | Runs ML pipelines on Kubernetes | 80, 443 | `/pipeline/apis/v1beta1/pipelines` |
| Ray | Distributed AI compute and job dashboard | 8265, 8000 | Dashboard, `/api/jobs/` |
| Qdrant | Vector database | 6333 HTTP, 6334 gRPC | `/collections` |
| Weaviate | Vector database with GraphQL | 8080 | `/v1/meta`, `/v1/schema` |
| Milvus | Distributed vector database | 19530 gRPC | gRPC service |
| Jupyter | Interactive notebooks | 8888 | `/api/kernels`, `/api/contents` |
| MinIO | S3-compatible object storage | 9000, 9001 | Bucket / object APIs |
| Prometheus metrics | Monitoring data exposed by services | Often 8002 / 8082 | `/metrics` |

**Memory hook:** 5000 = MLflow, 8265 = Ray, 8888 = Jupyter, 11434 = Ollama, 6333 = Qdrant, 19530 = Milvus.

Ports are defaults, not proof. Services can be moved, proxied, or hidden.

## 3. Fingerprinting — “Who is running here?”

Fingerprinting identifies the technology behind an open port. Do not rely only on Nmap's service label.

Look at four clues:

1. **Headers** — e.g. `Server: TorchServe/...`, `server: uvicorn`, or Triton's `NV-Status`.
2. **JSON body** — framework-specific field names and response shapes.
3. **Errors** — verbose errors may reveal package names, paths, or framework namespaces.
4. **Routes** — `/v2/models`, `/api/2.0/mlflow/`, `/v1/schema`, `/api/tags`.

### Fast signature table

| Clue | Likely service / family |
|---|---|
| `Server: TorchServe/...` | TorchServe |
| `NV-Status` header | NVIDIA Triton |
| `server: uvicorn` plus `/predict` or `/embeddings` | FastAPI-based ML API (not definitive alone) |
| `object: "model"` from `/v1/models` | OpenAI-compatible API; could be vLLM, Ollama, LiteLLM, or a wrapper |
| `tensorinfo_map` in an error | TensorFlow Serving clue |
| `mlflow.server` / `mlflow.tracking` in errors | MLflow |
| `/api/2.0/mlflow/` route prefix | MLflow |
| `/v1/schema` and `/v1/meta` | Weaviate clues |

**Important:** A clue is not always conclusive. Correlate multiple indicators and avoid sending malformed requests to production systems unless explicitly authorised.

### gRPC fingerprinting

gRPC is a binary RPC protocol, so ordinary HTTP probes may not identify it. `grpcurl` can inspect services when reflection is enabled.

```bash
grpcurl -plaintext <host>:8001 list
grpcurl -plaintext <host>:8001 describe inference.GRPCInferenceService
```

- `list` = list reflected gRPC services.
- `describe` = show a service's methods and message definitions.
- `-plaintext` = connect without TLS; use only when the endpoint is configured for plaintext and the test is authorised.
- Reflection disabled? These commands may not reveal the schema.

### TLS fingerprints

JA3 / JA4 are fingerprints derived from TLS handshake characteristics. They can help distinguish automated client traffic from ordinary browser traffic and correlate activity across IPs. They do **not** identify a specific AI framework by themselves.

## 4. Enumeration — “What information does it expose?”

Fingerprinting = identify the service.
Enumeration = collect its accessible inventory and metadata.

### MLflow — high-value inventory

| What to inspect | Endpoint | What it may reveal |
|---|---|---|
| Experiments | `POST /api/2.0/mlflow/experiments/search` | Experiment names and IDs |
| Registered models | `GET /api/2.0/mlflow/registered-models/list` | Model inventory |
| Model versions | `GET /api/2.0/mlflow/model-versions/search` | Version, stage, creator, artifact URI |
| Training runs | `POST /api/2.0/mlflow/runs/search` | Parameters, metrics, tags, run metadata |
| Artifacts | `GET /api/2.0/mlflow/artifacts/list` | Artifact names / paths, depending on API and permissions |

**Why it matters:** model names can expose business goals; version metadata can reveal production status and cloud-storage paths; run tags may expose project names or source-control references.

### Model-serving metadata

- **Triton:** `/v2/models/<name>/config` — model inputs/outputs, tensor names, dimensions, data types, backend, and batching configuration.
- **TensorFlow Serving:** `/v1/models/<name>/metadata` — model signature and tensor metadata.
- This information can help a legitimate tester understand the expected request format. It should not be used to access or extract data outside the engagement scope.

### Vector database metadata

| Service | Endpoint | What it tells you |
|---|---|---|
| Weaviate | `/v1/meta` | Version and modules |
| Weaviate | `/v1/schema` | Classes, properties, vectorizer configuration |
| Qdrant | `/collections` | Collection names |
| Qdrant | `/collections/<name>` | Vector configuration, dimensions, point count |
| Chroma (version-dependent) | `/api/v1/collections` | Collection inventory |

A collection name such as `internal-hr-policies` can reveal the type of information indexed. Vector dimensions alone do **not** prove which embedding model was used.

### Monitoring, notebooks, and debug interfaces

- **Prometheus `/metrics`:** may reveal model names, versions, request counts, latency, batch sizes, and GPU usage. This can expose deployment topology.
- **FastAPI `/docs` and `/openapi.json`:** may reveal routes and request/response schemas.
- **Jupyter `/api/kernels`:** may reveal running kernel metadata.
- **Jupyter `/api/contents`:** may list notebook files when access is permitted.
- Notebook cells or configuration files may contain secrets. Treat any discovered credentials as sensitive evidence: do not use, copy, or test them beyond the explicit rules of engagement. Report and request secure handling.

## 5. Map the Attack Surface — Connect the Dots

Example trust chain:

```text
Jupyter notebook
   | credentials / configuration
   v
MLflow tracking + model registry
   | artifact URI
   v
S3 / MinIO object storage
   |
   v
Model-serving endpoint (Triton / vLLM / TorchServe)
   |
   +----> Vector database (RAG data)
   |
   +----> Prometheus metrics
```

Ask these questions:
- Which service can reach which other service?
- Are admin or management APIs exposed?
- Are credentials stored in notebooks, environment variables, or CI logs?
- Can a model registry point to cloud storage?
- Are metrics and debug endpoints reachable by ordinary users?
- Is internal traffic protected with authentication and, where appropriate, mTLS?

**Key idea:** one exposed component can reveal the location and access path of several others. The attack surface includes internal service-to-service connections, not just internet-facing ports.

## 6. Supply-chain Reconnaissance

AI systems rely on more than their own servers.

- **Tokens:** Hugging Face tokens or cloud keys can appear in `.env` files, notebooks, CI logs, or repositories.
- **Dependencies:** internal package names in `requirements.txt` may create dependency-confusion risk if package resolution is unsafe.
- **Model sources:** configuration files and build logs may show downloads from Hugging Face Hub, PyTorch Hub, or private registries.
- **Artifacts:** S3, GCS, and MinIO policies may accidentally expose model files or datasets.
- **Containers:** public images may contain configuration or secrets that should never have been baked in.

Review only repositories, registries, and assets that are in scope. Do not attempt to claim leaked tokens or modify upstream packages.

## 7. MITRE ATLAS — Room Mapping

ATLAS provides a shared vocabulary for adversarial ML activity.

| Recon activity | Technique from the room |
|---|---|
| Port scanning and endpoint probing | `AML.T0006` — Active Scanning |
| Finding model registries and training artifacts | `AML.T0007` — Discover ML Artifacts |
| Finding exposed tokens / risky dependencies | `AML.T0010` — ML Supply Chain Compromise |
| Identifying model family / API format | `AML.T0014` — Discover ML Model Family |
| Public research material discovery | `AML.T0000` — Search for Victim's Publicly Available Research Materials |
| Overall phase | `AML.TA0002` — Reconnaissance |

Use technique IDs as reporting labels, and verify the current ATLAS catalogue when writing a formal assessment because framework mappings can change.

## 8. Case Study Memory Hooks

### IBM X-Force Red — ML training infrastructure

**Chain:** compromised data-scientist access → notebook contained MLflow credentials → MLflow inventory exposed model details → artifact URIs pointed to storage.

**Lesson:** a secure model server is not enough if notebooks, credentials, registries, and storage are poorly protected.

### ShadowRay — Ray dashboard

**Chain:** exposed Ray dashboard on port 8265 → unauthenticated job submission → host reconnaissance and environment-variable theft → cloud credential abuse → lateral movement / compute hijacking.

**Lesson:** an exposed orchestration or job-submission interface can become a route into the wider environment. Restrict access and authenticate management interfaces.

## 9. Five-Phase AI Recon Method

1. **Passive recon:** review public service banners, company research, public repositories, container images, and job listings.
2. **Active scanning:** scan only authorised IPs and AI-specific ports; identify HTTP and gRPC services.
3. **API fingerprinting:** inspect headers, JSON, routes, and documented metadata endpoints.
4. **Metadata enumeration:** inventory experiments, models, tensor configs, vector collections, metrics, and notebook metadata within scope.
5. **Supply-chain review:** assess tokens, package dependencies, model download sources, container images, and artifact-storage permissions.

### What defenders may see in logs

- Rapid repeated requests to `/v2/models`.
- Scripted MLflow calls to model and version listing APIs.
- `/metrics` requests from unknown source addresses.
- Sequential scans across ports such as 5000, 8000, 8265, and 8888.
- Jupyter API requests without a valid session.
- Traversal-like strings such as `../` in artifact paths.

## 10. Defensive Quick Wins

- Put MLflow, Ray, Kubeflow, Jupyter, and management APIs behind strong authentication and private network access.
- Do not expose development dashboards or notebook servers directly to the internet.
- Restrict AI service ports and metrics endpoints to approved networks.
- Use scoped, short-lived secrets; never hardcode tokens in notebooks or images.
- Audit S3 / MinIO bucket policies and model-registry artifact permissions.
- Disable unnecessary debug output and verbose errors in production.
- Use TLS for service communication and mTLS where service identity is required.
- Log and alert on unusual enumeration, management API access, and unexpected metrics scraping.

---

# Command Cheat Sheet — AI Recon

> Run these only against systems you own or have explicit permission to test. Replace `<target>` and `<port>` with authorised values.

## A. Nmap — discover AI ports

```bash
nmap -p 5000,6333,8000,8001,8002,8080,8081,8082,8265,8500,8501,8888,9000,9001,11434,19530 -sV <target>
```

- `-p` = ports to scan.
- `-sV` = attempt service/version detection.
- `<target>` = authorised IP or hostname.

Optional HTTP title/header scripts:

```bash
nmap -p 5000,8000,8080,8265,8888 -sV --script http-title,http-headers <target>
```

- `--script` = run selected Nmap NSE scripts.
- `http-title` = retrieve page title where supported.
- `http-headers` = inspect HTTP response headers.

## B. curl — inspect an HTTP endpoint

```bash
curl -i http://<target>:5000/
```

- `curl` = make an HTTP request.
- `-i` = include response headers as well as the body.
- Useful for identifying server headers and response format.

```bash
curl -s http://<target>:11434/api/tags
```

- `-s` = silent mode (less progress/error noise).
- `/api/tags` = Ollama model listing endpoint.

```bash
curl -s http://<target>:8000/v2/health/ready
```

- Checks Triton readiness endpoint.
- A successful response suggests the server is ready; interpret status/body in context.

## C. curl — MLflow metadata examples

```bash
curl -s -X POST \
  http://<target>:5000/api/2.0/mlflow/experiments/search \
  -H "Content-Type: application/json" \
  -d '{}'
```

- `-X POST` = use POST.
- `-H` = add an HTTP header.
- `Content-Type: application/json` = tell the server the body is JSON.
- `-d '{}'` = send an empty JSON object.

```bash
curl -s http://<target>:5000/api/2.0/mlflow/registered-models/list
curl -s http://<target>:5000/api/2.0/mlflow/model-versions/search
```

- First request lists registered models.
- Second searches model versions; access and required parameters may vary by MLflow version/configuration.

## D. curl — model and vector metadata

```bash
curl -s http://<target>:8000/v2/models/<model-name>/config
```

- Triton model configuration: tensor names, shapes, data types, and backend details.

```bash
curl -s http://<target>:6333/collections
curl -s http://<target>:6333/collections/<collection-name>
```

- First lists Qdrant collections.
- Second returns details for one collection.

```bash
curl -s http://<target>:8888/api/kernels
curl -s http://<target>:8888/api/contents
```

- Jupyter kernel metadata and notebook contents listing.
- Authentication may be required; do not attempt to bypass it.

## E. grpcurl — inspect gRPC services

```bash
grpcurl -plaintext <target>:8001 list
grpcurl -plaintext <target>:8001 describe inference.GRPCInferenceService
```

- `-plaintext` = no TLS.
- `list` = list services if reflection is enabled.
- `describe` = inspect a service definition.
- Change port to `8500` when checking a TensorFlow Serving gRPC endpoint, if authorised.

## F. ffuf — discover known AI routes

Create a small text file named `ai-paths.txt` containing paths such as:

```text
v1/models
v2/models
v2/health/ready
api/2.0/mlflow/experiments/search
api/2.0/mlflow/registered-models/list
v1/schema
v1/meta
api/kernels
api/contents
openapi.json
docs
metrics
api/tags
collections
healthz
ping
```

Then:

```bash
ffuf -w ai-paths.txt -u http://<target>:<port>/FUZZ
```

- `-w` = wordlist file.
- `-u` = target URL.
- `FUZZ` = location where ffuf inserts each wordlist entry.
- Review status codes and response sizes; a `200` alone does not prove a route is sensitive or vulnerable.

## G. Useful shell helpers

```bash
for port in 5000 8000 8001 8002 8265 8888 11434; do
  echo "Checking $port"
done
```

- `for` loop = repeat an action for each listed port.
- `echo` = print the current port.
- This example only prints port numbers; it does not scan them.

```bash
cat ai-paths.txt
```

- `cat` = display the wordlist contents.

```bash
grep -i "server\\|nv-status\\|request-id" headers.txt
```

- `grep` = search text.
- `-i` = case-insensitive.
- `\\|` in this example is grep's OR operator in basic regular expressions.
- Searches saved headers for common fingerprint clues.

---

## 11. Final Recall — Test Yourself

1. What is the difference between fingerprinting and enumeration?
2. Which service commonly uses port 5000?
3. Which port is commonly associated with the Ray dashboard?
4. Which service commonly exposes `/api/tags`?
5. What can Triton's model config reveal?
6. What can Weaviate's `/v1/schema` reveal?
7. Why is an exposed Jupyter server risky?
8. What is the purpose of `grpcurl list`?
9. What does `/metrics` potentially reveal?
10. What is the five-phase workflow?

### Answer Key

1. Fingerprinting identifies the service; enumeration collects its accessible details.
2. MLflow.
3. 8265.
4. Ollama.
5. Tensor names, shapes, data types, backend, and batching configuration.
6. Classes, properties, and vectorizer configuration.
7. It may expose notebook metadata, code, configuration, or accidentally stored secrets.
8. Lists reflected gRPC services.
9. Model names, versions, traffic, latency, batch sizes, and GPU usage.
10. Passive recon → active scanning → API fingerprinting → metadata extraction → supply-chain review.

**One-line memory hook:** Find the AI service, prove what it is, learn what it exposes, connect it to the rest of the stack, then report the risk.
