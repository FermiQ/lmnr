# app-server/src/main.rs

## Overview

The `app-server/src/main.rs` file serves as the primary entry point and orchestrator for the application server. It is responsible for initializing a wide array of services and configurations, setting up database connections (PostgreSQL and ClickHouse), message queues (RabbitMQ or Tokio MPSC channels based on feature flags), caching mechanisms (Redis or in-memory), object storage (AWS S3 or a mock implementation), and clients for external gRPC services like Agent Manager and Machine Manager.

The application starts by loading environment variables, initializing cryptographic libraries and logging. It then establishes shared resources like database pools and message queue connections. Two main server components are launched in separate threads:
1.  An **Actix HTTP server**: Handles REST API requests, serves various endpoints for workspaces, projects, traces, datasets, evaluations, etc. It also spawuls multiple background Tokio tasks (workers) to process messages from different queues (e.g., spans, browser events, evaluators) asynchronously.
2.  A **Tonic gRPC server**: Primarily handles gRPC services, with a key example being the `TraceServiceServer` for ingesting trace data.

Feature flags heavily influence the application's setup, allowing components like full message queueing, S3 storage, and external service integrations to be conditionally compiled or enabled.

## Key Components

- `main()`: The core function that orchestrates the entire application startup. It performs:
    - Initialization of `sodiumoxide` and `rustls` crypto providers.
    - Loading of environment variables via `dotenv`.
    - Setup of `env_logger`.
    - Parsing of critical configurations like payload limits and port numbers from environment variables.
    - Initialization of shared application state:
        - **Cache**: `RedisCache` if `REDIS_URL` is set, otherwise `InMemoryCache`.
        - **Database (PostgreSQL)**: `sqlx::postgres::PgPool` connected via `DATABASE_URL`.
        - **Message Queues**:
            - `spans_message_queue`: RabbitMQ (`lapin`) if `Feature::FullBuild` is enabled and `RABBITMQ_URL` is set, otherwise `mq::tokio_mpsc::TokioMpscQueue`.
            - `browser_events_message_queue`: Similar setup as spans MQ.
            - `evaluators_message_queue`: Similar setup as spans MQ.
            - `agent_manager_workers`: A dedicated MPSC-based worker queue for agent tasks.
        - **Storage (S3)**: `storage::s3::S3Storage` if `Feature::Storage` is enabled, using `AWS_SDK_CONFIG` and `S3_TRACE_PAYLOADS_BUCKET`. Otherwise, `storage::mock::MockStorage`.
        - **ClickHouse Client**: Configured via `CLICKHOUSE_URL`, `CLICKHOUSE_USER`, and `CLICKHOUSE_PASSWORD`.
        - **MachineManager Client**: `MachineManagerServiceClient` (gRPC) if `Feature::MachineManager` is enabled, connecting to `MACHINE_MANAGER_URL_GRPC`. Otherwise, a mock manager.
        - **AgentManager Client**: `AgentManagerServiceClient` (gRPC) if `Feature::AgentManager` is enabled, connecting to `AGENT_MANAGER_URL`. Otherwise, a mock manager.
        - **HTTP Client for Evaluators**: `reqwest::Client` configured for online evaluators if `Feature::Evaluators` is enabled.
    - Spawning a new thread for the Actix `HttpServer`.
    - Spawning a new thread for the Tonic `Server` (gRPC).
    - Waits for these server threads to complete.

- `HttpServer::new(...)` (within the HTTP server thread): Configures the Actix web server. This includes:
    - Wrapping with middleware: `Logger` (for request logging), `NormalizePath` (for URL normalization), and `HttpAuthentication` (for bearer token auth using `auth::validator` and `auth::project_validator`).
    - Setting application-level data (`app_data`) like payload size limits (`JsonConfig`, `PayloadConfig`), and shared `Arc`-wrapped instances of the cache, DB pool, message queues, ClickHouse client, storage, service managers, etc.
    - Registering numerous API routes defined in `api::v1::*` and `routes::*` modules, organized under scopes like `/v1`, `/api/v1/workspaces`, `/api/v1/projects/{project_id}`, etc.
    - Spawning background Tokio tasks using `tokio::spawn` for processing messages from `spans_mq_for_http`, `browser_events_message_queue`, and `evaluators_message_queue`. These workers interact with the database, cache, ClickHouse, and external evaluator services.

- `Server::builder()` (within the gRPC server thread): Configures the Tonic gRPC server. This includes:
    - Adding the `TraceServiceServer` (from `traces::grpc_service::ProcessTracesService`) which uses the shared DB, cache, and spans message queue.
    - Enabling Gzip compression and setting message size limits.
    - Setting up a graceful shutdown mechanism triggered by `wait_stop_signal`.

- **Modules**: The `main.rs` file heavily relies on a modular architecture, utilizing numerous internal modules such as:
    - `db`: Database interaction logic.
    - `mq`: Message queue abstractions and implementations (RabbitMQ, Tokio MPSC).
    - `cache`: Caching abstractions and implementations (Redis, InMemory).
    - `storage`: Object storage abstractions (S3, Mock).
    - `agent_manager`, `machine_manager`: Clients and mocks for external gRPC services.
    - `api::v1::*`, `routes::*`: HTTP route handlers and API definitions.
    - `features`: Feature flag management.
    - `traces`, `browser_events`, `evaluators`: Logic for processing specific types of asynchronous tasks.
    - `auth`: Authentication logic.
    - `runtime`: Tokio runtime creation.

## Important Variables/Constants

Many configuration values are loaded from environment variables. Key ones include:
- `RUST_LOG`: Controls logging verbosity.
- `HTTP_PAYLOAD_LIMIT`: Max size for HTTP request bodies (defaults to 5MB).
- `GRPC_PAYLOAD_LIMIT`: Max size for gRPC messages (defaults to 25MB).
- `PORT`: Listening port for the HTTP server (defaults to 8000).
- `GRPC_PORT`: Listening port for the gRPC server (defaults to 8001).
- `REDIS_URL`: If provided, Redis is used for caching; otherwise, an in-memory cache is used.
- `DATABASE_URL`: Connection string for the PostgreSQL database.
- `DATABASE_MAX_CONNECTIONS`: Maximum number of connections in the PostgreSQL pool.
- `RABBITMQ_URL`: Connection string for RabbitMQ. Used if `Feature::FullBuild` is enabled.
- `AWS_REGION`: AWS region for services like S3.
- `S3_TRACE_PAYLOADS_BUCKET`: Name of the S3 bucket for storing trace payloads. Used if `Feature::Storage` is enabled.
- `CLICKHOUSE_URL`, `CLICKHOUSE_USER`, `CLICKHOUSE_PASSWORD`: Credentials for ClickHouse database.
- `MACHINE_MANAGER_URL_GRPC`: gRPC endpoint for the Machine Manager service.
- `AGENT_MANAGER_URL`: gRPC endpoint for the Agent Manager service.
- `ONLINE_EVALUATORS_SECRET_KEY`, `PYTHON_ONLINE_EVALUATOR_URL`: Configuration for evaluator services.
- `NUM_SPANS_WORKERS_PER_THREAD`, `NUM_BROWSER_EVENTS_WORKERS_PER_THREAD`, `NUM_EVALUATORS_WORKERS_PER_THREAD`: Number of background workers spawned for each queue type within the HTTP server.

## Usage Examples

This file compiles into the main binary for the application server. It is typically run directly after building:

```bash
# (Assuming the binary is named app-server)
./app-server
```

Command-line arguments are not explicitly parsed by this `main.rs`; configuration is primarily through environment variables.

## Dependencies and Interactions

- **Web Framework**: `actix-web` for the HTTP server, `actix-web-httpauth` for authentication.
- **gRPC Framework**: `tonic` for the gRPC server.
- **Database**:
    - `sqlx` (specifically `sqlx::postgres`) for asynchronous PostgreSQL operations.
    - `clickhouse` client for interacting with a ClickHouse database.
- **Message Queues**:
    - `lapin` for RabbitMQ communication (if `Feature::FullBuild` is enabled).
    - `tokio::sync::mpsc` for local, in-process messaging as a fallback or for specific internal queues.
- **Caching**:
    - `redis` crate for connecting to a Redis server.
    - Internal `InMemoryCache` as an alternative.
- **Cloud Services**:
    - `aws-config` and `aws-sdk-s3` for interacting with AWS S3 for object storage.
- **HTTP Client**: `reqwest` for making outbound HTTP calls (e.g., to evaluator services).
- **Service Clients (gRPC)**:
    - `AgentManagerServiceClient` for `agent-manager`.
    - `MachineManagerServiceClient` for `machine-manager`.
    (These are generated by Tonic from `.proto` definitions, likely within their respective modules).
- **Concurrency**: `tokio` for the asynchronous runtime, `std::thread` for separating the HTTP and gRPC server execution environments.
- **Configuration**: `dotenv` for loading `.env` files.
- **Logging**: `env_logger` and `log` facade.
- **Cryptography**: `sodiumoxide` for general cryptographic functions, `rustls` for TLS.
- **Feature Flags**: A custom `features` module (e.g., `is_feature_enabled(Feature::X)`) is used to conditionally compile or enable different components (e.g., using mock services vs. real clients, S3 vs. mock storage).
- **Internal Modules**: The application is heavily modularized. `main.rs` integrates functionalities from modules like `db`, `mq`, `cache`, `storage`, `api`, `routes`, `traces`, `browser_events`, `evaluators`, `agent_manager`, `machine_manager`, `auth`, `runtime`, etc.
- **Orchestration**: `main()` initializes and holds `Arc` (Atomically Referenced Counter) handles to shared resources like database pools, cache instances, and message queue producers/consumers. These are then cloned and passed into the Actix application state (for HTTP handlers and workers) and the Tonic service constructors (for gRPC services). This ensures safe concurrent access to shared state.
```
