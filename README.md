# SSC Architecture Playbook

**Service + Store + Coordinator** — a practical SwiftUI architecture pattern.

This playbook explains how to structure the layer between your views and your backend in a way that stays clean as your app grows. It's one opinionated approach, born out of building a real app (Rally), not a theoretical exercise.

Two documents:

- **[playbook.md](playbook.md)** — the architecture: what the three layers are, how they connect, when to use each one, and the mistakes worth avoiding.
- **[conventions.md](conventions.md)** — how it looks in code: file layout, MARK groupings, environment wiring, doc comments, error handling.
