---
title: "Performance & Persistence"
order: 4
description: "Debouncing, throttling, caching, and graceful fallback patterns."
---

## Performance & Persistence

### 27. Debounce and Throttle

Debounce and throttle side effects. Scroll handlers throttled with RAF. Save operations debounced 300-1200ms. Explicit flags like `isScrollUpdate` to distinguish event sources.

### 28. LocalStorage Persistence

LocalStorage persistence with prefixed keys. Pattern: `'skjv-<feature>'`. Lazy load in constructor, save on change with debounce.

### 29. Graceful Fallbacks

Graceful fallbacks with silent failures for non-critical operations. Try-catch with console warnings, not user-facing errors. Cache fallbacks when network fails.
