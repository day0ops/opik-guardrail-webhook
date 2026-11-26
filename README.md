# Opik Guardrail Webhook

[![Release](https://github.com/day0ops/opik-guardrail-webhook/actions/workflows/release.yml/badge.svg)](https://github.com/day0ops/opik-guardrail-webhook/actions/workflows/release.yml)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

FastAPI webhook for [agentgateway](https://agentgateway.dev)'s `promptGuard` policy: inspects and filters LLM request/response traffic, with [Opik](https://www.comet.com/docs/opik/) wired in for guardrail metrics (Sentiment, Tone) and trace observability.

## What it does

- `POST /request`: scans a prompt before it reaches the model.
- `POST /response`: scans the model's reply before it reaches the caller.

### Pipeline

For `/request`, each message is checked in order, stopping at the first match:

1. Toxic phrase match → reject
2. Banned word match → reject
3. Opik `Sentiment` (VADER) compound score below `SENTIMENT_THRESHOLD` → reject
4. Opik `Tone` score below `0.5` → reject
5. PII pattern match → mask that message's content (does not stop the loop)

For `/response`, each choice is checked and never rejected, only masked:

1. PII pattern match → mask that choice's content
2. Sentiment score below `SENTIMENT_THRESHOLD` → replace the content with a safety-policy notice

> [!IMPORTANT]
> A pass/mask/reject decision always answers HTTP 200, with the decision in the `action` JSON body (`{"action": {"body"?, "status_code"?, "reason"}}`). A malformed request body answers 422. Sentiment/Tone scoring failures fail open (they fall back to a neutral score rather than erroring), so an unhandled 500 would only come from a bug in the guardrail-check code itself, not from Opik. Opik writes are queued and sent on a background thread by the SDK, so a failed or slow Opik call never blocks or fails the request, it just risks that trace being lost.

Every call also emits an Opik SDK trace and a local OpenTelemetry span fanned out to `LOCAL_OTEL_ENDPOINT`, correlated to each other via agentgateway's `traceparent` header. Both only run when `OPIK_API_KEY` is set: unsetting it skips the local OTEL span too, not just the Opik trace (both code paths return before either is created).

## Configuration

| Env var | Default | Description |
| --- | --- | --- |
| `OPIK_API_KEY` | unset | Opik API key. Unset disables both Opik tracing and the local OTEL fan-out. |
| `OPIK_PROJECT_NAME` | `agentgateway-guardrails` | Opik project traces are recorded under. |
| `SENTIMENT_THRESHOLD` | `-0.5` | VADER compound score below which content is rejected (request) or masked (response). |
| `BANNED_WORDS` | `violence,drugs,weapons,terrorism,exploit,abuse` | Comma-separated banned-word list. |
| `LOCAL_OTEL_ENDPOINT` | `http://fan-out-collector.telemetry:4318` | OTLP/HTTP endpoint for the local trace fan-out. Set to an empty string to disable it. |

> [!NOTE]
> The Opik SDK also reads `OPIK_WORKSPACE`, `OPIK_URL_OVERRIDE`, and `~/.opik.config` directly. This service doesn't set any of them explicitly, so they fall back to the SDK's own defaults (cloud Opik, default workspace) unless present in the environment.

## Wiring it into agentgateway

Point an `EnterpriseAgentgatewayPolicy`'s `promptGuard` webhook at this service (abbreviated; check your agentgateway/Solo.io docs for the full policy spec, e.g. `targetRefs`):

```yaml
spec:
  backend:
    ai:
      promptGuard:
        request:
          - webhook: { backendRef: { name: opik-guardrail-webhook, port: 8000 } }
        response:
          - webhook: { backendRef: { name: opik-guardrail-webhook, port: 8000 } }
```

## Usage

```bash
# Build image locally
make build IMAGE_TAG=latest

# Push to registry
make push IMAGE_REPO=<your-registry> IMAGE_TAG=<tag>

# Deploy to Kubernetes
make deploy IMAGE_REPO=<your-registry> IMAGE_TAG=<tag>

# Tail logs
make logs
```

## Requirements

- Kubernetes cluster with agentgateway installed (`agentgateway-system` namespace)
- `kubectl` configured for the target cluster

## Development

```bash
pip install -r server/requirements.txt
python -c "import sys; sys.path.insert(0, 'server'); import main"  # import sanity check, there's no test suite yet
docker build -t opik-guardrail-webhook server/
```
