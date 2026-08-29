---
name: triage-a-stuck-render-job
description: >-
  Find out why an OpenCue render job is not finishing and take the smallest corrective action, using
  the OpenCue REST Gateway. Read first, act second, and prefer the reversible action every time.
api: academy-software-foundation:academy-software-foundation-jobs-api
generated: '2026-08-29'
method: generated
source: >-
  grpc/academy-software-foundation-opencue-job.proto (verbatim from
  github.com/AcademySoftwareFoundation/OpenCue) and
  https://github.com/AcademySoftwareFoundation/OpenCue/blob/master/docs/_docs/reference/rest-api-reference.md
operations:
  - JobInterface.GetJobs
  - JobInterface.GetJob
  - JobInterface.GetFrames
  - JobInterface.GetComments
  - JobInterface.Pause
  - JobInterface.Resume
  - JobInterface.RetryFrames
  - JobInterface.KillFrames
  - JobInterface.EatFrames
---

# Triage a stuck render job

## Before you start

- **The base URL is the studio's, not ASWF's.** ASWF operates no OpenCue instance. The gateway runs
  wherever the studio deployed it; the documented default is `http://your-gateway:8448`.
- **Every call is `POST`**, including reads. The path is derived mechanically from the protobuf
  service and method: `POST /job.JobInterface/GetJobs`. There are no resource-shaped REST paths.
- **Every call needs a JWT.** `Authorization: Bearer <token>`, HS256, claims `sub` and `exp`.
  A missing header is `401`; an expired or bad token is `403`.
- **There is no idempotency key and no dry-run mode.** A retried write may fire twice. Read the
  state back rather than assuming a write landed.

## Steps

1. **Find the job.** `POST /job.JobInterface/GetJobs` with the show's search criteria in the body.
   Send `{}` for an unfiltered call. Field names are camel-cased from the protobuf message.
2. **Read its current state.** `POST /job.JobInterface/GetJob` with the job id. Note whether it is
   paused, and what its priority and resource ceilings are.
3. **Look at the frames, not just the job.** `POST /job.JobInterface/GetFrames`. A job that looks
   stuck is usually a small set of frames in a dead or waiting state. Use the frame search criteria
   fields on the request message to narrow — there is **no pagination**, so an unfiltered call on a
   large job returns the whole collection.
4. **Read the comments before changing anything.** `POST /job.JobInterface/GetComments`. Someone may
   have already paused the job deliberately, and the reason is usually recorded here.
5. **Choose the smallest action.**
   - Frames died on a transient failure → `POST /job.JobInterface/RetryFrames`. This is the reversible
     one: it re-queues the frames.
   - Frames are wedged and holding cores → `POST /job.JobInterface/KillFrames`, then
     `RetryFrames` when the cause is fixed.
   - Frames should never run → `POST /job.JobInterface/EatFrames`. Reversible via `RetryFrames`, but
     it discards the work, so confirm with a human first.
   - The whole job should stop consuming the farm → `POST /job.JobInterface/Pause`, reversed by
     `POST /job.JobInterface/Resume`.
6. **Verify.** Re-run step 2 or 3. Nothing in this API returns a confirmation you can trust without
   reading state back.

## What you must not do

- **Never call `JobInterface.Kill` to "fix" a job.** Killing a job is the one action in this flow
  with no reversal — the job must be resubmitted from scratch, and any progress is gone. Escalate to
  a human instead.
- **No reversal here has a published window.** OpenCue documents *that* Retry, Resume and Unlock
  exist; it does not state a deadline inside which they still work. Do not tell a user an action can
  be undone "within N hours" — that number does not exist.

## Errors you will actually see

| Status | Body | What it means |
|---|---|---|
| `401` | — | No `Authorization` header |
| `403` | — | Token invalid or expired |
| `404` | `{"code":5,"message":"Not Found","details":[]}` | The resource is missing **or** the interface is one the gateway does not route |
| `500` | — | Gateway or Cuebot failure — retry with backoff |

Rate limiting is 100 requests/second per client by default, signalled with `X-RateLimit-Limit`,
`X-RateLimit-Remaining` and `X-RateLimit-Reset`. The status code returned on exhaustion is not
documented.
