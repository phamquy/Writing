---
description: >-
  My personal note and reminder base on my experience building and operate one
  of the most foundation service in AWS
icon: triangle-exclamation
---

# Error Handling in Microservices: Best Practices

Error handling in microservices architecture requires special attention due to the distributed nature of these systems. Here are the best practices for effective error handling in microservices:

### 1. Standardize Error Formats

Create a consistent error response format across all microservices:

```json
{
  "status": 400,
  "code": "VALIDATION_ERROR",
  "message": "Invalid input parameters",
  "details": [
    {
      "field": "email",
      "message": "Must be a valid email address"
    }
  ],
  "timestamp": "2023-07-15T10:30:45Z",
  "traceId": "abc123xyz456"
}
```

### 2. Use Proper HTTP Status Codes

* `2xx` - Success responses
* `4xx` - Client errors (bad requests, unauthorized)
* `5xx` - Server errors (internal failures)

### 3. Implement Circuit Breakers

Prevent cascading failures by implementing circuit breakers that:

* Monitor for failures
* Trip when failure threshold is reached
* Provide fallback responses
* Auto-reset after timeout period

```python
# Example using Python with resilience4j-like pattern
@circuit_breaker(failure_threshold=5, reset_timeout=30)
def call_dependent_service():
    try:
        return http_client.get('http://other-service/api/resource')
    except Exception:
        return fallback_response()
```

### 4. Implement Retry Mechanisms

For transient failures, implement intelligent retry logic:

* Use exponential backoff
* Set maximum retry attempts
* Add jitter to prevent thundering herd

Beside the common practice, considered for **higher availability**

* Adaptive retry: adjust token base on success rate
* Avoid waste work by request deadline
* Improve latency by request hedging (watch out for idempotency of the api)
* Backpressure when in control of both client and server
* Metastable state, blackhole condition
* When in multiple AZ: auto weighing away and az alignment
* Adaptive queueing strategy: switching to stack when under duress
* Finding tipping point of all components of your services, bottleneck/chokepoint can be in any of those
* Noisy neighbor protection
* Throttling, and circuit breaker

### 5. Implement Timeouts

Set appropriate timeouts for all service-to-service communications:

```python
# Example with timeout
def call_service():
    try:
        return http_client.get('http://other-service/api/resource', timeout=3)
    except TimeoutError:
        # Handle timeout specifically
        log.warning("Service call timed out")
        return error_response(503, "Service temporarily unavailable")
```

### 6. Centralized Logging and Monitoring

Use correlation IDs across service boundaries

#### Implement structured logging

Structured logging is a critical practice in microservices architecture that transforms traditional text-based logs into structured data formats (typically JSON), making them machine-readable and easier to process. Here's a detailed explanation:

**What is Structured Logging?**

Structured logging formats log entries as data objects with consistent fields rather than unstructured text strings. This approach makes logs searchable, filterable, and analyzable at scale.

**Benefits of Structured Logging in Microservices**

* **Improved Searchability**: Easily query specific fields across distributed systems
* **Better Correlation**: Link related events across multiple services
* **Enhanced Analysis**: Perform statistical analysis on log data
* **Easier Parsing**: No need for complex regex patterns to extract information
* **Consistent Format**: Standardized format across different services and languages

**Key Components of Structured Logging**

**1. Standard Log Fields**

Every log entry should include:

```json
{
  "timestamp": "2023-07-15T10:30:45.123Z",
  "level": "ERROR",
  "service": "payment-service",
  "traceId": "abc123xyz456",
  "spanId": "span-789",
  "message": "Payment processing failed",
  "context": {
    "userId": "user-123",
    "orderId": "order-456",
    "paymentProvider": "stripe"
  },
  "exception": {
    "type": "PaymentDeclinedException",
    "message": "Card declined by issuer",
    "stackTrace": "..."
  }
}
```

**2. Correlation IDs**

Include identifiers that track requests across service boundaries:

```python
def process_order(order_id, trace_id=None):
    # Generate or propagate trace ID
    trace_id = trace_id or generate_trace_id()
    
    logger.info("Processing order", {
        "orderId": order_id,
        "traceId": trace_id
    })
    
    # Pass trace ID to other services
    payment_result = payment_service.process(
        order_id, 
        headers={"X-Trace-ID": trace_id}
    )

```

**3. Contextual Information**

Add relevant business context to logs:

```python
def update_user_profile(user_id, profile_data):
    logger.info("Updating user profile", {
        "userId": user_id,
        "updatedFields": list(profile_data.keys()),
        "source": request.headers.get("User-Agent")
    })

```

**Implementation Examples**

Python with structlog

```python
import structlog
import time

# Configure structured logger
structlog.configure(
    processors=[
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer()
    ]
)

logger = structlog.get_logger()

# Log with structured data
def process_payment(payment_id, amount, user_id):
    logger = logger.bind(
        payment_id=payment_id,
        user_id=user_id,
        service="payment-service"
    )
    
    logger.info("Payment processing started", amount=amount)
    
    try:
        # Process payment logic
        result = payment_gateway.charge(payment_id, amount)
        logger.info("Payment processed successfully", 
                   transaction_id=result.transaction_id)
        return result
    except Exception as e:
        logger.error("Payment processing failed", 
                    error_type=type(e).__name__,
                    error_message=str(e))
        raise
```

**Best Practices for Structured Logging**

* **Consistent Schema**: Define and enforce a standard log schema across all services
* **Appropriate Log Levels**: Use the right level (DEBUG, INFO, WARN, ERROR) for each message
* **Sensitive Data Handling**: Mask or exclude sensitive information (PII, credentials)
* **Performance Considerations**: Use sampling for high-volume logs
* **Contextual Binding**: Bind context early and carry it through the request lifecycle
* **Centralized Configuration**: Manage logging configuration centrally

**Integration with Observability Tools**

Structured logs integrate well with:

* **Log Aggregation**: Elasticsearch, Splunk, AWS CloudWatch
* **Distributed Tracing**: Jaeger, Zipkin, AWS X-Ray
* **Metrics Systems**: Prometheus, Grafana
* **APM Solutions**: Datadog, New Relic, Dynatrace

By implementing structured logging in your microservices, you create a foundation for effective troubleshooting, monitoring, and analysis across your distributed system.

#### Centralize logs for analysis

#### Set up alerts for error patterns

### 7. Graceful Degradation

Design services to function with reduced capabilities when dependencies fail:

```python
def get_user_with_recommendations(user_id):
    user = user_service.get_user(user_id)
    
    try:
        recommendations = recommendation_service.get_for_user(user_id)
    except ServiceUnavailableError:
        # Degrade gracefully
        recommendations = []
        log_error("Recommendation service unavailable")
    
    return {
        "user": user,
        "recommendations": recommendations
    }
```

### 8. Implement Bulkheads

Isolate failures to prevent system-wide outages:

* Separate thread pools
* Resource isolation
* Independent scaling

### 9. Use Dead Letter Queues

For asynchronous processing, implement dead letter queues:

* Move failed messages to separate queue
* Set up retry policies
* Alert on DLQ items
* Provide manual intervention tools

### 10. Document Error Responses

Ensure API documentation includes:

* All possible error codes
* Error response formats
* Recommended client handling strategies

### 11. Health Checks and Self-Healing

* Implement meaningful health checks
* Set up automated recovery processes
* Design for self-healing where possible

### 12. Test Failure Scenarios

* Implement chaos engineering principles
* Test service failures regularly
* Validate circuit breakers and fallbacks
* Try to choke every component in the processing path

By implementing these best practices, you can build resilient microservices that handle errors gracefully and maintain system stability even when individual components fail.
