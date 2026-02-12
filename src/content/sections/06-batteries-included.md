---
title: "Batteries Included"
order: 6
description: "Build it in, don't bolt it on — prefer custom implementations over dependencies."
---

## Batteries Included

### 31. Build It In

Build it in, don't bolt it on. Prefer custom implementations with sensible defaults over external dependencies. Examples: custom scroll detection over intersection-observer libraries, built-in localStorage caching over cache libraries, native CSS transitions over animation frameworks. The codebase should work out of the box with zero configuration. Every feature ships with its own persistence, theming support, offline fallback, and error handling baked in. External dependencies are a last resort, not a first reach.
