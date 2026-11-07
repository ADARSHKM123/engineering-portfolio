# Backend case study: re-architecting an image-ingestion pipeline

> **DRAFT. Requires confidentiality review and company approval before publishing.** Describes my own contribution to a private codebase; contains no company source code, endpoints, identifiers or infrastructure names.

[← Back to portfolio](../README.md) · Companion: [Frontend case study](../frontend/contributions.md)

## 1. Overview

An event-photography platform lives or dies on one flow: a photographer uploads a large batch of high-resolution photos, the backend turns each into display-ready versions, and guests are matched to their photos by face recognition. Every other feature depends on that ingestion path being reliable.

Over roughly eight months (June 2025 to February 2026) I took that path through **five architectural stages**, each driven by a concrete failure of the previous one: from the API server receiving and resizing images in memory, to chunked uploads, to a hardened job queue, to clients uploading **directly to object storage with presigned URLs**, to server-tracked processing jobs with **atomic quota enforcement**. I made these changes in a Node.js (Express) API with MongoDB, Redis-backed job queues and AWS services, and I wrote the large majority of the surviving code in that service.

## 2. Engineering problem

The first working version accepted uploads through the API process: multipart in, resize in memory, store, queue. That is fine for a demo and fails with real events:

- **Memory pressure.** Decoding and resizing several large images at once in the Node process caused the server to crash or return gateway errors under load.
- **Request lifetime.** Long uploads outlived router time limits, so requests were cut off.
- **Disk and temp files.** Staging data on local disk without cleanup eventually fills the disk.
- **Queue reliability.** Background jobs need retries, bounded retention, unique identities and a Redis connection that behaves during start-up and brief outages.
- **Business rules.** A free tier limits uploads per event. Enforcing a limit while many requests are in flight is a race-condition problem, not an `if` statement.
- **Observability of async work.** The client needs to know when processing finishes, and "done" must mean the server's view, not a guess.

## 3. My responsibilities

I owned the ingestion path end to end on the server: the upload endpoints, the staging and queue design, the storage integration, the processing-job model, the quota middleware, and the operational configuration (process manager, logging, disk monitoring). I also built the browser side of the final design (see the [frontend case study](../frontend/contributions.md)), so the API contract and the client were shaped together.

## 4. Technical implementation

### Stage 1: in-request processing (June 2025)
Upload endpoints accepted files, produced several image sizes using Sharp (progressive JPEG original, cropped thumbnail, mid-size version), uploaded to object storage and used a Bull queue for background work. It worked, and it is the baseline whose failures drove everything after.

### Stage 2: chunked uploads (September 2025)
To get large files through without one huge request, I added an initialize / receive-chunk / merge protocol: the server issues an upload ID, stores chunks in a per-upload directory, reports progress, and merges asynchronously when the last chunk arrives. I raised the upload size ceiling, and I added a disk-usage monitor that checks every few minutes and triggers an emergency cleanup of temporary files when usage becomes critical, because staging data on disk is a liability if nothing removes it.

### Stage 3: making the queue trustworthy (October 2025)
Under real use the server still crashed during uploads, so I raised router time limits and then worked through the queue layer:
- **Unique job identities** (UUID + timestamp + random component) so concurrent jobs cannot collide.
- **Retries with exponential backoff** (three attempts), and **bounded retention** of completed and failed jobs so Redis does not grow without limit but there is still history to debug with.
- **Redis connection behavior:** commands queue while connecting, retries are bounded, environment variables load before anything else in the process, and the Redis initialization path logs what it is doing so production start-up problems can be diagnosed from logs.
- **Cluster-safe keys.** Bull performs multi-key operations, and in a Redis Cluster the keys involved must hash to the same slot. I gave the queues a **hash-tagged key prefix** so all of a queue's keys land in one slot.
- A PM2 process configuration (multiple processes in cluster mode, memory-based restarts) so the service restarts predictably.

### Stage 4: presigned direct-to-storage uploads (October to November 2025)
The decisive change: **stop receiving file bytes at all.** The API authenticates the user, validates and sanitizes file names, and returns short-lived presigned PUT URLs; the browser uploads straight to object storage; the browser then confirms completion; the server triggers image-processing functions **in batches** (ten keys per invocation, per-batch timeout, short spacing between batches, per-batch results recorded so partial failure is visible). Because the client now prepares the display image and thumbnail, I removed the server's resize step and the mid-size variant, which eliminated the CPU and memory work from the request path.

### Stage 5: processing jobs and quota enforcement (January to April 2026)
- **A processing-job document in MongoDB** with a status, per-folder counters and an error list, plus a **TTL index** that expires old jobs after 24 hours. The client polls a status endpoint backed by this record.
- **Batch URL issuance** is the quota checkpoint. For a free-tier event, the handler computes available slots and **approves up to that many files (partial approval)**. It reserves them with an **atomic conditional update**: the increment applies only if the counter is still low enough, and if another request won the race, the handler re-reads and approves what is actually left. If nothing remains, it answers with a structured payload (current usage, limit, upgrade hint) that the client turns into a modal.
- Subscription expiry is checked before the limit; paid and admin users bypass the limit. Later changes made monthly event counting consistent in UTC and added a per-event image counter to the data model.

## 5. Architectural decisions

| Decision | Why | Trade-off I accepted |
|---|---|---|
| Presigned direct-to-storage uploads | Removes file bytes and image processing from the API process; memory use no longer grows with upload size | More moving parts (client must confirm; storage events/triggers needed); URLs are bearer credentials until they expire |
| Atomic conditional update for quota reservation | Correct under concurrent requests without locks or transactions | Slots are reserved when URLs are issued, so a file that later fails to upload is not credited back in this version |
| Partial approval instead of all-or-nothing | A user near the limit still gets value; the client can show exactly what happened | A more complex response contract for the client to handle |
| Hash-tagged queue keys | Makes the queue safe on a clustered Redis | All of one queue's keys live on one shard, which limits horizontal scaling of that queue |
| Job record with a TTL | Gives the UI an authoritative status and cleans itself up | Status is only available for a limited time |
| Batching the processing triggers | Bounds per-call duration and makes partial failure observable | Throughput limited by batch size and spacing; sequential batches |

## 6. Challenges and how I solved them

- **A crash that only reproduced under real uploads.** The fix was not one change but a sequence: lift router limits to stop requests being cut, move bytes off the server, then harden the queue. Each stage is a commit series I can walk through.
- **Start-up and configuration order.** Services that read configuration from the environment break quietly if something initializes before the environment is loaded. I moved environment loading to the very start of the process and added logging around Redis initialization so production behavior is visible instead of guessed.
- **Queue keys spread across Redis Cluster slots.** Bull's keys for one queue were landing in different slots; a hash-tagged prefix puts them in one. The lesson was to check how a library's data model maps onto the deployment topology, not only whether it works on a single node.
- **Races on the free-tier counter.** A read-then-write check lets two simultaneous batches both pass. Making the increment itself conditional pushed the decision into a single atomic database operation.

## 7. Results and impact

What the repository supports:

- The API process no longer receives or resizes image bytes in the primary upload path; it issues URLs, confirms, triggers batches and reports status.
- Queue configuration is explicit about retries, backoff, retention and key placement, and start-up behavior is observable in logs.
- The free-tier limit is enforced atomically, with a well-defined partial-approval response that the client handles.
- Asynchronous processing has an authoritative, self-cleaning status record.

What I will not claim: the repository has no load tests or production metrics, and some internal documentation uses phrases like "zero memory" as design goals, not measurements. I do not quote throughput, latency or error-rate improvements.

## 8. Limitations, and what I would do next

- **Release reserved slots** when an upload fails or is abandoned (a reconciliation job or confirmation-time true-up).
- **Persist chunk-upload state.** Chunk metadata lives in process memory. It does not survive a restart, and because the service runs as several processes, it is only correct if every request for one upload reaches the same process. Redis or the database would fix that, and multipart uploads directly to storage would remove the need for server-side chunking, which is one more reason the direct-to-storage design is the better one.
- **Process uploads via storage events** (notifications to a queue) instead of an explicit client confirmation, so a closed tab cannot strand files.
- **Add automated tests** around the quota handler (concurrent requests, boundary values) and the processing-job state transitions.
- **Move secrets out of the repository** and into a secrets manager; I would also add secret scanning in CI.

## 9. Key engineering takeaways

1. Diagnose before you re-architect: each stage here is a response to an observed failure, and the final design (do not touch the bytes) only made sense after the earlier stages showed why.
2. Treat limits and quotas as concurrency problems; make the check-and-update a single atomic operation.
3. Background jobs need identity, retries, retention and a clear definition of "done" that the UI can trust.
4. Initialization order and environment configuration are real sources of production-only bugs; make start-up behavior observable.
5. Good designs name their trade-offs, and I would rather list the gaps than hide them.
