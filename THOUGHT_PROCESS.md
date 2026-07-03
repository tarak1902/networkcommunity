# Thought Process — Concurrency & Edge Cases

> **DRAFT.** This is a working draft of the concurrency/edge-case writeup. It captures the
> reasoning the design is built on; we can tighten the wording and add demo notes together
> before submission.

The central design choice — **server-authoritative state with a single broadcast** — is
what makes the hard parts (sync and concurrency) fall out almost for free. The notes below
name the situations explicitly, because naming them with a sensible answer is the part most
submissions skip.

## 1. Race condition — two receptionists click "Call Next" at the same instant

The queue lives **only on the server**, and Node.js is single-threaded: Socket.IO events
are processed **one at a time, in order**. So two near-simultaneous `callNext` events are
handled sequentially — the first does `queue.shift()` and takes token N; the second then
does `queue.shift()` and takes token N+1. From the clients' perspective each `shift()` is
atomic: the two clicks call the next **two** tokens in order, and the **same token is never
called twice**.

This is the concrete reason server-authoritative state matters. If each client held its own
copy of the queue and decided locally who's next, two clients could both pick the same
person. Centralizing the queue and the decision removes that possibility structurally — not
with locks, but by having a single place where the mutation happens.

## 2. Reconnection — a screen drops and rejoins

Socket.IO automatically attempts to reconnect a dropped client. On every `connection`
(including reconnects) the server immediately emits the full current state to that socket.
So a screen that lost Wi-Fi for a moment comes back **fully correct without a refresh** — it
doesn't replay missed events, it just receives the current truth. Because every broadcast
carries the entire state (not a diff), there's nothing to "catch up" on.

## 3. Single broadcast of the whole state

Every change emits one event, `queueUpdated`, with the entire state object, to all clients.
Sending the whole state each time is a deliberate trade: at this scale it's effectively
free, and it eliminates an entire class of bugs where partial updates arrive out of order or
a client misses one and silently drifts. There is exactly one code path that mutates state
and exactly one that ships it, which keeps the system easy to reason about.

## 4. Edge cases (named explicitly, handled in code)

- **Empty queue.** "Call Next" is disabled in the UI *and* the server treats `callNext` as a
  no-op when the queue is empty — the client is never trusted as the only guard. The patient
  screen shows an explicit "No one waiting" state.
- **First patient (nowServing was `null`).** Calling the first patient just assigns
  `nowServing = queue.shift()`; the previous `null` needs no special case. The patient screen
  renders a placeholder while `nowServing` is null.
- **Patient added mid-session.** `addPatient` appends to the queue and broadcasts, whether or
  not someone is currently being served — same code path either way.
- **Average time changed mid-queue.** Wait time is derived on the client
  (`position × avgConsultMin`), so changing the average rebroadcasts state and every wait
  recomputes live. Nothing stored needs migrating.
- **Name optional.** A walk-in can be added with no name; the entry still gets a token and is
  rendered as "walk-in." Patient identity is never required for the queue to function.

## 5. What we deliberately did *not* build

No database, auth, Redis, or client-side routing library. At this scope each would add
surface area and bugs without improving the core guarantee — live, consistent sync across
screens. Restraint keeps the part that's actually graded (real-time correctness) rock solid.
