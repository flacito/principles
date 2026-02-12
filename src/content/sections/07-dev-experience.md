---
title: "Dev Experience"
order: 7
description: "Single command to satisfaction, standard ports, and operational simplicity."
---

## Dev Experience

### 32. Single Command to Satisfaction

Single command to satisfaction or help. Modern creative devs rarely read the README. "What command do I run?" in a DM is all they want. The dev command is the one command that runs everything needed to develop the system. If something isn't right on the developer's workstation, it tells them, they fix, they repeat. This means running a checklist script every time to make sure the workstation is good. If not, help. The single command is the chain of commands that used to appear in the README in big steps. Base setup, install, build, run — these are all phases of the DevX pipeline that need a script with checklist and help at each step.

### 33. Standard Ports, Custom Ports

Dev workstations have limited capacity and benefit from operational simplicity. When starting up workloads on a developer workstation, use common services as a preference. Postgres, MongoDB, CouchDB, Splunk, ChromaDB, etc. all have a standard default port. Use them. Web apps, APIs, etc. that are specific to the app get a PORT RANGE, but look like the default if possible. Example: Vite is 5173 should be 17173, keeping the last 3 the same if possible. There will always be collision. ChromaDB and Splunk on 8000 for instance. The dev env startup (FROM A SINGLE COMMAND) should reconcile this and configure away from the conflict as needed.
