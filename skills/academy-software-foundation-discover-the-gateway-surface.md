---
name: discover-the-gateway-surface
description: >-
  Work out what a specific OpenCue REST Gateway deployment can actually do, before calling it —
  including which documented endpoints that build does not route.
api: academy-software-foundation:academy-software-foundation-jobs-api
generated: '2026-08-29'
method: generated
source: >-
  https://github.com/AcademySoftwareFoundation/OpenCue/blob/master/docs/_docs/reference/rest-api-reference.md
  and https://github.com/AcademySoftwareFoundation/OpenCue/blob/master/docs/news/2026-08-26-rest-gateway-swagger-ui.md
operations:
  - GET /swagger/
  - GET /swagger/specs/{proto}.swagger.json
---

# Discover the gateway surface

Because OpenCue is self-deployed, no two gateways necessarily expose the same thing: the definitions
are generated at Docker build time from the `.proto` files that build was cut from. Ask the running
gateway rather than assuming.

## Steps

1. **`GET /swagger/`** — unauthenticated. Returns the Swagger UI and, in its definition menu, the
   list of documents this binary was built with. This is the only route on the gateway that does not
   require a JWT.
2. **`GET /swagger/specs/<proto>.swagger.json`** — also unauthenticated. Fetch the OpenAPI 2.0
   document for one proto, e.g. `show.swagger.json`. Eighteen documents cover 304 endpoints. A
   single document can hold several interfaces: `job.swagger.json` holds `JobInterface` (42),
   `LayerInterface` (36), `GroupInterface` (20) and `FrameInterface` (18).
3. **Subtract the endpoints that are described but not routed.** The definitions are generated with
   `generate_unbound_methods=true`, so they document every method in every proto, including six
   interfaces the gateway does not register. Calling one returns `404` with
   `{"code":5,"message":"Not Found","details":[]}` **even with a valid token**:

   | Interface | Why it is not routed |
   |---|---|
   | `CueInterface` | Internal Cuebot statistics |
   | `MonitoringInterface` | Farm history is read from Prometheus/Grafana instead |
   | `RenderPartitionInterface` | Not registered on the gateway |
   | `RqdReportInterface` | Agent-facing; RQD calls it to report in |
   | `RqdInterface` | Implemented by the RQD agent on each host, not by Cuebot |
   | `RunningFrame` | As above |

   273 of the 304 endpoints, across 22 interfaces, are actually routed.
4. **If `/swagger/` returns `401`**, the documentation routes are disabled on this deployment or
   `SWAGGER_DIR` does not exist, and the request fell through to the authenticated handler. Fall
   back to the `.proto` files in `grpc/` — they are the same source the definitions are generated
   from.

## Note on the OpenAPI documents in this repository

The four files in `openapi/` were **written by API Evangelist from documentation**, not published by
ASWF, and they describe resource-shaped paths (`/api/host`, `/api/show/{show_id}/job`) that this
gateway does not serve. Do not build a client from them. The contract of record is `grpc/`, and the
authoritative HTTP description is whatever the running gateway serves at `/swagger/specs/`.
