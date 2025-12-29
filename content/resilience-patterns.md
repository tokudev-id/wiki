---
title: Pola Ketahanan
description: Panduan implementasi pola-pola ketahanan (resilience patterns) untuk membangun aplikasi .NET yang tahan terhadap kegagalan menggunakan Polly v8.
published: true
date: 2025-12-29T00:00:00.000Z
tags: resilience, polly, circuit-breaker
editor: markdown
dateCreated: 2025-12-29T00:00:00.000Z
---

# Pola Ketahanan (Resilience Patterns)

Dokumen ini menjelaskan pola-pola ketahanan untuk membangun aplikasi .NET yang tahan terhadap kegagalan.

## Mengapa Resilience Penting?

Dalam sistem terdistribusi, kegagalan adalah hal yang pasti terjadi:
- Network timeout
- Service tidak tersedia
- Database overload
- External API down

Aplikasi yang baik harus bisa menangani kegagalan dengan graceful, bukan crash.

## Polly v8 - Library Resilience untuk .NET

### Instalasi

```bash
dotnet add package Microsoft.Extensions.Http.Resilience
dotnet add package Polly.Extensions
```

### Resilience Pipeline

Di Polly v8, strategi resilience dikonfigurasi sebagai pipeline:

```csharp
// Program.cs
builder.Services.AddResiliencePipeline("default", builder =>
{
    builder
        .AddRetry(new RetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            Delay = TimeSpan.FromSeconds(1),
            BackoffType = DelayBackoffType.Exponential
        })
        .AddCircuitBreaker(new CircuitBreakerStrategyOptions
        {
            FailureRatio = 0.5,
            SamplingDuration = TimeSpan.FromSeconds(30),
            MinimumThroughput = 10,
            BreakDuration = TimeSpan.FromSeconds(30)
        })
        .AddTimeout(TimeSpan.FromSeconds(10));
});
```

## Pola-pola Resilience

### 1. Retry Pattern

Mencoba ulang operasi yang gagal secara otomatis.

```csharp
// Konfigurasi retry dengan exponential backoff
builder.Services.AddResiliencePipeline("retry-pipeline", builder =>
{
    builder.AddRetry(new RetryStrategyOptions
    {
        // Jumlah maksimum retry
        MaxRetryAttempts = 3,

        // Delay awal
        Delay = TimeSpan.FromMilliseconds(500),

        // Tipe backoff: Constant, Linear, Exponential
        BackoffType = DelayBackoffType.Exponential,

        // Jitter untuk menghindari thundering herd
        UseJitter = true,

        // Retry hanya untuk exception tertentu
        ShouldHandle = new PredicateBuilder()
            .Handle<HttpRequestException>()
            .Handle<TimeoutException>()
            .HandleResult<HttpResponseMessage>(r =>
                r.StatusCode == HttpStatusCode.ServiceUnavailable ||
                r.StatusCode == HttpStatusCode.TooManyRequests),

        // Callback saat retry
        OnRetry = args =>
        {
            Console.WriteLine($"Retry {args.AttemptNumber} setelah {args.RetryDelay.TotalSeconds}s");
            return ValueTask.CompletedTask;
        }
    });
});

// Penggunaan
public class ExternalApiService
{
    private readonly ResiliencePipeline _pipeline;
    private readonly HttpClient _httpClient;

    public ExternalApiService(
        ResiliencePipelineProvider<string> pipelineProvider,
        HttpClient httpClient)
    {
        _pipeline = pipelineProvider.GetPipeline("retry-pipeline");
        _httpClient = httpClient;
    }

    public async Task<string> GetDataAsync()
    {
        return await _pipeline.ExecuteAsync(async ct =>
        {
            var response = await _httpClient.GetAsync("https://api.example.com/data", ct);
            response.EnsureSuccessStatusCode();
            return await response.Content.ReadAsStringAsync(ct);
        });
    }
}
```

### 2. Circuit Breaker Pattern

Mencegah aplikasi terus mencoba operasi yang kemungkinan akan gagal.

```
          ┌──────────┐
          │  CLOSED  │ <- Normal, request mengalir
          └────┬─────┘
               │ Failure threshold exceeded
               v
          ┌──────────┐
          │   OPEN   │ <- Fail fast, reject requests
          └────┬─────┘
               │ Break duration elapsed
               v
          ┌──────────┐
          │HALF-OPEN │ <- Test dengan limited requests
          └────┬─────┘
               │
       ┌───────┴───────┐
       │               │
    Success          Failure
       │               │
       v               v
   ┌──────────┐   ┌──────────┐
   │  CLOSED  │   │   OPEN   │
   └──────────┘   └──────────┘
```

```csharp
builder.Services.AddResiliencePipeline("circuit-breaker-pipeline", builder =>
{
    builder.AddCircuitBreaker(new CircuitBreakerStrategyOptions
    {
        // Buka circuit jika 50% request gagal
        FailureRatio = 0.5,

        // Dalam window 30 detik
        SamplingDuration = TimeSpan.FromSeconds(30),

        // Minimal 10 request sebelum evaluasi
        MinimumThroughput = 10,

        // Durasi circuit terbuka
        BreakDuration = TimeSpan.FromSeconds(30),

        // Exception yang dianggap failure
        ShouldHandle = new PredicateBuilder()
            .Handle<HttpRequestException>()
            .Handle<TimeoutException>(),

        // Callback saat state berubah
        OnOpened = args =>
        {
            Console.WriteLine($"Circuit OPENED! Break duration: {args.BreakDuration}");
            // Kirim alert ke monitoring
            return ValueTask.CompletedTask;
        },
        OnClosed = args =>
        {
            Console.WriteLine("Circuit CLOSED. Service recovered.");
            return ValueTask.CompletedTask;
        },
        OnHalfOpened = args =>
        {
            Console.WriteLine("Circuit HALF-OPEN. Testing...");
            return ValueTask.CompletedTask;
        }
    });
});
```

### 3. Timeout Pattern

Membatasi waktu tunggu operasi.

```csharp
builder.Services.AddResiliencePipeline("timeout-pipeline", builder =>
{
    // Timeout per attempt
    builder.AddTimeout(new TimeoutStrategyOptions
    {
        Timeout = TimeSpan.FromSeconds(10),

        OnTimeout = args =>
        {
            Console.WriteLine($"Timeout setelah {args.Timeout.TotalSeconds}s");
            return ValueTask.CompletedTask;
        }
    });
});

// Kombinasi dengan retry (timeout per attempt + total timeout)
builder.Services.AddResiliencePipeline("combined-timeout", builder =>
{
    // Total timeout untuk seluruh operasi (termasuk retry)
    builder.AddTimeout(TimeSpan.FromSeconds(60));

    // Retry
    builder.AddRetry(new RetryStrategyOptions
    {
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromSeconds(2)
    });

    // Timeout per attempt
    builder.AddTimeout(TimeSpan.FromSeconds(10));
});
```

### 4. Bulkhead Pattern

Membatasi jumlah concurrent calls untuk mencegah resource exhaustion.

```csharp
builder.Services.AddResiliencePipeline("bulkhead-pipeline", builder =>
{
    builder.AddConcurrencyLimiter(new ConcurrencyLimiterOptions
    {
        // Maksimum concurrent executions
        PermitLimit = 10,

        // Queue untuk request yang menunggu
        QueueLimit = 20,

        // Timeout menunggu di queue
        QueueProcessingOrder = QueueProcessingOrder.OldestFirst
    });
});

// Penggunaan untuk isolasi resource
public class PaymentService
{
    private readonly ResiliencePipeline _bulkhead;

    public async Task<PaymentResult> ProcessPaymentAsync(Payment payment)
    {
        // Hanya 10 payment yang diproses bersamaan
        // Sisanya menunggu di queue (max 20)
        // Lebih dari itu akan di-reject
        return await _bulkhead.ExecuteAsync(async ct =>
        {
            return await _paymentGateway.ProcessAsync(payment, ct);
        });
    }
}
```

### 5. Fallback Pattern

Menyediakan alternatif ketika operasi utama gagal.

```csharp
builder.Services.AddResiliencePipeline<string>("fallback-pipeline", builder =>
{
    builder.AddFallback(new FallbackStrategyOptions<string>
    {
        ShouldHandle = new PredicateBuilder<string>()
            .Handle<HttpRequestException>()
            .HandleResult(r => string.IsNullOrEmpty(r)),

        FallbackAction = args =>
        {
            // Return cached/default value
            return Outcome.FromResultAsValueTask("Default Value");
        },

        OnFallback = args =>
        {
            Console.WriteLine($"Fallback digunakan karena: {args.Outcome.Exception?.Message}");
            return ValueTask.CompletedTask;
        }
    });
});

// Contoh dengan cached fallback
public class ProductService
{
    private readonly IDistributedCache _cache;
    private readonly ResiliencePipeline<Product?> _pipeline;

    public async Task<Product?> GetProductAsync(int id)
    {
        return await _pipeline.ExecuteAsync(async ct =>
        {
            try
            {
                // Coba ambil dari API
                return await _externalApi.GetProductAsync(id, ct);
            }
            catch
            {
                // Fallback ke cache
                var cached = await _cache.GetStringAsync($"product:{id}", ct);
                return cached != null
                    ? JsonSerializer.Deserialize<Product>(cached)
                    : null;
            }
        });
    }
}
```

### 6. Rate Limiter

Membatasi jumlah request dalam periode waktu tertentu.

```csharp
builder.Services.AddResiliencePipeline("rate-limiter-pipeline", builder =>
{
    builder.AddRateLimiter(new RateLimiterStrategyOptions
    {
        DefaultRateLimiterOptions = new FixedWindowRateLimiterOptions
        {
            // 100 request per menit
            PermitLimit = 100,
            Window = TimeSpan.FromMinutes(1),
            QueueLimit = 10,
            QueueProcessingOrder = QueueProcessingOrder.OldestFirst
        },

        OnRejected = args =>
        {
            Console.WriteLine("Rate limit exceeded!");
            return ValueTask.CompletedTask;
        }
    });
});

// Sliding window rate limiter
builder.Services.AddRateLimiter(options =>
{
    options.AddSlidingWindowLimiter("api-limit", config =>
    {
        config.PermitLimit = 100;
        config.Window = TimeSpan.FromMinutes(1);
        config.SegmentsPerWindow = 6;  // 6 segments of 10 seconds
        config.QueueLimit = 10;
    });
});
```

## Kombinasi Strategi (Resilience Pipeline)

### Urutan yang Direkomendasikan

```csharp
builder.Services.AddResiliencePipeline("production-pipeline", builder =>
{
    // 1. Rate Limiter - Proteksi dari overload
    builder.AddRateLimiter(new RateLimiterStrategyOptions
    {
        DefaultRateLimiterOptions = new FixedWindowRateLimiterOptions
        {
            PermitLimit = 1000,
            Window = TimeSpan.FromMinutes(1)
        }
    });

    // 2. Total Timeout - Batas waktu keseluruhan
    builder.AddTimeout(new TimeoutStrategyOptions
    {
        Timeout = TimeSpan.FromMinutes(2),
        Name = "TotalTimeout"
    });

    // 3. Retry - Coba ulang untuk transient failures
    builder.AddRetry(new RetryStrategyOptions
    {
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromSeconds(1),
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = true
    });

    // 4. Circuit Breaker - Cegah cascade failure
    builder.AddCircuitBreaker(new CircuitBreakerStrategyOptions
    {
        FailureRatio = 0.5,
        SamplingDuration = TimeSpan.FromSeconds(30),
        MinimumThroughput = 10,
        BreakDuration = TimeSpan.FromSeconds(30)
    });

    // 5. Bulkhead - Isolasi resource
    builder.AddConcurrencyLimiter(new ConcurrencyLimiterOptions
    {
        PermitLimit = 25,
        QueueLimit = 50
    });

    // 6. Per-attempt Timeout
    builder.AddTimeout(new TimeoutStrategyOptions
    {
        Timeout = TimeSpan.FromSeconds(10),
        Name = "AttemptTimeout"
    });
});
```

## HttpClient dengan Resilience

```csharp
// Program.cs
builder.Services
    .AddHttpClient<IPaymentGatewayClient, PaymentGatewayClient>(client =>
    {
        client.BaseAddress = new Uri("https://api.payment.com");
        client.Timeout = TimeSpan.FromMinutes(2);
    })
    .AddStandardResilienceHandler(options =>
    {
        // Konfigurasi built-in resilience
        options.Retry.MaxRetryAttempts = 3;
        options.Retry.Delay = TimeSpan.FromSeconds(1);

        options.CircuitBreaker.FailureRatio = 0.5;
        options.CircuitBreaker.SamplingDuration = TimeSpan.FromSeconds(30);
        options.CircuitBreaker.BreakDuration = TimeSpan.FromSeconds(30);

        options.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(60);
        options.AttemptTimeout.Timeout = TimeSpan.FromSeconds(10);
    });

// Atau dengan custom pipeline
builder.Services
    .AddHttpClient<IExternalApiClient, ExternalApiClient>()
    .AddResilienceHandler("custom", builder =>
    {
        builder
            .AddRetry(new HttpRetryStrategyOptions
            {
                MaxRetryAttempts = 3,
                BackoffType = DelayBackoffType.Exponential
            })
            .AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
            {
                BreakDuration = TimeSpan.FromSeconds(30)
            })
            .AddTimeout(TimeSpan.FromSeconds(10));
    });
```

## Praktik Terbaik

### 1. Selalu Set Timeout

```csharp
// Jangan pernah biarkan operasi berjalan tanpa batas waktu
builder.AddTimeout(TimeSpan.FromSeconds(30));
```

### 2. Gunakan Jitter untuk Retry

```csharp
// Mencegah thundering herd
builder.AddRetry(new RetryStrategyOptions
{
    UseJitter = true,
    BackoffType = DelayBackoffType.Exponential
});
```

### 3. Monitor Circuit Breaker State

```csharp
// Log dan alert saat circuit opens
OnOpened = args =>
{
    _logger.LogWarning("Circuit opened for {Duration}", args.BreakDuration);
    _alertService.SendAlert("Circuit breaker opened!");
    return ValueTask.CompletedTask;
}
```

### 4. Test Resilience

```csharp
[Fact]
public async Task Service_ShouldRetryOnTransientFailure()
{
    // Arrange
    var attempts = 0;
    var mockHandler = new MockHttpMessageHandler((request, ct) =>
    {
        attempts++;
        if (attempts < 3)
            throw new HttpRequestException("Transient failure");
        return new HttpResponseMessage(HttpStatusCode.OK);
    });

    // Act
    var result = await _service.GetDataAsync();

    // Assert
    Assert.Equal(3, attempts);
    Assert.NotNull(result);
}
```

## Referensi

- [Polly Documentation](https://www.pollydocs.org/)
- [Microsoft.Extensions.Http.Resilience](https://learn.microsoft.com/en-us/dotnet/core/resilience/)
- [Cloud Design Patterns](https://learn.microsoft.com/en-us/azure/architecture/patterns/)
- [Release It! by Michael Nygard](https://pragprog.com/titles/mnee2/release-it-second-edition/)

---

*Terakhir diperbarui: Desember 2025*
