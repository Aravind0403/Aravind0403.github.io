---
layout: post
title: "Know Your Blast Radius Before You Deploy: Introducing Ripple"
date: 2026-07-12
---

![Introducing Ripple Banner](/assets/images/ripple_banner.png)

A developer changes an environment variable in a payment service deployment manifest. Ten minutes later, checkout fails in production.

The post-mortem reveals that a downstream notification service was resolving its checkout host dynamically using that exact environment variable. No static code analysis tool flagged the change. No compiler could catch it. The dependency was implicit, undocumented, and entirely tribal.

In distributed systems, the hardest question isn't writing the code; it’s answering: *“If I deploy this change, what breaks?”*

Traditionally, you had two choices: spend millions implementing and maintaining a heavy service mesh (like Istio) to trace traffic at runtime, or rely on outdated architecture diagrams. **Ripple** is the third way. It maps your microservices statically, resolves dynamic runtime bindings in sub-50ms, and hits an F1 score of 1.000 on standard benchmarks using local LLM inference when code path analysis hits a wall.

Zero network calls. Zero runtime agent overhead. Fully local blast-radius analysis.

---

## The Origin: Why Tribal Knowledge Fails at Scale

Over the last 7+ years building distributed infrastructure at Amazon and Microsoft, I learned one hard truth: architecture diagrams are always lying. The tribal knowledge of "what talks to what" is incredibly tough to acquire, and it only gets worse as you scale.

Building a metadata platform that orchestrated over 17,000 microservices at Microsoft was the ultimate testament to this. When a system is that massive, undocumented dependencies aren't just an annoyance—they are a systemic risk. I built Ripple because I was tired of finding out about hidden dependencies during 2:00 AM pager duty.

---

## The Architecture of Static Discovery

Ripple maps dependencies by tracing references across the entire codebase lifecycle—from raw source code to deployment manifests. To handle the concurrency of parsing thousands of files and running local LLM inferences without blocking, the pipeline is orchestrated via FastAPI and Celery, with Redis managing the task queues.

![Ripple Architecture Pipeline](/assets/images/ripple_architecture.png)

The pipeline runs in three core stages:

### 1. The Polyglot AST Extractor
Instead of regex-based pattern matching, Ripple uses Tree-sitter and native AST parsers to traverse the abstract syntax tree of Python, Go, Java, JavaScript/TypeScript, and C# (.NET). It extracts:
* Outbound HTTP call sites (e.g., `requests`, `axios`, `fetch`, `.NET HttpClient`).
* gRPC stubs, connection channels, and protocol buffers.
* Config lookups mapping to environment variables (e.g., `os.getenv("CHECKOUT_SERVICE_URL")`).

AST extraction runs at a blistering **~190 files/second** on Apple Silicon, converting code paths into structured endpoints or dynamic configuration bindings (`<dynamic:VAR_NAME>`).

### 2. The Cross-Layer Manifest Linker
A static code parser cannot resolve `<dynamic:VAR_NAME>`. That value only materializes inside a container runtime.

To bridge this gap, Ripple parses Kubernetes manifests, Helm charts (`values.yaml` + templates), and Docker Compose files. It extracts environment variable mappings, traces how values are injected into container definitions, and resolves hostnames deterministically. The linker completes this cross-layer resolution in under 50ms with 1.0 confidence, mapping the vast majority of dynamic microservice connections.

### 3. Local LLM Fallback (gemma3:4b)
For legacy codebases or complex dynamic setups where static tracing hits a wall, Ripple falls back to a local LLM (`gemma3:4b` running via Ollama). By feeding the LLM a highly structured, service-aware prompt containing the repository's discovered services, it resolves opaque bindings locally without ever leaking source code or metadata to external APIs.

Crucially, to prevent hallucinations, the LLM is strictly constrained via structured JSON decoding (Pydantic validation), forcing it to only output edges between already discovered services rather than inventing new ones.

---

## The Verification Benchmarks

We validated Ripple on standard reference microservice topologies, running completely locally on Apple Silicon with zero external API calls. The extracted graph data is stored in Neo4j for interactive traversal and PostgreSQL for relational querying.

| Repository | Primary Languages | Total Files | Linker-only F1 | Linker + LLM F1 | E2E Duration |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `karpathy/nanochat` | Python | 36 | 0.000 | **1.000** | 12.3s |
| `robusta-dev/robusta` | Python | 394 | 0.000 | **0.880** | 104.1s |
| `GoogleCloudPlatform/microservices-demo` | Go, Python | 13 | 0.966 | **1.000** | 3.0s |
| `open-telemetry/opentelemetry-demo` | Py, Go, TS, C# | 39 | 0.970 | **1.000** | 6.0s |

### Key Takeaways from the Data:
* **The Static Linker does the heavy lifting:** On enterprise demos (Google Microservices and OpenTelemetry), the deterministic manifest linker resolves over 96% of connections instantly. The LLM only needed to run once to close the remaining gaps.
* **LLM Fallback is the missing puzzle piece:** In single-repository setups like `nanochat`, the static linker fails completely (F1 = 0.000) because the project relies entirely on in-memory dynamic routing without externalized manifest definitions. The LLM fallback achieves a perfect F1 score by reasoning about the code-level connection patterns.
* **Scale Capability:** On large source codebases like `django/django` (2,886 files, 1,323 calls), the static extraction pipeline completes in 559 seconds, proving the tool scales to enterprise-level monorepos.

---

## Why No Service Mesh?

Service meshes are brilliant for traffic routing and telemetry, but using them for blast-radius analysis introduces critical blind spots:
1. **Runtime Only:** You only map relationships that are actively receiving traffic. Rare, high-severity code paths (e.g., billing reconciliation runs, disaster recovery endpoints) remain invisible until they fail.
2. **Late Feedback Loop:** Finding out a service depends on yours *after* you deploy it to staging or production is too late. Ripple brings this feedback loop directly into your pre-commit hooks or PR checks.
3. **Infrastructure Overhead:** Running sidecars across thousands of pods consumes significant CPU and memory resources. Ripple runs entirely in CLI/CI pipelines, leaving your runtime footprint untouched.

---

## Get Started

The code is fully open-source and structured to run locally on your development machine.

```bash
# Clone the repository
git clone -b product_semiversion https://github.com/Aravind0403/ServiceScope-v2
cd ServiceScope-v2

# Run the local scanner against any target directory
python main.py --target /path/to/your/microservices-repo
```

You can view the interactive dependency graph and run natural language queries (e.g., *"I'm changing payment_service—what breaks?"*) via the terminal or the storage visualization dashboards (Neo4j and PostgreSQL).

The repository is live at [github.com/Aravind0403/ServiceScope-v2](https://github.com/Aravind0403/ServiceScope-v2/tree/product_semiversion). Feel free to run it against your team's microservices and let me know: *how many undocumented dependencies did it find?*
