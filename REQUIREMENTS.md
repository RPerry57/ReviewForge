# Software Requirement Specification (SRS)

## Project Title: ReviewForge

**Automated Static Analysis & Code Quality Engine for GitHub**

---

## 1. Executive Summary

**ReviewForge** is an asynchronous, developer-focused code review automation platform. It listens for GitHub Pull Request (PR) webhooks, executes static code analysis and linting within isolated, ephemeral Docker containers, and uses LLMs/rule engines to generate inline suggestions on specific lines of code.

### Primary Resume Value Drivers

* **Decoupled Asynchronous Architecture:** Demonstrates mastery of event-driven design, message brokers, and queue worker patterns.
* **Security & Isolation:** Demonstrates containerization and safe execution of untrusted code within sandboxed environments.
* **Developer Platform Integration:** Demonstrates working with REST/GraphQL APIs, HMAC authentication, rate limiting, and webhook ingestion.

---

## 2. System Architecture & Data Flow

```
+------------------+         HMAC Signature      +----------------------+
|                  | ---------------------------> |  FastAPI Webhook     |
|   GitHub App     |                              |     Ingress          |
|                  | <--------------------------- |                      |
+------------------+     200 OK (Immediate)       +----------------------+
                                                            |
                                                            | Push Job
                                                            v
+------------------+        Worker Poll           +----------------------+
|                  | <--------------------------- |    Redis / Celery    |
|   Celery Worker  |                              |     Task Queue       |
|                  |                              +----------------------+
+------------------+
        |
        | 1. Spin up ephemeral container
        v
+-------------------------------------------------+
| Docker Sandbox Container                        |
|   - Git Clone PR Branch                         |
|   - Run Linters (Ruff/Flake8/ESLint)            |
|   - Generate Structured Diff Analysis           |
+-------------------------------------------------+
        |
        | 2. Aggregate & Map Output to Diff Lines
        v
+------------------+        Post Inline           +----------------------+
|  Review Service  | ---------------------------> | GitHub REST/GraphQL  |
|                  |          Comments            |        API           |
+------------------+                              +----------------------+

```

---

## 3. Functional Requirements

### 3.1 Webhook Ingestion & Security

* **Req 1.1 (Payload Verification):** Verify incoming GitHub Webhook requests using SHA-256 HMAC signatures (`X-Hub-Signature-256`) against a stored secret.
* **Req 1.2 (Rapid Response):** Acknowledge incoming `pull_request` events (`opened`, `synchronize`) with an immediate `200 OK` within 200ms to prevent webhook timeouts.
* **Req 1.3 (Job Dispatch):** Parse PR metadata (Repository ID, PR Number, Head SHA, Base Branch) and publish a structured job payload to the Redis queue.

### 3.2 Task Processing & Worker Pool

* **Req 2.1 (Queue Management):** Process incoming jobs asynchronously using Celery/Redis worker nodes.
* **Req 2.2 (Deduplication):** Prevent redundant analysis runs if a rapid succession of pushes occurs on the same branch.

### 3.3 Sandboxed Execution

* **Req 3.1 (Isolation):** Programmatically spawn a restricted Docker container per job using the Docker SDK.
* **Req 3.2 (Resource Limits):** Enforce strict CPU ($0.5$ cores), Memory ($512\text{ MB}$), execution timeout ($120\text{ seconds}$), and disabled network access post-clone to prevent resource exhaustion or malicious activity.
* **Req 3.3 (Linter Runner):** Execute static analysis tools (e.g., `ruff` for Python, `eslint` for JavaScript) against only modified/added files in the git diff.

### 3.4 Review Aggregation & GitHub API Feedback

* **Req 4.1 (Line Mapping):** Map linter/analysis outputs strictly to valid line numbers present in the pull request diff snippet.
* **Req 4.2 (API Commenting):** Call the GitHub GraphQL/REST API to post inline comments on specific diff lines, as well as a summary comment summarizing overall code health.
* **Req 4.3 (Rate Limit Handling):** Implement exponential backoff and token-bucket strategies to stay within GitHub's primary and secondary API rate limits.

---

## 4. Technical Stack

| Layer | Technology Choice |
| --- | --- |
| **Language** | Python 3.11+ |
| **API Framework** | FastAPI |
| **Message Queue / Broker** | Redis + Celery |
| **Sandboxing & Compute** | Docker (via Python `docker` SDK) |
| **External Integration** | GitHub REST API v3 / GraphQL API |
| **Development Tunneling** | `ngrok` or `localtunnel` (for testing webhooks locally) |

---

## 5. Development Roadmap & Implementation Steps

### Phase 1: Ingestion & Webhook Infrastructure

1. Create a GitHub App in GitHub Developer Settings with `Pull Requests: Read & Write` permissions.
2. Build a FastAPI app with a POST `/webhooks/github` route.
3. Add HMAC cryptographic signature validation.
4. Set up `ngrok` to forward GitHub payloads to `localhost:8000`.

### Phase 2: Asynchronous Processing & Queue

1. Spin up a local Redis instance via Docker (`docker run -p 6379:6379 redis`).
2. Initialize Celery worker task definitions in Python.
3. Configure FastAPI to dispatch webhook payloads directly to Celery task queues.

### Phase 3: Docker Sandbox Execution

1. Write a Python worker task that receives a repo URL and commit SHA.
2. Use the Docker SDK (`docker.from_env()`) to spin up a container with `git` and linter tools pre-installed.
3. Mount a temporary volume, clone the branch, run static analysis tools, and parse raw outputs into structured JSON.

### Phase 4: GitHub API Integration & Inline Commenting

1. Obtain an Installation Access Token from the GitHub App JWT key.
2. Parse the git diff to identify modified line numbers.
3. Send POST requests to `/repos/{owner}/{repo}/pulls/{pull_number}/reviews` to submit inline suggestions and review status (`APPROVE` vs `REQUEST_CHANGES`).

---