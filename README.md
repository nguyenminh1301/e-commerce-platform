![Per](assets/img/Per.svg)

An e-commerce backend for a perfume shop, with change data capture, full-text search and an AI shopping assistant.

[Quick start](#quick-start) ·
[API docs](docs/modules) ·
[Postman collection](postman/per-api-collection.json) ·
[Knowledge base](knowledge)

[![Tests](https://github.com/nguyenminh1301/e-commerce-platform/actions/workflows/test.yml/badge.svg)](https://github.com/nguyenminh1301/e-commerce-platform/actions/workflows/test.yml)

## About

Per is the backend of an online perfume store.

- Shoppers browse by brand, category and country of origin, search the catalogue, fill a cart
  and pay through PayOS.
- Every product carries perfume details: fragrance family, sillage, longevity and seasonality.
- An AI assistant answers questions such as "a fresh scent for summer" with product
  recommendations, drawing on the catalogue and the shop's own guides.
- Administrators manage the catalogue, images, users and the assistant's knowledge base through
  the same API.

It is one Spring Boot application (a modular monolith) on Java 21. Every endpoint lives under
`/per`.

## Architecture

### How a request flows through the system

[![Runtime architecture: client, edge, Spring Boot modular monolith, Kafka, data stores and external services](docs/asset/runtime.png)](docs/asset/runtime.png)

A request takes this path:

1. In production it arrived through a Cloudflare Tunnel and Nginx on a single VPS.
2. Inside the application, a JWT filter, admin role checks and Resilience4j rate limits, circuit
   breakers and retries run first. These live in the app itself; there is no separate gateway.
3. One of thirteen modules handles it:

| Group | Modules |
|-------|---------|
| Identity | `auth`, `user` |
| Catalogue | `product`, `brand`, `category`, `made-in` |
| Commerce | `cart`, `order`, `payment` |
| Platform | `media`, `mail`, `rag` |

PostgreSQL is the system of record, and the other stores are rebuilt from it:

| Store | Holds |
|-------|-------|
| PostgreSQL 16 | All business data, migrated by Flyway, with logical WAL for Debezium |
| Redis 7 | Cached catalogue lookups and refresh tokens |
| Elasticsearch 7.17 | Full-text search over products, brands and categories |
| Qdrant | Embeddings for the AI assistant, in four collections |

Changes reach search and the assistant through Kafka:

- Debezium reads PostgreSQL's write-ahead log for the catalogue tables and publishes each change.
- The application publishes its own index events and email jobs.
- Consumers inside the application update Elasticsearch and Qdrant, or send the email. A failed
  email goes to a dead-letter topic.

Checkout locks each product variant row before it deducts stock, so two shoppers cannot buy the
same last unit. PayOS then reports the payment by webhook: the order becomes paid, or it is
cancelled and its stock returned. A scheduler cancels payments that expire.

The application calls four external services: OpenAI for embeddings and answers, PayOS for
payments, Cloudinary for product and brand images, and an SMTP server for welcome and
password-reset email.

### How the AI assistant answers

[![RAG pipeline: catalogue and knowledge ingest into Qdrant, and the question-to-answer loop](docs/asset/rag.png)](docs/asset/rag.png)

The assistant uses retrieval-augmented generation. Two pipelines fill Qdrant:

- Catalogue changes arrive through Debezium and Kafka. Each changed product, brand or category
  is flattened to text, embedded with OpenAI and written to its collection.
- The markdown files in [`knowledge/`](knowledge) (policies, customer-service guide, fragrance
  guide) are split into 3,200-character chunks with 400 characters of overlap, embedded and
  stored in `knowledge_vectors`. This runs at startup when the collection is empty, or when an
  administrator calls `POST /per/rag/index/knowledge`.

A question sent to `POST /per/rag/chat` goes through five steps:

1. The question is embedded with the same model used at ingest.
2. All four collections are searched, keeping the top 3 matches from each that score 0.3 or
   higher.
3. The matches are merged into one context block and placed in a prompt with the instructions
   from `knowledge/system-prompt.txt`, the question and the expected JSON format.
4. The OpenAI chat model writes the answer. The application parses the JSON, or falls back to a
   canned answer if parsing fails.
5. The reply holds a summary, a detailed answer, recommended products and next steps.

`GET /per/rag/chat/stream` streams the same answer as plain-text server-sent events.

### How code reaches production and how it is watched

[![Delivery, logging and monitoring: GitHub, GitHub Actions, Jenkins, Docker Hub, ELK, Prometheus and Grafana](docs/asset/ops.png)](docs/asset/ops.png)

GitHub Actions runs the unit tests for every push and pull request to `main` and `release`.
Every push also starts Jenkins, which delivers the change:

1. Builds the application inside a multi-stage Docker build.
2. Pushes the image to Docker Hub as `nguyenminh1301/per`, tagged with the project version and
   `latest`.
3. Restarts the stack on the VPS with Docker Compose.

The running system is watched like this:

- Logs: the application sends JSON logs over TCP to Logstash, which indexes them into
  Elasticsearch (the same instance that serves product search). Kibana searches them.
- Metrics: Prometheus scrapes `/actuator/prometheus` and the host, Redis and PostgreSQL
  exporters. Grafana turns them into dashboards and alerts.
- Access: in production, Kibana and Grafana sat behind a Cloudflare Zero Trust subdomain.

## Tech stack

| Area | Technology |
|------|------------|
| Application | Java 21, Spring Boot 3.5, Spring Security with JWT, Spring Data JPA, Flyway, MapStruct, Resilience4j; tests with JUnit 5 and JaCoCo |
| AI | Spring AI 1.0 (milestone), OpenAI models, Qdrant |
| Data and messaging | PostgreSQL 16 with pgvector, Redis 7, Elasticsearch 7.17, Kafka (Confluent 7.6), Debezium 2.5 |
| External services | PayOS, Cloudinary, SMTP |
| Delivery and observability | GitHub Actions, Jenkins, Docker Hub, Docker Compose, Logstash, Kibana, Prometheus, Grafana |

## Documentation

| Document | Covers |
|----------|--------|
| [`docs/modules/`](docs/modules) | One guide per module: auth, user, product, cart, order, payment, media, rag and common |
| [`docs/kafka/`](docs/kafka) | Topics and events, retries and the dead-letter queue |
| [`docs/elasticsearch/`](docs/elasticsearch) | Indexing strategy, re-indexing and queries |
| [`docs/cache/`](docs/cache) | Redis cache configuration and the token blacklist |
| [`docs/resilience-patterns/`](docs/resilience-patterns) | Rate limiting and circuit breakers |

## Quick start

You need Java 21 and Docker Desktop.

1. Clone the repository:

   ```bash
   git clone https://github.com/nguyenminh1301/e-commerce-platform.git
   cd e-commerce-platform
   ```

2. Create `.env` and fill in the credentials. The application and Docker Compose both read it.

   ```bash
   cp .env.example .env
   ```

   > Important: the AI assistant needs `OPENAI_API_KEY`, `OPENAI_CHAT_MODEL` and
   > `OPENAI_EMBEDDING_MODEL`. Payments, image uploads and email need the PayOS, Cloudinary
   > and SMTP credentials.

3. Point the application at your machine. The example uses container host names, which only
   resolve inside Docker, so change these five lines in `.env`:

   ```dotenv
   POSTGRES_URL=jdbc:postgresql://localhost:5432/postgres
   REDIS_HOST=localhost
   QDRANT_HOST=localhost
   KAFKA_BOOTSTRAP_SERVERS=localhost:29092
   ELASTICSEARCH_URI=http://localhost:9200
   ```

4. Start the infrastructure. The Debezium connector registers itself once Kafka Connect is up.

   ```bash
   docker compose up -d
   ```

5. Start the application. On first start, Flyway creates the schema and loads demo data, and
   the assistant indexes the catalogue and the knowledge base into Qdrant.

   ```bash
   ./mvnw spring-boot:run
   ```

   `make run` does steps 4 and 5 together after checking that Java, Docker and `.env` are
   present.

The API is now on `http://localhost:8080/per`, and Swagger UI on
`http://localhost:8080/per/api-docs`.

### Local consoles

Data and messaging:

- Kafka UI: `http://localhost:8090`
- Debezium UI: `http://localhost:8084`
- Qdrant: `http://localhost:6333/dashboard`

Logs and metrics:

- Kibana: `http://localhost:5601`
- Grafana: `http://localhost:3000`
- Prometheus: `http://localhost:9091`

### Running the tests

```bash
./mvnw clean test
```

## License

Released under the [MIT License](LICENSE).
