# DevLite Monitor SDK

> Enterprise-grade Node.js observability agent for Express applications.  
> Lightweight · Real-time · Production-ready

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
- [API Reference](#api-reference)
  - [devLiteMonitorRealtime()](#devlitemonitorrealtime)
  - [DevLite.recordMetric()](#devliterecordmetric)
  - [DevLite.startSpan() / endSpan()](#devlitestartspan--endspan)
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

DevLite Monitor SDK is a zero-friction observability agent that plugs into any Express 4 or 5 application. It collects request metrics, distributed traces, system health, database query spans, and custom business metrics — then streams them in real time to the DevLite dashboard.

```js
import { devLiteMonitorRealtime } from "devlite-monitor";

const { middleware, DevLite } = devLiteMonitorRealtime({ apiKey: "YOUR_KEY" });
app.use(middleware);
```

That's all you need to get started.

---

## Features

| Category | Capability |
|---|---|
| **Request Monitoring** | Response time, status codes, payload sizes, slow request detection |
| **Distributed Tracing** | W3C `traceparent`, B3 headers, `x-trace-id`, AsyncLocalStorage propagation |
| **Adaptive Sampling** | Configurable sample rate · auto-reduce under heap pressure · always capture errors |
| **System Metrics** | CPU %, event-loop lag, event-loop utilization, memory, load average |
| **Anomaly Detection** | Latency spikes, error rate spikes, throughput drops via rolling stddev |
| **Error Intelligence** | SHA256 fingerprinting · rate-limited error reporting · stack traces |
| **Database Tracing** | Mongoose, PostgreSQL (`pg`), MySQL (`mysql2`) query spans |
| **Custom Metrics** | `DevLite.recordMetric()` for business events |
| **Background Spans** | `DevLite.startSpan()` / `DevLite.endSpan()` for async jobs |
| **Plugin Hooks** | `onRequestStart`, `onRequestFinish`, `onMetricBuild`, `onFlush` |
| **OpenTelemetry** | Optional OTEL semantic conventions + ResourceSpan export |
| **Compression** | Brotli → gzip → identity fallback on all HTTP payloads |
| **WebSocket Streaming** | Batched 500ms real-time feed to dashboard |
| **Dynamic Config** | Backend can update `sampleRate`, `slowThreshold`, feature flags at runtime |
| **Environment Detection** | Docker, Kubernetes, AWS Lambda, GCP, Azure auto-detected |
| **Cold Start Detection** | First-request flag per process lifetime |
| **Memory Leak Detection** | Rolling heap growth heuristic |
| **SDK Self-Monitoring** | Internal health metrics every 30s |

---

## Installation

```bash
npm install devlite-monitor
```

**Peer requirements:** Node.js ≥ 18, Express 4 or 5.

---

## Quick Start

### Minimal setup

```js
import express from "express";
import { devLiteMonitorRealtime, devLiteErrorMiddleware } from "devlite-monitor";

const app = express();

const { middleware } = devLiteMonitorRealtime({
  apiKey: "YOUR_API_KEY",
});

// Mount BEFORE your routes
app.use(middleware);

// Your routes here
app.get("/users/:id", (req, res) => res.json({ id: req.params.id }));

// Mount AFTER your routes for 5xx error capture
app.use(devLiteErrorMiddleware);
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message });
});

app.listen(3000);
```

### Full setup

```js
import express from "express";
import mongoose from "mongoose";
import {
  devLiteMonitorRealtime,
  devLiteErrorMiddleware,
  getCurrentTraceContext,
} from "devlite-monitor";

const app = express();
app.use(express.json());

const { middleware, statusMiddleware, DevLite } = devLiteMonitorRealtime({
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

  instrument: { mongoose },

  plugins: [
    {
      onRequestFinish: (req, res, metric) => {
        metric.userId = req.user?.id ?? null;
      },
    },
  ],

  debug: "info",
});

// Status endpoint first so it is not self-monitored
app.use(statusMiddleware);
app.use(middleware);

// Routes
app.post("/orders", async (req, res) => {
  const order = await createOrder(req.body);

  // Record a business metric
  DevLite.recordMetric("order_placed", 1, { region: "eu" });

  res.json(order);
});

app.use(devLiteErrorMiddleware);
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message });
});

app.listen(3000);
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
| `maxQueue` | `number` | `5000` | Max in-memory metrics before backpressure drops |
| `flushThreshold` | `number` | `50` | Flush immediately when queue reaches this count |
| `slowThreshold` | `number` | `1000` | Milliseconds — flag request as slow |
| `sampleRate` | `number` | `1.0` | Fraction of requests to record (0–1). Errors and slow requests are always recorded regardless |
| `maxErrorsPerMinute` | `number` | `100` | Rate-limit identical errors per fingerprint per minute |
| `otelCompatibility` | `boolean` | `false` | Add OpenTelemetry semantic convention fields and OTEL span blob |
| `enableStatusEndpoint` | `boolean` | `false` | Expose `GET /__devlite/status` |
| `ignoreRoutes` | `Array<string\|RegExp>` | `[]` | Routes to exclude from monitoring |
| `tags` | `object` | `{}` | Custom tags added to every metric (e.g. `service`, `version`, `region`) |
| `plugins` | `Array` | `[]` | Plugin objects or legacy `(req, res, metric) => void` functions |
| `debug` | `boolean\|string` | `false` | Log level: `true` / `"verbose"` / `"info"` / `"warn"` / `"error"` |
| `instrument` | `object` | `{}` | Database clients to auto-instrument: `{ mongoose, pg, mysql2 }` |

---

## API Reference

### `devLiteMonitorRealtime(options)`

Factory function. Returns `{ middleware, statusMiddleware, DevLite }`.

```js
const { middleware, statusMiddleware, DevLite } = devLiteMonitorRealtime({
  apiKey: "YOUR_KEY",
});
```

| Return value | Description |
|---|---|
| `middleware` | Express middleware — mount with `app.use(middleware)` |
| `statusMiddleware` | Status endpoint middleware — mount before `middleware` |
| `DevLite` | Public API object — `recordMetric`, `startSpan`, `endSpan` |

---

### `DevLite.recordMetric(name, value, tags?)`

Record a custom business metric. Batched and sent with regular metrics.

```js
// Track order volume
DevLite.recordMetric("orders_placed", 1, { region: "eu", plan: "pro" });

// Track payment amount
DevLite.recordMetric("revenue_usd", 49.99, { currency: "USD" });

// Track queue depth
DevLite.recordMetric("email_queue_depth", queue.length);
```

| Parameter | Type | Description |
|---|---|---|
| `name` | `string` | Metric name (non-empty string) |
| `value` | `number` | Numeric value |
| `tags` | `object` | Optional key-value tags merged with global tags |

---

### `DevLite.startSpan(name, tags?)` / `DevLite.endSpan(span)`

Track async background jobs with named spans. Spans are automatically correlated to the active request trace when started inside a request's async scope.

```js
// Track a background job
const span = DevLite.startSpan("send_welcome_email", { userId: user.id });

try {
  await sendWelcomeEmail(user);
} finally {
  DevLite.endSpan(span);
}
```

```js
// Track a scheduled task
cron.schedule("*/5 * * * *", async () => {
  const span = DevLite.startSpan("sync_inventory");
  await syncInventory();
  DevLite.endSpan(span);
});
```

| Parameter | Type | Description |
|---|---|---|
| `name` | `string` | Span name |
| `tags` | `object` | Optional per-span tags |

`startSpan()` returns a span object. Pass it to `endSpan()` to record the duration.

---

### `devLiteErrorMiddleware`

Express error-handling middleware. Mount it **after** your routes and **before** your own error handler.

```js
import { devLiteErrorMiddleware } from "devlite-monitor";

app.use(devLiteErrorMiddleware);  // ← DevLite captures the error
app.use(myOwnErrorHandler);       // ← your handler still runs
```

It attaches the error object to `res.locals.__devliteError` so the request middleware can include the error message, stack trace, and fingerprint in the 5xx metric. It always calls `next(err)` and never swallows errors.

You can also set the error manually in any middleware:

```js
app.use((err, req, res, next) => {
  res.locals.__devliteError = err; // DevLite picks this up
  res.status(500).json({ error: err.message });
});
```

---

### `getCurrentTraceContext()`

Returns the active trace context from anywhere in the async call chain — useful for attaching trace IDs to logs.

```js
import { getCurrentTraceContext } from "devlite-monitor";

app.get("/orders", async (req, res) => {
  const ctx = getCurrentTraceContext();
  logger.info("Processing request", { traceId: ctx?.traceId });

  const orders = await db.query("SELECT * FROM orders");
  res.json(orders);
});
```

Returns `{ traceId, spanId, parentSpanId?, dbQueries[] }` or `null` if called outside a request context.

---

## Database Instrumentation

Pass your database client(s) to the `instrument` option. The SDK wraps query methods and attaches timing spans to the active request trace automatically — no code changes required.

### Mongoose

```js
import mongoose from "mongoose";
import { devLiteMonitorRealtime } from "devlite-monitor";

const { middleware } = devLiteMonitorRealtime({
  apiKey: "YOUR_KEY",
  instrument: { mongoose },
});
```

Captures: `find`, `findOne`, `findOneAndUpdate`, `findOneAndDelete`, `updateOne`, `updateMany`, `deleteOne`, `deleteMany`, `save`, `count`.

### PostgreSQL (`pg`)

```js
import pg from "pg";

const { middleware } = devLiteMonitorRealtime({
  apiKey: "YOUR_KEY",
  instrument: { pg },
});
```

### MySQL (`mysql2`)

```js
import mysql2 from "mysql2";

const { middleware } = devLiteMonitorRealtime({
  apiKey: "YOUR_KEY",
  instrument: { mysql2 },
});
```

### Multiple databases

```js
const { middleware } = devLiteMonitorRealtime({
  apiKey: "YOUR_KEY",
  instrument: { mongoose, pg },
});
```

Each DB query appears in the trace under `database: [{ type, operation, collection, queryTime }]`.

---

## Plugin System

Plugins extend metrics with lifecycle hooks. All hooks are optional and errors inside plugins are caught and logged — a broken plugin will never crash the SDK or your application.

```js
const { middleware } = devLiteMonitorRealtime({
  apiKey: "YOUR_KEY",
  plugins: [
    {
      // Called at the start of every request
      onRequestStart(req, res) {
        res.locals.requestStartedAt = Date.now();
      },

      // Called after the response is sent, before the metric is queued
      onRequestFinish(req, res, metric) {
        metric.userId      = req.user?.id ?? null;
        metric.tenantId    = req.headers["x-tenant-id"] ?? null;
        metric.featureFlag = req.flags?.betaEnabled ?? false;
      },

      // Called after the metric object is fully built
      onMetricBuild(metric) {
        // Strip PII from the route
        metric.route = metric.route.replace(/\/emails\/[^/]+/, "/emails/:email");
      },

      // Called just before a batch is sent to the backend
      onFlush(batch) {
        console.log(`Flushing ${batch.length} metrics`);
      },
    },
  ],
});
```

Legacy function plugins `(req, res, metric) => void` are also supported for backwards compatibility and are treated as `onRequestFinish` hooks.

---

## Dynamic Configuration

The backend can push runtime configuration changes to any connected SDK agent over WebSocket — no restart required.

**Config update** (changes `sampleRate` and `slowThreshold`):
```json
{ "sampleRate": 0.1, "slowThreshold": 500 }
```

**Feature flag update** (toggles individual features):
```json
{
  "enableHistogram": true,
  "enableDbInstrumentation": true,
  "enableSampling": true,
  "enableAnomalyDetection": true,
  "enableDeduplication": false
}
```

To push config from your backend:
```js
import { pushSdkConfig, pushSdkFlags } from "./server.js";

// Reduce sampling for project under load
pushSdkConfig(projectId, { sampleRate: 0.05 });

// Disable histogram during incident
pushSdkFlags(projectId, { enableHistogram: false });
```

---

## Debug Status Endpoint

Enable the built-in status endpoint to inspect SDK internals at runtime:

```js
const { middleware, statusMiddleware } = devLiteMonitorRealtime({
  apiKey: "YOUR_KEY",
  enableStatusEndpoint: true,
});

app.use(statusMiddleware); // mount BEFORE middleware
app.use(middleware);
```

```bash
curl http://localhost:3000/__devlite/status
```

```json
{
  "sdk": "devlite-node-sdk",
  "version": "4.0.0",
  "runtime": "node",
  "pid": 12345,
  "uptime": 3600,
  "env": "production",
  "queueSize": 12,
  "droppedMetrics": 0,
  "wsConnected": true,
  "wsQueueSize": 3,
  "sampleRate": 0.5,
  "slowThreshold": 1000,
  "transportFailures": 0,
  "eventLoopLagMs": 1,
  "heapPressure": false,
  "featureFlags": {
    "enableHistogram": true,
    "enableDbInstrumentation": true,
    "enableSampling": true,
    "enableAnomalyDetection": true,
    "enableDeduplication": true
  }
}
```

> **Note:** Only enable this in environments where the route is not publicly accessible, or protect it with authentication middleware.

---

## OpenTelemetry Compatibility

Enable OTEL semantic conventions and span export with one option:

```js
const { middleware } = devLiteMonitorRealtime({
  apiKey: "YOUR_KEY",
  otelCompatibility: true,
  tags: { service: "payment-api", version: "1.0.0" },
});
```

When enabled, each `request` metric gains:

- OTEL semantic convention fields: `http.method`, `http.route`, `http.status_code`, `http.url`, `http.host`, `service.name`, `service.version`
- An `otel` field containing a W3C-compatible `ResourceSpan` blob suitable for forwarding to an OpenTelemetry Collector HTTP JSON receiver

Distributed trace headers are ingested in priority order:
1. `traceparent` (W3C Trace Context)
2. `x-b3-traceid` / `x-b3-spanid` (Zipkin B3)
3. `x-trace-id` / `x-request-id`
4. Auto-generated

Outgoing responses always carry a well-formed `traceparent` header for downstream propagation.

---

## Metric Types Reference

The SDK emits the following metric types to the backend:

| Type | Description | Cadence |
|---|---|---|
| `request` | HTTP request span with full trace context | Per request (sampled) |
| `latency_histogram` | Bucket counts + average latency | Every 10s |
| `route_stats` | Per-route count / avg / max latency | Every 60s |
| `throughput` | Requests/s and errors/s | Every 10s |
| `anomaly_detected` | Statistical signal spike (latency / errorRate / throughput) | On detection |
| `sdk_health` | Internal SDK diagnostics | Every 30s |
| `custom_metric` | User-defined `DevLite.recordMetric()` events | On call |
| `background_span` | Named async job duration | On `endSpan()` |
| `error_suppressed` | Rate-limited error tombstone | On suppression |
| `uncaughtException` | Global process error | On occurrence |
| `unhandledRejection` | Global promise rejection | On occurrence |

---

## Runtime Support

| Runtime | Support |
|---|---|
| Node.js ≥ 18 | Full support |
| Node.js 16–17 | Supported (event-loop utilization unavailable) |
| Bun | Supported (detected automatically) |
| Deno | Supported (detected automatically) |
| AWS Lambda | Supported — serverless mode auto-detected |
| Google Cloud Functions | Supported — serverless mode auto-detected |
| Azure Functions | Supported — serverless mode auto-detected |
| Docker | Container ID captured automatically from `/proc/self/cgroup` |
| Kubernetes | Pod name and namespace captured automatically |

---

## Performance

The SDK is designed to add less than **5ms overhead** per request on the hot path.

Key design decisions that achieve this:

- **System metrics are cached** — `os.cpus()`, `process.memoryUsage()` are collected at most once every 10 seconds, never per-request
- **All I/O is fire-and-forget** — HTTP batches and WebSocket emissions are async and never awaited on the request path
- **Metric building runs in `res.on("finish")`** — executes after the response is already sent to the client
- **`process.hrtime.bigint()`** — nanosecond timing with no object allocation
- **Adaptive sampling** — automatically reduces sample rate under heap pressure or high SDK CPU overhead
- **Backpressure protection** — metrics are dropped gracefully (with counters) rather than crashing or blocking when the queue is full

If the SDK's internal processing ever exceeds 5ms, a warning is logged with the actual duration so you can investigate.

---

## License

MIT © [Kevin Ishimwe](https://github.com/Ishimwe-Kevin)
