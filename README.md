# microservices-docker-k8s-mongodb

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

An event-driven microservices system built **from scratch** — no framework
magic — to learn how service decomposition, an event bus, and a CQRS-style
read model actually fit together.

It's a tiny blog (posts + comments + moderation), deliberately
over-architected: the point isn't the blog, it's the architecture around it.

## Status & honest limitations

Earlier versions of this README described the finished vision. The code
today is a **working local demo with a partial Kubernetes story**:

- **No MongoDB, despite the repo name.** There is not a single MongoDB or
  Mongoose reference in the codebase. Every service keeps its state in
  in-memory JavaScript objects (`const posts = {}`, …). Restart a service
  and its state is gone — with one exception below.
- **Services talk to `localhost`, so it runs as local processes.**
  Posts → event-bus is `http://localhost:4005/events`, the bus fans out to
  `localhost:4001/4002/4003`, the React client calls `localhost:4000`
  (posts) and `localhost:4002` (query) directly. Inside separate Kubernetes
  pods none of that resolves — cluster DNS names would be needed.
- **Kubernetes is one manifest, not a deployment.** Every service has a
  Dockerfile, but `infra/k8s/` contains exactly one file —
  `posts-depl.yaml`, a single-replica Deployment for posts (image
  `amancalledaman/posts`) with no Service object. There are no manifests
  for the other five services, no ClusterIP Services, and no Skaffold
  config (an earlier README said `skaffold dev` — there is nothing here
  for Skaffold to run). Finishing the K8s story means: replace the
  localhost URLs with service DNS, then write the missing manifests.
- **Moderation is a keyword demo** — a comment is rejected if its content
  contains the word `orange`, approved otherwise.
- **No tests, no CI.** The event log itself is an in-memory array in the
  event-bus process, so event replay works only while the bus stays up.

## Architecture

```
                    ┌─────────────┐
                    │   client    │  React frontend (CRA)
                    └──────┬──────┘
                 writes    │    reads
              ┌────────────┴────────────┐
              ▼                         ▼
          ┌───────┐                 ┌───────┐
          │ posts │                 │ query │  read model (CQRS-style)
          └───┬───┘                 └───▲───┘
              │ PostCreated             │ all events
              ▼                         │
          ┌───────────┐   fan-out   ┌───┴──────┐
          │ event-bus │────────────▶│ comments │
          │  (:4005)  │────────────▶│  (:4001) │
          └─────┬─────┘             └──────────┘
                │                   ┌──────────┐
                └──────────────────▶│moderation│  keyword rule,
                                    │  (:4003) │  emits verdicts
                                    └──────────┘
```

## Services

| Service | Port | Responsibility |
|---|---|---|
| `client/` | 3000 (CRA dev) | React frontend — create posts/comments, render the query model |
| `posts/` | 4000 | Create posts, emit `PostCreated` |
| `comments/` | 4001 | Create comments (`pending`), emit `CommentCreated`; apply moderation verdicts, emit `CommentUpdated` |
| `query/` | 4002 | Consume every event, build the denormalized posts+comments read model |
| `moderation/` | 4003 | Consume `CommentCreated`, apply the keyword rule, emit `CommentModerated` |
| `event-bus/` | 4005 | Append each event to its in-memory log, fan out to comments/query/moderation, serve the log on `GET /events` |
| `infra/` | — | Per-service Dockerfiles + the single posts Deployment manifest |

## Ideas this demonstrates

- **Event-driven communication** — services never call each other directly
  for writes; they publish facts (`PostCreated`, `CommentCreated`,
  `CommentModerated`, `CommentUpdated`) and react to the ones they own.
- **CQRS-lite** — writes go through posts/comments; reads come only from
  the query service's event-built view.
- **Event replay** — on startup the query service fetches the full event
  log (`GET :4005/events`) and replays it, rebuilding its read model from
  scratch. This is the one piece of state that survives a restart — and
  only while the event-bus process itself hasn't restarted.

## Run it (locally — the mode that works today)

Start each service in its own terminal:

```bash
for s in event-bus posts comments moderation query; do
  (cd $s && npm install && npm start &)
done
cd client && npm install && npm start   # UI on :3000
```

Open http://localhost:3000, create a post, add comments — and try one
containing the word "orange" to watch moderation reject it through the
event flow.

## License

MIT — see [LICENSE](LICENSE).

---

*One of [Aman Thakkar's](https://github.com/amanthakkar) architecture
learning builds, documented as it actually is.*
