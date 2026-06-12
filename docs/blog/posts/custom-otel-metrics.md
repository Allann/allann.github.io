--- 
title: Custom OTEL Metrics
date: 2025-05-29
categories: [programming, git]
tags: [otel, aspire]
---

# Creating Meaningful Custom Metrics in .NET with OpenTelemetry and .NET Aspire Dashboard

**Observability**—the ability to understand what is happening inside your software systems—has become essential in
today’s complex applications. While logs and traces provide valuable technical information, **custom metrics** unlock
deeper insights by focusing on what truly matters to your business.

<!-- more -->

Throughout this article we will anchor concepts to a *real‐world scenario* taken from the waste‑management domain: an
API that coordinates requests for video footage captured by garbage‑truck camera units.

> **Scenario in Brief**
> 
> - Requests arrive from multiple channels—an internal support portal, council‑facing public website, and
service‑to‑service calls.
> - Each request identifies a truck, resolves the truck’s current video‑provider integration, triggers the provider’s
API, and queues the job.
> - A background worker later downloads the footage, applies watermarking, stores it on‑prem, and notifies the requestor
by email.
> - Access to the video is time‑boxed: downloads/views can be restricted per user type; content expires after 30 days
and is deleted if never accessed within seven days to conserve storage.

We will use this workflow to show *why* and *how* to craft business‑centric OpenTelemetry metrics that surface genuine
operational and customer value.

## What You Will Learn

* Why custom metrics are vital for business‑focused monitoring
* How they differ from built‑in technical metrics
* Step‑by‑step guidance on creating custom metrics with OpenTelemetry in .NET 9+
* How to visualise those metrics in real time with the **.NET Aspire Dashboard**
* Proven practices to maximise the value of your metrics

## Why Custom Metrics Matter More Than Technical Metrics Alone

Application Performance Monitoring (or Management) suites refers to a commercial or open-source toolset that continuously
collects and analyses telemetry (CPU, memory, request latency, error rates, traces, etc.) from your applications and
infrastructure so teams can detect and diagnose performance issues in real time.
These suites ship technical or infrastructure‑level metrics out of the box—CPU usage, memory consumption, HTTP
latency. These figures tell you **how well the runtime is behaving**, but seldom **what the numbers mean for the
business workflow**.  Aspire uses OpenTelemetry out the box and the one we will be targeting in this article.

### What Are Custom Metrics?

Custom metrics are values you define and emit based on **domain events**. In our garbage‑truck video service, meaningful
metrics include:

* `video_requests.registered`– every new footage request (tagged with `origin=portal|council|s2s`)
* `video_jobs.queued` and `video_jobs.failed`
* `videos.ready`– footage processed, watermarked, and stored
* `video_notifications.sent` (tagged with `userType` and `delivery=result|error`)
* `videos.expired.deleted`– storage reclaimed after retention rules fire

### Why Choose Custom Metrics?

* **Align With Business KPIs** – Instead of “CPU 85 %”, operations staff see “background‑queue backlog = 27” and can
  judge SLA risk.
* **Contextual Alerting** – An alert on `videos.ready` dropping to zero during business hours pinpoints a real service
  failure.
* **Shared Language** – Support, developers and council stakeholders discuss the same graphs without translating tech
  jargon.

### Scenario Illustration

A sudden spike in CPU could be routine watermark encoding. But a rise in `video_jobs.failed` accompanied by a stall in
`videos.ready` immediately flags that footage is not reaching requestors—an outcome with contractual implications for
councils.

---

## How to Create Custom Metrics in .NET Using OpenTelemetry

Aspire and OpenTelemetry provides a vendor‑neutral, extensible API for metrics. The high‑level workflow is:

1. **Add and configure OpenTelemetry** in your .NET 9 application (worker service, API, or both).
2. **Define a `Meter` and instruments** (counters, histograms, up‑down counters) that represent domain events.
3. **Instrument code paths** — increment counters when a request is registered; observe durations while footage is being
   processed.
4. **Export metrics** (OTLP) and visualise them in the Aspire Dashboard.

```csharp
// ServiceDefaults/Extensions.cs

builder.Services.AddOpenTelemetry()
       .WithMetrics(metrics =>
       {
           metrics
                  .AddAspNetCoreInstrumentation()
                  .AddHttpClientInstrumentation();
           
           metrics.AddMeter(VideoMetrics.Meter.Name);
           metrics.AddOtlpExporter();
       })       
       // ...other trace setup
```

```csharp
// program.cs

builder.Services.AddSingleton<VideoMetrics>();

// code removed for brevity

app.MapPost("/api/footage", async ([FromBody] VideoRequest request, [FromServices] VideoMetrics metrics, CancellationToken ct) =>
{
    metrics.VideoRequestsRegistered.Add(1, new("request.origin", request.Source));

    // …business logic…
});
```

```csharp
// VideoMetrics.cs – strongly‑typed wrapper keeps metric names consistent
public sealed class VideoMetrics(MeterProvider meterProvider)
{
    public const string Name = "Video Api";
    
    public static Meter Meter = new(Name, "1.0.0");

    public Counter<int> VideoRequestsRegistered { get; } =
        Meter.CreateCounter<int>("video_requests.registered");

    public Counter<int> VideoJobsQueued { get; } =
        Meter.CreateCounter<int>("video_jobs.queued");

    public Counter<int> VideosReady { get; } =
        Meter.CreateCounter<int>("videos.ready");

    public Histogram<int> VideoProcessingDurationMs { get; } =
        Meter.CreateHistogram<int>("video_processing.duration_ms");
}
```

## Best Practices When Designing Custom Metrics

* **Clear Names**– prefer `video_jobs.failed` over `counter42`.
* **Tag Only What You Query** – high‑cardinality tags (e.g., truck VIN) balloon storage; stick to dimensions that
  matter (`origin`, `userType`).
* **Emit on Meaningful Boundaries** – increment `videos.ready` once per finished file, *not* inside a polling loop.
* **Unit Consistency** – store durations in milliseconds; sizes in bytes.
* **Guard Against Over‑Instrumentation** – track the video life‑cycle stages, not every line of code.

## Visualising Metrics with .NET Aspire Dashboard

Spin up the Aspire dashboard alongside your service and you can:

* **Watch live graphs** of `video_jobs.queued` vs. `videos.ready` to verify the pipeline is keeping up.
* **Drill into tags** — filter `video_requests.registered` by `origin` to see when council portals are busiest.
* **Build KPI pages** — combine counters into a *Video Fulfilment* dashboard that shows *average time‑to‑ready* and
  *failure ratio* at a glance.

<aside>
👉🏻 Create a bar chart comparing `videos.expired.deleted` to total storage used; sudden drops in deletions can warn that retention logic is mis‑configured.
</aside>

## Getting Started – Checklist

1. List key business events (request registered, job queued, video ready, notification sent, retention processed).
2. Sketch the metric instruments and tags you need.
3. Add OpenTelemetry packages and register a `Meter` with those instruments.
4. Increment/observe at the appropriate service boundaries.
5. Export to OTLP and explore in Aspire.
6. Review with support and council stakeholders, then iterate.

## Final Thoughts

Custom metrics elevate observability beyond technical health checks by embedding business context directly into the
monitoring fabric. In our garbage‑truck footage service that means knowing *how many videos are late*, *which users/services
generate the most requests*, and *how long watermarking really takes*. Armed with these insights, engineering and
operations teams can act proactively — long before an irate support call lands.

OpenTelemetry plus the .NET Aspire Dashboard delivers a vendor‑neutral, future‑proof tool‑chain for surfacing those
insights in .NET 9 and beyond. Start small, instrument the events that matter, and let the data guide your next
optimisation.

## NOTE
> .NET Aspire is designed primarily for local development, offering a streamlined dashboard for viewing logs,
traces, and metrics with minimal configuration. However, its dashboard is generally not intended for production use due
to limitations such as in-memory event storage and lack of persistent storage for telemetry data. For robust production
observability, several established alternatives are commonly used.
> 
>Leading Production-Grade Alternatives
> - Grafana + Prometheus + Loki + Jaeger
>     - Grafana provides powerful, customizable dashboards for metrics, logs, and traces.
>     - Prometheus is widely used for metrics collection.
>     - Loki handles log aggregation.
>     - Jaeger (or Zipkin) provides distributed tracing.
>     - This stack is a standard in production environments and integrates well with OpenTelemetry, which Aspire also supports.
> - ELK Stack (Elasticsearch, Logstash, Kibana)
>     - ELK is a popular choice for log aggregation, search, and visualization.
>     - Kibana offers advanced dashboarding capabilities for production monitoring.
> - Seq
>     - Seq is a log server with support for structured logging and, more recently, traces.
>     - It is used for centralized log management in .NET environments.
> - Cloud-hosted Services
>     - Azure Monitor, AWS CloudWatch, and Google Cloud Operations Suite offer managed observability solutions with dashboards,
alerting, and storage.
>     - These services are often recommended for production deployments, especially when hosting in the cloud.

| Tool/Stack | Metrics | Logs | Traces | Dashboard | Production Ready | Storage | Cloud Integration |
|---|---|---|---|---|---|---|---|
| Grafana + Prometheus etc. | Yes | Yes | Yes | Yes | Yes | Yes | Yes |
| ELK Stack | No | Yes | No | Yes | Yes | Yes | Yes |
| Seq | No | Yes | Yes | Yes | Yes | Yes | Limited |
| .NET Aspire Dashboard | Yes | Yes | Yes | Yes | Limited | No (in-memory) |    Azure (best) |