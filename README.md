# Microservice Demo

[![License](https://img.shields.io/badge/license-SATA-blue)](./LICENSE.md)

This repository is a documentation-maintained fork of [Joker666/microservice-demo](https://github.com/Joker666/microservice-demo). It preserves the upstream design and SATA license while making the project easier to evaluate, run, and maintain as a demo repository.

## Overview

`microservice-demo` is a polyglot task-management demo built as a small microservice system:

- **User service** for registration, login, and JWT verification
- **Project service** for projects and tags
- **Task service** for tasks and task assignment
- **API service** as a gRPC gateway/router across the backend services
- **HTTP gateway** for REST-like access generated from protobuf annotations

It is intended as a learning/demo project rather than a production-ready deployment.

## Architecture

### Service map

| Service | Stack | Data store | Internal port | Docker host port | Responsibility |
| --- | --- | --- | --- | --- | --- |
| `userService` | Node.js | MongoDB | `50051` | `8080` | User registration, login, token verification |
| `projectService` | Python | MySQL | `50052` | `8081` | Project and tag management |
| `taskService` | Ruby | PostgreSQL | `50053` | `8082` | Task creation, updates, and assignment |
| `apiService` | Go | None | `50059` | `8083` | gRPC API aggregation and routing |
| `api-gateway` | Go gRPC-Gateway | None | `9090` | `9090` | HTTP-to-gRPC proxy and Swagger UI |

### Service dependencies

- `projectService` depends on `userService`
- `taskService` depends on `userService` and `projectService`
- `apiService` depends on all backend services
- `api-gateway` depends on `apiService`

### Repository layout

```text
.
├── apiService/        # Go API server, proxy, Swagger UI assets
├── projectService/    # Python gRPC project service
├── taskService/       # Ruby gRPC task service
├── userService/       # Node.js gRPC user service
├── protos/            # Shared protobuf definitions and generated Go code
├── build.sh           # Regenerates protobuf artifacts for all services
├── up.sh              # Runs build.sh, then starts Docker Compose
├── docker-compose.yml # Demo stack orchestration
└── LICENSE.md         # Upstream SATA license
```

## Prerequisites

### For the recommended Docker Compose flow

- Docker Engine
- Docker Compose v2 (`docker compose`)

### For local per-service development

- Node.js and npm for `userService`
- Python 3.8 for `projectService` (`pipenv` is optional but supported)
- Ruby 2.7 and Bundler for `taskService`
- Go 1.15 with `GOPATH` configured for `apiService`

### For protobuf regeneration

Root-level `./build.sh` regenerates protobuf/client artifacts and requires additional tooling that is **not** needed for a normal Docker Compose run:

- `protoc`
- `grpc_tools_node_protoc`
- Python `grpcio-tools`
- `grpc_tools_ruby_protoc`
- `pipenv`
- Go gRPC / grpc-gateway plugins available in `GOPATH`

## Quick start

The lowest-friction way to run the demo is Docker Compose:

```bash
docker compose up --build
```

Then open:

- HTTP gateway: `http://localhost:9090`
- Swagger UI: `http://localhost:9090/swagger-ui/`
- OpenAPI document: `http://localhost:9090/swagger.json`

To stop the stack:

```bash
docker compose down
```

To remove the persisted demo databases as well:

```bash
docker compose down -v
```

### About `./up.sh`

`./up.sh` is a convenience wrapper for contributors who want to regenerate artifacts first:

```bash
./up.sh
```

It runs `./build.sh` before Compose, so it needs the full protobuf toolchain listed above. If you only want to start the demo containers, use `docker compose up --build` directly.

## Running services individually

The committed `.env` files contain demo defaults. Review and replace them before using the repo outside local experimentation.

### User service

```bash
cd userService
npm install
npm run start
```

Needs MongoDB and the following settings in `.env`:

- `DB_URI`
- `DB_NAME`
- `TOKEN_SECRET`
- `TOKEN_LIFE`
- `HOST`
- `PORT`

### Project service

```bash
cd projectService
pipenv install
pipenv run python service.py
```

If you do not use `pipenv`:

```bash
pip install -r requirements.txt
python service.py
```

Needs MySQL and these settings in `.env`:

- `DB_URI`
- `HOST`
- `PORT`

### Task service

```bash
cd taskService
bundle install
ruby server.rb
```

Needs PostgreSQL plus running user/project services. Configure:

- `DB_URI`
- `HOST`
- `PORT`
- `USER_ADDRESS`
- `PROJECT_ADDRESS`

When run in Docker Compose, the container entrypoint uses `./init` to prepare the service before startup.

### API service and proxy

```bash
cd apiService
./build.sh
./run.sh
```

That starts the gRPC API server. To start the HTTP proxy instead:

```bash
cd apiService
./proxy.sh
```

The API service reads:

- `HOST`
- `PORT`
- `USER_ADDRESS`
- `PROJECT_ADDRESS`
- `TASK_ADDRESS`
- `PROXY_PORT` (proxy mode)

## Configuration reference

### Docker Compose defaults

`docker-compose.yml` wires the demo with these container-level defaults:

- MongoDB for `userService`
- MySQL 5.7 for `projectService`
- PostgreSQL for `taskService`
- `api-gateway` published on port `9090`

Host-mapped gRPC ports are:

- `8080` → `userService`
- `8081` → `projectService`
- `8082` → `taskService`
- `8083` → `apiService`

Note that the standalone `apiService/.env` uses `PROXY_PORT=8081`, while Docker Compose overrides the proxy to `9090` to avoid host-port conflicts.

## API and protobuf workflow

The protobuf source of truth lives under `protos/`.

- `protos/user/*.proto` defines the user service contract
- `protos/project/*.proto` defines the project service contract
- `protos/task/*.proto` defines the task service contract
- `protos/api/api.proto` defines the aggregated API and HTTP annotations

Running the root build script:

```bash
./build.sh
```

regenerates:

- Node.js stubs in `userService/proto/`
- Python stubs in `projectService/proto/`
- Ruby stubs in `taskService/proto/`
- Go protobuf/gRPC code in `protos/`
- gRPC-Gateway bindings and `apiService/www/api.swagger.json`

Documented HTTP endpoints are generated from `protos/api/api.proto`, including:

- `POST /v1/user/register`
- `POST /v1/user/login`
- `POST /v1/project/create`
- `GET /v1/project/get/{project_id}`
- `POST /v1/task/create`
- `POST /v1/task/update`
- `GET /v1/project/{project_id}/task/list`

## Troubleshooting

- **`./up.sh` fails with missing tools**: use `docker compose up --build` or install the full protobuf toolchain required by the root `build.sh`.
- **Compose warns that `version` is obsolete**: this is a Docker Compose v2 warning from the current file format and does not block startup.
- **Databases are still initializing**: on a first run, wait briefly and restart the affected service or rerun `docker compose up --build`.
- **Services cannot reach each other in local mode**: verify each `.env` file uses the correct local service addresses and ports.
- **Swagger UI is blank or missing**: regenerate artifacts with the root `./build.sh` so `apiService/www/api.swagger.json` is present.

## Keeping the fork current

If you want to sync this fork with upstream while preserving fork-specific docs:

```bash
git remote add upstream https://github.com/Joker666/microservice-demo.git
git fetch upstream
git merge upstream/main
```

After syncing, re-check:

- `README.md` for fork-specific guidance
- `LICENSE.md` to ensure upstream notices remain intact
- generated protobuf and Swagger artifacts if upstream proto files changed

## Contributing

Issues and pull requests are welcome for documentation fixes, demo usability improvements, and non-breaking maintenance.

If a change is better suited to the original project, consider contributing it upstream as well:

- Upstream repository: https://github.com/Joker666/microservice-demo
- This fork: https://github.com/ricardojjulia/microservice-demo

## License and attribution

This fork keeps the upstream [SATA license](./LICENSE.md) and original attribution to **MD Ahad Hasan**. Documentation updates in this repository do not replace or relicense the upstream work.

See also:

- Upstream project: https://github.com/Joker666/microservice-demo
- Google Cloud microservices demo: https://github.com/GoogleCloudPlatform/microservices-demo
- microservices-demo reference project: https://github.com/microservices-demo/microservices-demo
