# Postiz Self-Hosting some known Report & Resolution Guide

## Overview

This repository provides a complete, structured report of issues related to login while self-hosting **Postiz** using Docker Compose, including:

* Initial database and backend startup failures
* Temporal orchestration errors
* Reverse proxy and routing challenges
* Authentication and session issues
* Final working architecture

It includes real logs, configuration examples, root cause analysis, and final resolutions based on my testing.



## Initial Architecture

The stack included:

* **Postiz App** (frontend + backend + orchestrator via PM2)
* **PostgreSQL (Postiz DB)**
* **Redis**
* **Temporal (workflow engine)**
* **Temporal PostgreSQL**
* **Temporal Elasticsearch (later required)**



## Phase 1: Database / Backend Not Starting

### Symptoms

* Frontend accessible
* Backend failing to start
* Requests returning `502 Bad Gateway`

### Logs

```text
ERROR [Error: 14 UNAVAILABLE: No connection established.
Error: connect ECONNREFUSED ::1:7233
```

### Root Cause

* Backend could not connect to **Temporal**
* Temporal service not reachable or not properly resolved



## Fix 1: Networking & Service Dependencies

### Changes

* Unified all services under **one Docker network**
* Fixed `depends_on` with proper health checks

```yaml
depends_on:
  postiz-postgres:
    condition: service_healthy
  postiz-redis:
    condition: service_healthy
  temporal:
    condition: service_started
```

### Why

* Docker DNS resolution was failing across multiple networks
* Services were starting before dependencies were ready



## Phase 2: Temporal DNS & Startup Failure

### Logs

```text
Name resolution failed for target dns:temporal:7233
```

### Root Cause

* Temporal service not discoverable via Docker DNS
* Incorrect network separation

### Resolution

* Moved all services to a **single shared network**



## Phase 3: Backend Crash Due to Temporal Limits

### Logs

```text
INVALID_ARGUMENT: Unable to create search attributes:
cannot have more than 3 search attribute of type Text.
Backend failed to start on port 3000
```

### Root Cause

* Temporal running in **SQL-only visibility mode**
* Postiz requires more than 3 `Text` search attributes



## Fix 2: Enable Elasticsearch for Temporal

### Changes

Added Elasticsearch service:

```yaml
temporal-elasticsearch:
  image: elasticsearch:7.17.27
  environment:
    discovery.type: "single-node"
    xpack.security.enabled: "false"
```

Updated Temporal:

```yaml
environment:
  ENABLE_ES: "true"
  ES_SEEDS: "temporal-elasticsearch"
  ES_VERSION: "v7"
```

### Why

* Temporal SQL visibility has strict limits
* Elasticsearch removes search attribute limitations



## Phase 4: 502 Bad Gateway (Backend Not Reachable)

### Symptoms

* Frontend loads
* API requests return 502

### Logs

```text
connect() failed (111: Connection refused) while connecting to upstream
upstream: "http://127.0.0.1:3000"
```

### Root Cause

* Backend crashed → nginx could not proxy

### Resolution

Fixing Temporal (above) resolved backend crash.



## Phase 5: CORS & URL Mismatch

### Symptoms

```text
Access-Control-Allow-Origin error
```

### Root Cause

* Mixed usage of:

  * `localhost`
  * LAN IP
  * domain

### Fix

Standardized environment variables (can choose between IP, Domain or localhost):

```yaml
MAIN_URL: "http://domain:5000"
FRONTEND_URL: "http://domain:5000"
NEXT_PUBLIC_BACKEND_URL: "http://domain:5000/api"
```


## Phase 6: Authentication Loop (Login Refresh)

### Symptoms

* Login succeeds visually
* Page refreshes, user not authenticated
* Onboarding not progressing

### Root Cause

* Session cookies not being stored
* Domain mismatch (IP vs domain vs localhost)
* Known Postiz cookie-domain issue

## Fix 3: Use Proper Domain or localhost

### Postiz Env

```yaml
MAIN_URL: "https://example.com"
FRONTEND_URL: "https://example.com"
NEXT_PUBLIC_BACKEND_URL: "https://example.com/api"
BACKEND_INTERNAL_URL: "http://localhost:3000"
```

### Why

* Cookies require consistent domain
* IP-based access breaks auth/session flow
