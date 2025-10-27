### Day 1 : Build a Microservice for the Demo

**Goal**

Create a simple FastAPI microservice (with observability) as the foundation for our demo.
It will simulate real business logic and expose endpoints for health checks and observability.

#### Part 1 – Setup & Tool Installation

Endpoints Overview
	•	/checkout → simulates real business logic
	•	/healthz → readiness/liveness probe


**Install Required Tools**

```
brew install minikube
brew install python3 pipx
pipx install fastapi uvicorn
brew install --cask lens
```

**Local Cluster Setup**

`minikube start --cpus=4 --memory=8192`

We now have a running local cluster you can deploy to.


**Build a FastAPI application**

Project structure

```
fastapi-demo/
├── app/
│   ├── main.py
│   ├── __init__.py
│   └── otel_config.py
├── requirements.txt
├── Dockerfile
```

**app/main.py**

```
from fastapi import FastAPI
import time, random
from app.otel_config import init_tracer
from opentelemetry import trace
from prometheus_fastapi_instrumentator import Instrumentator

# Initialize tracing
init_tracer()
tracer = trace.get_tracer(__name__)

# Create app
app = FastAPI()

# ✅ Initialize Prometheus metrics BEFORE startup
instrumentator = Instrumentator().instrument(app).expose(app)

@app.get("/checkout")
async def checkout():
    with tracer.start_as_current_span("checkout-operation"):
        time.sleep(random.uniform(0.2, 0.8))
        return {"status": "success"}

@app.get("/healthz")
def health():
    return {"status": "healthy"}
```

**app/otel_config.py**

```
import os
from opentelemetry import trace
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter

def init_tracer():
    # Allow override via env for flexibility during local tests
    endpoint = os.getenv("OTEL_EXPORTER_OTLP_ENDPOINT", "http://localhost:4318/v1/traces")
    service_name = os.getenv("OTEL_SERVICE_NAME", "fastapi-demo")

    provider = TracerProvider(resource=Resource(attributes={"service.name": service_name}))
    trace.set_tracer_provider(provider)
    exporter = OTLPSpanExporter(endpoint=endpoint)
    provider.add_span_processor(BatchSpanProcessor(exporter))
```

**requirements.txt**

```
fastapi
uvicorn
opentelemetry-api
opentelemetry-sdk
opentelemetry-exporter-otlp
prometheus-fastapi-instrumentator
```

**Dockerfile**

```
FROM python:3.10-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8080"]
```



#### Part 2 – Add Built-In Observability

We don’t just want an app that runs, we want one that explains itself while it runs.

Step 1: Add Tracing (OpenTelemetry)
	•	Integrate OpenTelemetry SDK to export traces for each request.
	•	Traces should appear in Jaeger with span name, duration, and trace ID.

Step 2: Add Metrics (Prometheus)
	•	Use prometheus-fastapi-instrumentator to export metrics at /metrics.
	•	This provides real-time counts, latencies, and error rates for Prometheus scraping.

#### Part 3 – Run & Test Locally

Step 1: Run the App

```
# from fastapi-demo/
python3 -m venv .venv
source ./.venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8080
```

Step 2: Test Endpoints

`curl -s http://localhost:8080/healthz`

`curl -s http://localhost:8080/checkout`

You should see JSON responses.

```
{"status":"healthy"}%
{"status":"success"}%
```


#### Part 4 – Enable Traces (Jaeger)

Step 1: Run Jaeger All-in-One


```
docker run -p 16686:16686 -p 4318:4318 --name jaeger \
  jaegertracing/all-in-one:1.57
```

Step 2: Send Requests to Generate Traces

`for i in {1..5}; do curl -s http://localhost:8080/checkout > /dev/null; done`

Step 3: View Traces

Open the Jaeger UI → http://localhost:16686
Find service: fastapi-demo → view spans and trace timelines.

#### Part 5 – Enable Prometheus Metrics

Step 1: Install Dependencies

`pip install prometheus-fastapi-instrumentator`

Add to requirements.txt:

```
fastapi
uvicorn
opentelemetry-api
opentelemetry-sdk
opentelemetry-exporter-otlp
prometheus-fastapi-instrumentator
```

Step 2: Verify Endpoints

```
curl -s http://localhost:8080/healthz
curl -s http://localhost:8080/checkout
```

Open http://localhost:8080/metrics

Example output:

```
# HELP http_request_duration_seconds Request duration
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_count{handler="/checkout",method="GET",status="200"} 5.0
http_request_duration_seconds_sum{handler="/checkout",method="GET",status="200"} 2.61
```


#### Part 6: Visualize metrics with Prometheus

If Prometheus is installed via Homebrew:

Step 1: Create prometheus.yml

```
scrape_configs:
  - job_name: 'fastapi-demo'
    static_configs:
      - targets: ['localhost:8080']
```

Step 2: Start Prometheus

`prometheus --config.file=prometheus.yml`

Step 3: Open Prometheus UI


Visit http://localhost:9090
Query: http_request_duration_seconds_count

✅ You’re now seeing live FastAPI metrics from your app.

Remember:
	•	Metrics show the what → e.g., 500 req/sec, 2% errors
	•	Traces show the why → e.g., checkout calls payment 3× due to retries

Together, they form the core of observability.
	

#### Part 7 – Validate Your Dockerfile

Step 1: Build and Run


```
docker build -t fastapi-demo:local .
docker run -p 8080:8080 --rm fastapi-demo:local
curl -s http://localhost:8080/checkout
```

Step 2: Export Traces from Container


```
# Ensure Jaeger from step D is running
docker run -e OTEL_EXPORTER_OTLP_ENDPOINT=http://host.docker.internal:4318/v1/traces \
  -e OTEL_SERVICE_NAME=fastapi-demo \
  -p 8080:8080 --rm fastapi-demo:local
```


#### Part 8 – Enable Traces Inside Docker (Networking Fix)

If traces fail to export, create a shared network.

Step 1: Create Network

`docker network create observability`

Step 2: Run Jaeger on That Network

```
docker run -d --name jaeger \
  --network observability \
  -e COLLECTOR_OTLP_ENABLED=true \
  -p 16686:16686 -p 4318:4318 \
  jaegertracing/all-in-one:1.57
```

Step 3: Run Your App on Same Network

```
docker run --rm -p 8080:8080 \
  --network observability \
  -e OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger:4318/v1/traces \
  -e OTEL_SERVICE_NAME=fastapi-demo \
  fastapi-demo:local
```

Step 4: Test

`for i in {1..5}; do curl -s http://localhost:8080/checkout > /dev/null; done`

✅ Traces now appear in http://localhost:16686


Optional – One-Command Startup with Docker Compose

Create a `docker-compose.yml`

Create a docker-compose.yml:

```
version: "3.8"
services:
  jaeger:
    image: jaegertracing/all-in-one:1.57
    ports:
      - "16686:16686"
      - "4318:4318"
    environment:
      - COLLECTOR_OTLP_ENABLED=true

  fastapi-demo:
    build: .
    ports:
      - "8080:8080"
    environment:
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger:4318/v1/traces
      - OTEL_SERVICE_NAME=fastapi-demo
    depends_on:
      - jaeger
```


Then run: `docker compose up --build`

✅ Both the app and Jaeger start and connect automatically.


#### Day 1 Outcome

By the end of Day 1, you have:

	•	✅ A FastAPI service running locally
	•	✅ /checkout and /healthz endpoints working
	•	✅ Traces visible in Jaeger
	•	✅ Metrics exposed at /metrics
	•	✅ A validated Docker build
	•	✅ Ready foundation for automation (Day 2)

#### Next Step

In day2, you’ll containerize this app fully and deploy it to a local Kind Kubernetes cluster with observability enabled.





