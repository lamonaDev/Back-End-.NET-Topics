# HTTP Status Codes — Production Playbook

## Overview

HTTP status codes are standardized response codes that indicate the result of an HTTP request. Proper use of status codes is essential for building professional, self-documenting APIs that clients can interact with reliably.

## Status Code Families

| Family | Range | Description |
|--------|-------|-------------|
| 1xx | 100-199 | Informational — Request received, processing continues. Rarely seen in APIs. |
| 2xx | 200-299 | Success — Request was received, understood, and accepted. |
| 3xx | 300-399 | Redirection — Further action needed to complete the request. |
| 4xx | 400-499 | Client Error — The request contains bad syntax or cannot be fulfilled. |
| 5xx | 500-599 | Server Error — The server failed to fulfil a valid request. |

### Why Status Codes Matter

```
Good API Response:
  404 Not Found → "Product not found" → Client knows to show "not found" UI
  
Bad API Response:
  500 Internal Server Error → "Something went wrong" → Client doesn't know what to do
  200 OK → { error: "not found" } → Client must parse body to understand error
```

## 2xx Success Codes

| Code | Name | When to Use |
|------|------|-------------|
| 200 | OK | GET request successful, response in body |
| 201 | Created | POST/PUT created new resource, include Location header |
| 202 | Accepted | Request accepted for processing, will complete async |
| 204 | No Content | DELETE or successful update, no response body |
| 206 | Partial Content | GET with Range header, returning partial data |

### Example: 201 Created

```http
POST /api/products HTTP/1.1
Host: api.example.com
Content-Type: application/json

{ "name": "New Product", "price": 99.99 }

HTTP/1.1 201 Created
Location: /api/products/42
Content-Type: application/json

{ "id": 42, "name": "New Product", "price": 99.99, "createdAt": "..." }
```

### Example: 204 No Content

```http
DELETE /api/products/42 HTTP/1.1
Host: api.example.com

HTTP/1.1 204 No Content
```

## 3xx Redirection Codes

| Code | Name | When to Use |
|------|------|-------------|
| 301 | Moved Permanently | Resource has a new permanent URL |
| 302 | Found | Temporary redirect (or Moved Temporarily) |
| 304 | Not Modified | CDN cache hit — client has current version |
| 307 | Temporary Redirect | Temporary redirect, keep HTTP method |
| 308 | Permanent Redirect | Permanent redirect, keep HTTP method |

### 301 vs 302

- **301 (Moved Permanently)**: Tell clients to update bookmarks. Search engines transfer SEO value.
- **302 (Found)**: Temporary redirect. Keep original URL in bookmarks.

## 4xx Client Error Codes

| Code | Name | Real API Case | Anti-Pattern |
|------|------|---------------|--------------|
| 400 | Bad Request | Invalid JSON body, missing required field, constraint violation | Using 400 for auth failures |
| 401 | Unauthorized | Missing/invalid JWT token on protected endpoint | Using 403 when 401 is correct |
| 403 | Forbidden | Valid token but user lacks permission (e.g., non-admin) | Using 404 to hide existence |
| 404 | Not Found | GET /products/999 — product doesn't exist | Returning 404 for auth errors |
| 405 | Method Not Allowed | Using POST on endpoint that only supports GET | - |
| 409 | Conflict | POST /users — email already registered | Using 400 for all conflicts |
| 415 | Unsupported Media Type | Wrong Content-Type header | - |
| 422 | Unprocessable Entity | Business rule violation (order quantity exceeds stock) | Using 500 for validation |
| 429 | Too Many Requests | Rate limit exceeded — include Retry-After header | Using 503 for rate limiting |

### 401 vs 403 — The Common Mistake

```
401 Unauthorized:
  - You are not authenticated (no token, invalid token)
  - Client needs to log in first

403 Forbidden:
  - You ARE authenticated, but not authorized
  - You have a valid token, but can't perform this action
  - Example: Regular user trying to access admin endpoint
```

### 400 vs 422

```
400 Bad Request:
  - Syntax error in request
  - Invalid JSON structure
  - Missing required field names

422 Unprocessable Entity:
  - Syntax is correct, but semantic errors
  - Email format is valid but already exists
  - Quantity is valid (integer) but exceeds stock
```

## 5xx Server Error Codes

| Code | Name | Real API Case | Anti-Pattern |
|------|------|---------------|--------------|
| 500 | Internal Server Error | Unhandled exception, DB crash, null ref | Leaking stack traces in response |
| 502 | Bad Gateway | Reverse proxy couldn't reach upstream API | N/A |
| 503 | Service Unavailable | App restarting, maintenance window, circuit breaker open | Using 500 for planned downtime |
| 504 | Gateway Timeout | Reverse proxy timed out waiting for API | N/A |

## Consistent Error Response Strategy

ASP.NET Core's built-in **Problem Details** (RFC 7807) is the industry standard for error responses:

### Standard Problem Details Format

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "traceId": "00-abc123-def456-00",
  "errors": {
    "Email": ["The Email field is required."],
    "Price": ["Price must be greater than 0."]
  }
}
```

### Custom Error Types

```json
{
  "type": "https://api.example.com/errors/insufficient-stock",
  "title": "Insufficient Stock",
  "status": 422,
  "detail": "Product 'Laptop Pro' only has 5 units in stock, but you requested 10.",
  "availableStock": 5,
  "requestedQuantity": 10
}
```

### Program.cs — enable Problem Details globally

```csharp
builder.Services.AddProblemDetails();

// Minimal API — return ProblemHttpResult
app.MapPost("/products", (Product p) => {
    if (p.Price <= 0)
        return Results.Problem(
            detail: "Price must be positive",
            statusCode: StatusCodes.Status422UnprocessableEntity);
    return Results.Created($"/products/{p.Id}", p);
});
```

### Exception Handling Middleware

```csharp
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        context.Response.StatusCode = 500;
        context.Response.ContentType = "application/problem+json";
        
        var exceptionHandlerFeature = context.Features.Get<IExceptionHandlerFeature>();
        var exception = exceptionHandlerFeature?.Error;
        
        await context.Response.WriteAsJsonAsync(new
        {
            type = "https://tools.ietf.org/html/rfc7231#section-6.6.1",
            title = "Internal Server Error",
            status = 500,
            detail = "An unexpected error occurred."
            // Don't expose exception details in production!
        });
    });
});
```

## Status Code Selection Guide

```
                    ┌─────────────────────────────────────────┐
                    │     What happened with the request?    │
                    └─────────────────────────────────────────┘
                                  │
       ┌──────────────────────────┼──────────────────────────┐
       │                          │                          │
       v                          v                          v
┌─────────────┐          ┌─────────────┐          ┌─────────────┐
│  Request   │          │  Request    │          │  Request    │
│  Received  │          │  Failed     │          │  Crashed    │
│  Successfully│          │  (Client)  │          │  (Server)   │
└─────────────┘          └─────────────┘          └─────────────┘
       │                          │                          │
       v                          v                          v
  ┌────┴────┐              ┌──────┴──────┐            ┌──────┴──────┐
  │         │              │             │            │             │
  v         v              v             v            v             v
 GET OK   POST OK        400-417       401-403     500-504
 PUT OK   Created       Client Error  Auth Error   Server Error
 DELETE   202/204       (Bad Request) (Forbidden)
          2xx          4xx           4xx          5xx
```

## Best Practices

1. Use the correct status code for each scenario
2. Return consistent error formats (RFC 7807 Problem Details)
3. Include Location header for 201 Created
4. Don't return 200 OK with error in body
5. Don't expose stack traces in production
6. Use 404 for missing resources, 403 for forbidden
7. Use 429 for rate limiting, not 503
8. Include meaningful error messages for debugging
9. Log all 5xx errors for investigation
10. Return 204 No Content for successful DELETE

## References

- [MDN — HTTP Status Codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
- [RFC 7807 — Problem Details for HTTP APIs](https://datatracker.ietf.org/doc/html/rfc7807)
- [Microsoft Docs — Handle errors in ASP.NET Core web APIs](https://learn.microsoft.com/en-us/aspnet/core/web-api/handle-errors)