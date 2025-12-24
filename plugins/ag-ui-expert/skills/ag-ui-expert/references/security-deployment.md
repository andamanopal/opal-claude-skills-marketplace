# Security & Deployment

## Security Architecture

### BFF (Backend-for-Frontend) Pattern

Never expose agent runtime directly to clients:

```
┌──────────┐      ┌──────────────────┐      ┌─────────────┐
│  Client  │─────▶│   BFF (FastAPI)  │─────▶│ Agent       │
│          │◀─────│   - Auth         │◀─────│ Runtime     │
│          │  SSE │   - Rate limit   │  SSE │             │
└──────────┘      │   - Validation   │      └─────────────┘
                  │   - Audit log    │
                  └──────────────────┘
```

### Input Validation

```python
from pydantic import validator
from ag_ui.core import RunAgentInput

class SecureRunAgentInput(RunAgentInput):
    """Validated input with security checks."""
    
    @validator('messages')
    def validate_messages(cls, messages):
        for msg in messages:
            # Size limit
            if len(msg.content) > 100_000:  # 100KB
                raise ValueError("Message exceeds size limit")
            
            # Prompt injection patterns
            injection_patterns = [
                "ignore previous instructions",
                "system:",
                "<|im_start|>",
                "ADMIN:",
                "override:",
            ]
            content_lower = msg.content.lower()
            for pattern in injection_patterns:
                if pattern.lower() in content_lower:
                    raise ValueError(f"Invalid content pattern detected")
        
        return messages
    
    @validator('thread_id', 'run_id')
    def validate_ids(cls, v):
        # Ensure IDs are safe strings
        if not v.replace('-', '').replace('_', '').isalnum():
            raise ValueError("Invalid ID format")
        if len(v) > 128:
            raise ValueError("ID too long")
        return v
```

### Authentication Middleware

```python
from fastapi import Request, HTTPException
from typing import Callable, AsyncGenerator
import jwt

class AuthMiddleware:
    """Validate JWT tokens before processing."""
    
    def __init__(self, secret_key: str, algorithm: str = "HS256"):
        self.secret_key = secret_key
        self.algorithm = algorithm
    
    async def __call__(
        self,
        input_data: RunAgentInput,
        next_handler: Callable
    ) -> AsyncGenerator[BaseEvent, None]:
        # Extract token from forwarded_props
        token = input_data.forwarded_props.get("auth_token")
        
        if not token:
            yield RunErrorEvent(
                type=EventType.RUN_ERROR,
                message="Authentication required",
                code="AUTH_ERROR"
            )
            return
        
        try:
            payload = jwt.decode(token, self.secret_key, algorithms=[self.algorithm])
            input_data.forwarded_props["user_id"] = payload.get("sub")
            input_data.forwarded_props["user_roles"] = payload.get("roles", [])
        except jwt.InvalidTokenError:
            yield RunErrorEvent(
                type=EventType.RUN_ERROR,
                message="Invalid authentication token",
                code="AUTH_ERROR"
            )
            return
        
        # Continue to next handler
        async for event in next_handler(input_data):
            yield event
```

### Rate Limiting

```python
from slowapi import Limiter
from slowapi.util import get_remote_address
from collections import defaultdict
import time

# IP-based limiter
limiter = Limiter(key_func=get_remote_address)

# User-based limiter
class UserRateLimiter:
    def __init__(self, max_requests: int = 30, window_seconds: int = 60):
        self.max_requests = max_requests
        self.window = window_seconds
        self.requests: dict[str, list[float]] = defaultdict(list)
    
    def check(self, user_id: str) -> bool:
        now = time.time()
        # Clean old requests
        self.requests[user_id] = [
            t for t in self.requests[user_id]
            if now - t < self.window
        ]
        # Check limit
        if len(self.requests[user_id]) >= self.max_requests:
            return False
        self.requests[user_id].append(now)
        return True

user_limiter = UserRateLimiter(max_requests=30, window_seconds=60)

@app.post("/jarvis")
@limiter.limit("10/minute")  # IP limit
async def jarvis_endpoint(request: Request, input_data: SecureRunAgentInput):
    user_id = input_data.forwarded_props.get("user_id", "anonymous")
    
    if not user_limiter.check(user_id):
        return JSONResponse(
            status_code=429,
            content={"error": "Rate limit exceeded", "code": "RATE_LIMITED"}
        )
    
    # Process request...
```

### Tool Authorization

```python
class ToolAuthorizationMiddleware:
    """Filter tools based on user permissions."""
    
    ROLE_PERMISSIONS = {
        "admin": ["*"],  # All tools
        "user": ["search_lifelogs", "summarize", "export"],
        "viewer": ["search_lifelogs"],
    }
    
    async def __call__(
        self,
        input_data: RunAgentInput,
        next_handler: Callable
    ) -> AsyncGenerator[BaseEvent, None]:
        user_roles = input_data.forwarded_props.get("user_roles", [])
        
        # Get allowed tools
        allowed_tools = set()
        for role in user_roles:
            perms = self.ROLE_PERMISSIONS.get(role, [])
            if "*" in perms:
                allowed_tools = None  # All allowed
                break
            allowed_tools.update(perms)
        
        # Filter tools if not admin
        if allowed_tools is not None:
            input_data.tools = [
                t for t in input_data.tools
                if t.name in allowed_tools
            ]
        
        async for event in next_handler(input_data):
            yield event
```

### Audit Logging

```python
import logging
import json
from datetime import datetime

audit_logger = logging.getLogger("jarvis.audit")

class AuditMiddleware:
    """Log all agent interactions for compliance."""
    
    async def __call__(
        self,
        input_data: RunAgentInput,
        next_handler: Callable
    ) -> AsyncGenerator[BaseEvent, None]:
        start_time = datetime.utcnow()
        user_id = input_data.forwarded_props.get("user_id", "anonymous")
        
        # Log request
        audit_logger.info(json.dumps({
            "event": "agent_request",
            "thread_id": input_data.thread_id,
            "run_id": input_data.run_id,
            "user_id": user_id,
            "timestamp": start_time.isoformat(),
            "message_count": len(input_data.messages),
            "tool_count": len(input_data.tools)
        }))
        
        tool_calls = []
        error = None
        
        async for event in next_handler(input_data):
            # Track tool calls
            if event.type == EventType.TOOL_CALL_START:
                tool_calls.append(event.tool_call_name)
            elif event.type == EventType.RUN_ERROR:
                error = event.message
            
            yield event
        
        # Log completion
        end_time = datetime.utcnow()
        audit_logger.info(json.dumps({
            "event": "agent_complete",
            "thread_id": input_data.thread_id,
            "run_id": input_data.run_id,
            "user_id": user_id,
            "timestamp": end_time.isoformat(),
            "duration_ms": (end_time - start_time).total_seconds() * 1000,
            "tool_calls": tool_calls,
            "error": error
        }))
```

## Production Deployment

### Nginx SSE Configuration

```nginx
upstream jarvis_backend {
    server 127.0.0.1:8000;
    keepalive 32;
}

server {
    listen 443 ssl http2;
    server_name jarvis.example.com;
    
    ssl_certificate /etc/ssl/certs/jarvis.crt;
    ssl_certificate_key /etc/ssl/private/jarvis.key;
    
    # AG-UI endpoint with SSE settings
    location /ai-api/jarvis/ {
        proxy_pass http://jarvis_backend/;
        
        # Critical SSE settings
        proxy_buffering off;
        proxy_cache off;
        proxy_http_version 1.1;
        proxy_set_header Connection '';
        chunked_transfer_encoding off;
        
        # Extended timeout for long streams
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
        
        # Headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # CORS
        add_header Access-Control-Allow-Origin $http_origin;
        add_header Access-Control-Allow-Credentials true;
        add_header Access-Control-Allow-Methods "GET, POST, OPTIONS";
        add_header Access-Control-Allow-Headers "Authorization, Content-Type, Accept";
    }
    
    # Health check
    location /health {
        proxy_pass http://jarvis_backend/health;
    }
}
```

### Docker Configuration

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install dependencies
COPY pyproject.toml poetry.lock ./
RUN pip install poetry && \
    poetry config virtualenvs.create false && \
    poetry install --no-dev --no-interaction

# Copy application
COPY . .

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Run with uvicorn
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

### Docker Compose

```yaml
version: "3.8"

services:
  jarvis:
    build: .
    ports:
      - "8000:8000"
    environment:
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - JWT_SECRET=${JWT_SECRET}
      - LOG_LEVEL=info
    volumes:
      - ./logs:/app/logs
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  nginx:
    image: nginx:alpine
    ports:
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/ssl:ro
    depends_on:
      - jarvis
    restart: unless-stopped
```

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jarvis-agent
spec:
  replicas: 3
  selector:
    matchLabels:
      app: jarvis-agent
  template:
    metadata:
      labels:
        app: jarvis-agent
    spec:
      containers:
      - name: jarvis
        image: jarvis-agent:latest
        ports:
        - containerPort: 8000
        env:
        - name: ANTHROPIC_API_KEY
          valueFrom:
            secretKeyRef:
              name: jarvis-secrets
              key: anthropic-api-key
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 10
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
---
apiVersion: v1
kind: Service
metadata:
  name: jarvis-agent
spec:
  selector:
    app: jarvis-agent
  ports:
  - port: 80
    targetPort: 8000
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jarvis-ingress
  annotations:
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/proxy-buffering: "off"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
spec:
  rules:
  - host: jarvis.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: jarvis-agent
            port:
              number: 80
```

### Graceful Shutdown

```python
import asyncio
from contextlib import asynccontextmanager
from fastapi import FastAPI

active_connections: set[str] = set()

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    yield
    # Shutdown - wait for active connections
    if active_connections:
        print(f"Waiting for {len(active_connections)} active connections...")
        for _ in range(30):  # 30 second timeout
            if not active_connections:
                break
            await asyncio.sleep(1)
        
        # Cancel remaining
        for conn_id in list(active_connections):
            print(f"Cancelling connection: {conn_id}")
            active_connections.discard(conn_id)

app = FastAPI(lifespan=lifespan)

@app.post("/jarvis")
async def jarvis_endpoint(input_data: RunAgentInput):
    conn_id = input_data.run_id
    active_connections.add(conn_id)
    
    try:
        async def stream():
            # ... streaming logic
            pass
        return StreamingResponse(stream(), media_type="text/event-stream")
    finally:
        active_connections.discard(conn_id)
```

### OpenTelemetry Observability

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

# Setup
trace.set_tracer_provider(TracerProvider())
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter(endpoint="http://jaeger:4317"))
)
tracer = trace.get_tracer("jarvis-agent")

@app.post("/jarvis")
async def jarvis_endpoint(input_data: RunAgentInput):
    with tracer.start_as_current_span(
        "agent_run",
        attributes={
            "thread_id": input_data.thread_id,
            "run_id": input_data.run_id,
            "message_count": len(input_data.messages)
        }
    ) as span:
        async def stream():
            async for event in process_workflow(input_data):
                span.add_event(event.type.value)
                yield encoder.encode(event)
        
        return StreamingResponse(stream(), media_type="text/event-stream")
```

### Health Check Endpoint

```python
from fastapi import FastAPI
from datetime import datetime

app = FastAPI()
start_time = datetime.utcnow()

@app.get("/health")
async def health_check():
    return {
        "status": "healthy",
        "uptime_seconds": (datetime.utcnow() - start_time).total_seconds(),
        "version": "1.0.0",
        "active_connections": len(active_connections)
    }
```
