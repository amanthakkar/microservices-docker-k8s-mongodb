# microservices-docker-k8s-mongodb

An event-driven microservices system built **from scratch** — no framework magic — to learn how Docker, Kubernetes, and an event bus actually fit together.

It's a tiny blog (posts + comments + moderation), deliberately over-architected: the point isn't the blog, it's the architecture around it.

## Architecture

```
                    ┌─────────────┐
                    │   client    │  React frontend
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
          ┌───────┐   ┌──────────┐  ┌───────┐
          │ posts │   │ comments │  │ query │  read model (CQRS-style)
          └───┬───┘   └────┬─────┘  └───▲───┘
              │            │            │
              └─────┬──────┘            │
                    ▼                   │
              ┌───────────┐   events    │
              │ event-bus │─────────────┘
              └─────┬─────┘
                    ▼
              ┌────────────┐
              │ moderation │  flags comments, emits verdict events
              └────────────┘
```

## Services

| Service | Responsibility |
|---|---|
| `client/` | React frontend |
| `posts/` | Create posts, emit `PostCreated` events |
| `comments/` | Create comments, emit `CommentCreated` events |
| `moderation/` | Consume comment events, moderate, emit `CommentModerated` |
| `query/` | Consume all events, build the denormalized read model the client actually queries |
| `event-bus/` | The dumb, honest center: receives events, fans them out to every service |
| `infra/` | Dockerfiles + Kubernetes manifests for every service |

## Ideas this demonstrates

- **Event-driven communication** — services never call each other directly for writes; they publish facts.
- **CQRS-lite** — writes go through posts/comments; reads come from the query service's event-built view.
- **Event replay** — the query service can rebuild its state by replaying the event bus history after a restart.
- **Kubernetes basics that matter** — a Deployment + ClusterIP Service per microservice, config in `infra/k8s/`.

## Run it

With Kubernetes (Skaffold drives the build/deploy loop):

```bash
skaffold dev
```

Or run a single service locally:

```bash
cd posts && npm install && npm start
```

## License

MIT — see [LICENSE](LICENSE).
