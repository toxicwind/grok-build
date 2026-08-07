# Serialization Error Fix

## Problem

```
invalid type: null, expected u32 at line 1 column 354
```

This error occurs when xAI API returns error responses that don't match the expected OpenAI-compatible schema. The Rust/serde deserializer expects `u32` fields (like `created_at`) but gets `null`.

## Root Causes

### 1. Deprecated `stream_options` Field

xAI removed support for `stream_options` in the request body. Including it causes:
- 422 Unprocessable Entity
- Non-standard error responses that crash the deserializer

### 2. API Timeout/Overload

When xAI is overloaded, it returns:
```json
{"code":"The service is currently unavailable","error":"Timed out waiting for first token"}
```

This response lacks `created_at` (or has `null`), causing the `u32` deserialization error.

### 3. Rate Limiting

429 responses may have malformed bodies that don't match the expected schema.

## Fix Implementation

### Request Pre-processing

Strip `stream_options` before sending:

```rust
// In request builder
if body.contains_key("stream_options") {
    body.remove("stream_options");
    log::warn!("Removed deprecated stream_options from request");
}
```

### Response Pre-check

Check HTTP status before attempting deserialization:

```rust
if !response.status().is_success() {
    let error_text = response.text().await?;
    return Err(ApiError::HttpError {
        status: response.status(),
        body: error_text,
    });
}
```

### Lenient Parsing

Use `Option<u32>` instead of `u32` for fields that may be null:

```rust
#[derive(Deserialize)]
struct ChatCompletion {
    id: String,
    #[serde(default)]
    created_at: Option<u32>,  // Was: created_at: u32
    choices: Vec<Choice>,
}
```

### Retry Logic

Add exponential backoff for transient failures:

```rust
const MAX_RETRIES: u32 = 3;
const BASE_DELAY_MS: u64 = 1000;

async fn with_retry<T, F, Fut>(f: F) -> Result<T, ApiError>
where
    F: Fn() -> Fut,
    Fut: std::future::Future<Output = Result<T, ApiError>>,
{
    for attempt in 0..MAX_RETRIES {
        match f().await {
            Ok(result) => return Ok(result),
            Err(e) if e.is_transient() && attempt < MAX_RETRIES - 1 => {
                let delay = BASE_DELAY_MS * 2u64.pow(attempt);
                tokio::time::sleep(Duration::from_millis(delay)).await;
            }
            Err(e) => return Err(e),
        }
    }
    unreachable!()
}
```

## Testing

Run the fix verification:

```bash
cargo test --test serialization_fix
```

## References

- xAI API documentation: https://docs.x.ai/api
- OpenAI compatibility notes: https://docs.x.ai/docs/openai-compatibility
- Original issue: https://github.com/xai-org/grok-build/issues/ (search "serialization")
