# DevLite Monitor SDK

> Enterprise-grade Node.js observability agent — one line to get started.  
> Lightweight · Real-time · Production-ready · OpenTelemetry-compatible

[![npm version](https://img.shields.io/npm/v/devlite-monitor)](https://www.npmjs.com/package/devlite-monitor)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Node.js](https://img.shields.io/badge/node-%3E%3D18.0.0-brightgreen)](https://nodejs.org)

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Usage Patterns](#usage-patterns)
- [API Reference](#api-reference)
  - [devLiteMonitorRealtime()](#devlitemonitorrealtime)
  - [DevLite.recordMetric()](#devliterecordmetric)
  - [DevLite.increment() · timing() · gauge()](#devliteincrement--timing--gauge)
  - [DevLite.startSpan() · endSpan() · withSpan()](#devlitestartspan--endspan--withspan)
  - [DevLite.log()](#devlitelog)
  - [DevLite.registerHealthCheck()](#devliteregisterhealthcheck)
  - [devLiteErrorMiddleware](#devliteerrormiddleware)
  - [getCurrentTraceContext()](#getcurrenttracecontext)
- [Database Instrumentation](#database-instrumentation)
- [Plugin System](#plugin-system)
- [Dynamic Configuration](#dynamic-configuration)
- [Debug Status Endpoint](#debug-status-endpoint)
- [OpenTelemetry Compatibility](#opentelemetry-compatibility)
- [Metric Types Reference](#metric-types-reference)
- [Runtime Support](#runtime-support)
- [Performance](#performance)

---

## Overview

DevLite Monitor SDK is a drop-in observability agent for Express applications. It collects request metrics, distributed traces, system health, database query spans, structured logs, and custom business metrics — then streams them in real time to the DevLite dashboard with zero configuration required.

```js
// The entire setup — one line
app.use(devLiteMonitorRealtime({ apiKey: process.env.DEVLITE_API_KEY }));
```

---

## Features

| Category | Capability |
|---|---|
| **Request Monitoring** | Response time, status codes, payload sizes, slow request detection, PII scrubbing |
| **Distributed Tracing** | W3C `traceparent`, B3 headers, `x-trace-id`, AsyncLocalStorage propagation |
| **Adaptive Sampling** | Configurable rate · auto-reduce under heap/CPU pressure · always capture errors |
| **System Metrics** | CPU %, event-loop lag, event-loop utilization, memory, load average, free/total mem |
| **Anomaly Detection** | Latency spikes, error rate spikes, throughput drops — Welford online algorithm |
| **Error Intelligence** | SHA256 fingerprinting · rate-limited reporting · stack traces · `onError` hook |
| **Database Tracing** | Mongoose, PostgreSQL (`pg`), MySQL (`mysql2`) — query spans with p95 |
| **Custom Metrics** | `recordMetric` · `increment` · `timing` · `gauge` |
| **Background Spans** | `startSpan` · `endSpan` · `withSpan` auto-wrapper |
| **Structured Logs** | `DevLite.log()` ships correlated log entries with trace IDs attached |
| **Health Checks** | `registerHealthCheck()` — recurring uptime probes emitted as metrics |
| **Plugin Hooks** | `onRequestStart` · `onRequestFinish` · `onMetricBuild` · `onFlush` · `onError` |
| **OpenTelemetry** | Full OTEL semantic conventions + ResourceSpan export with DB span events |
| **Compression** | Brotli (quality 4) → gzip → identity fallback on all HTTP payloads |
| **Payload Integrity** | SHA256 checksum on every batch (`X-Payload-Checksum` header) |
| **WebSocket Streaming** | Batched 500ms real-time feed to dashboard |
| **Dynamic Config** | Backend pushes `sampleRate`, `slowThreshold`, feature flags at runtime |
| **Environment Detection** | Docker, Kubernetes, AWS Lambda, GCP, Azure — auto-detected |
| **Cold Start Detection** | First-request flag per process lifetime |
| **Memory Leak Detection** | Rolling heap growth heuristic |
| **SDK Self-Monitoring** | Internal health metrics, queue size, dropped counts — every 30s |

---

## Installation

```bash
npm install devlite-monitor
```

**Peer requirements:** Node.js ≥ 18, Express 4 or 5.

---

## Quick Start

### Minimal — one line

```js
import express from "express";
import { devLiteMonitorRealtime } from "devlite-monitor";

const app = express();

app.use(devLiteMonitorRealtime({ apiKey: process.env.DEVLITE_API_KEY }));

app.get("/users/:id", (req, res) => res.json({ id: req.params.id }));

app.listen(3000);
```

### With error capture

```js
import express from "express";
import { devLiteMonitorRealtime, devLiteErrorMiddleware } from "devlite-monitor";

const app = express();

const monitor = devLiteMonitorRealtime({ apiKey: process.env.DEVLITE_API_KEY });

app.use(monitor);

// Your routes
app.get("/orders/:id", getOrder);

// Mount AFTER routes for 5xx error detail in traces
app.use(monitor.errorMiddleware);
app.use((err, req, res, next) => res.status(500).json({ error: err.message }));

app.listen(3000);
```

### Full production setup

```js
import express    from "express";
import mongoose   from "mongoose";
import pg         from "pg";
import {
  devLiteMonitorRealtime,
  devLiteErrorMiddleware,
  getCurrentTraceContext,
  DevLite,
} from "devlite-monitor";

const app = express();
app.use(express.json());

const monitor = devLiteMonitorRealtime({
  apiKey:               process.env.DEVLITE_API_KEY,
  sampleRate:           0.5,
  slowThreshold:        800,
  otelCompatibility:    true,
  enableStatusEndpoint: true,

  tags: {
    service: "orders-api",
    version: "2.1.0",
    region:  "eu-west-1",
  },

  ignoreRoutes: ["/health", "/ping", /^\/internal\//],

  instrument: { mongoose, pg },

  plugins: [{
    onRequestFinish: (req, res, metric) => {
      metric.userId   = req.user?.id   ?? null;
      metric.tenantId = req.headers["x-tenant-id"] ?? null;
    },
    onError: (err, req, res) => {
      alertChannel.critical(err.message, { traceId: req.headers["x-trace-id"] });
    },
  }],

  debug: "info",
});

app.use(monitor);

// Register health checks
DevLite.registerHealthCheck("database", () => mongoose.connection.db.admin().ping());
DevLite.registerHealthCheck("redis",    () => redis.ping(), 15_000);

// Routes
app.post("/orders", async (req, res) => {
  const result = await DevLite.withSpan("create_order", () => createOrder(req.body));

  DevLite.increment("orders_created", { plan: req.user.plan });
  DevLite.gauge("pending_orders", await getPendingCount());
  DevLite.log.info("Order created", { orderId: result.id, userId: req.user.id });

  res.json(result);
});

app.use(monitor.errorMiddleware);
app.use((err, req, res, next) => res.status(500).json({ error: err.message }));

app.listen(3000, () => console.log("Server started on port 3000"));
```

---

## Configuration

All options are passed to `devLiteMonitorRealtime()`.

| Option | Type | Default | Description |
|---|---|---|---|
| `apiKey` | `string` | **required** | Your DevLite project API key |
| `backendUrl` | `string` | DevLite cloud | Override the metrics HTTP endpoint |
| `socketUrl` | `string` | DevLite cloud | Override the WebSocket endpoint |
| `batchInterval` | `number` | `5000` | Milliseconds between periodic HTTP flushes |
| `maxQueue` | `number` | `10000` | Max in-memory metrics before backpressure |
| `flushThreshold` | `number` | `50` | Eager-flush queue size |
| `slowThreshold` | `number` | `1000` | Flag requests slower than this (ms) |
| `sampleRate` | `number` | `1.0` | Fraction to sample (0–1). Errors, slow requests, and cold starts are always sampled |
| `maxErrorsPerMinute` | `number` | `100` | Rate-limit per error fingerprint per minute |
| `otelCompatibility` | `boolean` | `false` | Add OTEL semantic fields and ResourceSpan blobs |
| `enableStatusEndpoint` | `boolean` | `false` | Expose `GET /__devlite/status` automatically |
| `ignoreRoutes` | `Array<string\|RegExp>` | `[]` | Routes to skip entirely |
| `tags` | `object` | `{}` | Static tags on every metric (`service`, `version`, `region`, etc.) |
| `plugins` | `Array` | `[]` | Plugin objects or legacy `(req, res, metric) => void` functions |
| `debug` | `boolean\|string` | `false` | Log level: `true` · `"verbose"` · `"info"` · `"warn"` · `"error"` |
| `instrument` | `object` | `{}` | DB clients: `{ mongoose, pg, mysql2 }` |

---

## Usage Patterns

### Pattern 1 — Drop-in (one line)

```js
app.use(devLiteMonitorRealtime({ apiKey: "YOUR_KEY" }));
```

### Pattern 2 — Instance with error capture

```js
const monitor = devLiteMonitorRealtime({ apiKey: "YOUR_KEY" });
app.use(monitor);
app.use(monitor.errorMiddleware); // after routes
```

### Pattern 3 — Full access via instance

```js
const monitor = devLiteMonitorRealtime({ apiKey: "YOUR_KEY" });
app.use(monitor);
app.use(monitor.errorMiddleware);

monitor.DevLite.recordMetric("checkout", 1);
monitor.DevLite.registerHealthCheck("db", () => db.ping());
```

### Pattern 4 — Module-level singleton (no reference needed)

```js
// app.js
app.use(devLiteMonitorRealtime({ apiKey: "YOUR_KEY" }));

// anywhere_else.js — works after the above has run once
import { DevLite } from "devlite-monitor";
DevLite.recordMetric("payment_processed", 1);
```

### Pattern 5 — Destructuring (backwards compatible)

```js
const { middleware, DevLite } = devLiteMonitorRealtime({ apiKey: "YOUR_KEY" });
app.use(middleware);
```

---

## API Reference

### `devLiteMonitorRealtime(options)`

Factory function. Returns an Express middleware function with extra capabilities attached.

```js
const monitor = devLiteMonitorRealtime({ apiKey: "YOUR_KEY" });

// The function itself is the middleware
app.use(monitor);

// Named properties for advanced usage
monitor.middleware        // raw request middleware (no status endpoint routing)
monitor.statusMiddleware  // standalone /__devlite/status handler
monitor.errorMiddleware   // 5xx error-capture middleware
monitor.DevLite           // full API object
monitor.getCurrentTrace   // alias for getCurrentTraceContext()
```

---

### `DevLite.recordMetric(name, value, tags?)`

Record any named numeric metric. Batched and sent with regular metrics.

```js
DevLite.recordMetric("orders_placed",    1,     { region: "eu", plan: "pro" });
DevLite.recordMetric("revenue_usd",      49.99, { currency: "USD" });
DevLite.recordMetric("cart_item_count",  req.body.items.length);
```

---

### `DevLite.increment()` · `timing()` · `gauge()`

Typed convenience wrappers around `recordMetric`.

```js
// Increment a counter by 1 (or n)
DevLite.increment("api_calls",           { endpoint: "/checkout" });
DevLite.increment("failed_logins",       { reason: "wrong_password" }, 1);

// Record a duration in milliseconds
DevLite.timing("stripe_charge_ms",       324, { currency: "usd" });
DevLite.timing("email_send_ms",          241, { provider: "sendgrid" });

// Set a gauge to a current absolute value
DevLite.gauge("active_connections",      pool.totalCount);
DevLite.gauge("pending_jobs",            queue.length);
DevLite.gauge("cache_hit_ratio",         hits / (hits + misses));
```

---

### `DevLite.startSpan()` · `endSpan()` · `withSpan()`

Track async background jobs with named spans. Spans started inside a request's async scope are automatically correlated to the parent trace.

**Manual span:**
```js
const span = DevLite.startSpan("process_refund", { orderId: order.id });
try {
  await processRefund(order);
} finally {
  DevLite.endSpan(span);
}
```

**Auto-wrap with `withSpan` (recommended):**
```js
// Wraps any async function — span ends automatically on success or error
const charge = await DevLite.withSpan("stripe_charge",  () => stripe.charge(data));
const user   = await DevLite.withSpan("fetch_user",     () => db.users.findById(id));
const sent   = await DevLite.withSpan("send_invoice",   () => mailer.send(invoice), { customerId });
```

**Cron jobs:**
```js
cron.schedule("*/5 * * * *", async () => {
  await DevLite.withSpan("sync_inventory", syncInventory);
});
```

---

### `DevLite.log()`

Ship structured log entries to the DevLite dashboard with trace context automatically attached. Logs are batched (50 per flush, max 5s delay).

```js
// Using level string
DevLite.log("info",  "Payment processed",   { orderId: "123", amount: 49.99 });
DevLite.log("warn",  "Retry attempt",       { attempt: 3, endpoint: "/payments" });
DevLite.log("error", "DB connection failed",{ host: "postgres-1", port: 5432 });

// Using convenience shortcuts
DevLite.log.info("Server started",          { port: 3000 });
DevLite.log.warn("Rate limit approaching",  { remaining: 5 });
DevLite.log.error("Unhandled error",        { stack: err.stack });
```

Every log entry automatically includes `traceId` and `spanId` when called inside a request's async scope — giving you correlated logs and traces in the dashboard.

---

### `DevLite.registerHealthCheck()`

Register a recurring health check. Results are emitted as `health_check` metrics with pass/fail status and response time.

```js
// Basic — returns true (healthy), false (unhealthy), or throws (unhealthy)
DevLite.registerHealthCheck("database", () => db.ping());
DevLite.registerHealthCheck("redis",    () => redis.ping(), 15_000);  // every 15s

// Custom check
DevLite.registerHealthCheck("payments", async () => {
  const response = await fetch("https://api.stripe.com/v1/balance");
  return response.ok;
}, 60_000); // every 60s

// Multiple checks registered at startup
DevLite.registerHealthCheck("postgres",    () => pgPool.query("SELECT 1"));
DevLite.registerHealthCheck("elasticsearch", () => esClient.ping());
DevLite.registerHealthCheck("external_api", () => externalApi.health());
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `name` | `string` | required | Health check name shown in dashboard |
| `fn` | `async () => boolean\|void` | required | Check function — throw or return `false` = unhealthy |
| `intervalMs` | `number` | `60000` | How often to run the check |

---

### `devLiteErrorMiddleware`

Express error-handling middleware. Mount **after** your routes and **before** your own error handler.

```js
// Named import
import { devLiteErrorMiddleware } from "devlite-monitor";
app.use(devLiteErrorMiddleware);
app.use(myErrorHandler);

// From instance (equivalent)
app.use(monitor.errorMiddleware);
app.use(myErrorHandler);
```

It attaches the error to `res.locals.__devliteError` so the request middleware enriches the 5xx metric with the error message, truncated stack trace, and SHA256 fingerprint. Always calls `next(err)` — never swallows errors.

You can also set it manually:

```js
app.use((err, req, res, next) => {
  res.locals.__devliteError = err; // DevLite picks this up
  res.status(500).json({ error: err.message });
});
```

---

### `getCurrentTraceContext()`

Returns the active trace context from anywhere in the async call chain — useful for attaching trace IDs to your own logger.

```js
import { getCurrentTraceContext } from "devlite-monitor";

app.get("/orders", async (req, res) => {
  const { traceId } = getCurrentTraceContext() ?? {};
  logger.info("Fetching orders", { traceId }); // correlated log

  const orders = await Order.find();
  res.json(orders);
});
```

Returns `{ traceId, spanId, parentSpanId?, dbQueries[] }` or `null` outside a request context.

---

## Database Instrumentation

Pass your database client(s) to the `instrument` option. The SDK wraps query methods and attaches timing spans to the active request trace — no code changes required in your queries.

```js
// Mongoose
const monitor = devLiteMonitorRealtime({
  apiKey:     "YOUR_KEY",
  instrument: { mongoose },
});
```

```js
// PostgreSQL
const monitor = devLiteMonitorRealtime({
  apiKey:     "YOUR_KEY",
  instrument: { pg },
});
```

```js
// MySQL
const monitor = devLiteMonitorRealtime({
  apiKey:     "YOUR_KEY",
  instrument: { mysql2 },
});
```

```js
// Multiple databases
const monitor = devLiteMonitorRealtime({
  apiKey:     "YOUR_KEY",
  instrument: { mongoose, pg },
});
```

Each query appears in the request trace under `database: [{ type, operation, collection, queryTime }]`. Mongoose also captures `aggregate` operations in v5.

---

## Plugin System

Plugins extend metrics with lifecycle hooks. All hooks are optional. Errors inside plugins are caught per-plugin — a broken plugin never crashes the SDK or your application.

```js
const monitor = devLiteMonitorRealtime({
  apiKey: "YOUR_KEY",
  plugins: [{
    // Called synchronously at request start
    onRequestStart(req, res) {
      res.locals.startedAt = Date.now();
    },

    // Called after response is sent — enrich the metric
    onRequestFinish(req, res, metric) {
      metric.userId      = req.user?.id     ?? null;
      metric.tenantId    = req.headers["x-tenant-id"] ?? null;
      metric.featureFlag = req.flags?.betaEnabled      ?? false;
    },

    // Called after metric is fully built — last mutation chance
    onMetricBuild(metric) {
      metric.route = metric.route.replace(/\/emails\/[^/]+/, "/emails/:email");
    },

    // Called just before HTTP flush — inspect or log the batch
    onFlush(batch) {
      console.log(`Flushing ${batch.length} metrics`);
    },

    // Called when a 5xx error is captured
    onError(err, req, res) {
      pagerDuty.trigger(err.message, { traceId: req.headers["x-trace-id"] });
    },
  }],
});
```

Legacy `(req, res, metric) => void` functions are treated as `onRequestFinish` hooks for backwards compatibility.

---

## Dynamic Configuration

The backend can push runtime configuration to connected SDK agents over WebSocket — no restarts required.

**Update runtime config** (`sampleRate`, `slowThreshold`):
```json
{ "sampleRate": 0.1, "slowThreshold": 500 }
```

**Toggle feature flags:**
```json
{
  "enableHistogram":         true,
  "enableDbInstrumentation": true,
  "enableSampling":          true,
  "enableAnomalyDetection":  true,
  "enableDeduplication":     false,
  "enableLogCapture":        true,
  "enableHealthChecks":      true
}
```

**Pushing config from your backend:**
```js
import { pushSdkConfig, pushSdkFlags } from "./server.js";

pushSdkConfig(projectId, { sampleRate: 0.05 });
pushSdkFlags(projectId,  { enableHistogram: false });
```

---

## Debug Status Endpoint

Enable the built-in status endpoint to inspect SDK internals at runtime. When `enableStatusEndpoint: true`, it is handled automatically by the combined middleware — no extra mount needed.

```js
const monitor = devLiteMonitorRealtime({
  apiKey:               "YOUR_KEY",
  enableStatusEndpoint: true,
});

app.use(monitor); // /__devlite/status is served automatically
```

```bash
curl http://localhost:3000/__devlite/status
```

```json
{
  "sdk": "devlite-node-sdk",
  "version": "5.0.0",
  "runtime": "node",
  "pid": 12345,
  "uptime": 3600,
  "env": "production",
  "detectedEnv": { "docker": true, "containerId": "a1b2c3d4..." },
  "queueSize": 12,
  "droppedMetrics": 0,
  "wsConnected": true,
  "wsQueueSize": 3,
  "sampleRate": 0.5,
  "slowThreshold": 800,
  "transportFailures": 0,
  "eventLoopLagMs": 1,
  "heapPressure": false,
  "featureFlags": {
    "enableHistogram": true,
    "enableDbInstrumentation": true,
    "enableSampling": true,
    "enableAnomalyDetection": true,
    "enableDeduplication": true,
    "enableLogCapture": true,
    "enableHealthChecks": true
  }
}
```

> Protect this route in production with authentication middleware or restrict it to internal networks.

---

## OpenTelemetry Compatibility

```js
const monitor = devLiteMonitorRealtime({
  apiKey:           "YOUR_KEY",
  otelCompatibility: true,
  tags:             { service: "payment-api", version: "1.0.0" },
});
```

When enabled, every `request` metric gains OTEL semantic convention fields:

- `http.method`, `http.route`, `http.status_code`, `http.url`, `http.host`
- `http.request_content_length`, `http.response_content_length`
- `http.user_agent`, `http.flavor`
- `service.name`, `service.version`, `service.namespace`
- `deployment.environment`, `net.host.name`
- `telemetry.sdk.name`, `telemetry.sdk.version`, `telemetry.sdk.language`

And an `otel` field containing a W3C-compatible `ResourceSpan` blob with DB span events — suitable for forwarding to an OpenTelemetry Collector HTTP JSON receiver.

**Trace header priority (ingest):**
1. `traceparent` (W3C Trace Context)
2. `x-b3-traceid` / `x-b3-spanid` (Zipkin B3)
3. `x-trace-id` / `x-request-id`
4. Auto-generated

**Outgoing responses** always carry a well-formed `traceparent` header for downstream service propagation.

---

## Metric Types Reference

| Type | Description | Cadence |
|---|---|---|
| `request` | HTTP request span — full trace, system snapshot, DB spans | Per request (sampled) |
| `latency_histogram` | 7-bucket counts + avg + p95 | Every 10s |
| `route_stats` | Per-route count / avg / p95 / max / error rate | Every 60s |
| `throughput` | Requests/s and errors/s | Every 10s |
| `anomaly_detected` | Statistical spike — latency / errorRate / throughput | On detection |
| `sdk_health` | Internal SDK diagnostics | Every 30s |
| `custom_metric` | `recordMetric` · `increment` · `timing` · `gauge` | On call |
| `background_span` | Named async job duration from `startSpan`/`endSpan`/`withSpan` | On `endSpan` |
| `log_batch` | Structured log entries with trace correlation | Batched (50 or 5s) |
| `health_check` | Named uptime check result with response time | Per check interval |
| `error_suppressed` | Rate-limit tombstone for silenced identical errors | On suppression |
| `uncaughtException` | Global process error | On occurrence |
| `unhandledRejection` | Global promise rejection | On occurrence |

---

## Runtime Support

| Runtime | Support |
|---|---|
| Node.js ≥ 18 | Full support |
| Node.js 16–17 | Supported (event-loop utilization unavailable) |
| Bun | Supported — detected automatically |
| Deno | Supported — detected automatically |
| AWS Lambda | Supported — serverless mode auto-detected |
| Google Cloud Functions | Supported — serverless mode auto-detected |
| Azure Functions | Supported — serverless mode auto-detected |
| Docker | Container ID captured from `/proc/self/cgroup` |
| Kubernetes | Pod name + namespace captured from service account mount |

---

## Performance

The SDK adds less than **5ms overhead** per request on the hot path.

| Technique | Effect |
|---|---|
| System metrics cached (10s TTL) | `os.cpus()` and `process.memoryUsage()` never called per-request |
| `process.hrtime.bigint()` timing | Nanosecond precision, zero object allocation |
| `res.on("finish")` execution | All metric work runs after response is sent — zero client latency impact |
| Fire-and-forget transport | HTTP flushes and WebSocket emissions never awaited on the request path |
| Adaptive sampling | Rate auto-reduces under heap pressure (>85%) or SDK CPU overhead (>1%) |
| Welford online algorithm | O(1) memory anomaly detection regardless of traffic volume |
| Backpressure protection | Metrics dropped gracefully with counters — never crashes or blocks |
| Deduplication window | Identical request metrics aggregated within 5s windows at high QPS |
| PII scrubbing | URL patterns replaced before recording — compliance with zero code changes |

If internal processing exceeds 5ms, a warning is logged with the actual duration.

---

## License

MIT © [Kevin Ishimwe](https://github.com/Ishimwe-Kevin)
