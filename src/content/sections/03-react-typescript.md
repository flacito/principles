---
title: "React & TypeScript Conventions"
order: 3
description: "Functional components, naming conventions, and state management patterns."
---

## React & TypeScript Conventions

### 19. Functional Components Only

Functional React components only. Use `observer()` wrapper for MobX integration. No class components.

### 20. Boolean Naming

Boolean state/variable names prefixed with `is`, `has`, `should`, or `can`. Examples: `isVisible`, `shouldRender`, `hasBookmark`.

### 21. Handler Naming

Handler functions prefixed with `handle`. Examples: `handleBookmark`, `handleDelete`, `handleToggleFavorite`.

### 22. Function Naming by Verb

Async/data functions prefixed by verb indicating operation. Use `fetch` for API calls, `parse` for transformations, `format` for display formatting, `get` for retrieval. Examples: `fetchChapterHtml`, `parseReference`, `formatChapterReference`.

### 23. State Management

MobX for global state, refs for mutable non-reactive state. Single observable class instance pattern (e.g., `bibleState`). Use `useRef` for values that shouldn't trigger re-renders (timers, flags, last-known values).

### 24. Animation Pattern

Three-state animation pattern: `'entering' | 'visible' | 'exiting'`. Separate `shouldRender` boolean controls DOM presence while `animationState` controls CSS classes.

### 25. Zod-First Schemas

Zod schemas for model validation. Derive TypeScript types via `z.infer<typeof Schema>` rather than manual type definitions.

### 26. Types vs Interfaces

Prefer `interface` for object shapes, `type` for unions and aliases.
