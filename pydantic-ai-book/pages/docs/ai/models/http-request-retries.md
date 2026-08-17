---
type: Web Page
title: HTTP Request Retries | Pydantic Docs
resource: https://pydantic.dev/docs/ai/models/http-request-retries
timestamp: '2026-08-17T07:03:21.217446+00:00'
---

# HTTP Request Retries

Pydantic AI provides retry functionality for HTTP requests made by model providers through custom HTTP transports. This is particularly useful for handling transient failures like rate limits, network timeouts, or temporary server errors.

This is the lowest of the [several layers that can retry](/docs/ai/core-concepts/retries/) in an agent run, and the only one the model never sees.

The retry functionality is built on top of the [tenacity](https://github.com/jd/tenacity) library and integrates
seamlessly with httpx clients. You can configure retry behavior for any provider that accepts a custom HTTP client.

To use the retry transports, you need to install `tenacity`, which you can do via the `retries` dependency group:

Here’s an example of adding retry functionality with smart retry handling:

The `wait_retry_after` function is a smart wait strategy that automatically respects HTTP `Retry-After` headers:

This wait strategy:

- Automatically parses `Retry-After` headers from HTTP 429 responses
- Supports both seconds format (`"30"` ) and HTTP date format (`"Wed, 21 Oct 2015 07:28:00 GMT"` )
- Falls back to your chosen strategy when no header is present
- Respects the `max_wait` limit to prevent excessive delays

For asynchronous HTTP clients (recommended for most use cases):

For synchronous HTTP clients:

The `wait_retry_after` function automatically detects `Retry-After` headers in 429 (rate limit) responses and waits for the specified time. If no header is present, it falls back to exponential backoff.

The retry transports work with any provider that accepts a custom HTTP client:

1. 
**Start Conservative** : Begin with a small number of retries (3-5) and reasonable wait times.
2. 
**Use Exponential Backoff** : This helps avoid overwhelming servers during outages.
3. 
**Set Maximum Wait Times** : Prevent indefinite delays with reasonable maximum wait times.
4. 
**Handle Rate Limits Properly** : Respect`Retry-After` headers when possible.
5. 
**Log Retry Attempts** : Add logging to monitor retry behavior in production. (This will be picked up by Logfire automatically if you instrument httpx.)
6. 
**Consider Circuit Breakers** : For high-traffic applications, consider implementing circuit breaker patterns.

The retry transports will re-raise the last exception if all retry attempts fail. Make sure to handle these appropriately in your application:

- Retries add latency to requests, especially with exponential backoff
- Consider the total timeout for your application when configuring retry behavior
- Monitor retry rates to detect systemic issues
- Use async transports for better concurrency when handling multiple requests

For more advanced retry configurations, refer to the [tenacity documentation](https://tenacity.readthedocs.io/).

The AWS Bedrock provider uses boto3’s built-in retry mechanisms instead of httpx. To configure retries for Bedrock, use boto3’s `Config`:

```
from botocore.config import Config
config = Config(retries={'max_attempts': 5, 'mode': 'adaptive'})
```
See [Bedrock: Configuring Retries](/docs/ai/models/bedrock/#configuring-retries) for complete examples.

# Citations

1. Source page: https://pydantic.dev/docs/ai/models/http-request-retries
