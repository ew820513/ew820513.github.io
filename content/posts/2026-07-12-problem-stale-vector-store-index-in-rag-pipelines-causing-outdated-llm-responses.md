---
title: "Problem: Stale vector store index in RAG pipelines causing outdated LLM responses"
date: 2026-07-12
draft: true
tags: ["architecture", "cloud"]
---

## Problem

In Retrieval-Augmented Generation (RAG) pipelines, the vector store index is used to store embeddings of documents for efficient retrieval. However, if the index is not kept up-to-date, the retrieved documents become stale, leading to outdated responses from the LLM. This is a common issue when the index is built once and then never refreshed, especially in environments where data changes frequently.

## Architecture Decisions

1. **Cache Invalidation Strategy**: We decided to implement a periodic refresh of the vector store index rather than real-time updates to avoid the overhead of constantly rebuilding the index. This balances between freshness and performance.

2. **Refresh Mechanism**: We introduced a scheduled job that refreshes the index at regular intervals (e.g., every 24 hours) or when a certain threshold of new data is detected. This ensures that the index remains current without overburdening the system.

## Implementation

```python
# Example: Periodic refresh of vector store index in a RAG pipeline
import time
from datetime import datetime, timedelta

# Assume we have a function to rebuild the vector store index
def rebuild_vector_store_index():
    # ... code to rebuild the index ...

# Schedule the next refresh
last_refresh = datetime.now() - timedelta(hours=24)
while True:
    if datetime.now() - last_refresh > timedelta(hours=24):
        rebuild_vector_store_index()
        last_refresh = datetime.now()
    time.sleep(3600)  # Check every hour
```

