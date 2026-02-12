---
title: "Progressive Degradation"
order: 11
description: "The app works without its dependencies — each layer adds capability, none is required."
---

## Progressive Degradation

### 37. The App Works Without Its Dependencies

The app works without its infrastructure dependencies. The core experience functions without Ollama, ChromaDB, or network access. AI features degrade gracefully with warnings, not crashes. Every service dependency is wrapped in a health check — if it's down, skip it and tell the user what's unavailable. The app is useful at every layer: static files alone, static + API, static + API + AI. Each layer adds capability but none is required for the layer below it.
