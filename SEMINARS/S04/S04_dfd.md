# S04 — Data Flow Diagram (DFD)

## Контекст

DFD описывает архитектуру сервиса на уровне контуров (Client ↔ API ↔ Service ↔ Storage ↔ External) с выделением **границ доверия** и **ключевых потоков данных**.
Эта диаграмма будет использоваться в STRIDE-анализе и приоритизации рисков L×I.

## Мини-DFD (Mermaid)

```mermaid
flowchart LR
  %% ========= Trust boundaries (subgraphs) =========
  classDef boundary stroke-dasharray: 4 4,stroke:#888,color:#333

  subgraph Internet["Trust Boundary: Internet"]
    Client[Client / Front]
  end
  Internet:::boundary

  subgraph Service["Trust Boundary: Service"]
    API[API / Controller]
    SearchSvc[Search Service]
    OrdersSvc[Orders Service]
    PaySvc[Payment Service]
  end
  Service:::boundary

  subgraph Storage["Trust Boundary: Storage"]
    DB[(DB / RDBMS)]
    SearchIndex[(Search Index)]
  end
  Storage:::boundary

  subgraph External["Trust Boundary: External Integrations"]
    PayProvider[External Payment Provider]
    Email[Email/SMS Provider]
    Q[[Queue / Events]]
  end
  External:::boundary

  %% ========= Edges: Internet <-> Service =========
  Client -->|HTTPS, JWT (AuthN), query, correlation_id (NFR: Security-AuthN; RateLimiting; Observability/Logging; API-Contract/RFC7807)| API
  API -->|HTTP 200/4xx/5xx, RFC7807, correlation_id (NFR: API-Contract/Errors; Observability)| Client

  %% ========= Search (US-007) =========
  API -->|DTO: SearchRequest(q, filters), tenant-id (NFR: InputValidation; Limits/PageSize; AuthZ(optional))| SearchSvc
  SearchSvc -->|Query DSL / DTO (NFR: Timeouts; Retry)| SearchIndex
  SearchIndex -->|SearchResult docs (no PII) (NFR: Data-Minimization)| SearchSvc
  SearchSvc -->|DTO: SearchResponse (NFR: Performance/SLO)| API

  %% ========= Order & Payment (US-008) =========
  API -->|DTO: Cart/OrderCreate (PII), tenant-id, idempotency-key (NFR: Security-AuthZ/RBAC; Data-Integrity; Idempotency; InputValidation; Auditability)| OrdersSvc
  OrdersSvc -->|SQL (create order), PII (NFR: Privacy/PII; Encryption-at-Rest)| DB
  DB -->|OrderRecord| OrdersSvc

  OrdersSvc -->|Domain Event: order.created (NFR: Observability/Tracing)| Q
  Q -->|Event: notify.user| Email
  Email -->|Delivery status (NFR: Timeouts; Retry; CircuitBreaker)| Q

  %% Payment session init
  API -->|DTO: PaymentSessionRequest(order_id) (NFR: AuthZ/RBAC; InputValidation; Idempotency)| PaySvc
  PaySvc -->|HTTPS, payment-init (payment), correlation_id (NFR: Timeouts; Retry; CircuitBreaker; TLS)| PayProvider
  PayProvider -->|Redirect URL / session_id (NFR: Data-Minimization)| PaySvc
  PaySvc -->|DTO: PaymentSessionResponse (NFR: API-Contract)| API

  %% Webhook
  PayProvider -->|HTTPS, webhook (status), HMAC signature (NFR: Webhook-Auth (HMAC/secret-rotation); Replay-Protection; RateLimiting)| API
  API -->|DTO: PaymentStatus, idempotency-key (NFR: Idempotency; Auditability)| PaySvc
  PaySvc -->|SQL (update payment/order), PII (NFR: Data-Integrity; Encryption-at-Rest)| DB
  PaySvc -->|Event: payment.succeeded/failed (NFR: Observability)| Q
  Q -->|Event: order.fulfill / notify (NFR: Reliability)| OrdersSvc
