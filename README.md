# Adarsh KM: Full-Stack Engineering Portfolio

> **DRAFT. Requires confidentiality review and company approval before publishing.**

I'm a full-stack software engineer. I build React front ends and Node.js / Python back ends, and I like the unglamorous parts: queues, quotas, upload paths, auth flows and failure handling.

From 2023 to 2026 I worked on the engineering of an **event-photography platform**: a photographer uploads thousands of photos from an event, and each guest registers with a selfie and receives only the photos they appear in. I worked across the web client, the API, the processing pipeline, a desktop application and the marketing site.

**The company's repositories are private.** Nothing here is source code from them. This repository contains written case studies of work I did personally, with the architecture described at a level the company has approved for public discussion. Where I worked alongside other developers, I say so, and I only claim what I built.

---

## Selected work

| | Case study | One-line summary |
|---|---|---|
| **Frontend** | [Quota-aware bulk photo uploader in React](frontend/contributions.md) | An app-wide upload manager that resizes images in the browser, sends them straight to object storage through a bounded pool of parallel workers, and handles a server-side quota mid-batch without corrupting UI state. |
| **Backend** | [Image-ingestion pipeline: from request-path processing to direct-to-storage uploads](backend/contributions.md) | Five stages over roughly eight months that took a Node API from processing uploads in memory (and crashing) to issuing presigned URLs, tracking processing jobs, and enforcing quotas with atomic database operations. |

The two case studies describe the two ends of the same system, so they can be read together.

## What I personally owned

- **Front end:** most of a React + Vite single-page application (roughly 25,000 lines still in the codebase were written by me), including the upload system, the guest registration and selfie-capture flow, the photographer's event workspace, and the gallery / selection screens.
- **Back end:** the large majority of a Node.js (Express) API (roughly 17,000 surviving lines), including the upload pipeline, authentication, face-recognition integration and job workers, and subscription / quota enforcement.
- **Desktop product and web presence:** part of a local-first desktop application and its cloud relay (sign-in with OAuth 2.0 / OIDC + PKCE, an event-lifecycle state machine, relay hardening), co-developed with a colleague, and the public marketing site (static React with prerendering and per-route SEO).

These are my own measurements of authorship from version-control history; I am happy to walk through them in an interview.

## Technologies demonstrated

| Area | Technologies |
|---|---|
| Front end | React, Vite, React Router, Redux Toolkit, TanStack Query, Tailwind CSS, Context API, Canvas / `createImageBitmap`, `getUserMedia`, XHR upload progress, static prerendering (SSG) |
| Back end | Node.js, Express, MongoDB / Mongoose, Redis, Bull (job queues), AWS S3 (presigned URLs), AWS Lambda, SQS, Rekognition, Sharp, Socket.IO, Winston |
| Auth and security | JWT, OAuth 2.0 / OIDC with PKCE, RS256 / JWKS verification, rate limiting, input validation |
| Delivery | PM2, GitHub Actions CI/CD |
| Practices | Atomic database updates, idempotent job design, optimistic UI updates, graceful degradation, documentation of upload architecture |

## How to read these case studies

Each one covers the problem, my responsibilities, the implementation, the architectural decisions, the challenges, the outcome and what I would change. I include the limits of the work on purpose. For example, I did not benchmark the upload pipeline, so I don't quote speed-up numbers.

## Contact

- Email: *[add]*
- LinkedIn: *[add]*
