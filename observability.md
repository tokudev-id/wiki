---
title: Observability
description: Panduan implementasi observability di aplikasi .NET menggunakan OpenTelemetry, mencakup distributed tracing, metrics, logging, dan monitoring stack dengan Jaeger, Prometheus, dan Grafana.
published: true
date: 2025-12-29T00:00:00.000Z
tags: observability, opentelemetry, monitoring
editor: markdown
dateCreated: 2025-12-29T00:00:00.000Z
---

# Observability

Dokumen ini menjelaskan cara mengimplementasikan observability di aplikasi .NET menggunakan OpenTelemetry dan tools pendukung.

## Tiga Pilar Observability

```
┌─────────────────────────────────────────────────────────────┐
│                      OBSERVABILITY                          │
├───────────────────┬───────────────────┬────────────────────┤
│      LOGS         │     METRICS       │      TRACES        │
│                   │                   │                    │
│ • What happened   │ • Aggregated data │ • Request flow     │
│ • Event details   │ • Trends          │ • Latency          │
│ • Errors          │ • Alerting        │ • Dependencies     │
└───────────────────┴───────────────────┴────────────────────┘
```

| Pilar | Fungsi | Contoh |
|-------|--------|--------|
| **Logs** | Catatan kejadian diskret | Error messages, audit trail |
| **Metrics** | Data numerik agregat | Request count, latency, CPU usage |
| **Traces** | Alur request antar service | Distributed tracing |

## OpenTelemetry Setup

### Instalasi Packages

```bash
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
dotnet add package OpenTelemetry.Instrumentation.Http
dotnet add package OpenTelemetry.Instrumentation.SqlClient
dotnet add package OpenTelemetry.Instrumentation.EntityFrameworkCore
dotnet add package OpenTelemetry.Exporter.OpenTelemetryProtocol
dotnet add package OpenTelemetry.Exporter.Console
```

### Konfigurasi Dasar

```csharp
// Program.cs
using OpenTelemetry.Logs;
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

var builder = WebApplication.CreateBuilder(args);

// Resource information
var resourceBuilder = ResourceBuilder.CreateDefault()
    .AddService(
        serviceName: "MyApi",
        serviceVersion: "1.0.0",
        serviceInstanceId: Environment.MachineName)
    .AddAttributes(new Dictionary<string, object>
    {
        ["deployment.environment"] = builder.Environment.EnvironmentName
    });

// Configure OpenTelemetry
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService("MyApi"))
    .WithTracing(tracing =>
    {
        tracing
            .SetResourceBuilder(resourceBuilder)
            .AddAspNetCoreInstrumentation(options =>
            {
                options.RecordException = true;
                options.Filter = httpContext =>
                {
                    // Jangan trace health checks
                    return !httpContext.Request.Path.StartsWithSegments("/health");
                };
            })
            .AddHttpClientInstrumentation()
            .AddSqlClientInstrumentation(options =>
            {
                options.SetDbStatementForText = true;
                options.RecordException = true;
            })
            .AddEntityFrameworkCoreInstrumentation()
            .AddSource("MyApi.CustomActivities")
            .AddOtlpExporter(options =>
            {
                options.Endpoint = new Uri("http://localhost:4317");
            });

        if (builder.Environment.IsDevelopment())
        {
            tracing.AddConsoleExporter();
        }
    })
    .WithMetrics(metrics =>
    {
        metrics
            .SetResourceBuilder(resourceBuilder)
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddRuntimeInstrumentation()
            .AddProcessInstrumentation()
            .AddMeter("MyApi.CustomMetrics")
            .AddOtlpExporter(options =>
            {
                options.Endpoint = new Uri("http://localhost:4317");
            });

        if (builder.Environment.IsDevelopment())
        {
            metrics.AddConsoleExporter();
        }
    });

// Logging dengan OpenTelemetry
builder.Logging.AddOpenTelemetry(options =>
{
    options.SetResourceBuilder(resourceBuilder);
    options.IncludeFormattedMessage = true;
    options.IncludeScopes = true;
    options.AddOtlpExporter(otlp =>
    {
        otlp.Endpoint = new Uri("http://localhost:4317");
    });
});
```

## Distributed Tracing

### Custom Tracing

```csharp
using System.Diagnostics;

public class PesananService : IPesananService
{
    private static readonly ActivitySource ActivitySource = new("MyApi.PesananService");
    private readonly ILogger<PesananService> _logger;

    public async Task<PesananDto> BuatPesananAsync(BuatPesananRequest request)
    {
        // Buat span untuk operasi ini
        using var activity = ActivitySource.StartActivity("BuatPesanan");

        // Tambahkan attributes (tags)
        activity?.SetTag("pelanggan.id", request.PelangganId);
        activity?.SetTag("items.count", request.Items.Count);

        try
        {
            // Validasi
            using (var validationActivity = ActivitySource.StartActivity("ValidasiPesanan"))
            {
                await ValidasiAsync(request);
            }

            // Proses items
            using (var itemsActivity = ActivitySource.StartActivity("ProsesItems"))
            {
                itemsActivity?.SetTag("items.total", request.Items.Count);

                foreach (var item in request.Items)
                {
                    await ProsesItemAsync(item);
                }
            }

            // Simpan ke database
            using (var dbActivity = ActivitySource.StartActivity("SimpanKeDatabase"))
            {
                var pesanan = await _repository.SimpanAsync(request);
                activity?.SetTag("pesanan.id", pesanan.Id);
                activity?.SetTag("pesanan.total", pesanan.Total);

                return pesanan;
            }
        }
        catch (Exception ex)
        {
            // Rekam exception di span
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            activity?.RecordException(ex);
            throw;
        }
    }
}

// Registrasi ActivitySource
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing.AddSource("MyApi.PesananService");
    });
```

### Propagasi Context

```csharp
// Untuk HttpClient, context otomatis dipropagasi
// Untuk messaging (RabbitMQ, Kafka), perlu manual:

public class MessagePublisher
{
    public async Task PublishAsync<T>(string queue, T message)
    {
        var activity = Activity.Current;

        var headers = new Dictionary<string, string>();

        // Inject trace context ke headers
        if (activity != null)
        {
            headers["traceparent"] = $"00-{activity.TraceId}-{activity.SpanId}-{(activity.Recorded ? "01" : "00")}";
            if (!string.IsNullOrEmpty(activity.TraceStateString))
            {
                headers["tracestate"] = activity.TraceStateString;
            }
        }

        // Publish dengan headers
        await _messageBus.PublishAsync(queue, message, headers);
    }
}

// Consumer
public class MessageConsumer
{
    public async Task ConsumeAsync(Message message)
    {
        // Extract trace context dari headers
        var traceparent = message.Headers.GetValueOrDefault("traceparent");

        using var activity = ActivitySource.StartActivity(
            "ProcessMessage",
            ActivityKind.Consumer,
            traceparent);

        activity?.SetTag("message.type", message.Type);

        await ProcessMessageAsync(message);
    }
}
```

## Metrics

### Custom Metrics

```csharp
using System.Diagnostics.Metrics;

public class PesananMetrics
{
    private readonly Counter<long> _pesananDibuat;
    private readonly Counter<long> _pesananGagal;
    private readonly Histogram<double> _waktuPemrosesan;
    private readonly UpDownCounter<int> _pesananAntrian;

    public PesananMetrics(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("MyApi.Pesanan");

        _pesananDibuat = meter.CreateCounter<long>(
            "pesanan.dibuat",
            unit: "{pesanan}",
            description: "Jumlah pesanan yang berhasil dibuat");

        _pesananGagal = meter.CreateCounter<long>(
            "pesanan.gagal",
            unit: "{pesanan}",
            description: "Jumlah pesanan yang gagal");

        _waktuPemrosesan = meter.CreateHistogram<double>(
            "pesanan.waktu_proses",
            unit: "ms",
            description: "Waktu pemrosesan pesanan");

        _pesananAntrian = meter.CreateUpDownCounter<int>(
            "pesanan.antrian",
            unit: "{pesanan}",
            description: "Jumlah pesanan dalam antrian");
    }

    public void PesananBerhasil(string kategori)
    {
        _pesananDibuat.Add(1, new KeyValuePair<string, object?>("kategori", kategori));
    }

    public void PesananGagal(string kategori, string alasan)
    {
        _pesananGagal.Add(1,
            new KeyValuePair<string, object?>("kategori", kategori),
            new KeyValuePair<string, object?>("alasan", alasan));
    }

    public void CatatWaktuProses(double milliseconds)
    {
        _waktuPemrosesan.Record(milliseconds);
    }

    public void TambahAntrian() => _pesananAntrian.Add(1);
    public void KurangiAntrian() => _pesananAntrian.Add(-1);
}

// Registrasi
builder.Services.AddSingleton<PesananMetrics>();
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics =>
    {
        metrics.AddMeter("MyApi.Pesanan");
    });

// Penggunaan
public class PesananService
{
    private readonly PesananMetrics _metrics;

    public async Task<Pesanan> BuatPesananAsync(Request request)
    {
        var sw = Stopwatch.StartNew();
        _metrics.TambahAntrian();

        try
        {
            var pesanan = await ProsesAsync(request);
            _metrics.PesananBerhasil(pesanan.Kategori);
            return pesanan;
        }
        catch (Exception ex)
        {
            _metrics.PesananGagal(request.Kategori, ex.GetType().Name);
            throw;
        }
        finally
        {
            sw.Stop();
            _metrics.CatatWaktuProses(sw.Elapsed.TotalMilliseconds);
            _metrics.KurangiAntrian();
        }
    }
}
```

### Metrics Endpoint untuk Prometheus

```csharp
// Install package
// dotnet add package OpenTelemetry.Exporter.Prometheus.AspNetCore

builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics =>
    {
        metrics.AddPrometheusExporter();
    });

var app = builder.Build();
app.MapPrometheusScrapingEndpoint();  // Endpoint: /metrics
```

## Logging Best Practices

### Structured Logging dengan Correlation

```csharp
// Middleware untuk menambahkan CorrelationId
public class CorrelationIdMiddleware
{
    private readonly RequestDelegate _next;

    public async Task InvokeAsync(HttpContext context)
    {
        var correlationId = context.Request.Headers["X-Correlation-ID"].FirstOrDefault()
            ?? Activity.Current?.TraceId.ToString()
            ?? Guid.NewGuid().ToString();

        context.Response.Headers["X-Correlation-ID"] = correlationId;

        using (LogContext.PushProperty("CorrelationId", correlationId))
        {
            await _next(context);
        }
    }
}

// Log dengan context
public class PesananService
{
    private readonly ILogger<PesananService> _logger;

    public async Task<Pesanan> BuatPesananAsync(Request request)
    {
        using (_logger.BeginScope(new Dictionary<string, object>
        {
            ["PelangganId"] = request.PelangganId,
            ["RequestId"] = Activity.Current?.SpanId.ToString()
        }))
        {
            _logger.LogInformation("Memulai pembuatan pesanan untuk pelanggan {PelangganId}", request.PelangganId);

            var pesanan = await ProsesAsync(request);

            _logger.LogInformation(
                "Pesanan berhasil dibuat: {PesananId}, Total: {Total}",
                pesanan.Id,
                pesanan.Total);

            return pesanan;
        }
    }
}
```

## Observability Stack

### Docker Compose untuk Development

```yaml
# docker-compose.observability.yml
version: "3.8"

services:
  # OpenTelemetry Collector
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "8889:8889"   # Prometheus metrics
    networks:
      - observability

  # Jaeger untuk Tracing
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # UI
      - "14250:14250"  # gRPC
    environment:
      - COLLECTOR_OTLP_ENABLED=true
    networks:
      - observability

  # Prometheus untuk Metrics
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    networks:
      - observability

  # Grafana untuk Dashboards
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana
    networks:
      - observability

  # Loki untuk Logs
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    networks:
      - observability

volumes:
  grafana-data:

networks:
  observability:
    driver: bridge
```

### OpenTelemetry Collector Config

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024

exporters:
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true

  prometheus:
    endpoint: "0.0.0.0:8889"

  loki:
    endpoint: http://loki:3100/loki/api/v1/push

  logging:
    verbosity: detailed

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [jaeger, logging]

    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]

    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [loki]
```

## .NET Aspire Dashboard (Alternatif Simple)

```csharp
// Install package
// dotnet add package Aspire.Dashboard

// Program.cs
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing.AddOtlpExporter(options =>
        {
            options.Endpoint = new Uri("http://localhost:18889");
        });
    })
    .WithMetrics(metrics =>
    {
        metrics.AddOtlpExporter(options =>
        {
            options.Endpoint = new Uri("http://localhost:18889");
        });
    });

// Jalankan Aspire Dashboard standalone
// docker run -p 18888:18888 -p 18889:18889 mcr.microsoft.com/dotnet/aspire-dashboard:latest
// Akses: http://localhost:18888
```

## Health Checks

```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy())
    .AddDbContextCheck<ApplicationDbContext>("database")
    .AddRedis(builder.Configuration.GetConnectionString("Redis")!, "redis")
    .AddUrlGroup(new Uri("https://api.external.com/health"), "external-api");

var app = builder.Build();

// Endpoint untuk liveness probe
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false  // Hanya cek app running
});

// Endpoint untuk readiness probe
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});

// Endpoint detail
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = async (context, report) =>
    {
        context.Response.ContentType = "application/json";

        var result = new
        {
            status = report.Status.ToString(),
            duration = report.TotalDuration.TotalMilliseconds,
            checks = report.Entries.Select(e => new
            {
                name = e.Key,
                status = e.Value.Status.ToString(),
                duration = e.Value.Duration.TotalMilliseconds,
                description = e.Value.Description,
                error = e.Value.Exception?.Message
            })
        };

        await context.Response.WriteAsJsonAsync(result);
    }
});
```

## Alerting

### Contoh Alert Rules (Prometheus)

```yaml
# alerts.yml
groups:
  - name: api-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_requests_total{status=~"5.."}[5m]))
          /
          sum(rate(http_server_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Error rate tinggi (> 5%)"
          description: "Error rate: {{ $value | humanizePercentage }}"

      - alert: HighLatency
        expr: |
          histogram_quantile(0.95, rate(http_server_request_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Latency P95 > 1 detik"

      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.instance }} down"
```

## Checklist Observability

- [ ] OpenTelemetry terkonfigurasi untuk traces, metrics, logs
- [ ] Correlation ID ada di setiap request
- [ ] Custom metrics untuk business operations
- [ ] Health check endpoints tersedia
- [ ] Dashboards untuk monitoring key metrics
- [ ] Alerts terkonfigurasi untuk critical issues
- [ ] Distributed tracing untuk cross-service calls
- [ ] Logs terstruktur dengan konteks yang cukup

## Referensi

- [OpenTelemetry .NET](https://opentelemetry.io/docs/languages/dotnet/)
- [.NET Observability with OpenTelemetry](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/observability-with-otel)
- [Grafana Documentation](https://grafana.com/docs/)
- [Prometheus Documentation](https://prometheus.io/docs/)

---

*Terakhir diperbarui: Desember 2025*
