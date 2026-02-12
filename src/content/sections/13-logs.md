---
title: "Logs Are First-Class"
order: 13
description: "Every long-running command logs with timestamps, structured in production, readable in dev."
---

## Logs Are First-Class

### 39. Every Long-Running Command Logs

Every long-running command logs to `.logs/` with timestamps. Logs are gitignored but always available for debugging. Use `tee` for real-time terminal output plus persistent record. Log filenames include the operation and timestamp: `deploy-20250206-143022.log`, `dev-20250206-090000.log`. Clean up logs older than a configurable retention period. Structured JSON logging in production (for machine parsing), human-readable in development.
