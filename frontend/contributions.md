# Frontend case study: a quota-aware bulk photo uploader in React

> **DRAFT. Requires confidentiality review and company approval before publishing.** Describes my own contribution to a private codebase; contains no company source code.

[← Back to portfolio](../README.md) · Companion: [Backend case study](../backend/contributions.md)

## 1. Overview

On an event-photography platform, a photographer uploads a whole event, often hundreds or thousands of full-resolution photos, and then waits for the platform to process them so guests can find their pictures. The upload screen is the first thing a paying customer touches and the place where things most visibly go wrong: browser tabs freeze, uploads stop when you navigate away, and a limit reached halfway through a batch leaves the UI claiming something that isn't true.

I designed and built the browser-side upload system that replaced page-local uploads with an **app-wide upload manager**. It prepares images in the browser, uploads them directly to object storage with a bounded pool of parallel workers, reports speed and time remaining, handles a server-enforced quota in the middle of a batch, and then tracks server-side processing until it finishes.

It took shape over about three months of iteration (November 2025 to February 2026), building on earlier work I did on chunked uploads for large files (June to September 2025).

## 2. Engineering problem

1. **Uploads belonged to a page.** When uploads were driven by a component, navigating away or re-rendering made progress disappear or restart. A photographer wants to keep working on another event while photos upload.
2. **Original files are large and the browser is a constrained machine.** Decoding hundreds of multi-megapixel images and holding them in memory at the same time would freeze the tab. Uploading them serially would take far too long.
3. **The server must not carry the bytes.** The backend had previously crashed when processing uploads in its own memory (see the backend case study), so the client had to upload straight to storage using short-lived presigned URLs, which means the client has to coordinate with the API on *every batch*.
4. **A quota is enforced by the server, in the middle of a batch.** A free-tier event can accept only so many photos. For example, a request for 300 files when 40 slots remain should upload 40, tell the user clearly, and not claim that 300 photos are "being matched".
5. **Processing is asynchronous.** After the files land in storage, the platform processes them in the background. The UI has to show that stage truthfully, even after a page reload.

## 3. My responsibilities

I owned this feature end to end on the front end: the upload manager, the React context and indicator that expose it to every screen, the upload page, the quota-limit UX, and the integration with the API contracts for batch URL issuance, upload confirmation and processing status. I also built the server side of the same flow (see the backend case study), so the endpoint shapes were designed with both ends in mind.

## 4. Technical implementation

**State lives above the router.** The upload manager is created inside a React Context provider mounted at the application root, and a small indicator component reads from it. Because it isn't owned by a page, uploads continue while the user navigates, and any screen can start an upload through the context.

**A queue drained by a bounded worker pool.** Files go into a queue. A fixed number of workers (five) each repeatedly take the next item until the queue is empty. This gives concurrency without the "fire everything with `Promise.all`" failure mode, where hundreds of simultaneous decodes and requests overwhelm the browser.

```js
// Illustrative pattern only (not production code).
async function drain(items, concurrency, handle) {
  let next = 0;
  const worker = async () => {
    while (next < items.length && running) {
      const item = items[next++];
      try { await handle(item); } catch (e) { recordFailure(item, e); }
    }
  };
  await Promise.all(Array.from({ length: Math.min(concurrency, items.length) }, worker));
}
```

**Presigned URLs are requested in batches** (twenty files at a time) rather than one request per file or one giant request for the whole event. This keeps requests small, lets the server make quota decisions per batch, and means the next batch's URLs are fetched only if the previous one was accepted.

**Image preparation in the browser.** Each file is decoded with `createImageBitmap`, drawn to a canvas, and encoded as two JPEG variants: a display-size version with the long edge capped, and a square, center-cropped thumbnail. Both variants upload in parallel. The decoded bitmap is explicitly closed in a `finally` block so memory is released even when a step throws. Moving this work to the client also let me remove a server-side resize step (see the backend case study).

**Uploads use `XMLHttpRequest`, not `fetch`,** because XHR exposes upload progress events. The manager aggregates bytes sent, derives throughput and estimated time remaining from deltas, and reports active, queued, completed and failed counts, which feed the progress indicator.

**Quota handling is a first-class error type.** When the API answers that a batch exceeds the event's allowance, the manager raises a dedicated error carrying the current usage, stops requesting further batches, and lets files already accepted finish. A modal explains what happened and offers the upgrade path. If the server accepts only part of a batch, the accepted files proceed normally.

**Confirmation and processing status.** When uploads finish, the client confirms them to the API, which creates a processing job. The client then polls the job status once a second and shows folder-level progress. The job identifier is stored in `localStorage`, so after a page reload the app re-attaches to a job that is still running instead of showing nothing.

## 5. Architectural decisions

| Decision | Why | Trade-off I accepted |
|---|---|---|
| Upload manager in a root-level context, not a page hook | Uploads must outlive navigation; one source of truth for every screen | A long-lived object holds references to files; the manager needs explicit cancel and cleanup paths |
| Direct-to-storage upload with presigned URLs | Keeps file bytes off the API; scales with the client count, not the server's memory | The client has more responsibility: it must prepare images, coordinate batches and confirm completion |
| Batched URL issuance (20) | Lets the server enforce the quota per batch and fail early | More round trips than a single request for the whole event |
| Bounded worker pool (5) | Predictable memory and network use in the browser | A fixed number isn't adaptive; a very fast or very slow connection could use a different value |
| Image variants generated on the client | Removes CPU-heavy work from the server | Output quality depends on browser canvas encoding; older browsers need the `createImageBitmap` path |
| Dedicated quota error type | Keeps the "limit reached" path out of the generic error handler and drives a specific UX | One more error contract that must be kept in sync with the server |

## 6. Challenges and how I solved them

**Making large-batch uploads not freeze the tab.** The first versions of the page processed and uploaded in separate phases. I restructured the flow so processing and uploading happen in the same worker loop, one image at a time per worker, and reduced the work done per file (including dropping a third image variant that the product no longer needed). The result is that memory stays bounded by the pool size and not by the batch size. *I did not run benchmarks, so I make no claim about how much faster it is.*

**Quota hit mid-batch.** The tricky part is not the modal; it's making sure the rest of the application's state stays consistent. A late fix I made (February 2026) addressed a case where the server rejected a batch completely (no slots left): the client still called the confirm step and showed a "matching in progress" state for photos that were never uploaded. I added an early return that applies only when the limit was hit *and* nothing was accepted, and left the partially-accepted path untouched. I framed the change around three cases (normal flow, partial approval, full rejection) and scoped it so only the full-rejection case behaves differently.

**Keeping the user oriented after a reload.** Persisting just the processing-job identifier gave a cheap, robust recovery for the long tail of the process (server-side processing) without trying to resume half-transferred files.

## 7. Results and impact

What I can state from the codebase and its history:

- Uploads continue while the photographer moves around the application; progress, speed and time remaining are visible from every screen.
- The browser sends files directly to object storage; the API only issues URLs, confirms and reports status, so a large upload no longer passes through the application server.
- A free-tier limit reached part-way through a batch results in a partial upload, a clear explanation and consistent UI state, rather than a failed batch or a false "processing" indicator.
- This system is the front end of the pipeline described in the [backend case study](../backend/contributions.md); the two were designed against each other.

What I cannot state: I have no measurements (throughput, error rates, time-to-complete) that I'm able to share, and the repository contains no performance benchmarks. I'd rather say that than invent numbers.

## 8. Limitations, and what I would do next

- **No per-file retry.** A failed file is recorded as failed. A production-grade version should retry transient failures with backoff and let the user re-queue failures.
- **Transfers are not resumable.** Only the processing stage is re-attached after a reload. Multipart uploads would make large files resumable.
- **No automated tests for the manager.** I would extract the queue and pool logic into a pure module and unit-test it (ordering, concurrency limit, quota stop, cancel).
- **Debug logging.** The manager logs heavily during development; this should be gated behind a flag for production.

## 9. Key engineering takeaways

1. Put long-lived work above the router, and make its lifecycle (start, progress, cancel, cleanup) explicit.
2. A bounded worker pool beats `Promise.all` for anything that touches memory-heavy resources such as images.
3. Treat expected business failures (like quotas) as typed errors with their own UX, not as generic exceptions.
4. When you change an edge-case path, enumerate the cases first. The late fix was six lines because the cases were clear.
5. Moving work to the client is a trade, not a free win: it relocates complexity, and the interface contract between client and server becomes the thing to design carefully.
