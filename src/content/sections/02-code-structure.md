---
title: "Code Structure"
order: 2
description: "Pure functions, small files, and barrel-driven directory conventions."
---

## Code Structure

### 13. Pure Functions in Small Files

Pure functions in small files with single main function. Helper functions rarely needed but allowed. More often helpers are reusable pure functions themselves and should be in utils/

### 14. File Size

Files should not exceed the low 100s of lines of code in size, though this can be larger for files like React components, which have a lot of tags. Here's some perspective: I am a human, I've never seen the source, I know about Fastify, I prefer to follow a route in and out, understanding the code to enhance or fix it. Anywhere along this path if I run into a file that LOTS of functions or a single function with LOTS of lines and concerns my cognitive burden increases and my effectiveness diminishes.

### 15. Barrel-Driven Directory Structure

A standard, barrel driven directory structure shall be used and mapped in the tsconfig with directory paths where @ is the src/ dir and several allowable first class paths follow: @utils to /src/utils and @utils/\_ to /src/utils/\_ and so on for model, services, state, components, pages, screens, commands, routes, providers, hooks, api, plugins, db, and so on. It depends on the repo type.

### 16. Subdirectory Grouping

Subdirectories beneath those of Principle 15 should be well-used where each subdirectory groups a common theme. Example: src/model/user/\*.ts, where this directory contains all the model files that describe a user. Reason: code is read in browsers and this makes for very clean links.

### 17. File Naming

Prefer file names with lower case, hyphen separated. Example src/model/user/user-context.ts

### 18. Testing

Very good unit tests and e2e testing should be coded with every change. Cypress preferred for Web e2e.
