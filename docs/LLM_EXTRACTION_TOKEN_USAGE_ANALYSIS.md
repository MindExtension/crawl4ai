# LLM Extraction Token Usage - Analysis & Strategy

## Overview

This document analyzes how token usage (prompt + completion tokens) is tracked in crawl4ai's LLM extraction strategy and proposes a strategy to expose this data via the HTTP API.

## Current Implementation

### Token Usage Model

crawl4ai has a `TokenUsage` dataclass in `crawl4ai/models.py`:

```python
@dataclass
class TokenUsage:
    completion_tokens: int = 0
    prompt_tokens: int = 0
    total_tokens: int = 0
    completion_tokens_details: Optional[dict] = None
    prompt_tokens_details: Optional[dict] = None
```

### LLMExtractionStrategy Token Tracking

The `LLMExtractionStrategy` class (`crawl4ai/extraction_strategy.py`) tracks token usage automatically:

1. **Instance attributes** (lines 581-582):
   ```python
   self.usages = []  # Store individual usages per chunk
   self.total_usage = TokenUsage()  # Accumulated usage
   ```

2. **After each LLM call** (lines 657-673):
   ```python
   usage = TokenUsage(
       completion_tokens=response.usage.completion_tokens,
       prompt_tokens=response.usage.prompt_tokens,
       total_tokens=response.usage.total_tokens,
       completion_tokens_details=response.usage.completion_tokens_details.__dict__
       if response.usage.completion_tokens_details else {},
       prompt_tokens_details=response.usage.prompt_tokens_details.__dict__
       if response.usage.prompt_tokens_details else {},
   )
   self.usages.append(usage)
   
   # Update totals
   self.total_usage.completion_tokens += usage.completion_tokens
   self.total_usage.prompt_tokens += usage.prompt_tokens
   self.total_usage.total_tokens += usage.total_tokens
   ```

3. **Helper method** `show_usage()` (lines 974-989) prints a detailed report.

### Python SDK Usage

When using crawl4ai as a Python library, token usage is accessible:

```python
from crawl4ai import LLMExtractionStrategy

strategy = LLMExtractionStrategy(
    llm_config=your_llm_config,
    instruction="Extract product info"
)

result = await crawler.arun(url, extraction_strategy=strategy)

# Access token usage
print(strategy.total_usage.prompt_tokens)      # Total prompt tokens
print(strategy.total_usage.completion_tokens)  # Total completion tokens
print(strategy.total_usage.total_tokens)       # Combined total

# Per-request breakdown
for i, usage in enumerate(strategy.usages):
    print(f"Request {i}: {usage.prompt_tokens} prompt, {usage.completion_tokens} completion")

# Or use the built-in helper
strategy.show_usage()
```

## Problem: HTTP API Does Not Expose Token Usage

### Current HTTP Response Structure

Looking at `deploy/docker/api.py` (lines 602-643), the `/crawl` endpoint returns:

```python
response = {
    "success": True,
    "results": processed_results,  # CrawlResult.model_dump()
    "server_processing_time_s": end_time - start_time,
    "server_memory_delta_mb": mem_delta_mb,
    "server_peak_memory_mb": peak_mem_mb
}
```

The `CrawlResult.model_dump()` does **NOT** include token usage - it only contains:
- `url`, `html`, `success`, `extracted_content`, `markdown`, `links`, `media`, `screenshot`, etc.

### Root Cause

In `deploy/docker/api.py` (lines 162-183), the extraction strategy is created, used, and then **discarded** without extracting the usage:

```python
llm_strategy = LLMExtractionStrategy(...)
result = await crawler.arun(url=url, config=CrawlerRunConfig(extraction_strategy=llm_strategy, ...))
# llm_strategy.total_usage is available here but NOT returned in the response!
```

## Proposed Solution

### Option 1: Modify HTTP API Response (Recommended)

Add token usage to the response in `deploy/docker/api.py`:

#### For `/llm` endpoint (`process_llm_extraction` function):

```python
# After line 207, before storing result in Redis
result_data = {"extracted_content": content}

# Add token usage if available
if hasattr(llm_strategy, 'total_usage'):
    result_data["token_usage"] = {
        "prompt_tokens": llm_strategy.total_usage.prompt_tokens,
        "completion_tokens": llm_strategy.total_usage.completion_tokens,
        "total_tokens": llm_strategy.total_usage.total_tokens,
    }
    # Optionally include per-chunk breakdown
    if llm_strategy.usages:
        result_data["token_usage"]["chunks"] = [
            {
                "prompt_tokens": u.prompt_tokens,
                "completion_tokens": u.completion_tokens,
                "total_tokens": u.total_tokens,
            }
            for u in llm_strategy.usages
        ]
```

#### For `/crawl` endpoint with LLM extraction:

This is more complex because the extraction strategy is embedded in `CrawlerRunConfig`. Options:

1. **Modify `CrawlResult` model** to include an optional `token_usage` field
2. **Return strategy reference** from `arun()` alongside the result
3. **Add callback mechanism** to capture usage during extraction

### Option 2: Add Token Usage to CrawlResult Model

Modify `crawl4ai/models.py` to include token usage in `CrawlResult`:

```python
class CrawlResult(BaseModel):
    # ... existing fields ...
    llm_token_usage: Optional[TokenUsage] = None  # NEW
```

Then in `async_webcrawler.py`, after extraction:

```python
if config.extraction_strategy and hasattr(config.extraction_strategy, 'total_usage'):
    crawl_result.llm_token_usage = config.extraction_strategy.total_usage
```

### Option 3: Webhook Enhancement

For async jobs using webhooks, include token usage in the webhook payload:

```python
# In webhook notification
await webhook_service.notify_job_completion(
    task_id=task_id,
    task_type="llm_extraction",
    status="completed",
    urls=[url],
    webhook_config=webhook_config,
    result=result_data,
    token_usage={  # NEW
        "prompt_tokens": llm_strategy.total_usage.prompt_tokens,
        "completion_tokens": llm_strategy.total_usage.completion_tokens,
        "total_tokens": llm_strategy.total_usage.total_tokens,
    }
)
```

## Expected HTTP Response Format

After implementing the solution, the HTTP response should include:

```json
{
  "success": true,
  "results": [
    {
      "url": "https://example.com",
      "success": true,
      "extracted_content": "...",
      "token_usage": {
        "prompt_tokens": 1500,
        "completion_tokens": 250,
        "total_tokens": 1750,
        "chunks": [
          {"prompt_tokens": 750, "completion_tokens": 125, "total_tokens": 875},
          {"prompt_tokens": 750, "completion_tokens": 125, "total_tokens": 875}
        ]
      }
    }
  ],
  "server_processing_time_s": 2.5,
  "server_memory_delta_mb": 50
}
```

## Implementation Priority

1. **High Priority**: Add token usage to `/llm` endpoint response (simplest change)
2. **Medium Priority**: Add token usage to `/crawl` endpoint when using LLMExtractionStrategy
3. **Low Priority**: Include in webhook payloads for async jobs

## Files to Modify

| File | Change |
|------|--------|
| `deploy/docker/api.py` | Add token_usage to response in `process_llm_extraction` and `handle_crawl_request` |
| `crawl4ai/models.py` | Optionally add `llm_token_usage` field to `CrawlResult` |
| `crawl4ai/async_webcrawler.py` | Optionally populate token usage in CrawlResult |
| `deploy/docker/webhook.py` | Include token usage in webhook payloads |

## Notes

- Token usage depends on the LLM provider returning usage data in the response
- Some providers may not return detailed token breakdowns
- Chunked content will have multiple usage entries (one per chunk)
- The `litellm` library used internally handles provider-specific response formats
