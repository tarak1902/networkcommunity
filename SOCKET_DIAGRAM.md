# Socket Event Diagram

One server holds the queue (the single source of truth). Every change produces exactly one
broadcast — `queueUpdated` — carrying the **entire** state to **every** connected screen.
Both screens are dumb renderers of that state.

```
 RECEPTIONIST SCREEN            SERVER (source of truth)             PATIENT SCREEN
                                state = {
                                  nowServing: null,
                                  queue: [],
                                  avgConsultMin: 10,
                                  lastUpdated
                                }
                                nextToken = 1

  ── addPatient(name) ───────►  queue.push({ token: nextToken++, name })
                                broadcast queueUpdated ───────────────►  re-render
  ◄────────── queueUpdated ─────┤  (whole state to everyone)

  ── setAvgTime(min) ────────►  avgConsultMin = min
                                broadcast queueUpdated ───────────────►  re-render
  ◄────────── queueUpdated ─────┤                                        (waits recompute)

  ── callNext ───────────────►  nowServing = queue.shift()
                                broadcast queueUpdated ───────────────►  re-render (instant)
  ◄────────── queueUpdated ─────┤

  ── undoCall ───────────────►  queue.unshift(nowServing); nowServing = null
                                broadcast queueUpdated ───────────────►  re-render
  ◄────────── queueUpdated ─────┤

  (on connect / reconnect) ──►  socket.emit('queueUpdated', state) ────►  resync immediately
                                (full current state to the new socket)    (no refresh needed)
```

## The one rule that makes it correct

- **Exactly one broadcast event**, `queueUpdated`, carrying the **whole** state object.
- Sent to **everyone** on **every** change.
- Sent to a socket the **moment it connects or reconnects**.

Because of this, no screen can drift out of sync, and a screen that drops and rejoins is
instantly correct again — it just receives the current truth. Clients never compute or
store authoritative state; they only render what arrives.

## Events at a glance

| Direction        | Event          | Payload   | Server effect                                        |
| ---------------- | -------------- | --------- | ---------------------------------------------------- |
| client → server  | `addPatient`   | `name`    | push `{ token, name }` (name optional) → broadcast   |
| client → server  | `setAvgTime`   | `minutes` | set `avgConsultMin` (validated) → broadcast          |
| client → server  | `callNext`     | —         | `nowServing = queue.shift()` if non-empty → broadcast |
| client → server  | `undoCall`     | —         | put `nowServing` back at front → broadcast           |
| server → clients | `queueUpdated` | `state`   | the single broadcast of the full state object        |
