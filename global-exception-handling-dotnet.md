# Global Exception Handling in ASP.NET Core - Technical Documentation

**Version:** .NET 10  
**Last Updated:** January 2026  
**Author:** Principal .NET Architect

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Exception Handling Overview](#exception-handling-overview)
3. [Architectural Approaches](#architectural-approaches)
4. [Implementation Guide](#implementation-guide)
5. [Advanced Patterns](#advanced-patterns)
6. [Production Best Practices](#production-best-practices)
7. [Performance Considerations](#performance-considerations)
8. [Appendix](#appendix)

---

## Executive Summary

### Quick Decision Matrix

| Approach | Use Case | .NET Version | Recommendation |
|----------|----------|--------------|----------------|
| Try-Catch Blocks | Localized error recovery only | All | ❌ Not for global handling |
| UseExceptionHandler Lambda | Quick prototypes, simple APIs | All | ⚠️ Limited use |
| Custom Middleware | Legacy projects | .NET 7 and earlier | ⚠️ Migrate when possible |
| **IExceptionHandler** | **Production applications** | **.NET 8+** | ✅ **Recommended** |

### Key Takeaways

- **IExceptionHandler** is the modern, recommended approach for .NET 8+
- Provides clean separation of concerns with testable, chainable handlers
- .NET 10 introduces `SuppressDiagnosticsCallback` for better log control
- Use RFC 9457 Problem Details for standardized error responses
- Consider Result Pattern for expected failures to improve performance

---

## Exception Handling Overview

### What Are Exceptions?

Exceptions in .NET are objects inheriting from `System.Exception` that represent runtime errors. They propagate up the call stack until caught.

### Common Exception Types

| Exception Type | Description | Typical Scenario |
|----------------|-------------|------------------|
| `NullReferenceException` | Object reference not set | Accessing null object |
| `ArgumentNullException` | Required argument is null | Method parameter validation |
| `InvalidOperationException` | Operation invalid for current state | State machine violations |
| `KeyNotFoundException` | Dictionary lookup failed | Missing key in collection |
| `HttpRequestException` | HTTP call failed | External API communication |

### Exception Handling Goals

1. **Catch** unhandled exceptions before reaching the client
2. **Log** exceptions with appropriate context
3. **Return** consistent, safe error responses
4. **Protect** sensitive information from exposure

---

## Architectural Approaches

### 1. Try-Catch Blocks (❌ Not Recommended for Global Handling)

**Description:** Manual exception handling in each endpoint.

**Code Example:**

```csharp
[HttpGet("{id}")]
public IActionResult GetProduct(int id)
{
    try
    {
        var product = _productService.GetById(id);
        return Ok(product);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Error fetching product {ProductId}", id);
        return StatusCode(500, "An error occurred");
    }
}
```

**Pros:**
- Simple to understand
- Fine-grained control per endpoint

**Cons:**
- ❌ Code duplication across endpoints
- ❌ Inconsistent error handling
- ❌ Difficult to maintain
- ❌ No centralized logging

**Verdict:** Use only for specific, localized error recovery scenarios.

---

### 2. Built-in UseExceptionHandler Middleware

**Description:** ASP.NET Core's built-in exception handling middleware.

**Code Example:**

```csharp
app.UseExceptionHandler(options =>
{
    options.Run(async context =>
    {
        context.Response.StatusCode = StatusCodes.Status500InternalServerError;
        context.Response.ContentType = "application/json";

        var exceptionFeature = context.Features.Get<IExceptionHandlerFeature>();
        if (exceptionFeature is not null)
        {
            var error = new { message = "An unexpected error occurred" };
            await context.Response.WriteAsJsonAsync(error);
        }
    });
});
```

**How It Works:**

| Line | Description |
|------|-------------|
| 1 | Adds exception handling middleware to pipeline |
| 3 | Registers terminal middleware (doesn't call next) |
| 5-6 | Sets HTTP status and content type |
| 8 | Retrieves exception details from feature |
| 11-12 | Serializes error response as JSON |

**Pros:**
- ✅ Centralized error handling
- ✅ Built into ASP.NET Core

**Cons:**
- ❌ Inline code in Program.cs
- ❌ Difficult to test
- ❌ Limited type-specific handling

---

### 3. Custom Middleware (Pre-.NET 8)

**Description:** Custom middleware class for exception handling.

**Code Example:**

```csharp
public class ExceptionHandlingMiddleware(
    RequestDelegate next, 
    ILogger<ExceptionHandlingMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await next(context);
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Unhandled exception occurred");
            await HandleExceptionAsync(context, ex);
        }
    }

    private static async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        context.Response.ContentType = "application/json";
        context.Response.StatusCode = exception switch
        {
            KeyNotFoundException => StatusCodes.Status404NotFound,
            ArgumentException => StatusCodes.Status400BadRequest,
            _ => StatusCodes.Status500InternalServerError
        };

        var problemDetails = new ProblemDetails
        {
            Status = context.Response.StatusCode,
            Title = "An error occurred",
            Detail = exception.Message
        };

        await context.Response.WriteAsJsonAsync(problemDetails);
    }
}
```

**Registration:**

```csharp
app.UseMiddleware<ExceptionHandlingMiddleware>();
```

**Architecture Breakdown:**

| Component | Purpose |
|-----------|---------|
| Primary Constructor | Injects dependencies (RequestDelegate, ILogger) |
| Try Block | Wraps next middleware call |
| Switch Expression | Maps exception types to HTTP status codes |
| ProblemDetails | RFC 7807 standard error response |

**Pros:**
- ✅ Clean separation of concerns
- ✅ Testable
- ✅ Type-specific handling

**Cons:**
- ❌ More boilerplate than IExceptionHandler
- ❌ Manual integration with framework

---

### 4. IExceptionHandler (✅ Recommended - .NET 8+)

**Description:** Modern, framework-integrated exception handling interface.

**Why It's Superior:**

| Feature | Benefit |
|---------|---------|
| **Dependency Injection** | Handlers registered in DI container |
| **Testable** | Easy unit testing without HTTP pipeline |
| **Chainable** | Multiple handlers for different exception types |
| **Framework-Integrated** | Leverages Microsoft's maintained middleware |

---

## Implementation Guide

### Step 1: Create Custom Exception Classes

#### Base Exception Class

```csharp
public abstract class AppException : Exception
{
    public HttpStatusCode StatusCode { get; }

    protected AppException(
        string message, 
        HttpStatusCode statusCode = HttpStatusCode.InternalServerError)
        : base(message)
    {
        StatusCode = statusCode;
    }
}
```

**Design Rationale:**

| Design Choice | Reason |
|---------------|--------|
| `abstract` class | Forces creation of specific exception types |
| `HttpStatusCode` property | Exception knows its HTTP response code |
| Default status 500 | Safe default for unexpected errors |

#### Specific Exception Classes

**NotFoundException:**

```csharp
public sealed class NotFoundException : AppException
{
    public NotFoundException(string resourceName, object key)
        : base(
            $"{resourceName} with identifier '{key}' was not found.", 
            HttpStatusCode.NotFound)
    {
    }
}
```

**BadRequestException:**

```csharp
public sealed class BadRequestException : AppException
{
    public BadRequestException(string message)
        : base(message, HttpStatusCode.BadRequest)
    {
    }
}
```

**ConflictException:**

```csharp
public sealed class ConflictException : AppException
{
    public ConflictException(string message)
        : base(message, HttpStatusCode.Conflict)
    {
    }
}
```

**ValidationException:**

```csharp
public sealed class ValidationException : AppException
{
    public IDictionary<string, string[]> Errors { get; }

    public ValidationException(IDictionary<string, string[]> errors)
        : base("One or more validation errors occurred.", HttpStatusCode.BadRequest)
    {
        Errors = errors;
    }

    public ValidationException(string field, string error)
        : base("One or more validation errors occurred.", HttpStatusCode.BadRequest)
    {
        Errors = new Dictionary<string, string[]>
        {
            { field, [error] }
        };
    }
}
```

**Exception Design Patterns:**

| Pattern | Implementation | Purpose |
|---------|----------------|---------|
| Sealed classes | Prevents further inheritance | Clear intent, optimization |
| Descriptive constructors | `NotFoundException(resourceName, key)` | Self-documenting errors |
| Field-level errors | `ValidationException` with dictionary | Form validation support |

### Step 2: Implement IExceptionHandler

#### Production-Ready Handler

```csharp
using Microsoft.AspNetCore.Diagnostics;
using Microsoft.AspNetCore.Mvc;

public sealed class GlobalExceptionHandler(
    ILogger<GlobalExceptionHandler> logger,
    IProblemDetailsService problemDetailsService) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        // Log with trace identifier for correlation
        logger.LogError(
            exception, 
            "Unhandled exception occurred. TraceId: {TraceId}",
            httpContext.TraceIdentifier);

        // Map exception to HTTP status
        var (statusCode, title) = MapException(exception);

        httpContext.Response.StatusCode = statusCode;

        // Build RFC 9457 Problem Details
        var problemDetails = new ProblemDetails
        {
            Status = statusCode,
            Title = title,
            Type = GetProblemType(statusCode),
            Instance = httpContext.Request.Path,
            Detail = GetSafeErrorMessage(exception, httpContext)
        };

        // Add debugging extensions
        problemDetails.Extensions["traceId"] = httpContext.TraceIdentifier;
        problemDetails.Extensions["timestamp"] = DateTime.UtcNow;

        // Use framework service for response
        return await problemDetailsService.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = httpContext,
            ProblemDetails = problemDetails
        });
    }

    private static (int StatusCode, string Title) MapException(Exception exception) 
        => exception switch
    {
        AppException appEx => ((int)appEx.StatusCode, appEx.Message),
        ArgumentNullException => (StatusCodes.Status400BadRequest, "Invalid argument"),
        ArgumentException => (StatusCodes.Status400BadRequest, "Invalid argument"),
        UnauthorizedAccessException => (StatusCodes.Status401Unauthorized, "Unauthorized"),
        _ => (StatusCodes.Status500InternalServerError, "An unexpected error occurred")
    };

    private static string GetProblemType(int statusCode) => statusCode switch
    {
        400 => "https://tools.ietf.org/html/rfc9110#section-15.5.1",
        401 => "https://tools.ietf.org/html/rfc9110#section-15.5.2",
        403 => "https://tools.ietf.org/html/rfc9110#section-15.5.4",
        404 => "https://tools.ietf.org/html/rfc9110#section-15.5.5",
        409 => "https://tools.ietf.org/html/rfc9110#section-15.5.10",
        _ => "https://tools.ietf.org/html/rfc9110#section-15.6.1"
    };

    private static string? GetSafeErrorMessage(Exception exception, HttpContext context)
    {
        // Only expose details in development
        var env = context.RequestServices.GetRequiredService<IHostEnvironment>();
        
        if (env.IsDevelopment())
        {
            return exception.Message;
        }

        // In production, only expose messages from our own exceptions
        return exception is AppException ? exception.Message : null;
    }
}
```

#### Method Breakdown

**TryHandleAsync Method:**

| Section | Lines | Purpose |
|---------|-------|---------|
| Logging | 13-17 | Record exception with correlation ID |
| Status Mapping | 19-20 | Determine HTTP response code |
| Problem Details | 24-31 | Build RFC 9457 compliant response |
| Extensions | 34-35 | Add debug metadata |
| Response Writing | 37-41 | Use framework service for serialization |

**MapException Method:**

```csharp
private static (int StatusCode, string Title) MapException(Exception exception)
```

Uses pattern matching to map exception types to HTTP status codes:

| Exception Type | Status Code | Title |
|----------------|-------------|-------|
| `AppException` | From exception | From exception |
| `ArgumentNullException` | 400 | "Invalid argument" |
| `ArgumentException` | 400 | "Invalid argument" |
| `UnauthorizedAccessException` | 401 | "Unauthorized" |
| Default | 500 | "An unexpected error occurred" |

**GetSafeErrorMessage Method:**

Security-critical method with environment-aware behavior:

| Environment | Custom Exceptions | System Exceptions |
|-------------|-------------------|-------------------|
| Development | Full message | Full message |
| Production | Full message | `null` (hidden) |

### Step 3: Register in Program.cs

```csharp
var builder = WebApplication.CreateBuilder(args);

// Register Problem Details service
builder.Services.AddProblemDetails();

// Register exception handler
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();

var app = builder.Build();

// Add exception handling middleware
app.UseExceptionHandler();

app.Run();
```

**Registration Order:**

1. `AddProblemDetails()` - Registers IProblemDetailsService
2. `AddExceptionHandler<T>()` - Registers handler as singleton
3. `UseExceptionHandler()` - Adds middleware to pipeline

---

## Advanced Patterns

### .NET 10: SuppressDiagnosticsCallback

**.NET 10 Change:** By default, middleware no longer emits diagnostic logs when handler returns `true`.

#### Revert to .NET 8/9 Behavior

```csharp
app.UseExceptionHandler(new ExceptionHandlerOptions
{
    SuppressDiagnosticsCallback = _ => false  // Always emit diagnostics
});
```

#### Selective Suppression

```csharp
app.UseExceptionHandler(new ExceptionHandlerOptions
{
    SuppressDiagnosticsCallback = context =>
        context.Exception is NotFoundException or BadRequestException
});
```

**Use Cases:**

| Scenario | Configuration |
|----------|---------------|
| Legacy compatibility | `_ => false` (always log) |
| Skip business errors | Suppress `NotFoundException`, `BadRequestException` |
| Log unexpected only | Suppress custom exceptions, log system exceptions |

### Handler Chaining

#### NotFoundExceptionHandler

```csharp
public sealed class NotFoundExceptionHandler(
    ILogger<NotFoundExceptionHandler> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        // Check if this handler can process the exception
        if (exception is not NotFoundException notFound)
        {
            return false; // Pass to next handler
        }

        // Log at appropriate level
        logger.LogWarning("Resource not found: {Message}", notFound.Message);

        httpContext.Response.StatusCode = StatusCodes.Status404NotFound;
        
        await httpContext.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = 404,
            Title = "Resource Not Found",
            Detail = notFound.Message
        }, cancellationToken);

        return true; // Handled successfully
    }
}
```

#### ValidationExceptionHandler

```csharp
public sealed class ValidationExceptionHandler(
    ILogger<ValidationExceptionHandler> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        if (exception is not ValidationException validation)
        {
            return false;
        }

        logger.LogWarning("Validation failed: {Message}", validation.Message);

        httpContext.Response.StatusCode = StatusCodes.Status400BadRequest;
        
        await httpContext.Response.WriteAsJsonAsync(new ValidationProblemDetails
        {
            Status = 400,
            Title = "Validation Failed",
            Errors = validation.Errors
        }, cancellationToken);

        return true;
    }
}
```

#### Registration Order

```csharp
// Order matters - first to last
builder.Services.AddExceptionHandler<NotFoundExceptionHandler>();
builder.Services.AddExceptionHandler<ValidationExceptionHandler>();
builder.Services.AddExceptionHandler<GlobalExceptionHandler>(); // Fallback
```

**Execution Flow:**

```
Exception Thrown
    ↓
NotFoundExceptionHandler
    ├─ NotFoundException? → Handle (return true)
    └─ No? → Pass (return false)
        ↓
ValidationExceptionHandler
    ├─ ValidationException? → Handle (return true)
    └─ No? → Pass (return false)
        ↓
GlobalExceptionHandler
    └─ Handle ALL (return true)
```

### Silencing Middleware Logs

**appsettings.json:**

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.AspNetCore.Diagnostics.ExceptionHandlerMiddleware": "None"
    }
  }
}
```

**Effect:** Middleware catches exceptions and calls handlers, but doesn't log.

---

## Production Best Practices

### Project Structure

```
YourApi/
├── Exceptions/
│   ├── AppException.cs
│   ├── NotFoundException.cs
│   ├── BadRequestException.cs
│   ├── ConflictException.cs
│   └── ValidationException.cs
├── Handlers/
│   ├── GlobalExceptionHandler.cs
│   ├── NotFoundExceptionHandler.cs
│   └── ValidationExceptionHandler.cs
├── Program.cs
├── appsettings.json
└── appsettings.Development.json
```

### Global Problem Details Customization

```csharp
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = ctx =>
    {
        // Add to ALL problem detail responses
        ctx.ProblemDetails.Extensions["traceId"] = ctx.HttpContext.TraceIdentifier;
        ctx.ProblemDetails.Extensions["timestamp"] = DateTime.UtcNow;
        ctx.ProblemDetails.Instance = 
            $"{ctx.HttpContext.Request.Method} {ctx.HttpContext.Request.Path}";
    };
});
```

### Example Response

**Request:**
```http
GET /products/123e4567-e89b-12d3-a456-426614174000
```

**Response (404):**
```json
{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.5.5",
  "title": "Product with identifier '123e4567-e89b-12d3-a456-426614174000' was not found.",
  "status": 404,
  "instance": "GET /products/123e4567-e89b-12d3-a456-426614174000",
  "traceId": "0HN123ABC:00000001",
  "timestamp": "2026-01-29T10:30:00.000Z"
}
```

### Service Layer Usage

```csharp
public class ProductService
{
    private readonly IProductRepository _repository;

    public Product GetById(Guid id)
    {
        var product = _repository.Find(id);
        
        if (product is null)
        {
            throw new NotFoundException("Product", id);
        }

        return product;
    }

    public void Create(ProductRequest request)
    {
        if (string.IsNullOrWhiteSpace(request.Name))
        {
            throw new BadRequestException("Product name is required");
        }

        // Check for duplicates
        if (_repository.ExistsByName(request.Name))
        {
            throw new ConflictException(
                $"Product with name '{request.Name}' already exists");
        }

        _repository.Add(request);
    }

    public void ValidateProduct(ProductRequest request)
    {
        var errors = new Dictionary<string, string[]>();

        if (string.IsNullOrWhiteSpace(request.Name))
        {
            errors["name"] = ["Product name is required"];
        }

        if (request.Price <= 0)
        {
            errors["price"] = ["Product price must be greater than zero"];
        }

        if (errors.Any())
        {
            throw new ValidationException(errors);
        }
    }
}
```

---

## Performance Considerations

### The Cost of Exceptions

**Problem:** Exceptions are expensive operations involving:
- Stack unwinding
- Memory allocation
- Performance degradation under load

**David Fowler (Microsoft ASP.NET Core Architect):** "Exceptions should be exceptional."

### Result Pattern Alternative

For **expected** error conditions (validation, not found), use the Result pattern:

#### Result<T> Implementation

```csharp
public record Result<T>
{
    public T? Value { get; init; }
    public string? Error { get; init; }
    public bool IsSuccess => Error is null;

    public static Result<T> Success(T value) => new() { Value = value };
    public static Result<T> Failure(string error) => new() { Error = error };
}
```

#### Service Usage

```csharp
public Result<Product> GetById(Guid id)
{
    var product = _repository.Find(id);
    
    return product is not null
        ? Result<Product>.Success(product)
        : Result<Product>.Failure($"Product {id} not found");
}
```

#### Endpoint Usage

```csharp
app.MapGet("/products/{id:guid}", (Guid id, ProductService service) =>
{
    var result = service.GetById(id);
    
    return result.IsSuccess
        ? Results.Ok(result.Value)
        : Results.NotFound(result.Error);
});
```

### Performance Comparison

| Approach | Cost | Use Case |
|----------|------|----------|
| Exceptions | High (stack unwinding) | Unexpected errors |
| Result Pattern | Low (simple record) | Expected failures |

**Guideline:**

- **Use Exceptions:** Database failures, network errors, file system errors
- **Use Result Pattern:** Not found, validation failures, business rule violations

---

## Appendix

### Complete Working Example

```csharp
using Microsoft.AspNetCore.Diagnostics;
using Microsoft.AspNetCore.Mvc;
using System.Net;

var builder = WebApplication.CreateBuilder(args);

// Configure global problem details
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = ctx =>
    {
        ctx.ProblemDetails.Extensions["traceId"] = ctx.HttpContext.TraceIdentifier;
        ctx.ProblemDetails.Extensions["timestamp"] = DateTime.UtcNow;
        ctx.ProblemDetails.Instance = 
            $"{ctx.HttpContext.Request.Method} {ctx.HttpContext.Request.Path}";
    };
});

builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddOpenApi();

var app = builder.Build();

app.UseExceptionHandler();
app.MapOpenApi();

// Test endpoints
app.MapGet("/", () => "Global Exception Handling Demo - .NET 10");

app.MapGet("/products/{id:guid}", (Guid id) =>
{
    throw new NotFoundException("Product", id);
});

app.MapPost("/products", (ProductRequest request) =>
{
    if (string.IsNullOrWhiteSpace(request.Name))
    {
        throw new BadRequestException("Product name is required");
    }
    return Results.Created($"/products/{Guid.NewGuid()}", request);
});

app.MapGet("/error", () =>
{
    throw new InvalidOperationException("Something went terribly wrong!");
});

app.Run();

// Records
public record ProductRequest(string Name, decimal Price);

// Base Exception
public abstract class AppException(
    string message, 
    HttpStatusCode statusCode = HttpStatusCode.InternalServerError)
    : Exception(message)
{
    public HttpStatusCode StatusCode { get; } = statusCode;
}

// Specific Exceptions
public sealed class NotFoundException(string resourceName, object key)
    : AppException(
        $"{resourceName} with identifier '{key}' was not found.", 
        HttpStatusCode.NotFound);

public sealed class BadRequestException(string message)
    : AppException(message, HttpStatusCode.BadRequest);

// Exception Handler
public sealed class GlobalExceptionHandler(
    ILogger<GlobalExceptionHandler> logger,
    IProblemDetailsService problemDetailsService) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        logger.LogError(
            exception, 
            "Exception occurred. TraceId: {TraceId}", 
            httpContext.TraceIdentifier);

        var (statusCode, title) = exception switch
        {
            AppException appEx => ((int)appEx.StatusCode, appEx.Message),
            _ => (StatusCodes.Status500InternalServerError, "An unexpected error occurred")
        };

        httpContext.Response.StatusCode = statusCode;

        return await problemDetailsService.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = httpContext,
            ProblemDetails = new ProblemDetails
            {
                Status = statusCode,
                Title = title
            }
        });
    }
}
```

### FAQ

**Q: Should I use IExceptionHandler or custom middleware?**  
A: Use `IExceptionHandler` on .NET 8+. It's the recommended Microsoft approach with better testing and framework integration.

**Q: Can I use multiple IExceptionHandler implementations?**  
A: Yes. Register them in order, each can return `false` to pass to the next handler.

**Q: Should I expose exception details to clients?**  
A: Never in production. Only expose safe messages. Use development environment checks for debugging.

**Q: What's the difference between IExceptionHandler and ExceptionFilterAttribute?**  
A: `ExceptionFilterAttribute` only works in MVC controllers. `IExceptionHandler` works across the entire pipeline (minimal APIs, middleware, controllers).

**Q: What changed in .NET 10?**  
A: `SuppressDiagnosticsCallback` on `ExceptionHandlerOptions`. By default, .NET 10 doesn't emit diagnostic logs when handlers return `true`.

### HTTP Status Code Reference

| Code | Exception Type | Use Case |
|------|----------------|----------|
| 400 | `BadRequestException`, `ValidationException` | Invalid input |
| 401 | `UnauthorizedAccessException` | Missing/invalid auth |
| 403 | `ForbiddenException` | Insufficient permissions |
| 404 | `NotFoundException` | Resource not found |
| 409 | `ConflictException` | Duplicate/concurrent modification |
| 500 | Default | Unexpected errors |

### Additional Resources

- **RFC 9457:** Problem Details for HTTP APIs
- **RFC 9110:** HTTP Semantics (Status Codes)
- **Microsoft Docs:** Exception Handling in ASP.NET Core
- **David Fowler's Guidelines:** Performance Best Practices

---

## Summary

### Key Points

1. **Modern Approach:** Use `IExceptionHandler` for .NET 8+ applications
2. **Custom Exceptions:** Create specific exception types with embedded HTTP status codes
3. **Problem Details:** Use RFC 9457 standard for error responses
4. **Security:** Never expose internal error details in production
5. **.NET 10:** Leverage `SuppressDiagnosticsCallback` for log control
6. **Performance:** Consider Result Pattern for expected failures
7. **Testing:** Handler design makes unit testing straightforward
8. **Chaining:** Register multiple handlers for granular error handling

### Implementation Checklist

- [ ] Create custom exception hierarchy
- [ ] Implement `IExceptionHandler`
- [ ] Register `AddProblemDetails()` and `AddExceptionHandler<T>()`
- [ ] Add `UseExceptionHandler()` middleware
- [ ] Configure environment-aware error messaging
- [ ] Set up logging filters
- [ ] Add trace identifiers to responses
- [ ] Test all exception paths
- [ ] Document exception types for team

---

**Document Version:** 1.0  
**Target Framework:** .NET 10  
**Compatibility:** .NET 8, .NET 9, .NET 10

*This documentation is based on production-ready patterns and Microsoft recommended practices.*
