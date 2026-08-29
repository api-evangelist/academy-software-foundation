---
name: manage-farm-capacity
description: >-
  Inspect and adjust OpenCue render-farm capacity — hosts, procs, allocations and show subscriptions
  — through the OpenCue REST Gateway, keeping every change reversible.
api: academy-software-foundation:academy-software-foundation-hosts-api
generated: '2026-08-29'
method: generated
source: >-
  grpc/academy-software-foundation-opencue-host.proto,
  grpc/academy-software-foundation-opencue-facility.proto,
  grpc/academy-software-foundation-opencue-subscription.proto (verbatim from
  github.com/AcademySoftwareFoundation/OpenCue)
operations:
  - HostInterface.GetHosts
  - HostInterface.Lock
  - HostInterface.Unlock
  - ProcInterface.RedirectToJob
  - ProcInterface.ClearRedirect
  - AllocationInterface.GetHosts
  - ShowInterface.CreateSubscription
  - SubscriptionInterface.Delete
---

# Manage farm capacity

## Shape of the surface

`POST /host.HostInterface/GetHosts`, `POST /facility.AllocationInterface/GetHosts`, and so on — the
package name in the path is the `.proto` file the service is declared in, which is why allocation
calls live under `facility.` and proc calls under `host.`. Auth, error and rate-limit behaviour are
identical to every other gateway route.

## Steps

1. **See what exists.** `POST /host.HostInterface/GetHosts` for the whole farm, or
   `POST /facility.AllocationInterface/GetHosts` to scope to one allocation.
2. **Take a host out of service reversibly.** `POST /host.HostInterface/Lock` stops new work being
   booked onto it. Reverse it with `POST /host.HostInterface/Unlock`. Prefer this over deleting a
   host — `HostInterface.Delete` has no undo.
3. **Steer capacity at a job.** `POST /host.ProcInterface/RedirectToJob` (or `RedirectToGroup`)
   points a freed proc at specific work. Reverse it with
   `POST /host.ProcInterface/ClearRedirect`.
4. **Change a show's entitlement.** `POST /show.ShowInterface/CreateSubscription` binds a show to an
   allocation with a core count. `POST /subscription.SubscriptionInterface/Delete` removes it —
   there is no undelete, so read the subscription's current values first if you may need to restore
   them by re-creating it.

## Irreversible operations in this area

`Delete` exists on Host, Allocation, Facility, Owner, Deed and Subscription. **None of them has an
undelete, a trash state or a retention window.** Treat every `Delete` as permanent and require
explicit human confirmation naming the exact entity id.
