# INTELLIGENT LLM GATEWAY
## Cẩm nang triển khai MLOps & LLMOps từ con số 0 về Ops — bản tutorial thực hành

**Phiên bản tài liệu:** 1.0 · **Cập nhật:** 27/10/2026  
**Đối tượng:** Đã có nền tảng ML/DL/LLM và Python, nhưng chưa vững DevOps/MLOps/LLMOps.  
**Môi trường:** Ubuntu 24.04 LTS · không GPU · ưu tiên local, CPU-only.  
**Thời lượng:** 8 tuần chính + 1–2 tuần dự phòng, ước tính 12–18 giờ/tuần. Đây là kế hoạch học, không phải cam kết hoàn thành trong số giờ cố định.  
**Ngân sách:** 0 đồng cho phần cốt lõi dùng mock provider. Kết nối LLM thật là **tùy chọn**, có thể phát sinh phí API; không bắt buộc thuê VM/cloud.  
**Sản phẩm:** Một repository GitHub có source, tests, CI, Docker image, dashboards, evaluation reports, ML router và Kubernetes manifests.  
**Vai trò của file này:** **Tài liệu duy nhất để hiểu phạm vi, thứ tự làm, kỹ thuật, bài tập và tiêu chí nghiệm thu.** Các đoạn mã là hướng dẫn/mẫu để triển khai thành các file source trong repository; bản thân Markdown này không phải một chương trình có thể chạy trực tiếp.

---

# MỤC LỤC

1. [Mục tiêu, phạm vi và nguyên tắc](#1-mục-tiêu-phạm-vi-và-nguyên-tắc)
2. [Sản phẩm và use cases](#2-sản-phẩm-và-use-cases)
3. [Kiến trúc hệ thống](#3-kiến-trúc-hệ-thống)
4. [Tech stack: dùng gì, khi nào, vì sao](#4-tech-stack-dùng-gì-khi-nào-vì-sao)
5. [API contract, error contract và policies](#5-api-contract-error-contract-và-policies)
6. [Cấu trúc repository và quy ước kỹ thuật](#6-cấu-trúc-repository-và-quy-ước-kỹ-thuật)
7. [Khởi động project trên Ubuntu](#7-khởi-động-project-trên-ubuntu)
8. [Tuần 1: Linux, HTTP, networking, FastAPI, Docker](#8-tuần-1-linux-http-networking-fastapi-docker)
9. [Tuần 2: Reliability engineering](#9-tuần-2-reliability-engineering)
10. [Tuần 3: Logs, metrics, traces, load testing](#10-tuần-3-logs-metrics-traces-load-testing)
11. [Tuần 4: LLMOps, provider thật và evaluation](#11-tuần-4-llmops-provider-thật-và-evaluation)
12. [Tuần 5: MLOps và ML-based router](#12-tuần-5-mlops-và-ml-based-router)
13. [Tuần 6: CI/CD, images, release và rollback](#13-tuần-6-cicd-images-release-và-rollback)
14. [Tuần 7: Kubernetes với kind](#14-tuần-7-kubernetes-với-kind)
15. [Tuần 8: Hardening, runbook, capstone](#15-tuần-8-hardening-runbook-capstone)
16. [Bộ test matrix và các ngưỡng nghiệm thu](#16-bộ-test-matrix-và-các-ngưỡng-nghiệm-thu)
17. [Đánh giá, metrics và các hiểu lầm thường gặp](#17-đánh-giá-metrics-và-các-hiểu-lầm-thường-gặp)
18. [Thiết kế vận hành tiết kiệm tài nguyên](#18-thiết-kế-vận-hành-tiết-kiệm-tài-nguyên)
19. [Checklist cuối khóa và demo portfolio](#19-checklist-cuối-khóa-và-demo-portfolio)
20. [Troubleshooting và tài liệu chính thức](#20-troubleshooting-và-tài-liệu-chính-thức)

---

# 1. Mục tiêu, phạm vi và nguyên tắc

## 1.1. Bạn đang xây dựng chính xác cái gì?

Một **LLM API Gateway** đứng giữa ứng dụng và nhiều LLM provider. Client chỉ gọi **một endpoint**, Gateway chịu trách nhiệm:

- Xác thực client, giới hạn request và kiểm tra payload.
- Chọn provider theo rule, chi phí, độ sẵn sàng và chính sách; về sau bổ sung ML-based routing.
- Gọi API upstream bất đồng bộ, quản lý timeout, retry, fallback và circuit breaker.
- Trả response nhất quán; tùy chọn streaming qua Server-Sent Events (SSE).
- Quan sát bằng logs, metrics, traces; cảnh báo khi provider có lỗi.
- Theo dõi phiên bản policy/model, chạy evaluation trước khi release, rollback có kiểm soát.
- Có CI/CD, container, và triển khai thử trên Kubernetes local.

**Không xây:** chatbot UI lớn, RAG, vector database, training/fine-tune LLM, Kubernetes multi-node trên cloud, billing platform thật, hệ thống xác thực doanh nghiệp, hay tính năng hỗ trợ mọi provider giống LiteLLM. Những việc này tốn nguồn lực nhưng không làm tăng đáng kể mục tiêu học Ops ban đầu.

## 1.2. Những năng lực phải đạt sau project

| Nhóm | Bạn phải tự làm/giải thích được |
|---|---|
| Linux / Networking | Process, port, DNS, localhost vs container hostname, SSH cơ bản, curl, log |
| Backend / API | HTTP semantics, async/await, validation, status codes, authentication, cancellation |
| Docker | Dockerfile, image, container, Compose, network, volume, env/secrets |
| Reliability | Timeout, retry budget, exponential backoff, fallback, circuit breaker, rate limiting |
| Observability | JSON logs, correlation IDs, metrics, PromQL, trace spans, p95/p99 latency |
| LLMOps | Provider adapters, LLM usage/cost, prompt version, offline evaluation, streaming |
| MLOps | Data versioning, training pipeline, MLflow, model registry, reproducibility, promotion gates |
| CI/CD | PR test, build, GHCR release, immutable versions, staging, rollback |
| Kubernetes | Pod/Deployment/Service, probes, resource limits, rollout, port-forward, kubectl debug |
| Security | Secret hygiene, data masking, least privilege, admission/config validation, abuse protection |

## 1.3. Phạm vi ưu tiên theo nguyên tắc 80/20

**P0 — Bắt buộc:** mock providers, async gateway, routing rule, auth local, retry/fallback/timeout, tests, Docker Compose, metrics, trace/log, load test, CI, Kubernetes demo.  
**P1 — Rất nên có:** một real provider opt-in, prompt/eval dataset, DVC, MLflow, CPU routing model, immutable release manifest và rollback.  
**P2 — Nếu còn thời gian:** Redis-backed distributed limits, policy canary, OpenTelemetry collector riêng, Langfuse Cloud, ghép Grafana alerting, Docker image scanning, semantic cache.  
**Không ưu tiên:** autoscaler nhiều node, distributed tracing backend tự vận hành quá lớn, model serving GPU, infrastructure-as-code cho cloud, JWT/OAuth hoàn chỉnh.

**Định nghĩa 'production-ready' ở đây:** **production-minded/production-style learning project**, KHÔNG phải sản phẩm được chứng nhận an toàn để đưa lên Internet xử lý dữ liệu nhạy cảm. Bạn sẽ học các primitive vận hành giống production, triển khai trên máy cá nhân.

---

# 2. Sản phẩm và use cases

## 2.1. Câu chuyện sản phẩm

Giả sử một team có hai loại LLM backend:

- `cheap`: provider chi phí thấp/độ trễ nhỏ trong mô phỏng.
- `quality`: provider có thể được lựa chọn khi câu hỏi phức tạp trong mô phỏng.

**Lưu ý khoa học:** Đây chỉ là **nhãn tier dùng cho mock**, không phản ánh chất lượng model thật. Nếu sau này nói một model tốt hơn/rẻ hơn, bắt buộc có dữ liệu/benchmark đo thật.

Ví dụ client gửi:

```bash
curl -i http://127.0.0.1:8000/v1/chat/completions \
  -H 'Authorization: Bearer local-dev-key-change-me' \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "auto",
    "messages": [{"role": "user", "content": "Explain why this code fails"}],
    "stream": false,
    "max_tokens": 128
  }'
```

Gateway thực hiện các bước: validate → auth → giới hạn tài nguyên → route → call upstream → fallback nếu lỗi hợp lệ → ghi metric/trace → trả response + request ID.

## 2.2. Ba demo bắt buộc

**Demo A — Bình thường:** request đơn giản đến `cheap`; request mang từ khóa phức tạp đến `quality`; mỗi request có request ID để tra log.  
**Demo B — Failure engineering:** `cheap` trả 429/503 hoặc timeout; gateway chọn fallback theo policy, không treo và không retry vô hạn.  
**Demo C — Model release:** v2 ML router được train/đăng ký, kiểm tra không đạt gate thì không promote; release ứng dụng lỗi được rollback về image + router artifact trước đó.

## 2.3. Đầu ra cuối cùng

Một người khác có thể **clone repo và tái hiện**: setup → start → make requests → fault injection → view dashboard → run evaluation → build/release → deploy kind → rollback. Không cần đọc chat cũ hoặc phụ thuộc vào kiến thức ngầm của tác giả.

---

# 3. Kiến trúc hệ thống

```mermaid
flowchart TB
    C[Client: curl / Python / application] --> G[FastAPI Gateway]
    G --> AU[Auth + validation + request limits]
    AU --> RT[Router: rule or CPU ML model]
    RT --> R[Reliability policy: timeout / retry / fallback / circuit]
    R --> P1[Mock cheap provider]
    R --> P2[Mock quality provider]
    R -. optional .-> PR[Real LLM API]
    G -. metrics .-> PM[Prometheus --> Grafana]
    G -. traces .-> OT[OpenTelemetry / optional Langfuse]
    D[Dataset and DVC] --> TR[Offline training]
    TR --> MF[MLflow + Registry]
    MF --> B[Versioned router artifact]
    B -. loaded on startup .-> RT
    GH[GitHub Actions] --> IMG[Docker image / GHCR]
    IMG --> DEP[Docker Compose / kind]
    B -. release manifest .-> DEP
```

**Giải thích đường dữ liệu và đường điều khiển:**

1. **Online serving (request path):** client → gateway → provider → client. Cần ít dependency nhất có thể để lỗi monitoring không làm ứng dụng chết.
2. **Observability (side path):** metrics/traces/logs phục vụ phân tích, không phải phần bắt buộc để tạo câu trả lời.
3. **Offline MLOps (training path):** data → train → evaluate → registry → artifact → release. **Không train model bên trong request.**
4. **Release path:** GitHub Actions build app image, chạy gates; local staging deploy version đã được test; kind demo ở cuối.

## 3.1. Tách module ngay từ đầu

- `api`: schemas, routes, error handling.
- `routing`: route policy theo rule hoặc model, không phụ thuộc transport HTTP.
- `providers`: adapter interface + mock + real provider; mỗi provider chịu trách nhiệm chuẩn hóa dữ liệu.
- `reliability`: retry, fallback, deadline, circuit breaker, concurrency limits.
- `observability`: logs, metrics, tracing; không log raw prompt/secret mặc định.
- `ml`: dữ liệu, train, eval, registry, load versioned model.
- `infra`: Compose, monitoring, Kubernetes, workflow scripts.

## 3.2. Ba mức triển khai

- **Level 1 / laptop:** Gateway + 2 mocks bằng Docker Compose (rất nhẹ).
- **Level 2 / observability lab:** Compose + Prometheus/Grafana, MLflow chạy riêng khi cần.
- **Level 3 / Kubernetes lab:** kind cluster + gateway + mock providers; dừng những container không cần trước khi tạo cluster.

Không cần chạy đồng thời tất cả service để được tính là hoàn thành project.

---

# 4. Tech stack: dùng gì, khi nào, vì sao

| Lớp | Công nghệ chọn | Thời điểm | Vai trò / quyết định |
|---|---|---|---|
| Runtime | Python 3.11+ | Ngay đầu | Phù hợp starter, thư viện ổn định; có thể nâng sau khi test |
| Web/API | FastAPI + Pydantic v2 + Uvicorn | Tuần 1 | Async HTTP, validation, schema, OpenAPI |
| HTTP client | HTTPX AsyncClient | Tuần 1 | Connection pooling, connect/read/write/pool timeout |
| Testing | pytest + HTTPX MockTransport | Tuần 1 | Unit/integration tests không cần gọi API ngoài |
| Code quality | Ruff; type checking tùy chọn | Tuần 2 | Lint/format để CI có tiêu chuẩn |
| Container | Docker Engine + Docker Compose v2 | Tuần 1 | Build/deploy local, service DNS, networks |
| Limits (ban đầu) | In-process limiter + asyncio.Semaphore | Tuần 2 | Hiểu semantics; **chỉ đúng cho một worker** |
| Limits (nâng cao) | Redis | Tùy chọn tuần 8 | Shared counters/quotas nếu nhiều replicas |
| Metrics | prometheus-client | Tuần 3 | Histogram/Counter/Gauge |
| Dashboard | Prometheus + Grafana | Tuần 3 | p95, errors, fallbacks, throughput |
| Distributed traces | OpenTelemetry Python SDK + HTTPX/FastAPI instrumentation | Tuần 3 | Correlate gateway và upstream |
| LLM tracing | Langfuse Cloud (tùy chọn) | Tuần 4 | Prompt management, generations, eval; không self-host |
| Load tests | Locust | Tuần 3 | Concurrent users, p95, failures |
| Data versioning | DVC + local remote | Tuần 5 | Gắn dữ liệu training với commit/artifact |
| ML model | scikit-learn TF-IDF + LogisticRegression | Tuần 5 | Routing classifier CPU nhẹ |
| Experiment/registry | MLflow + SQLite | Tuần 5 | Metrics, runs, registered model versions |
| CI/CD | GitHub Actions + GHCR | Tuần 6 | Test, build, publish immutable images |
| Cluster local | kind + kubectl | Tuần 7 | Deployment, Service, probes, rollout |
| Security | env/local secrets, container non-root | Xuyên suốt | Không gửi key lên repo; giảm rủi ro |

**Vì sao chưa dùng Kafka, PostgreSQL, Airflow, Kubeflow, Istio, ArgoCD?** Chúng rất hữu ích ở hệ thống lớn, nhưng làm project này phình to, khó debug, thiếu tính tập trung. Học xong vòng đời cơ bản rồi mới thay từng bộ phận nếu có nhu cầu cụ thể.

**Phiên bản:** Không tự động chạy `pip install --upgrade` tất cả liên tục; sau khi một tuần hoàn thành, tạo lockfile/pin dependency và chỉ nâng có test. Với Langfuse, luôn xem tài liệu SDK đang hiện hành; các API cũ có thể đã đổi tên. Với Kubernetes/kind, dùng bản ổn định ghi trên trang hướng dẫn chính thức.

---

# 5. API contract, error contract và policies

## 5.1. Public endpoints của v1

| Method/Path | Chức năng | Giai đoạn |
|---|---|---|
| `POST /v1/chat/completions` | Chat completion; non-streaming trước, SSE sau | 1–2 |
| `GET /health/live` | Kiểm tra process có sống | 1 |
| `GET /health/ready` | Kiểm tra có thể phục vụ request | 1, cải tiến 7 |
| `GET /version` | gateway/policy/router version | 1, nâng cấp 5 |
| `GET /metrics` | Prometheus scrape | 3 |
| `GET /docs` | Swagger UI chỉ mở trong môi trường local/development | 1 |

**Tương thích OpenAI:** Chỉ cam kết **subset của Chat Completions**, không nhận hỗ trợ đầy đủ tool calling, multimodal, embeddings, Responses API, logprobs hay mọi biến thể streaming. Khi gặp field chưa hỗ trợ, nên báo lỗi rõ ràng thay vì âm thầm bỏ qua hoặc biến đổi ý nghĩa.

## 5.2. Request schema tối thiểu

```json
{
  "model": "auto",
  "messages": [{"role": "user", "content": "Explain async Python"}],
  "stream": false,
  "max_tokens": 256
}
```

- `model`: `auto`, `cheap`, `quality` trong mock phase; real providers ánh xạ qua internal deployment IDs.
- `messages`: mảng có ít nhất một user message; giới hạn số lượng và tổng kích thước request.
- `stream`: false ban đầu; true ở Tuần 2.
- `max_tokens`: giới hạn bằng policy server, không hoàn toàn tin input client.
- `X-Request-ID`: phía server tạo nếu chưa có; nếu client đưa ID thì validate trước khi dùng.

## 5.3. Response và request tracing

Mock trả cấu trúc dạng Chat Completion có `choices`, `model`, `usage`; trong bản mock, `usage` bằng 0 vì **không tạo token thật**. Đừng báo cáo số này là token của LLM. Các headers có thể gồm `X-Request-ID`, `X-Selected-Provider`, `X-Fallback-Used`. Trong môi trường public phải đánh giá xem header metadata nội bộ nào được phép lộ.

## 5.4. Error mapping tối thiểu

| Tình huống | Gateway nên trả | Retry/fallback? |
|---|---|---|
| Không auth | `401` | Không |
| Request invalid | `422` (FastAPI validation) | Không |
| Rate limit chính gateway | `429`, `Retry-After` nếu áp dụng | Không gọi provider |
| Request body quá lớn | `413` | Không |
| Timeout tổng request | `504` hoặc documented `503` (chọn 1 convention) | Chỉ khi còn deadline |
| Upstream 429/502/503/504 | Response chuẩn hóa `502/503/504` phù hợp lỗi | Có thể retry/fallback có giới hạn |
| Upstream 400/401 vì cấu hình/request | `502` hoặc `4xx` được document rõ | Thường không retry mù |
| Cả hai provider không sẵn sàng | `503` | Không còn fallback |
| Client ngắt stream | Dừng upstream, cleanup; không cố gửi response mới | Không retry |

**Quan trọng:** Starter v0.1 có mapping đơn giản (`503` nếu cả hai provider fail, `502` nếu upstream lỗi không retryable). Bảng trên là **đặc tả mong muốn của bản nâng cấp**, không tự nhận starter đã làm được.

## 5.5. Routing policy (từ dễ tới khó)

- **Rule v1:** Nếu query chứa `explain`, `debug`, `compare`, ... → `quality`; còn lại → `cheap`. Mục tiêu học routing, không tối ưu benchmark.
- **Rule v2:** Thêm điều kiện `allowed_providers`, `tenant_budget`, `max_latency`, `provider_healthy` và fallback eligibility.
- **ML v1:** Classifier chỉ dự đoán **nhóm yêu cầu** (`simple`, `reasoning`, `code`, `formatting`, ...); sau đó policy map nhóm sang model tier, tuân thủ hard constraints.
- **Tùy chọn nâng cao:** Khi có ground truth chất lượng đầu ra của **cả hai provider** trên tập prompt riêng, mới thử học router theo quality–cost tradeoff.

Ví dụ decision semantics:

```text
request -> validation + auth
        -> quota / budget check
        -> select eligible providers
        -> rule or classifier recommendation
        -> hard policy override (allowlist, cost cap)
        -> bounded attempts under global deadline
        -> response + usage + trace
```

**Hạn chế bắt buộc:** Fallback từ `quality` sang `cheap` có thể **hạ chất lượng**; chỉ làm nếu request cho phép. Với yêu cầu safety/policy quan trọng, fail closed có thể đúng hơn fallback tự động.

---

# 6. Cấu trúc repository và quy ước kỹ thuật

## 6.1. Cấu trúc đích (tạo dần theo tuần)

```text
intelligent_llm_gateway/
├── app/
│   ├── main.py                  # FastAPI app factory + routes
│   ├── schemas.py               # Pydantic request/response
│   ├── settings.py              # Typed configuration
│   ├── routing/
│   │   ├── rules.py             # Rule-based router
│   │   ├── ml_router.py         # Load trained classifier (week 5)
│   │   └── policy.py            # Eligibility / business constraints
│   ├── providers/
│   │   ├── base.py              # Adapter contract
│   │   ├── mock.py              # Mock adapter
│   │   └── real.py              # Optional real provider adapter
│   ├── reliability/
│   │   ├── retry.py
│   │   ├── circuit.py
│   │   └── limits.py
│   └── observability/
│       ├── logging.py
│       ├── metrics.py
│       └── tracing.py
├── mock_provider/
│   └── main.py
├── ml/
│   ├── data/
│   │   ├── router_train.csv     # Artificial/labeled small set
│   │   ├── router_val.csv
│   │   └── router_test.csv
│   ├── train.py
│   ├── evaluate.py
│   ├── export.py
│   └── artifacts/                # Git-ignored; versioned externally
├── evaluations/
│   ├── mock_scenarios.yaml
│   ├── prompt_suite.jsonl
│   └── reports/
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── contract/
│   └── resilience/
├── loadtests/
│   └── locustfile.py
├── infra/
│   ├── monitoring/
│   │   ├── prometheus.yml
│   │   └── grafana/
│   └── k8s/
│       ├── namespace.yaml
│       ├── gateway-deployment.yaml
│       ├── gateway-service.yaml
│       └── providers.yaml
├── scripts/
│   ├── smoke-test.sh
│   ├── release.sh
│   └── rollback.sh
├── .github/workflows/
│   ├── ci.yml
│   └── release.yml
├── Dockerfile
├── compose.yaml
├── compose.fallback.yaml
├── compose.monitoring.yaml
├── pyproject.toml
├── .env.example
├── .gitignore
├── Makefile                       # tùy chọn
├── release-manifest.json          # tạo cuối tuần 5–6
└── README.md
```

**Nếu dùng starter v0.1 đã gửi trước đây:** file `app/router.py` hiện tại có thể được giữ đến hết Tuần 2; chỉ chuyển sang `app/routing/*` khi refactor. Không cần đổi tên mọi file trước khi hiểu chúng đang làm gì. Starter còn có `PLAN.md`, `README.md` và tests cơ bản, nhưng file hướng dẫn này là đặc tả mở rộng đầy đủ hơn.

## 6.2. Quy ước Git

- Branch ngắn theo feature: `feat/retry-policy`, `feat/metrics`, `feat/ml-router`.
- Mỗi PR có mô tả thay đổi, tests, cách rollback (nếu ảnh hưởng serving).
- Không commit `.env`, access keys, dữ liệu người dùng thật, model cache/dataset lớn, log chứa secrets.
- Commit/release tags: `v0.1.0`, `v0.2.0`, ...; model version độc lập với API version.
- Mọi lệnh tái lập phải nằm trong README hoặc scripts, không chỉ nằm trong history terminal.

## 6.3. Quy ước phiên bản release

Một release lý tưởng xác định đủ:

```json
{
  "gateway_image": "ghcr.io/OWNER/intelligent-llm-gateway:git-sha-example",
  "git_commit": "<full_git_sha>",
  "routing_policy_version": "rules-v2",
  "router_model_name": "gateway-route-classifier",
  "router_model_version": 3,
  "dataset_revision": "<dvc_or_dataset_hash>",
  "evaluation_report": "evaluations/reports/release-003.json"
}
```

Đây chỉ là **ví dụ cấu trúc manifest**, không phải giá trị đã build. Model/manifest thực tế sẽ được tạo trong tuần 5–6. Dùng immutable image digest cho triển khai quan trọng hơn một tag có thể bị ghi đè.

---

# 7. Khởi động project trên Ubuntu

## 7.1. Kiểm tra môi trường

Mở VS Code terminal trên Ubuntu:

```bash
lsb_release -a
python3 --version
git --version
docker --version
docker compose version
sudo systemctl status docker --no-pager
```

Nếu chưa có Docker, **cài theo hướng dẫn chính thức cho Ubuntu**: https://docs.docker.com/engine/install/ubuntu/ . Không dùng script cài đặt từ nguồn không rõ và không trộn nhiều nguồn `docker.io`/Docker CE khi chưa hiểu conflict.

Nếu `docker ps` bị `permission denied`:

```bash
sudo docker ps
```

Có thể cấu hình Docker group theo docs, nhưng hãy hiểu rằng **quyền truy cập Docker socket gần tương đương quyền root**. Không tùy tiện thêm user khác vào group Docker trên máy dùng chung.

## 7.2. Cách nhanh nhất: dùng starter v0.1

Nếu đã tải gói **`intelligent_llm_gateway_starter.zip`** trong cuộc trò chuyện trước:

```bash
mkdir -p ~/projects
cd ~/projects
unzip ~/Downloads/intelligent_llm_gateway_starter.zip
cd intelligent_llm_gateway_starter

cp .env.example .env
docker compose up --build -d
docker compose ps
```

Đường dẫn `~/Downloads/...` là **ví dụ**; nếu file được lưu chỗ khác, dùng đường dẫn thực tế. Tên repo có thể đổi thành `intelligent_llm_gateway` sau khi kiểm tra chạy tốt.

Nếu muốn học Python trước Docker:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -e '.[dev]'
pytest -q
```

**Bản starter hiện có 6 tests cơ bản**; chúng không chứng minh đã hoàn thành reliability, observability hay security. Mục tiêu bạn phải tự thêm những tests tiếp theo.

## 7.3. Gửi request đầu tiên

```bash
curl -i http://127.0.0.1:8000/health/live
curl -i http://127.0.0.1:8000/version
curl -i http://127.0.0.1:8000/v1/chat/completions \
  -H 'Authorization: Bearer local-dev-key-change-me' \
  -H 'Content-Type: application/json' \
  -d '{"model":"auto","messages":[{"role":"user","content":"hello"}]}'
```

Kỳ vọng (starter v0.1): status 200; header `X-Selected-Provider: cheap`, `X-Fallback-Used: false`. Mock trả nội dung có marker `SYNTHETIC`, không phải sinh bằng LLM.

Thử route phức tạp:

```bash
curl -i http://127.0.0.1:8000/v1/chat/completions \
  -H 'Authorization: Bearer local-dev-key-change-me' \
  -H 'Content-Type: application/json' \
  -d '{"model":"auto","messages":[{"role":"user","content":"explain this bug"}]}'
```

Kỳ vọng: chọn `quality` theo rule ban đầu.

## 7.4. Bài tập networking cơ bản

```bash
docker compose ps
docker compose logs -f gateway
docker compose exec gateway python -c \
  "import socket; print(socket.gethostbyname('cheap'))"
ss -ltnp | grep ':8000' || true
curl -v http://127.0.0.1:8000/health/live
```

**Cần hiểu:** `localhost` bên trong container là chính container đó; `cheap` là service DNS name trong Docker Compose network; máy host truy cập qua port được publish lên `127.0.0.1:8000`.

## 7.5. Dừng và dọn dẹp

```bash
docker compose down
# Không chạy 'docker system prune -a --volumes' bừa bãi nếu còn dữ liệu khác.
```

---

# 8. Tuần 1: Linux, HTTP, networking, FastAPI, Docker

**Mục tiêu:** Biết request được xử lý qua các thành phần nào; chạy và debug bằng command line mà không dựa vào IDE.

## Ngày 1–2: HTTP + FastAPI

1. Đọc `app/main.py`, `app/settings.py`, `app/schemas.py`, `app/router.py`, `mock_provider/main.py` trong starter.
2. Vẽ bằng tay vòng đời một request (bằng mũi tên + tên hàm).
3. Tạo ba request: valid, thiếu auth, sai schema; quan sát code `200`, `401`, `422`.
4. Dùng `curl -i`, `curl -v` để phân biệt header/body/status và địa chỉ kết nối.
5. Tìm hiểu `async def`, `await`, `httpx.AsyncClient`, connection pooling; không tạo `AsyncClient` mới cho mỗi request nếu không cần.

Bài tập: Thêm `GET /version` trả cả `gateway_version` và `policy_version`; viết test kiểm tra JSON schema.

## Ngày 3–4: Docker và Compose

1. Đọc Dockerfile từng dòng: base image, copy source, install dependencies, user không phải root, CMD.
2. Chạy `docker image ls`, `docker ps`, `docker inspect`, `docker logs`.
3. Quan sát ba service trong Compose: `gateway`, `cheap`, `quality`.
4. Thay `MOCK_LATENCY_MS=20` thành `200` cho `cheap`, recreate container và đo thay đổi.
5. Thử dừng `cheap`; quan sát response và log của Gateway. Phân biệt Compose startup dependency với việc upstream khỏe liên tục.

## Ngày 5–6: Testing cơ bản

Chạy tests:

```bash
source .venv/bin/activate
pytest -q
```

Viết thêm ít nhất 4 tests:

- Thiếu messages → `422`.
- `stream=true` ở starter → `422` (vì chưa hỗ trợ).
- Primary trả `429`, secondary `200` → fallback thành công.
- Primary/secondary đều `503` → gateway trả lỗi đúng; không treo.

Dùng `httpx.MockTransport` thay cho provider thật trong unit test để tests nhanh, rẻ và deterministic.

## Ngày 7: Báo cáo kỹ thuật nhỏ

Viết vào `README.md` (không cần tạo file báo cáo riêng) một mục "Week 1 learning notes":

- Một request đi qua những file/hàm nào?
- `401`, `422`, `502`, `503` khác gì nhau?
- Tại sao Gateway container không gọi provider qua `localhost`?
- `async` không có nghĩa là xử lý CPU-heavy song song; nó có lợi khi chờ I/O như HTTP.

**Definition of Done — Tuần 1**

- [ ] `docker compose up --build -d` chạy 3 services.
- [ ] Curl gọi được gateway và biết routing response.
- [ ] Unit tests xanh; có ít nhất 4 tests do bạn tự thêm.
- [ ] Giải thích được Docker networking, HTTPX, validation và auth.
- [ ] Không log access token, không commit `.env`.

---

# 9. Tuần 2: Reliability engineering

**Mục tiêu:** Gateway vẫn cư xử dự đoán được khi upstream chậm, lỗi, rate limited hoặc client hủy kết nối. Đây là phần quan trọng nhất về vận hành dịch vụ.

## 9.1. Refactor theo các interface nhỏ

Tạo adapter contract khái niệm:

```python
from typing import Protocol, Any

class Provider(Protocol):
    name: str
    async def complete(self, payload: dict[str, Any]) -> dict[str, Any]: ...
```

Đây là **interface minh họa**; adapter thật cần typed request/response, error classes và streaming interface riêng. Đặt phần gọi upstream ở provider layer; route/controller không chứa toàn bộ logic HTTPX/retry.

## 9.2. Thiết kế retry đúng

**Không retry bất kỳ lỗi nào một cách vô điều kiện.** Đặt chính sách ví dụ:

- Chỉ retry/fallback khi `429`, `502`, `503`, `504`, connection reset hoặc timeout **trước khi stream bắt đầu**; 500 có thể retry tùy provider semantics.
- Không retry `400`, `401`, `403`, schema validation; xử lý `404` theo hợp đồng provider, không mù quáng retry.
- Tối đa **2–3 upstream attempts tổng cộng** trên mỗi request (tính cả lần đầu và fallback).
- Tổng deadline request ≤ 3 giây khi test mock; các giá trị production thật phải được chọn theo SLO/provider.
- Exponential backoff + jitter, tôn trọng `Retry-After` nếu còn budget; không vượt deadline.
- Nếu request đã tạo side effect hoặc stream đã phát dữ liệu, **không replay tự động** vì có thể duplicate output/chi phí.
- Khi client hủy request, hủy upstream call và giải phóng semaphore/connection.

Công thức backoff để hiểu:

```text
backoff_n = min(max_backoff, base_delay * 2^n) + random_jitter
remaining_budget = deadline - monotonic_now
chỉ đợi nếu backoff_n < remaining_budget
```

Dùng đồng hồ **monotonic** cho deadline, không dùng wall clock dễ bị điều chỉnh.

## 9.3. Timeout là nhiều giai đoạn

HTTPX có thể cấu hình `connect`, `read`, `write`, `pool`. Chúng **không tự bảo đảm** toàn bộ end-to-end request deadline. Bao ngoài toàn bộ operation bằng timeout/deadline tổng (ví dụ `asyncio.timeout`) để tránh nhiều lần retry kéo dài ngoài dự kiến.

Nếu upstream trả quá chậm, request cần có kết thúc hữu hạn và metric ghi lỗi rõ ràng.

## 9.4. Circuit breaker

State machine tối thiểu:

```text
CLOSED --failure threshold--> OPEN
OPEN --cooldown elapsed--> HALF_OPEN
HALF_OPEN --probe success--> CLOSED
HALF_OPEN --probe failure--> OPEN
```

Gợi ý cấu hình lab: lỗi liên tục 3 lần → OPEN trong 5 giây → cho 1 probe vào HALF_OPEN. **Đây là ngưỡng dùng học/demo, không phải chuẩn production.** Test bằng fake clock/clock injection để tests chạy nhanh, không `sleep(5)` thật.

Tạo tests chứng minh:

1. Các lỗi liên tiếp làm circuit open.
2. Khi open, provider không được gọi tiếp.
3. Sau cooldown cho probe; thành công đóng circuit.
4. Gặp lỗi ở HALF_OPEN thì mở lại.

## 9.5. Rate limiting và concurrency limits

- Trước tiên: token bucket cố định **theo API key** hoặc sliding window trong RAM để học; **chỉ đảm bảo đúng trong một process**.
- Thêm `asyncio.Semaphore` giới hạn tổng requests đang gọi upstream.
- Thiết lập queue timeout; khi quá tải trả `429`/`503` có policy rõ ràng, không để request queue vô hạn.
- Nếu chạy nhiều Uvicorn workers hoặc nhiều replicas, chuyển quota shared sang Redis (P2). Đừng tuyên bố rate limit toàn cluster khi đang chỉ dùng RAM từng process.

Bài test: cùng API key gửi 12 request trong 1 giây với limit thử nghiệm 10/s → có request bị từ chối, counter tăng đúng; với 2 worker thì **minh họa** được điểm yếu của in-process limit.

## 9.6. Streaming bằng SSE

Ở Tuần 2 mới hỗ trợ `stream=true`:

- Trả `Content-Type: text/event-stream`.
- Mỗi chunk là `data: <json>\n\n`; kết thúc bằng `data: [DONE]\n\n` (nếu contract theo OpenAI subset).
- Không gom toàn bộ chunks thành response rồi giả streaming.
- Sau khi bắt đầu trả SSE 200, không thể đổi HTTP status thành 503 nếu provider lỗi giữa chừng. Cần quy ước error event và đóng stream.
- Ngắt kết nối phải cancel upstream stream và cleanup.
- Fallback chỉ có thể diễn ra **trước khi client nhận token/chunk đầu tiên** (hoặc khi có buffering rõ ràng); nếu không sẽ trộn nội dung hai provider.

Ví dụ **format SSE**, không phải output test thật:

```text
data: {"choices":[{"delta":{"content":"Hello"},"index":0}]}

data: {"choices":[{"delta":{"content":"!"},"index":0}]}

data: [DONE]

```

Thử:

```bash
curl -N http://127.0.0.1:8000/v1/chat/completions \
  -H 'Authorization: Bearer local-dev-key-change-me' \
  -H 'Content-Type: application/json' \
  -d '{"model":"auto","stream":true,"messages":[{"role":"user","content":"hi"}]}'
```

**Lưu ý:** Lệnh trên chỉ thành công sau khi bạn implement streaming; starter v0.1 cố ý trả 422.

## 9.7. Fault injection

Starter hỗ trợ override:

```bash
docker compose -f compose.yaml -f compose.fallback.yaml up --build -d
curl -i http://127.0.0.1:8000/v1/chat/completions \
  -H 'Authorization: Bearer local-dev-key-change-me' \
  -H 'Content-Type: application/json' \
  -d '{"model":"auto","messages":[{"role":"user","content":"hello"}]}'
```

Kỳ vọng starter: cheap mock `503` → gateway fallback sang quality; header `X-Fallback-Used: true`.

Tự mở rộng fault modes: `429`, `timeout`, `invalid_json`, `disconnect`, `slow_stream`, `partial_stream`, `flaky` (nói rõ seed nếu stochastic). Fake provider phải hỗ trợ bật/tắt lỗi qua environment hoặc endpoint **chỉ dùng cho test**.

**Definition of Done — Tuần 2**

- [ ] Retry có attempt budget, deadline, backoff và kiểm tra status retryable.
- [ ] Circuit breaker có tests, không cần sleep thật.
- [ ] Rate/concurrency limit có tests; hiểu giới hạn in-memory.
- [ ] SSE stream chạy bằng `curl -N`, ngắt kết nối không leak.
- [ ] Failure injection chứng minh không treo, không retry storm.
- [ ] Đối với streaming, không replay khi đã gửi token.

---
# 10. Tuần 3: Logs, metrics, traces, load testing

**Mục tiêu:** Khi hệ thống lỗi hoặc chậm, biết **đang xảy ra điều gì, ở đâu và vì sao**. Không chỉ in `print()` rồi đoán.

## 10.1. Phân biệt 3 trụ cột observability

| Tầng | Câu hỏi trả lời | Công cụ |
|---|---|---|
| Logs | Request nào lỗi? Lỗi gì? Provider nào? | Python structured JSON logging |
| Metrics | Trong 5 phút, bao nhiêu request lỗi? p95 tăng ra sao? | Prometheus + Grafana |
| Traces | Một request đã trải qua các bước nào, mỗi bước tốn bao lâu? | OpenTelemetry + backend phù hợp |

**Quan trọng:** Tracing SDK tự nó không tạo giao diện xem trace. Bắt đầu bằng Console exporter để xem span ở terminal; sau đó dùng Jaeger/Tempo hoặc Langfuse Cloud nếu muốn giao diện phân tích. Tránh tự host 3 tracing backends cùng lúc.

## 10.2. Structured logs và correlation IDs

Mỗi log event nên chứa:

```json
{
  "timestamp": "2026-10-27T10:00:00Z",
  "level": "INFO",
  "event": "provider_fallback",
  "request_id": "example-uuid",
  "provider": "cheap",
  "fallback_provider": "quality",
  "status_code": 503,
  "elapsed_ms": 310.2,
  "gateway_version": "v0.3"
}
```

**Không log:** `Authorization`, provider API key, raw request prompts mặc định, raw outputs chứa dữ liệu cá nhân, cookies. Hash/mask user identifiers nếu cần correlate và chỉ lưu trong phạm vi phù hợp. Traces cũng phải áp dụng chính sách tương tự.

Bài tập: Thay log dạng text trong gateway bằng JSON formatter; mỗi request có `request_id`; cùng ID xuất hiện ở log `routing_selected`, `upstream_error`, `fallback_selected`, `request_completed`.

## 10.3. Prometheus metrics cần thiết

Tạo các metric **với tên và đơn vị rõ ràng**:

| Metric | Loại | Labels gợi ý | Dùng để |
|---|---|---|---|
| `gateway_requests_total` | Counter | `route`, `status_class` | RPS, error rates |
| `gateway_request_duration_seconds` | Histogram | `route` | p50/p95/p99 |
| `gateway_upstream_requests_total` | Counter | `provider`, `outcome` | Theo dõi lỗi từng provider |
| `gateway_fallback_total` | Counter | `from_provider`, `to_provider` | Fallback frequency |
| `gateway_inflight_requests` | Gauge | Không hoặc `route` | Concurrency |
| `gateway_rate_limited_total` | Counter | `policy` | Throttling |
| `gateway_circuit_state` | Gauge | `provider` | 0=closed, 1=open, 2=half-open theo quy ước riêng |
| `gateway_reported_tokens_total` | Counter | `provider`, `direction` | Usage từ provider thật (không bao gồm mock như token thật) |

**Không đưa `request_id`, raw prompt, URL tùy ý hay API key làm label** — cardinality sẽ bùng nổ. Dùng route template (`/v1/chat/completions`) thay cho URL có tham số động.

Mẫu tối thiểu cho `app/observability/metrics.py`:

```python
from prometheus_client import Counter, Gauge, Histogram

REQUESTS = Counter(
    "gateway_requests_total",
    "Number of gateway HTTP requests",
    ["route", "status_class"],
)
DURATION = Histogram(
    "gateway_request_duration_seconds",
    "End-to-end request latency in seconds",
    ["route"],
    buckets=(0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2, 5, 10),
)
INFLIGHT = Gauge("gateway_inflight_requests", "Current in-flight gateway requests")
FALLBACKS = Counter(
    "gateway_fallback_total", "Number of fallback decisions",
    ["from_provider", "to_provider"],
)
```

Đăng ký endpoint chỉ cho local/internal Prometheus scrape:

```python
from fastapi import Response
from prometheus_client import CONTENT_TYPE_LATEST, generate_latest

@app.get("/metrics", include_in_schema=False)
def metrics():
    return Response(content=generate_latest(), media_type=CONTENT_TYPE_LATEST)
```

**Lưu ý tích hợp:** `@app.get` ví dụ đặt ở module tạo app tương ứng; không copy thẳng vào file khác nếu `app` chưa được khai báo. Khi dùng **nhiều Uvicorn worker**, python Prometheus client có multiprocess mode cần cấu hình khác; lab giữ **1 worker** trước.

## 10.4. Prometheus + Grafana qua Compose

Tạo `infra/monitoring/prometheus.yml`:

```yaml
global:
  scrape_interval: 5s
scrape_configs:
  - job_name: gateway
    static_configs:
      - targets: ["gateway:8000"]
    metrics_path: /metrics
```

Tạo `compose.monitoring.yaml` (Compose overlay; dùng cùng `compose.yaml`):

```yaml
services:
  prometheus:
    image: prom/prometheus:latest  # lab; cố định version/digest khi release
    command:
      - --config.file=/etc/prometheus/prometheus.yml
    volumes:
      - ./infra/monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports:
      - "127.0.0.1:9090:9090"
  grafana:
    image: grafana/grafana:latest   # lab; cố định version/digest khi release
    ports:
      - "127.0.0.1:3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
volumes:
  grafana-data:
```

**Sau khi đã implement `/metrics`**, chạy:

```bash
docker compose -f compose.yaml -f compose.monitoring.yaml up --build -d
curl -s http://127.0.0.1:8000/metrics | grep gateway_requests
```

Vào `http://127.0.0.1:9090` kiểm tra Targets → `gateway` UP. Vào `http://127.0.0.1:3000`, thiết lập tài khoản admin local, tạo Prometheus datasource URL **`http://prometheus:9090`** (không dùng `localhost` bên trong Grafana container).

Nếu không muốn sử dụng tag `latest`, thay bằng phiên bản đã kiểm tra và ghim phiên bản trong repository. Không đưa dashboard Grafana chưa được bảo vệ auth ra Internet.

## 10.5. PromQL cơ bản cần hiểu

Ví dụ:

```promql
# HTTP requests/s trong 5 phút
sum(rate(gateway_requests_total[5m]))

# Tỉ lệ 5xx trong 5 phút
sum(rate(gateway_requests_total{status_class="5xx"}[5m]))
/
clamp_min(sum(rate(gateway_requests_total[5m])), 1e-9)

# p95 (nếu Histogram buckets được export đầy đủ)
histogram_quantile(
  0.95,
  sum by (le) (rate(gateway_request_duration_seconds_bucket[5m]))
)

# Fallbacks/s
sum(rate(gateway_fallback_total[5m]))
```

Ghi chú: p95 phải tính từ histogram buckets, **không lấy trung bình các p95 của nhiều instance**. Dải 5 phút chỉ là window demo, không phải SLA sản phẩm. Request rate bằng 0 phải xử lý rõ trên dashboard để tránh hiển thị NaN gây hiểu nhầm.

## 10.6. OpenTelemetry tracing

Cài (vào virtual environment hoặc dependencies):

```bash
python -m pip install \
  opentelemetry-api opentelemetry-sdk \
  opentelemetry-instrumentation-fastapi \
  opentelemetry-instrumentation-httpx
```

Thực hành hai bước:

1. Khởi tạo `TracerProvider` + Console exporter để nhìn thấy spans.
2. Instrument FastAPI + HTTPX, thêm manual spans `route.select`, `provider.call`, `fallback.decision`.

Ví dụ manual span minh họa:

```python
from opentelemetry import trace
tracer = trace.get_tracer("llm-gateway")

with tracer.start_as_current_span("routing.select") as span:
    provider_name = "cheap"  # thay bằng policy thực tế
    span.set_attribute("gateway.provider", provider_name)
```

Xác minh một request có nhiều spans với chung trace ID; từ một span lỗi xác định đúng upstream, status, latency.

## 10.7. Load testing bằng Locust

Cài:

```bash
python -m pip install locust
```

Tạo `loadtests/locustfile.py`:

```python
from locust import HttpUser, task, between

class GatewayUser(HttpUser):
    wait_time = between(0.2, 0.7)

    @task
    def chat(self):
        self.client.post(
            "/v1/chat/completions",
            headers={"Authorization": "Bearer local-dev-key-change-me"},
            json={"model": "auto", "messages": [
                {"role": "user", "content": "hello"}
            ]},
            name="POST /v1/chat/completions",
        )
```

Chạy lab non-UI ví dụ:

```bash
locust -f loadtests/locustfile.py \
  --host http://127.0.0.1:8000 \
  --headless -u 10 -r 2 -t 60s
```

Lặp lại 1 → 5 → 10 → 25 simulated users; ghi throughput, p50/p95, tỷ lệ lỗi, CPU/RAM. Không coi simulated users bằng số request/s vì mỗi user có think-time.

## 10.8. Nghiệm thu tuần 3

- [ ] Log JSON có request ID, không chứa key/prompt nhạy cảm.
- [ ] Prometheus target UP; có 4 panel: RPS, error rate, p95, fallback rate.
- [ ] Có trace ít nhất một request bị fallback.
- [ ] Có baseline Locust và một kịch bản lỗi upstream.
- [ ] Có thể giải thích vì sao throughput cao không đồng nghĩa latency tốt.

---

# 11. Tuần 4: LLMOps, provider thật và evaluation

**Mục tiêu:** Thêm lớp LLM thực sự nhưng giữ project vẫn chạy hoàn toàn khi không có API key. Mock dùng cho reliability; LLM thật chỉ dùng cho đánh giá chất lượng và usage/cost.

## 11.1. Real provider adapter — bật tùy chọn

Đặt `ENABLE_REAL_PROVIDER=false` là mặc định. Khi muốn thử, dùng **một provider** có API phù hợp và key riêng, giới hạn chi phí. Không viết key vào source, Compose YAML hay README.

Contract adapter cần chuẩn hóa:

```text
ProviderRequest:
  system/user/assistant messages
  max_output_tokens
  stream
  timeout/deadline

ProviderResult:
  text/content
  provider_name, model_id
  provider_request_id
  latency_ms
  input_tokens, output_tokens (nullable)
  finish_reason
  estimated_cost_usd (nullable)
```

**Cần tách 3 trạng thái của tokens/cost:** `reported` (provider cung cấp), `estimated` (local tokenizer hoặc price table), `unknown` (không có dữ liệu). Đừng biến `unknown` thành 0.

## 11.2. Token pricing và budget

Nếu provider báo số token chính xác và bảng giá có hiệu lực tại thời điểm gọi, tính ví dụ:

```text
cost_usd =
  input_tokens / 1_000_000 * input_price_per_1m
  + output_tokens / 1_000_000 * output_price_per_1m
```

Thực tế giá còn có cache hit, batch, reasoning tokens, vùng region, negotiated pricing...; lưu `pricing_version` cùng số tiền và coi số tự tính là **estimate**. Giá provider thay đổi theo thời gian: không hardcode giá từ một bài blog cũ. Đo latency của request thật, không dùng latency từ mock để kết luận provider nào nhanh hơn.

Chính sách giới hạn lab:

- Chỉ gọi real provider khi `ENABLE_REAL_PROVIDER=true` và key tồn tại.
- Đặt cap số requests/ngày và `max_tokens` trong test suite.
- Không load-test provider thật; Locust chỉ target mocks.
- Tránh retry non-idempotent request không có chính sách rõ; một lần retry có thể phát sinh thêm chi phí dù client không nhận response.
- Nếu hết ngân sách/hết quota → trả lỗi cấu trúc, không âm thầm đổi sang provider trả phí khác.

## 11.3. Langfuse Cloud (tùy chọn)

Langfuse bổ sung phân tích prompt, LLM generations và đánh giá. Dùng Cloud để tránh self-host database/ClickHouse/Redis. Các phiên bản SDK phát triển nhanh; hãy xem docs tại thời điểm triển khai, không copy API cũ từ tutorial vài năm trước.

Quy trình:

1. Tạo project thử nghiệm trên https://langfuse.com/ (kiểm tra free-tier hiện hành).
2. Đưa public/secret keys qua environment hoặc secret store, không commit.
3. Instrument `provider.call`/LLM generation và ghi `model`, token usage/cost khi biết được.
4. Ẩn hoặc tắt ghi raw prompts/outputs nhạy cảm; chỉ dùng prompts giả lập.
5. Mở một trace và xác nhận có route decision, generation và outcome.
6. Giả lập Langfuse không hoạt động; API chính vẫn phải hoạt động, không block request chỉ vì exporter lỗi.

Langfuse có Python SDK dựa trên OpenTelemetry; tuân theo docs phiên bản mới tại: https://langfuse.com/docs/observability/sdk/overview . **Đừng bắt buộc sử dụng Langfuse nếu đã có OpenTelemetry + backend trace đủ mục tiêu học.**

## 11.4. Prompt versioning

Dùng một prompt ngắn có phiên bản:

```text
prompt_name: gateway-eval-system
prompt_version: v1
system: "Answer concisely. If uncertain, say so."
```

Thay prompt `v1` → `v2`, đánh giá cùng input set; không coi thay prompt là cải thiện nếu không có evaluation. Nếu sử dụng Langfuse prompt management, lưu `prompt_version` vào trace/release metadata.

## 11.5. Evaluation dataset (cực kỳ quan trọng)

Tạo `evaluations/prompt_suite.jsonl` với **ít nhất 30–50 prompts tự viết**, chia thành:

- 10–15 general/simple queries.
- 10–15 reasoning/comparison queries.
- 10–15 coding/debugging queries.
- 5–10 edge cases: ambiguous, malformed, long, forbidden/unsupported fields.

Mỗi record lưu ID, category, prompt, expected criteria và allowed tiers; **không cần tạo đáp án hoàn chỉnh nếu đánh giá bằng rubric**, nhưng rubric phải được ghi trước khi test.

Ví dụ:

```json
{"id":"q001","task_type":"simple","prompt":"What is a Python list?","rubric":"Defines ordered mutable sequence with one example","allowed_tiers":["cheap","quality"]}
{"id":"q002","task_type":"code","prompt":"Explain why an async function is not awaited","rubric":"Explains coroutine and await; provides correct fix","allowed_tiers":["quality"]}
```

Định nghĩa tách bạch:

- **Infrastructure correctness (mock):** route decision, fallback, status, retries, deadline, SSE, logs.
- **Output quality (real LLM):** correct/incorrect, rubric score, refusal appropriateness, consistency, human evaluation.
- **Operational cost (real LLM):** provider-reported tokens, latency, token cost.

Với **mock providers**, tuyệt đối không công bố số liệu về chất lượng LLM hoặc tối ưu chi phí thật. Có thể đánh giá routing policy trên nhãn do bạn gán, nhưng nhãn đó không phải sự thật khách quan về model quality.

## 11.6. Cách so sánh policy công bằng

Cùng một test set, cùng model/provider version, cùng generation configuration, ghi:

| Chỉ số | Ý nghĩa |
|---|---|
| `routing_accuracy` | Trùng nhãn route do annotator gán (proxy) |
| `task_success_rate` | Output đạt rubric trên tập có ground truth |
| `estimated_cost_usd` | Ước lượng chi phí dựa trên usage+pricing hiện hành |
| `p50/p95_latency_ms` | Độ trễ end-to-end, dùng real requests khi claim real |
| `fallback_rate` | Tỷ lệ request cần fallback |
| `provider_error_rate` | Lỗi upstream theo provider |

Nếu có 2 policies A/B, đo trên cùng prompts, seed nếu có randomness, report sample size; không trộn kết quả synthetic với real. Phần evaluation mock có thể chạy toàn bộ miễn phí; phần quality real chỉ chạy số ít yêu cầu và đặt budget cap.

**Definition of Done — Tuần 4**

- [ ] Có interface provider rõ ràng; mock mode chạy không cần key.
- [ ] Có ít nhất 30 prompts và evaluation script/report.
- [ ] Cost/token logging phân biệt `reported/estimated/unknown`.
- [ ] Có prompt/config version trong response metadata hoặc trace.
- [ ] Nếu dùng real provider: đã thử ít nhất một batch nhỏ, có báo cáo riêng.
- [ ] Nếu dùng Langfuse: trace hoạt động; Langfuse down không làm gateway down.

---

# 12. Tuần 5: MLOps và ML-based router

**Mục tiêu:** Tự trải nghiệm ML lifecycle thật, không chỉ sử dụng MLflow như một dashboard trang trí. Model rất nhỏ, train CPU được.

## 12.1. Bài toán ML chính xác

Không huấn luyện model để 'dự đoán LLM nào tốt nhất' khi **chưa có dữ liệu đánh giá đầu ra từng LLM**. Bước v1 hợp lý hơn:

```text
Input: prompt text
Output: class ∈ {simple, reasoning, code, formatting, unknown}
Downstream policy: class + customer constraints + provider health
                   -> select cheap/quality or refuse
```

Đây là **task-type classifier**; nó giúp học ML training, versioning, deployment. Không đồng nghĩa quality/cost-optimal routing.

## 12.2. Dataset version 1

Tạo `ml/data/router_train.csv`, `router_val.csv`, `router_test.csv`, với các cột:

```csv
prompt_id,text,task_type
001,"What is a list in Python?",simple
002,"Explain this deadlock and show a fix",code
003,"Compare two rollout strategies",reasoning
004,"Convert this object to YAML",formatting
```

Khuyến nghị ban đầu: 200–400 câu tổng cộng, 4–5 classes cân bằng tương đối. Có thể tự tạo synthetic prompts, nhưng phải ghi nguồn gốc và chấp nhận khả năng model chỉ học shortcut từ cụm từ. Phân lớp độc lập; dùng **stratified split** nếu các lớp đủ mẫu và loại bỏ duplicate/near-duplicate giữa train/test để tránh leakage.

**Tối thiểu bắt buộc:** train/val/test cố định từ đầu, không tuning trên test liên tục. Giữ test set cuối để final evaluation.

## 12.3. Baseline và model v1

- Baseline: rule keywords hiện tại.
- ML v1: `TfidfVectorizer(ngram_range=(1,2)) + LogisticRegression(max_iter=...)`.
- Tùy chọn: `class_weight='balanced'` nếu classes mất cân bằng, nhưng đo tác động trên validation.
- Metrics: Macro-F1, accuracy, confusion matrix, per-class recall, latency inference trên CPU.

Mẫu code train cơ bản (đặt ở `ml/train.py`, điều chỉnh đường dẫn theo dữ liệu đã tạo):

```python
from pathlib import Path
import json
import pandas as pd
from sklearn.pipeline import Pipeline
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score, f1_score, classification_report
import joblib

train = pd.read_csv("ml/data/router_train.csv")
val = pd.read_csv("ml/data/router_val.csv")

model = Pipeline([
    ("tfidf", TfidfVectorizer(ngram_range=(1, 2), min_df=1)),
    ("classifier", LogisticRegression(max_iter=1000, random_state=42)),
])
model.fit(train["text"], train["task_type"])
pred = model.predict(val["text"])
metrics = {
    "val_accuracy": float(accuracy_score(val["task_type"], pred)),
    "val_macro_f1": float(f1_score(val["task_type"], pred, average="macro")),
}
print(json.dumps(metrics, indent=2))
print(classification_report(val["task_type"], pred, zero_division=0))
Path("ml/artifacts").mkdir(parents=True, exist_ok=True)
joblib.dump(model, "ml/artifacts/router.joblib")
```

Cài bổ sung:

```bash
python -m pip install pandas scikit-learn joblib mlflow dvc
python ml/train.py
```

**Lưu ý bảo mật:** Chỉ load `joblib`/pickle artifact mà bạn tự build hoặc từ nguồn đáng tin cậy; file pickle độc hại có thể thực thi code. Artifact được cố định version và checksum trong release.

## 12.4. DVC cho data versioning

Từ repo root:

```bash
dvc init
mkdir -p ~/dvc-store-llm-gateway
dvc remote add -d localstore ~/dvc-store-llm-gateway
# Đảm bảo dữ liệu training chưa bị commit trực tiếp vào Git.
dvc add ml/data
dvc push
git add .dvc .dvcignore ml/data.dvc .gitignore
git commit -m "Track routing dataset with DVC"
```

Lưu ý:

- `dvc init` chạy trong repo Git.
- Nếu data đã được Git track, cần `git rm --cached` các file tương ứng trước khi DVC add.
- `~/dvc-store-llm-gateway` là local remote nằm **ngoài repo**, không phải backup cloud. GitHub Actions hosted runner không thể tự truy cập nó. Để CI chạy, chỉ dùng test fixture nhỏ có thể commit, hoặc sau này cấu hình remote có credentials được quản lý an toàn.
- Khi thay dataset: `dvc add ml/data`, `dvc push`, commit file `.dvc`; khi phục hồi: checkout commit và `dvc pull`.

## 12.5. MLflow Tracking và Model Registry

Khởi chạy local MLflow UI:

```bash
mlflow server \
  --backend-store-uri sqlite:///mlflow.db \
  --default-artifact-root ./mlruns \
  --host 127.0.0.1 \
  --port 5000
```

**Lưu ý:** Tuỳ phiên bản MLflow, cú pháp server/artifact config có thể có lựa chọn mới; xem `mlflow server --help`. Ở đây dùng local backend store SQLite và artifact root trên disk. `mlflow.db` và `mlruns/` cần bỏ vào `.gitignore`.

Sau đó trong `ml/train.py` bổ sung tracking ở training code:

```python
import mlflow
import mlflow.sklearn

mlflow.set_tracking_uri("http://127.0.0.1:5000")
mlflow.set_experiment("gateway-router")
with mlflow.start_run():
    mlflow.log_param("vectorizer", "tfidf_1_2gram")
    mlflow.log_param("classifier", "logistic_regression")
    mlflow.log_metrics(metrics)
    mlflow.sklearn.log_model(model, name="router_model")
```

Đây là **đoạn bổ sung để đặt trong chương trình train**, chạy sau khi có biến `model` và `metrics`; **không** paste độc lập rồi mong chạy được. Kiểm tra cú pháp `log_model` tương ứng version MLflow cài thực tế và pin version đã xác nhận hoạt động.

Model Registry workflow:

1. Chạy `rule baseline` và `ml v1`; lưu kết quả cùng dataset version và git SHA.
2. Register ML model thành `gateway-route-classifier`, version 1.
3. Thử một config/model khác → version 2.
4. So sánh trên validation; nếu tốt và không vi phạm latency gate → promote alias `candidate`/`champion` (tùy setup).
5. Khi tạo release, **resolve thành model version/digest bất biến**; không dựa vào alias động trong serving cho rollback.
6. Chứng minh rollback được cả container image lẫn router artifact/policy.

## 12.6. Evaluation gates: chất lượng và Ops

Ví dụ gate ban đầu (do bạn tự xác lập, không phải benchmark chung):

```yaml
router_quality_gate:
  minimum_macro_f1_on_validation: 0.75
  allowed_macro_f1_regression_vs_current: 0.02
serving_gate:
  max_p95_local_router_inference_ms: 20
  require_contract_tests_pass: true
```

Nếu dữ liệu nhỏ/khó, 0.75 có thể không đạt hoặc quá dễ — **đừng điều chỉnh gate sau khi thấy test để né failure**. Điều quan trọng là công bố threshold trước và minh họa một release bị chặn. Đánh giá chất lượng LLM thực tế là **gate khác**, không được thay bằng task classification score.

## 12.7. Serving ML router đúng cách

- Load model **một lần khi app startup** từ artifact do build/release chọn, không truy vấn registry remote mỗi request.
- Nếu model không load được, readiness fail hoặc chuyển sang rule fallback có chủ đích (phải báo `policy_version` rõ).
- Router classifier đưa ra class/confidence; nếu confidence thấp → rule fallback hoặc safe default, không đoán vô căn cứ.
- Áp dụng hard constraints (allowlist, max budget, provider status) **sau** model prediction.
- Ghi `router_model_version` vào `/version` và trace metadata.

## 12.8. Data drift (tuỳ chọn hữu ích)

Tạo dataset 'production-like' có nhiều code prompts hơn training distribution. So sánh:

- Tỷ lệ class dự đoán theo thời gian.
- Độ dài prompt, tỷ lệ unknown/low confidence.
- Feature distribution differences hoặc classifier confidence distribution.

Không có label production → đây chỉ là **proxy/drift signal**, không kết luận chất lượng suy giảm. Thực hành một lịch retraining thủ công; chỉ retrain/rollout tự động sau khi hiểu data quality và review gates.

**Definition of Done — Tuần 5**

- [ ] Có train/val/test rõ, ít nhất baseline và một model CPU.
- [ ] Có Macro-F1, per-class metrics, confusion matrix, latency.
- [ ] `dvc pull` tái tạo dataset từ local remote sau khi checkout đúng commit.
- [ ] MLflow chứa runs, artifact và ít nhất 2 registered model versions.
- [ ] Gateway expose model/policy version; rollback có thể pin artifact cũ.
- [ ] Không gọi classifier là quality-optimal LLM router khi chưa có evidence.

---

# 13. Tuần 6: CI/CD, images, release và rollback

**Mục tiêu:** Chuyển từ 'chạy được trên máy tôi' sang 'build và kiểm thử lặp lại được'.

## 13.1. Quy trình cần xây

```text
Pull request
  -> lint + type check (nếu dùng) + unit tests
  -> contract + reliability tests
  -> offline evaluation trên committed CI fixture
  -> docker build
  -> chỉ merge nếu PASS

Release/tag/manual workflow
  -> build pinned image -> GHCR
  -> kiểm tra image và lưu manifest
  -> deploy vào local staging (bằng lệnh trên Ubuntu)
  -> smoke test + load mini test
  -> promote local (có sự đồng ý)
  -> rollback nếu lỗi
```

**Không giả định GitHub-hosted Actions có thể kết nối vào `localhost` laptop của bạn.** CI build/publish chạy trên GitHub; local deploy sử dụng lệnh bạn chạy trên máy. Không tự mở Docker socket, SSH port hoặc runner công khai chỉ để tự động hóa demo.

## 13.2. Workflow CI tối thiểu

Tạo `.github/workflows/ci.yml`:

```yaml
name: ci
on:
  push:
  pull_request:
permissions:
  contents: read
jobs:
  test-build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: "pip"
      - name: Install
        run: python -m pip install -e '.[dev]'
      - name: Unit and contract tests
        run: pytest -q
      - name: Docker build
        run: docker build -t gateway:ci .
```

Đây là **CI skeleton**. Khi thêm DVC/MLflow/metrics mới, cập nhật `pyproject.toml` và dependency lockfile. `cache: pip` yêu cầu file dependency phù hợp; nếu repo chưa có requirements/lock mà workflow báo lỗi khi dò dependency file, bỏ dòng `cache` hoặc chỉ định file hợp lệ. Từng bước sau có thể thêm Ruff, secret scan, vulnerability scan, integration test trong Compose.

Hành động (`uses:`) trong production nên pin commit SHA đã xác minh thay vì chỉ major version. PR từ fork không được chạy job yêu cầu deployment secrets.

## 13.3. Tách CI và CD

**CI:** lint/tests/build mọi PR. **CD:** phát hành sau khi code được merge/tag và các gates pass. CD demo nên được kích hoạt thủ công (`workflow_dispatch`) để giảm rủi ro publish ngoài ý muốn.

Publish image lên GHCR:

- Dùng GitHub-provided `GITHUB_TOKEN` với `packages: write` ở release job và `contents: read`.
- Dùng `docker/login-action` / `docker/build-push-action` theo ví dụ mới nhất trong GitHub docs.
- Tag image bằng `git SHA` + `vX.Y.Z`, hạn chế `latest`.
- Lưu digest được registry trả về trong release manifest.
- Không build với API keys trong `ARG` hoặc Dockerfile.

Nguồn chính thức: https://docs.github.com/en/actions/tutorials/publish-packages/publish-docker-images

## 13.4. Docker production-minded

Kiểm tra Dockerfile:

- Base image slim phù hợp Python version, cố định version/digest ở release.
- Dependencies có lockfile; tránh image thay đổi ngẫu nhiên theo thời gian.
- `USER` không root; chỉ copy file cần thiết vào image.
- `.dockerignore` loại `.venv`, `.git`, `.env`, model cache, secrets, logs.
- Configure Uvicorn 1 worker cho lab; mở rộng worker cần xem lại rate limit, metrics, circuit state.
- `HEALTHCHECK` chỉ phản ánh local process; readiness policy độc lập.
- Nếu dùng model, đóng gói artifact bất biến hoặc pin và tải khi startup từ nguồn tin cậy, checksum xác minh; không tải version mơ hồ.

## 13.5. Smoke test bắt buộc

Tạo `scripts/smoke-test.sh` gọi:

1. `GET /health/live` → 200.
2. `GET /health/ready` → 200 khi config/artifact hợp lệ.
3. `GET /version` → đúng `gateway_version`, `router_model_version` mong đợi.
4. `POST /v1/chat/completions` → 200 trên mock.
5. Gửi một request lỗi → status/JSON error đúng.
6. Nếu release streaming thì SSE contract test phải pass.

Một bản release không được promote chỉ vì container status `running`; cần **smoke test**.

## 13.6. Rollback toàn bộ release

Đừng chỉ `git checkout` code. Rollback phải khôi phục:

- Image digest/tag trước đó.
- ML model artifact version/digest trước đó.
- Routing policy và config trước đó.
- Prompt template/version nếu thay.

**Không thay đổi dữ liệu bền vững một cách phá hủy** khi rollback nếu có database migrations (chưa dùng database nghiệp vụ trong bản học).

**Definition of Done — Tuần 6**

- [ ] Pull request lỗi không qua CI.
- [ ] CI tự động chạy tests và build image thành công.
- [ ] Tạo release image có immutable reference; manifest ghi đúng versions.
- [ ] Deploy staging local và chạy smoke tests trước promote.
- [ ] Có demo rollback v2 lỗi → v1 tốt, gồm model/policy.
- [ ] Không có secrets trong image, log, workflow output hoặc repo.

---

# 14. Tuần 7: Kubernetes với kind

**Mục tiêu:** Nắm các primitive Kubernetes cần cho MLOps/LLMOps, không phải xây cloud cluster quy mô lớn.

## 14.1. Học lý thuyết tối thiểu

- **Pod:** nhóm container nhỏ nhất được scheduler triển khai.
- **Deployment:** quản lý replica và rollout stateless workload.
- **Service:** endpoint ổn định cho một nhóm pods.
- **ConfigMap/Secret:** truyền cấu hình/secret. Secret mặc định không đồng nghĩa đã an toàn end-to-end; phải quan tâm RBAC/encryption/access.
- **Readiness probe:** có nhận traffic mới hay chưa.
- **Liveness probe:** process có cần restart hay không.
- **Resource requests/limits:** scheduler và quota tài nguyên.
- **Rolling update / rollback:** chuyển image phiên bản có kiểm soát.

Đừng thiết kế liveness phụ thuộc mọi upstream provider: upstream outage có thể khiến restart loop, làm tình hình tệ hơn. Readiness nên phản ánh khả năng phục vụ request và tính hợp lệ của artifact/cấu hình, có policy rõ ràng.

## 14.2. Cài kind và kubectl

Chỉ dùng hướng dẫn hiện hành:

- kind: https://kind.sigs.k8s.io/docs/user/quick-start/
- kubectl: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

Kiểm tra:

```bash
kind version
kubectl version --client
docker version
```

Tạo cluster local một node:

```bash
kind create cluster --name llm-gateway-lab --wait 60s
kubectl cluster-info --context kind-llm-gateway-lab
kubectl get nodes
```

Nếu thiếu RAM, tắt Compose monitoring stack trước:

```bash
docker compose -f compose.yaml -f compose.monitoring.yaml down
```

## 14.3. Load Docker image vào kind (không cần registry)

Ở root repository:

```bash
docker build -t gateway:lab-v1 .
kind load docker-image gateway:lab-v1 --name llm-gateway-lab
```

kind node có image riêng; `docker build` trên host không tự làm image xuất hiện bên trong kind. Tránh tag `latest` khi local load; dùng `imagePullPolicy: IfNotPresent` cho image đã load.

## 14.4. Tạo hai mock providers (lab đầu tiên)

```bash
kubectl create namespace llm-gateway

kubectl -n llm-gateway create deployment cheap \
  --image=gateway:lab-v1 -- \
  uvicorn mock_provider.main:app --host 0.0.0.0 --port 8000
kubectl -n llm-gateway set env deployment/cheap \
  MOCK_PROVIDER_NAME=cheap MOCK_LATENCY_MS=20 MOCK_FAIL_MODE=none
kubectl -n llm-gateway expose deployment cheap --port=8000 --target-port=8000

kubectl -n llm-gateway create deployment quality \
  --image=gateway:lab-v1 -- \
  uvicorn mock_provider.main:app --host 0.0.0.0 --port 8000
kubectl -n llm-gateway set env deployment/quality \
  MOCK_PROVIDER_NAME=quality MOCK_LATENCY_MS=70 MOCK_FAIL_MODE=none
kubectl -n llm-gateway expose deployment quality --port=8000 --target-port=8000
```

Các lệnh imperative này dùng để bạn hiểu Deployment/Service; khi mọi thứ chạy, chuyển nội dung sang `infra/k8s/providers.yaml` (declarative YAML) để có thể `kubectl apply -f` tái lập.

## 14.5. Deployment + Service cho Gateway

Tạo `infra/k8s/gateway-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gateway
  namespace: llm-gateway
spec:
  replicas: 1
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: gateway
  template:
    metadata:
      labels:
        app: gateway
    spec:
      containers:
        - name: gateway
          image: gateway:lab-v1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8000
          env:
            - name: GATEWAY_API_KEY
              valueFrom:
                secretKeyRef:
                  name: gateway-api-key
                  key: api-key
            - name: CHEAP_PROVIDER_URL
              value: http://cheap:8000
            - name: QUALITY_PROVIDER_URL
              value: http://quality:8000
            - name: PROVIDER_TIMEOUT_SECONDS
              value: "1.0"
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8000
            initialDelaySeconds: 2
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: gateway
  namespace: llm-gateway
spec:
  selector:
    app: gateway
  ports:
    - port: 8000
      targetPort: 8000
```

Tạo Secret **lab local** (đừng commit Secret YAML plaintext):

```bash
kubectl -n llm-gateway create secret generic gateway-api-key \
  --from-literal=api-key=local-dev-key-change-me
kubectl apply -f infra/k8s/gateway-deployment.yaml
kubectl -n llm-gateway get pods,svc
kubectl -n llm-gateway rollout status deployment/gateway
```

**Chú ý:** `kubectl create secret --from-literal` có thể đưa key vào shell history/process arguments. Giá trị ở đây chỉ là **demo giả**; với credential thật dùng secret store hoặc kênh nhập an toàn hơn. Kubernetes Secret trong local kind không thay thế secret management của production.

Port forward sang laptop:

```bash
kubectl -n llm-gateway port-forward svc/gateway 8000:8000
```

Terminal khác:

```bash
curl -i http://127.0.0.1:8000/health/live
curl -i http://127.0.0.1:8000/v1/chat/completions \
  -H 'Authorization: Bearer local-dev-key-change-me' \
  -H 'Content-Type: application/json' \
  -d '{"model":"auto","messages":[{"role":"user","content":"hello"}]}'
```

Nếu cổng `8000` đang bị Docker Compose/uvicorn chiếm, dừng nó hoặc forward sang `18000:8000` rồi curl `127.0.0.1:18000`.

## 14.6. Troubleshoot Kubernetes đúng quy trình

```bash
kubectl -n llm-gateway get pods -o wide
kubectl -n llm-gateway describe deployment gateway
kubectl -n llm-gateway describe pod <POD_NAME>
kubectl -n llm-gateway logs deployment/gateway --tail=100
kubectl -n llm-gateway get events --sort-by=.lastTimestamp
kubectl -n llm-gateway get endpointslice
```

Hiểu khác biệt `Pending`, `ContainerCreating`, `ImagePullBackOff`, `CrashLoopBackOff`, `Running but Unready`. Không phản xạ xóa/restart container trước khi xem events.

## 14.7. Demo rollout/rollback

Sau khi `gateway:lab-v1` chạy thành công và được ghi vào release manifest, thử release lỗi:

```bash
kubectl -n llm-gateway set image deployment/gateway \
  gateway=gateway:lab-broken
kubectl -n llm-gateway rollout status deployment/gateway --timeout=30s
kubectl -n llm-gateway describe deployment gateway
```

Expected: rollout mới không hoàn thành do thiếu image (đây là fault injection). Rollback:

```bash
kubectl -n llm-gateway rollout history deployment/gateway
kubectl -n llm-gateway rollout undo deployment/gateway
kubectl -n llm-gateway rollout status deployment/gateway --timeout=60s
```

Sau đó tự làm demo khó hơn: build `gateway:lab-v2` image tồn tại nhưng `/health/ready` fail khi thiếu model/config. Khi rollback, xác minh `/version` phản ánh cả gateway và model/policy đúng v1.

## 14.8. Cleanup

```bash
kind delete cluster --name llm-gateway-lab
```

**Definition of Done — Tuần 7**

- [ ] Biết tạo kind cluster và load image.
- [ ] Gateway gọi được 2 mock provider trong K8s qua Services.
- [ ] Có readiness/liveness, requests/limits, Kubernetes Secret local.
- [ ] Có thể debug `kubectl describe/logs/events`.
- [ ] Có demo rollout lỗi và rollback thành công.
- [ ] Hiểu deployment local không tương đương production cloud multi-zone.

---

# 15. Tuần 8: Hardening, runbook, capstone

**Mục tiêu:** Kết hợp thành một hệ thống có thể demo/review như một engineer.

## 15.1. Security baseline

- API auth mặc định bật; không chỉ so sánh một key demo khi public deployment.
- Cấm/giới hạn request body quá lớn, số messages, token max, request timeout.
- Không log `Authorization`, raw prompts, provider raw error có thể chứa secret.
- Provider URLs phải là **allowlisted config**; không chấp nhận URL tùy ý từ client để tránh SSRF.
- Header client đưa vào phải validate, tránh header injection và lộ internal metadata.
- Gateway container chạy non-root, mount files read-only khi phù hợp.
- Không mở Prometheus, Grafana, MLflow, Docker daemon ra Internet.
- GitHub Actions token least privilege, credentials chỉ ở môi trường cần thiết.
- Nếu dùng real provider, đặt budget cap, chạy evaluation nhỏ, audit thông tin bị gửi đi.
- Nếu có user-specific caching thì phải có tenant boundary; tốt nhất **chưa triển khai semantic cache** trong phiên bản đầu.

## 15.2. Một runbook tối thiểu phải viết

Trong `README.md`, tạo section "Operational runbook":

1. **Startup:** docker compose lệnh nào, cần env gì, service nào UP?
2. **Health:** health/live và ready khác nhau thế nào?
3. **Debug 5xx:** xem metrics → xác định provider → trace request → log request ID.
4. **Debug latency:** kiểm tra p95, upstream, retry counts, in-flight, CPU/memory.
5. **Incident provider down:** circuit open/fallback nếu hợp lệ, hoặc trả 503.
6. **Restore:** chọn image/model/policy manifest tốt, rollback, smoke test.
7. **Secret rotation:** cập nhật credential qua secret store/config, restart có kiểm soát, revoke key cũ.
8. **Cost spike:** giới hạn token/requests, stop real provider, điều tra retry loops.
9. **Cleanup:** dừng kind, Compose và các local volumes không cần.
10. **Known limitations:** một worker in-process limiter, no distributed Redis, mock ≠ real quality.

## 15.3. Incident drill bắt buộc

Tự tạo 5 tình huống:

| Incident | Cách kích hoạt | Nên quan sát được |
|---|---|---|
| Cheap provider outage | MOCK_FAIL_MODE=503 | Fallback/circuit events, 5xx nếu cả hai chết |
| Rate limiting | burst vượt limit | 429 của gateway, không call upstream cho request bị chặn |
| Long timeout | mock sleep > deadline | bounded response time, không retry vô hạn |
| Release lỗi | container image missing / readiness fail | rollout không complete, rollback về version trước |
| Prompt/cost regression | prompt v2 làm output dài hơn trong real test nhỏ | eval report nêu tradeoff, gate chặn khi cần |

**Đừng cố tình gọi một real provider hàng nghìn lần** để test billing/rate limit. Dùng mock trả 429/5xx là đủ để test logic reliability.

## 15.4. Demo capstone 15–20 phút

Chuẩn bị một buổi demo (quay video hoặc tự chạy trực tiếp):

1. 2 phút: giới thiệu kiến trúc, scope và giới hạn.
2. 2 phút: một request thành công, routing logic.
3. 3 phút: inject upstream 503 và quan sát fallback trong log/trace/dashboard.
4. 2 phút: chạy Locust nhỏ, xem p95/throughput/error rate.
5. 3 phút: hiển thị MLflow runs/model versions, DVC dataset revision.
6. 3 phút: cho thấy CI fail khi test lỗi; kiểm tra release manifest.
7. 3 phút: deploy kind + rollback release lỗi.
8. 2 phút: nêu tradeoffs, known limitations và cải tiến tương lai.

**Definition of Done — Tuần 8**

- [ ] Có kiến trúc và runbook trong README.
- [ ] Có ít nhất 5 incident drills có bằng chứng.
- [ ] Có bảng metrics trước/sau cho một thay đổi cụ thể.
- [ ] Có test, lint, build, release manifest, smoke test, rollback reproducible.
- [ ] Demo end-to-end, ghi rõ chỗ mock vs real.

---
# 16. Bộ test matrix và các ngưỡng nghiệm thu

Một project Ops chuẩn phải có **test strategy**. Không chỉ chạy test unit mà cần chứng minh hành vi hệ thống khi lỗi.

## 16.1. Test pyramid khuyến nghị

| Tầng | Số lượng gợi ý | Có network thật? | Thường chạy khi nào? |
|---|---:|---|---|
| Unit tests | 25–40 | Không | Mỗi PR |
| Provider/HTTP contract tests | 10–20 | MockTransport / ASGI | Mỗi PR |
| Reliability state machine tests | 10–15 | Không, fake clock | Mỗi PR |
| Compose integration tests | 8–12 | Docker network nội bộ | CI phù hợp / local trước release |
| Evaluation suite | 30–50 prompts | Mock; real opt-in | Trước release policy/prompt |
| Load tests | 3 scenarios | Local gateway + mocks | Trước release có thay đổi performance |
| Kubernetes smoke/rollback | 2–4 drills | kind local | Tuần 7 / release rehearsal |

Đây là **mục tiêu số lượng tương đối**, không phải cần viết đủ 100 tests ngay. Ưu tiên chất lượng test cho timeout/fallback/cancellation hơn là test chỉ để tăng coverage.

## 16.2. Bộ scenario cụ thể

| ID | Điều kiện | Kỳ vọng | Loại test |
|---|---|---|---|
| T01 | Request simple và auto | Chọn `cheap` theo rule v1 | Unit |
| T02 | Request code và auto | Chọn `quality` theo rule v1 | Unit |
| T03 | `model=quality` | Respect explicit selection hoặc document policy override | Contract |
| T04 | Không auth | 401, không gọi provider | Security |
| T05 | Invalid body | 422, không gọi provider | Contract |
| T06 | Payload quá lớn | 413, không gọi upstream | Security |
| T07 | Primary HTTP 503 | Fallback nếu còn budget và policy cho phép | Resilience |
| T08 | Primary HTTP 429 | Respect Retry-After/budget, fallback theo policy | Resilience |
| T09 | Primary HTTP 400 | Không retry vô hạn | Resilience |
| T10 | Hai providers đều fail | 503/contracted error, có request ID | Resilience |
| T11 | Read timeout | Request kết thúc trong global deadline | Resilience |
| T12 | Client disconnect | Upstream cancellation và cleanup | SSE |
| T13 | Stream đã phát chunk | Không chuyển provider âm thầm | SSE |
| T14 | Circuit đang OPEN | Không gọi upstream bị block | Resilience |
| T15 | Hết rate limit | Gateway 429, provider không được gọi | Security |
| T16 | Monitoring backend down | Request chính vẫn hoạt động | Observability |
| T17 | ML artifact checksum sai | Readiness fail hoặc safe policy explicitly | MLOps |
| T18 | ML router version mismatch | Không promote | Release |
| T19 | CI test fail | Không publish/promote release | CI/CD |
| T20 | Kubernetes v2 broken | Rollback phục hồi v1 | K8s |
| T21 | Random unknown request | Không crash; safe default documented | Routing |
| T22 | Concurrent burst | Không vượt in-flight limit | Load |
| T23 | Real provider key thiếu | Mock mode vẫn hoạt động | Configuration |
| T24 | Real provider có usage null | Cost ghi unknown, không ghi 0 giả | LLMOps |

## 16.3. Các ngưỡng đề xuất để tự học

**Các ngưỡng dưới đây là acceptance targets cho lab mock trên laptop, KHÔNG phải SLA/SLO ngành và chưa chắc phù hợp máy bạn:**

- Unit/contract tests: 100% tests thuộc gate phải pass.
- Error handling: không có infinite retry / request treo vô hạn.
- API auth: 100% request không key nhận 401 ở protected endpoint.
- Timeouts: request kết thúc trong deadline cấu hình, có cộng thêm overhead nhỏ của môi trường test.
- Fault isolation: một provider down không làm gateway chết khi provider khác còn đáp ứng và policy cho phép.
- p95 latency: đo baseline trước; định nghĩa mục tiêu giảm tương đối hoặc ngưỡng dựa trên mock latency bạn đã chọn, không bịa số p95 chung.
- Release: mỗi artifact gắn version; có rollback được về bản đã kiểm thử.
- Evaluation: rule và ML router phải so sánh trên cùng validation/test set, không trộn data leak.

## 16.4. Test coverage không đồng nghĩa correctness

Một bộ 100% code coverage vẫn có thể sai về timeout/rate limit/streaming. Khi review test, tự hỏi:

1. Test có tạo đúng tình huống failure không?
2. Có kiểm tra upstream **không bị gọi** khi phải fail fast không?
3. Có kiểm tra request hoàn thành trong budget không?
4. Có kiểm tra metric/log phản ánh đúng nguyên nhân không?
5. Có kiểm tra sau failure hệ thống phục hồi được không?

---

# 17. Đánh giá, metrics và các hiểu lầm thường gặp

## 17.1. Metrics cơ bản và công thức

**Throughput:** request hoàn thành/giây (RPS).  
**Error rate:** request lỗi / tổng request; phải quyết định có tính client 4xx hay chỉ server 5xx, và công bố rõ.  
**Latency:** thời gian từ lúc gateway nhận request đến khi hoàn tất response; với streaming tách **time-to-first-byte / time-to-first-token** và **time-to-last-byte**.  
**p95:** 95% requests trong tập đo có latency không vượt phân vị này. Không phải "95% tốc độ tối đa".  
**Fallback rate:** requests có thực hiện fallback / tổng requests đã xử lý.  
**Provider availability (lab):** tỷ lệ thử gọi upstream thành công trong giai đoạn đo; không suy rộng thành uptime của provider ngoài đời.  
**Cost/request:** chi phí thực tế hoặc estimate mỗi request; chia rõ provider-reported và tự tính.  
**Routing regret:** chỉ được đo khi có ground truth quality/cost cho các lựa chọn tương ứng; không dùng classifier accuracy thay thế.

## 17.2. Baseline report nên có mẫu như sau

| Run ID | Scenario | Provider latency mock | Users | RPS | p95 | 5xx | Fallback rate |
|---|---|---|---:|---:|---:|---:|---:|
| `baseline-01` | Healthy | cheap=20 ms, quality=70 ms | 1 | đo thực tế | đo thực tế | đo | đo |
| `baseline-02` | Healthy | như trên | 10 | đo thực tế | đo thực tế | đo | đo |
| `fault-01` | cheap=503 | quality=70 ms | 10 | đo thực tế | đo thực tế | đo | đo |
| `fault-02` | cheap timeout | quality=70 ms | 10 | đo thực tế | đo thực tế | đo | đo |

Tuyệt đối **không điền số tưởng tượng** khi đưa lên README/GitHub; chỉ điền kết quả từ Locust/Prometheus thực tế và ghi máy, số workers, config, phiên bản image.

## 17.3. Thử nghiệm A/B có giá trị học nhất

**Experiment 1 — Retry policy:** no retry vs bounded retry with fallback. Đo error rate, p95 và upstream attempts; xem vì sao availability có thể tăng trong khi tail latency tăng.  
**Experiment 2 — Circuit breaker:** circuit OFF vs ON trong fault burst, đo attempted requests đến provider chết và thời gian hồi phục.  
**Experiment 3 — Async concurrency:** mức users 1/5/10/25, đo điểm bão hòa và queueing latency.  
**Experiment 4 — Rule vs ML router:** cùng set validation; đo accuracy/F1 của class selection và inference latency CPU; chưa claim better LLM response quality.  
**Experiment 5 — Prompt versions:** chỉ với small real-provider sample, dùng cùng inputs/rubric, so sánh quality, token cost, p95; có caveat số mẫu nhỏ.  
**Experiment 6 — Rollout reliability:** deploy broken version, kiểm tra readiness/rollout, đo số request lỗi và khả năng rollback.

## 17.4. Những lỗi nhận thức nên tránh

- **"Mock quality provider tốt hơn"**: tên `quality` không chứng minh bất kỳ chênh lệch chất lượng nào.
- **"Router có 90% accuracy nên tiết kiệm 40% LLM cost"**: accuracy nhãn task không suy ra cost/quality tradeoff thật.
- **"Docker chạy được tức là production-ready"**: chưa có security, observability, rollback hay backups.
- **"Có Prometheus tức là đã monitoring tốt"**: metrics sai labels/units hoặc không có alert/runbook vẫn ít giá trị.
- **"Retry sẽ giúp hệ thống đáng tin cậy hơn"**: retry sai có thể khuếch đại quá tải và chi phí.
- **"Kubernetes tự scale khi tạo Deployment"**: replicas=1 không phải autoscaling; scaling cần resources, HPA/metrics, workload phù hợp.
- **"MLflow lưu model là có thể rollback ứng dụng"**: phải pin cả code, config, model artifact, policy.
- **"MLOps chỉ có MLflow"**: MLflow không thay thế CI/CD, data validation, model serving, operations và incident handling.

---

# 18. Thiết kế vận hành tiết kiệm tài nguyên

## 18.1. Yêu cầu máy tính (ước lượng, không phải benchmark)

| Cấu hình | Nên vận hành thế nào |
|---|---|
| Ubuntu + RAM 8 GB | Compose core bình thường; chỉ bật monitoring hoặc MLflow hoặc kind ở từng buổi |
| Ubuntu + RAM 16 GB | Dễ chạy core + Prometheus/Grafana; vẫn nên tách kind nếu máy đang nặng |
| CPU 4 cores | Đủ học; load test throughput thấp hơn máy nhiều cores là bình thường |
| GPU | Không cần ở mọi giai đoạn |
| Internet | Cần tải packages/images và dùng GitHub; request mock local không cần Internet |
| Chi phí API | 0 nếu chỉ mock; real provider hoàn toàn optional |

Đừng tải model NLP nhiều GB chỉ để train router: TF-IDF + Logistic Regression đủ minh họa một vòng MLOps. Nếu đang dùng Ubuntu laptop có RAM giới hạn, đo bằng `free -h`, `htop`, `docker stats` trước khi nâng quy mô.

## 18.2. Cách chạy theo từng buổi học

**Buổi backend/reliability:**

```bash
docker compose up --build -d
docker compose ps
```

**Buổi monitoring:**

```bash
docker compose -f compose.yaml -f compose.monitoring.yaml up -d
# sau khi xong
docker compose -f compose.yaml -f compose.monitoring.yaml down
```

**Buổi MLOps:**

```bash
source .venv/bin/activate
mlflow server --backend-store-uri sqlite:///mlflow.db \
  --default-artifact-root ./mlruns --host 127.0.0.1 --port 5000
# terminal khác: python ml/train.py
```

**Buổi Kubernetes:** dừng Compose nếu dùng cùng port/thiếu RAM, rồi tạo kind.

## 18.3. Chi phí ẩn cần hiểu

- Dữ liệu tracing ở SaaS có quota/retention theo gói; xem chính sách hiện hành.
- GitHub Actions/GHCR phụ thuộc loại tài khoản, repository visibility và quota; đừng giả định mọi loại usage miễn phí vô hạn.
- Real LLM API có thể tính phí riêng dù gateway local miễn phí.
- Prometheus/Grafana/MLflow dùng disk; dọn dữ liệu thí nghiệm nếu không cần.
- Docker images và kind nodes chiếm khá nhiều disk khi build nhiều lần.

## 18.4. Khi nào thật sự cần Redis hoặc cloud?

Chỉ thêm **Redis** khi muốn học giới hạn quota chính xác giữa nhiều gateway processes/replicas hoặc shared circuit state; học in-memory trước. Chỉ thêm **cloud VM** khi muốn học deployment/network/public TLS, không bắt buộc để chứng minh bạn hiểu CI/CD, Docker, kubectl và monitoring.

---

# 19. Checklist cuối khóa và demo portfolio

Đây là checklist quyết định project đã "ổn và chuẩn" ở mức học thực hành hay chưa.

## 19.1. Functional correctness

- [ ] Client gọi một gateway endpoint thay vì gọi trực tiếp các mock providers.
- [ ] Auth, validation, routing, provider abstraction hoạt động.
- [ ] SSE streaming đúng protocol subset đã document.
- [ ] Retry/fallback/circuit/rate limit/concurrency có tests.
- [ ] Request timeout/cancellation không leak và không retry storm.
- [ ] Response errors có status/JSON shape nhất quán.

## 19.2. Observability và reliability

- [ ] JSON logs + correlation ID, không lộ secrets.
- [ ] Có dashboard traffic, latency p95, errors, upstream failures, fallback.
- [ ] Traces phân tích được ít nhất một request thất bại.
- [ ] Locust report có bối cảnh cấu hình và error/fault scenario.
- [ ] Có runbook, alert criteria và incident drills.

## 19.3. MLOps/LLMOps

- [ ] Prompt evaluation suite có nhãn/rubric, cố định subset test.
- [ ] Quality LLM thật (nếu có) đánh giá độc lập với mock.
- [ ] Token/cost chia reported, estimated, unknown.
- [ ] ML router dùng CPU, có train/val/test, baseline và Macro-F1.
- [ ] DVC quản lý data version; MLflow quản lý training runs/model versions.
- [ ] Serving pin version model và policy; có đánh giá hồi quy khi promote.

## 19.4. DevOps/CI/CD/Kubernetes

- [ ] GitHub Actions có test/build gates.
- [ ] Dockerfile an toàn ở mức lab: no root, no secrets, pinned dependencies.
- [ ] Release có git SHA + image ref + model/data/policy version.
- [ ] Staging smoke test và có rollback.
- [ ] kind Deployment/Service/probes hoạt động, `kubectl` debug được.
- [ ] Có bằng chứng rollout lỗi và phục hồi.

## 19.5. README công khai nên gồm

1. Project overview: giải thích gateway và use cases.
2. Architecture diagram và các thành phần.
3. Tech stack và lý do chọn.
4. Local quickstart (CPU-only), yêu cầu môi trường.
5. API examples, auth demo, lỗi và streaming limitations.
6. How to run tests / evaluation / load tests.
7. Observability screenshots **từ số liệu thật**.
8. MLOps lineage: data revision, model version, release manifest.
9. CI/CD và Kubernetes demo + rollback instructions.
10. Lessons learned, limitations, những gì muốn cải tiến tiếp.

## 19.6. Chuẩn đầu ra cho portfolio và phỏng vấn

Bạn nên giải thích được ba quyết định kiến trúc bằng bằng chứng:

**Quyết định 1 — Tại sao global request deadline và retry budget?** Đưa ra failure test và so sánh p95/no. of attempts trước-sau.  
**Quyết định 2 — Tại sao tách policy from ML classifier?** Nêu ví dụ class prediction không được vượt qua provider allowlist/budget/safety constraint.  
**Quyết định 3 — Tại sao version cả image, model và prompt?** Cho xem một rollback manifest có thể tái dựng phiên bản chạy ổn định.

Đừng chỉ nói "biết Kubernetes/MLflow"; hãy nói cụ thể sự cố bạn tạo ra, điều bạn quan sát, cách bạn sửa và kết quả đo.

---

# 20. Troubleshooting và tài liệu chính thức

## 20.1. 12 lỗi dễ gặp nhất trên Ubuntu

| Triệu chứng | Khả năng cao | Cách kiểm tra/sửa |
|---|---|---|
| `docker: command not found` | Chưa cài Docker hoặc PATH | Theo Docker Engine Ubuntu docs; `docker --version` |
| `permission denied /var/run/docker.sock` | Quyền Docker daemon | Kiểm tra service; `sudo docker ps`; hiểu rủi ro Docker group |
| `port 8000 already allocated` | Compose/Uvicorn/kubectl đang dùng port | `ss -ltnp | grep :8000`; dừng service cũ hoặc dùng 18000 |
| Gateway không thấy `cheap` | Sai hostname/network | Compose DNS là `cheap`, không phải `localhost` trong container |
| `connection refused` | Service chưa nghe/port sai | `docker compose ps/logs`; `curl` trong network tương ứng |
| Gateway 503 dù quality đang chạy | Timeout/budget, policy, upstream URL | Check logs with request_id; test provider health từ container |
| `pytest` import error | Venv chưa active hoặc package thiếu | `source .venv/bin/activate`; `python -m pip install -e '.[dev]'` |
| Prometheus target DOWN | Sai target hoặc endpoint `/metrics` chưa có | Targets page; `curl localhost:8000/metrics`, service DNS |
| Grafana datasource cannot connect | Dùng `localhost:9090` bên trong Grafana | Dùng `http://prometheus:9090` cùng Compose network |
| MLflow UI không thấy runs | Tracking URI khác/MLflow server chưa chạy | In `mlflow.get_tracking_uri()`, xem server logs |
| kind `ImagePullBackOff` | Image chưa `kind load`/tag `latest` | `kind load docker-image gateway:lab-v1`; IfNotPresent |
| K8s rollout stuck | Probe fail, image lỗi, OOM, config thiếu | `kubectl describe pod`, logs, events, rollout status |

## 20.2. Quy trình debug 7 bước áp dụng cho mọi lỗi

1. **Reproduce:** tìm request nhỏ nhất gây lỗi.
2. **Localize:** client/Gateway/provider/config/network/cluster?
3. **Read exact error:** HTTP status + logs + stacktrace + Kubernetes events.
4. **Observe:** metric/trace nào thay đổi? request_id có khớp?
5. **Form hypothesis:** một giả thuyết cụ thể có thể kiểm chứng.
6. **Make one change:** thay đúng một biến rồi chạy lại tests.
7. **Regression test:** viết test để lỗi đó không tái diễn; ghi vào runbook.

## 20.3. Tài liệu gốc được khuyến nghị

Các nguồn dưới đây là **tài liệu chính thức**, ưu tiên hơn video hoặc blog cũ khi API/CLI thay đổi:

| Chủ đề | Tài liệu |
|---|---|
| Ubuntu Docker Engine | https://docs.docker.com/engine/install/ubuntu/ |
| Docker Compose | https://docs.docker.com/compose/ |
| FastAPI | https://fastapi.tiangolo.com/ |
| HTTPX async | https://www.python-httpx.org/async/ |
| HTTPX timeouts | https://www.python-httpx.org/advanced/timeouts/ |
| pytest | https://docs.pytest.org/en/stable/ |
| Prometheus histogram | https://prometheus.github.io/client_python/instrumenting/histogram/ |
| Prometheus practices | https://prometheus.io/docs/practices/ |
| Grafana | https://grafana.com/docs/grafana/latest/ |
| OpenTelemetry Python | https://opentelemetry.io/docs/languages/python/instrumentation/ |
| OpenTelemetry HTTPX | https://opentelemetry.io/docs/languages/python/libraries/ |
| Langfuse SDK | https://langfuse.com/docs/observability/sdk/overview |
| Locust | https://docs.locust.io/en/stable/writing-a-locustfile.html |
| DVC | https://doc.dvc.org/start |
| MLflow backend | https://mlflow.org/docs/latest/tracking/backend-stores/ |
| MLflow Registry | https://mlflow.org/docs/latest/model-registry |
| GitHub Actions | https://docs.github.com/en/actions |
| Publish Docker image | https://docs.github.com/en/actions/tutorials/publish-packages/publish-docker-images |
| Kubernetes concepts | https://kubernetes.io/docs/concepts/ |
| kind quick start | https://kind.sigs.k8s.io/docs/user/quick-start/ |
| LiteLLM architecture | https://docs.litellm.ai/docs/proxy/architecture |

## 20.4. Khi bạn muốn nâng cấp project sau 8 tuần

**Ưu tiên tốt nhất nếu muốn tiếp tục làm LLMOps/infrastructure:** thêm Redis-backed distributed rate limit, nhiều gateway workers/replicas, OpenTelemetry collector + Tempo, rolling/canary deployment, cost-aware router có outcome evaluation thật.  
**Ưu tiên tốt nhất nếu muốn chuyển sang MLOps:** thêm scheduling data pipeline, model drift với ground truth có độ trễ, model registry promotion workflow, retraining triggers và data quality checks.  
**Không nhất thiết làm cả hai**: portfolio tốt khi một luồng từ vấn đề → thiết kế → bằng chứng → rollout → đo lường thật sự rõ ràng.

---

# LỊCH HỌC TỔNG HỢP: một trang để tự theo dõi

| Tuần | Trọng tâm | Deliverable bắt buộc | Gate |
|---|---|---|---|
| 1 | HTTP, async, Docker, FastAPI | Gateway + mocks chạy local, tests | Request/HTTP/network explainable |
| 2 | Reliability + SSE | Retry/deadline/circuit/rate limit/streaming | Failure tests, bounded attempts |
| 3 | Observability + Load | Logs, Prometheus/Grafana, trace, Locust | Debug được failure bằng evidence |
| 4 | LLMOps + Evaluation | Optional LLM adapter, eval suite, prompt versions | Không nhập nhằng mock/real metrics |
| 5 | MLOps | CPU classifier, DVC, MLflow, registry | Reproducible train + ML model pin |
| 6 | CI/CD | PR gates, image/manifest, staging, rollback | Bad release blocked and reversible |
| 7 | Kubernetes | kind deployment, services, probes, rollback | Successful local K8s incident drill |
| 8 | Hardening + Demo | Runbook, security, incident evidence, README | 15–20 phút demo hoàn chỉnh |

## Phương pháp học hiệu quả mỗi buổi (2–3 giờ)

- **20–30 phút:** đọc đúng phần docs liên quan task hôm nay.
- **60–90 phút:** code và chạy thử, tránh copy–paste khi chưa hiểu.
- **30–45 phút:** tạo lỗi có chủ ý, debug, viết test regression.
- **10–15 phút:** cập nhật README / learning notes / commit.

Nếu bí ở một giai đoạn, **không cần học hết lý thuyết tool trước**: nắm khái niệm đủ dùng, làm một end-to-end path rồi mở rộng. Khi cần rút còn 4 tuần, ưu tiên Tuần 1–3 và phần CI/CD Tuần 6; MLflow/kind có thể học sau khi đã vận hành API chắc.

## Điều kiện kết thúc project

Project được coi là hoàn thành khi người khác có thể:

1. Clone repo, start services, gọi gateway không có GPU/API key.
2. Gây lỗi provider và theo dõi kết quả fallback trong logs/metrics/traces.
3. Chạy evaluation, xem policy/model/data versions và tái lập model CPU.
4. Xác nhận CI gate, build image, deploy staging, rollback release lỗi.
5. Dùng kind chạy lại gateway và giải thích một lần Kubernetes incident.

**Tuyên bố phạm vi cuối:** Đây là một project học chuyên sâu **MLOps + LLMOps dưới góc nhìn infrastructure**. Nó không phải RAG, không tự host LLM, không cần GPU. Bộ mock chứng minh engineering/reliability; nếu không có đánh giá trên real LLM, không đưa ra bất kỳ tuyên bố nào về chất lượng hay chi phí mô hình thật.

**Bắt đầu từ đâu ngay hôm nay:** Mở **Mục 7**, chạy starter v0.1 trên Ubuntu, hoàn thành tuần 1; chỉ chuyển sang tuần 2 khi có thể tự giải thích request path, Docker DNS và failure HTTP statuses.