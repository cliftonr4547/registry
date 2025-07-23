# MCP Registry

A community driven registry service for Model Context Protocol (MCP) servers.

## Development Status

This project is being built in the open and is currently in the early stages of development. Please see the [overview discussion](https://github.com/modelcontextprotocol/registry/discussions/11) for the project scope and goals. If you would like to contribute, please check out the [contributing guidelines](CONTRIBUTING.md).

## Overview

The MCP Registry service provides a centralized repository for MCP server entries. It allows discovery and management of various MCP implementations with their associated metadata, configurations, and capabilities.

## Features

- RESTful API for managing MCP registry entries (list, get, create, update, delete)
- Health check endpoint for service monitoring
- Support for various environment configurations
- Graceful shutdown handling
- MongoDB and in-memory database support
- Comprehensive API documentation
- Pagination support for listing registry entries
- **Seed data export/import composability with HTTP support**
- **Registry instance data sharing via HTTP endpoints**

## Getting Started

### Prerequisites

- Go 1.23.x (required - check with `go version`)
- MongoDB
- Docker (optional, but recommended for development)

For development:
- golangci-lint v1.61.0 - Install with:
  ```bash
  curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin v1.61.0
  ```

## Running

The easiest way to get the registry running is to use `docker compose`. This will setup the MCP Registry service, import the seed data and run MongoDB in a local Docker environment.

```bash
# Build the Docker image
docker build -t registry .

# Run the registry and MongoDB with docker compose
docker compose up
```

This will start the MCP Registry service and MongoDB with Docker, exposing it on port 8080.

### Registry Composability

The registry supports composability through seed data import/export, allowing registry instances to share data:

```bash
# Start first registry with seed data
MCP_REGISTRY_SEED_FILE_PATH=data/seed_2025_05_16.json ./registry

# Start second registry importing from first via HTTP
./registry --seed-file-path=http://localhost:8080/v0/seed.json
```

You can also use environment variables:
```bash
# Import from local file
MCP_REGISTRY_SEED_FILE_PATH=data/seed_2025_05_16.json ./registry

# Import from remote registry
MCP_REGISTRY_SEED_FILE_PATH=http://other-registry:8080/v0/seed.json ./registry
```

## Building

If you prefer to run the service locally without Docker, you can build and run it directly using Go.

```bash
# Build a registry executable
go build ./cmd/registry
```
This will create the `registry` binary in the current directory. You'll need to have MongoDB running locally or with Docker.

By default, the service will run on `http://localhost:8080`.

### Command Line Options

The registry supports command line flags for configuration:

```bash
# Specify seed file path via command line flag
./registry --seed-file-path=data/seed_2025_05_16.json

# Import from HTTP endpoint
./registry --seed-file-path=http://localhost:8080/v0/seed.json
```

Command line flags take precedence over environment variables.

## Development

### Linting

The project uses golangci-lint with extensive checks. Always run linting before pushing:

```bash
# Run all linters (same as CI)
golangci-lint run --timeout=5m

# Check formatting
gofmt -s -l .

# Fix formatting
gofmt -s -w .
```

### Git Hooks (Optional)

To automatically run linting before commits:

```bash
git config core.hooksPath .githooks
```

This will prevent commits that fail linting or have formatting issues.

## Project Structure

```
├── api/           # OpenApi specification
├── cmd/           # Application entry points
├── config/        # Configuration files
├── internal/      # Private application code
│   ├── api/       # HTTP server and request handlers
│   ├── config/    # Configuration management
│   ├── model/     # Data models
│   └── service/   # Business logic
├── pkg/           # Public libraries
├── scripts/       # Utility scripts
└── tools/         # Command line tools
    └── publisher/ # Tool to publish MCP servers to the registry
```

## API Documentation

The API is documented using Swagger/OpenAPI. You can access the interactive Swagger UI at:

```
/v0/swagger/index.html
```

This provides a complete reference of all endpoints with request/response schemas and allows you to test the API directly from your browser.

## API Endpoints

### Health Check

```
GET /v0/health
```

Returns the health status of the service:
```json
{
  "status": "ok"
}
```

### Registry Endpoints

#### List Registry Server Entries

```
GET /v0/servers
```

Lists MCP registry server entries with pagination support.

Query parameters:
- `limit`: Maximum number of entries to return (default: 30, max: 100)
- `cursor`: Pagination cursor for retrieving next set of results

Response example:
```json
{
  "servers": [
    {
      "id": "123e4567-e89b-12d3-a456-426614174000",
      "name": "Example MCP Server",
      "url": "https://example.com/mcp",
      "description": "An example MCP server",
      "created_at": "2025-05-17T17:34:22.912Z",
      "updated_at": "2025-05-17T17:34:22.912Z"
    }
  ],
  "metadata": {
    "next_cursor": "123e4567-e89b-12d3-a456-426614174000",
    "count": 30
  }
}
```

#### Get Server Details

```
GET /v0/servers/{id}
```

Retrieves detailed information about a specific MCP server entry.

Path parameters:
- `id`: Unique identifier of the server entry

Response example:
```json
{
  "id": "01129bff-3d65-4e3d-8e82-6f2f269f818c",
  "name": "io.github.gongrzhe/redis-mcp-server",
  "description": "A Redis MCP server (pushed to https://github.com/modelcontextprotocol/servers/tree/main/src/redis) implementation for interacting with Redis databases. This server enables LLMs to interact with Redis key-value stores through a set of standardized tools.",
  "repository": {
    "url": "https://github.com/GongRzhe/REDIS-MCP-Server",
    "source": "github",
    "id": "907849235"
  },
  "version_detail": {
    "version": "0.0.1-seed",
    "release_date": "2025-05-16T19:13:21Z",
    "is_latest": true
  },
  "packages": [
    {
      "registry_name": "docker",
      "name": "@gongrzhe/server-redis-mcp",
      "version": "1.0.0",
      "package_arguments": [
        {
          "description": "Docker image to run",
          "is_required": true,
          "format": "string",
          "value": "mcp/redis",
          "default": "mcp/redis",
          "type": "positional",
          "value_hint": "mcp/redis"
        },
        {
          "description": "Redis server connection string",
          "is_required": true,
          "format": "string",
          "value": "redis://host.docker.internal:6379",
          "default": "redis://host.docker.internal:6379",
          "type": "positional",
          "value_hint": "host.docker.internal:6379"
        }
      ]
    }
  ]
}
```

#### Publish a Server Entry

```
POST /v0/publish
```

Publishes a new MCP server entry to the registry. Authentication is required via Bearer token in the Authorization header.

Headers:
- `Authorization`: Bearer token for authentication (e.g., `Bearer your_token_here`)
- `Content-Type`: application/json

Request body example:
```json
{
    "description": "<your description here>",
    "name": "io.github.<owner>/<server-name>",
    "packages": [
        {
            "registry_name": "npm",
            "name": "@<owner>/<server-name>",
            "version": "0.2.23",
            "package_arguments": [
                {
                    "description": "Specify services and permissions.",
                    "is_required": true,
                    "format": "string",
                    "value": "-s",
                    "default": "-s",
                    "type": "positional",
                    "value_hint": "-s"
                }
            ],
            "environment_variables": [
                {
                    "description": "API Key to access the server",
                    "name": "API_KEY"
                }
            ]
        },{
            "registry_name": "docker",
            "name": "@<owner>/<server-name>-cli",
            "version": "0.123.223",
            "runtime_hint": "docker",
            "runtime_arguments": [
                {
                    "description": "Specify services and permissions.",
                    "is_required": true,
                    "format": "string",
                    "value": "--mount",
                    "default": "--mount",
                    "type": "positional",
                    "value_hint": "--mount"
                }
            ],
            "environment_variables": [
                {
                    "description": "API Key to access the server",
                    "name": "API_KEY"
                }
            ]
        }
    ],
    "repository": {
        "url": "https://github.com/<owner>/<server-name>",
        "source": "github"
    },
    "version_detail": {
        "version": "0.0.1-<publisher_version>"
    }
}
```

Response example:
```json
{
  "message": "Server publication successful",
  "id": "1234567890abcdef12345678"
}
```

#### Export Seed Data

```
GET /v0/seed.json
```

Exports all servers in the registry in seed format for import by other registry instances.

Response example:
```json
[
  {
    "id": "01129bff-3d65-4e3d-8e82-6f2f269f818c",
    "name": "io.github.gongrzhe/redis-mcp-server",
    "description": "A Redis MCP server implementation for interacting with Redis databases.",
    "repository": {
      "url": "https://github.com/GongRzhe/REDIS-MCP-Server",
      "source": "github",
      "id": "907849235"
    },
    "version_detail": {
      "version": "0.0.1-seed",
      "release_date": "2025-05-16T19:13:21Z",
      "is_latest": true
    },
    "packages": [
      {
        "registry_name": "docker",
        "name": "@gongrzhe/server-redis-mcp",
        "version": "1.0.0",
        "package_arguments": [
          {
            "description": "Docker image to run",
            "is_required": true,
            "format": "string",
            "value": "mcp/redis",
            "default": "mcp/redis",
            "type": "positional",
            "value_hint": "mcp/redis"
          }
        ]
      }
    ]
  }
]
```

### Ping Endpoint

```
GET /v0/ping
```

Simple ping endpoint that returns environment configuration information:
```json
{
  "environment": "dev",
  "version": "registry-<sha>"
}
```

## Configuration

The service can be configured using environment variables:

| Variable | Description | Default |
|----------|-------------|---------|
| `MCP_REGISTRY_APP_VERSION`           | Application version | `dev` |
| `MCP_REGISTRY_DATABASE_TYPE`         | Database type | `mongodb` |
| `MCP_REGISTRY_COLLECTION_NAME`       | MongoDB collection name | `servers_v2` |
| `MCP_REGISTRY_DATABASE_NAME`         | MongoDB database name | `mcp-registry` |
| `MCP_REGISTRY_DATABASE_URL`          | MongoDB connection string | `mongodb://localhost:27017` |
| `MCP_REGISTRY_GITHUB_CLIENT_ID`      | GitHub App Client ID |  |
| `MCP_REGISTRY_GITHUB_CLIENT_SECRET`  | GitHub App Client Secret |  |
| `MCP_REGISTRY_LOG_LEVEL`             | Log level | `info` |
| `MCP_REGISTRY_SEED_FILE_PATH`        | Path or URL to import seed file (supports local files and HTTP URLs) | `data/seed.json` |
| `MCP_REGISTRY_SERVER_ADDRESS`        | Listen address for the server | `:8080` |

### Command Line Flags

Command line flags take precedence over environment variables:

| Flag | Description | Environment Variable |
|------|-------------|---------------------|
| `--seed-file-path` | Path or URL to import seed file | `MCP_REGISTRY_SEED_FILE_PATH` |

**Note**: The `MCP_REGISTRY_SEED_IMPORT` environment variable has been removed. Seed data is now imported automatically when `MCP_REGISTRY_SEED_FILE_PATH` is set.

## Registry Composability

The MCP Registry supports composability through seed data export/import functionality, enabling registry instances to share data with each other via HTTP endpoints.

### Use Cases

1. **Distributed Registries**: Set up multiple registry instances that share a common dataset
2. **Registry Synchronization**: Import data from a central registry to local instances
3. **Development Environments**: Import production data to development instances
4. **Registry Migration**: Move data between different registry deployments

### Export Seed Data

Any registry instance can export its complete dataset:

```bash
# Export all servers in seed format
curl http://localhost:8080/v0/seed.json > exported_data.json
```

### Import Seed Data

Registry instances can import data from:

**Local files:**
```bash
# Via environment variable
MCP_REGISTRY_SEED_FILE_PATH=data/seed_2025_05_16.json ./registry

# Via command line flag
./registry --seed-file-path=data/seed_2025_05_16.json
```

**HTTP endpoints:**
```bash
# Via environment variable
MCP_REGISTRY_SEED_FILE_PATH=http://localhost:8080/v0/seed.json ./registry

# Via command line flag  
./registry --seed-file-path=http://other-registry:8080/v0/seed.json
```

### Composability Workflow Example

```bash
# Step 1: Start primary registry with initial seed data
MCP_REGISTRY_SEED_FILE_PATH=data/seed_2025_05_16.json ./registry

# Step 2: Start secondary registry importing from primary
./registry --seed-file-path=http://localhost:8080/v0/seed.json --server-address=:8081

# Step 3: Verify both registries have the same data
curl http://localhost:8080/v0/servers | jq '.servers | length'
curl http://localhost:8081/v0/servers | jq '.servers | length'
```

This enables true composability where registry instances can be distributed and synchronized, with new instances importing data from existing ones via HTTP.


## Testing

Run the test script to validate API endpoints:

```bash
./scripts/test_endpoints.sh
```

You can specify specific endpoints to test:

```bash
./scripts/test_endpoints.sh --endpoint health
./scripts/test_endpoints.sh --endpoint servers
```

## License

See the [LICENSE](LICENSE) file for details.

## Contributing

See the [CONTRIBUTING](CONTRIBUTING.md) file for details.
