# AI Image Analysis API

A scalable, production-ready API service for asynchronous image analysis powered by FastAPI, Redis, and Redis Queue (RQ).

## Overview

This project demonstrates a high-performance, distributed architecture for processing image analysis requests asynchronously. It features:

- **FastAPI Backend**: Asynchronous REST API with automatic OpenAPI documentation
- **Async Processing**: Redis Queue (RQ) for background job processing
- **Distributed Caching**: Redis-based caching layer for frequently analyzed images
- **Rate Limiting**: Sliding window algorithm with per-IP request throttling
- **Docker Containerization**: Multi-service orchestration with Docker Compose
- **Comprehensive Testing**: Unit and integration tests for all core functionality
- **Production-Ready**: Health checks, error handling, structured logging, and validation

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    FastAPI Server                        │
│         (Async Request Handling, Rate Limiting)          │
└────────────┬─────────────────────────────────────────────┘
             │
             ├──────────────────┬──────────────────┐
             │                  │                  │
             ▼                  ▼                  ▼
        ┌────────────┐    ┌────────────┐    ┌────────────┐
        │   Redis    │    │   Redis    │    │ RQ Worker  │
        │  (Caching) │    │  (Queues)  │    │ (Processing)
        └────────────┘    └────────────┘    └────────────┘
```

### Services

1. **FastAPI Service**: Handles HTTP requests, enqueues jobs, serves cached results
2. **Redis**: Manages job queues, caching, and rate limiting state
3. **RQ Worker**: Processes image analysis tasks asynchronously

## Project Structure

```
project-root/
├── app/
│   ├── __init__.py
│   ├── main.py                 # FastAPI application and routes
│   ├── models.py               # Pydantic models for validation
│   ├── services.py             # Business logic and model loading
│   ├── tasks.py                # RQ background task definitions
│   ├── dependencies.py         # Rate limiting, caching dependencies
│   └── core/
│       ├── config.py           # Configuration from environment
│       ├── security.py         # Optional auth utilities
│       └── utils.py            # Helper functions and logging
├── tests/
│   ├── test_api.py             # Integration tests
│   └── test_units.py           # Unit tests for core logic
├── Dockerfile                  # Multi-stage Docker build
├── docker-compose.yml          # Service orchestration
├── requirements.txt            # Python dependencies
├── .env.example                # Example environment variables
└── README.md                   # This file
```

## Getting Started

### Prerequisites

- Python 3.9+
- Docker and Docker Compose
- Redis (handled by Docker Compose)

### Local Development Setup

1. **Clone the repository**
   ```bash
   git clone https://github.com/Kishorekalingi/ai-image-analysis-api.git
   cd ai-image-analysis-api
   ```

2. **Create virtual environment**
   ```bash
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   ```

3. **Install dependencies**
   ```bash
   pip install -r requirements.txt
   ```

4. **Set up environment variables**
   ```bash
   cp .env.example .env
   ```

5. **Run with Docker Compose**
   ```bash
   docker-compose up --build
   ```

   The API will be available at `http://localhost:8000`

## API Endpoints

### 1. Submit Image Analysis Job

**Endpoint:** `POST /analyze-image`

**Request Body:**
```json
{
  "image_data": "base64_encoded_string_or_null",
  "image_url": "https://example.com/image.jpg"
}
```

Note: Provide either `image_data` or `image_url`, not both.

**Response (202 Accepted):**
```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "PENDING",
  "message": "Job queued for processing"
}
```

**Error Responses:**
- `400 Bad Request`: Invalid image format or missing data
- `429 Too Many Requests`: Rate limit exceeded (includes `Retry-After` header)

### 2. Get Analysis Status

**Endpoint:** `GET /analysis-status/{job_id}`

**Path Parameters:**
- `job_id` (string, UUID format): The job ID from submission

**Response (200 OK):**
```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "COMPLETED",
  "result": {
    "label": "cat",
    "confidence": 0.98
  },
  "cached": false,
  "processing_time_ms": 1234
}
```

**Status Values:**
- `PENDING`: Job is queued
- `PROCESSING`: Job is being processed
- `COMPLETED`: Analysis finished successfully
- `FAILED`: An error occurred during processing

**Error Responses:**
- `404 Not Found`: Job ID not found

### 3. Health Check

**Endpoint:** `GET /health`

**Response (200 OK):**
```json
{
  "status": "healthy",
  "timestamp": "2024-01-21T14:30:00Z"
}
```

## Configuration

Environment variables (see `.env.example`):

```bash
# Redis Configuration
REDIS_HOST=redis
REDIS_PORT=6379

# Rate Limiting
RATE_LIMIT_MAX_REQUESTS=5
RATE_LIMIT_INTERVAL_SECONDS=60

# Caching
CACHE_TTL_SECONDS=3600

# API
API_TITLE="AI Image Analysis API"
API_VERSION="1.0.0"
```

## Rate Limiting

The API implements distributed rate limiting using Redis with a sliding window algorithm:

- **Default Limit**: 5 requests per minute per IP address
- **Behavior**: Requests exceeding the limit receive a 429 response with a `Retry-After` header
- **Key Format**: `rate_limit:{ip_address}` in Redis

## Caching Strategy

The API uses a cache-aside pattern:

1. On receiving an image, compute its SHA-256 hash
2. Check if the hash exists in Redis cache
3. If cached, return the cached result immediately
4. If not cached, enqueue a background job
5. Upon completion, cache the result for future requests

**Cache Key Format:** `analysis_result:{image_hash}`

**Default TTL:** 1 hour

## Running Tests

```bash
# Run all tests
pytest

# Run with coverage
pytest --cov=app tests/

# Run specific test file
pytest tests/test_api.py

# Run specific test
pytest tests/test_units.py::test_rate_limiting
```

## Performance Metrics

- **Synchronous Operations**: <50ms (rate limiting, cache lookup, job enqueueing)
- **Asynchronous Processing**: ~2-5 seconds (model inference)
- **Concurrency**: Handles hundreds of concurrent requests

## Docker Deployment

### Building Images

```bash
docker-compose build
```

### Running Services

```bash
docker-compose up -d
```

### Viewing Logs

```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f api
```

### Stopping Services

```bash
docker-compose down
```

## Health Checks

Each service includes health checks:

- **FastAPI**: HTTP health endpoint
- **Redis**: Redis PING command
- **RQ Worker**: Monitors queue connectivity

View health status:
```bash
docker-compose ps
```

## Troubleshooting

### Port Already in Use

If port 8000 or 6379 is in use:

```bash
docker-compose -f docker-compose.yml up -d -e API_PORT=8001
```

### Redis Connection Errors

Ensure Redis is running:

```bash
docker-compose logs redis
```

### Worker Not Processing Jobs

Check worker logs:

```bash
docker-compose logs worker
```

## API Documentation

Swagger UI documentation is automatically generated and available at:

```
http://localhost:8000/docs
```

ReDoc documentation is available at:

```
http://localhost:8000/redoc
```

## Implementation Highlights

### 1. Asynchronous Processing

- FastAPI uses `async`/`await` for non-blocking I/O
- RQ enables background job processing without blocking the API
- Redis acts as the message broker

### 2. Error Handling

- Pydantic validation for request payloads
- Try-catch blocks around model inference
- Structured error responses with meaningful messages
- Graceful degradation on Redis failures

### 3. Logging

- Structured logging with context information
- Request/response logging
- Job processing event logging
- Error and exception logging

### 4. Security

- Input validation for all endpoints
- File size and format validation
- Rate limiting to prevent abuse
- CORS headers (can be configured)

## Performance Optimization

1. **Model Loading**: Pre-load the model once in worker initialization
2. **Image Processing**: Efficient resizing and preprocessing
3. **Caching**: Avoid redundant inference for identical images
4. **Async I/O**: Non-blocking operations for network and file access
5. **Worker Scaling**: Deploy multiple RQ worker instances for parallelism

## Testing Coverage

- **Unit Tests**: Rate limiting logic, cache operations, utility functions
- **Integration Tests**: API endpoints, job processing, error handling
- **Edge Cases**: Invalid inputs, timeout scenarios, concurrent requests

## Future Enhancements

1. Support for multiple ML models
2. Batch image analysis endpoint
3. WebSocket support for real-time status updates
4. Metrics collection (Prometheus)
5. Distributed tracing (Jaeger)
6. API authentication and authorization

## Contributing

Contributions are welcome! Please follow these steps:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Add tests
5. Submit a pull request

## License

MIT License - See LICENSE file for details

## Support

For issues and questions, please open an issue on GitHub.
