# Queue Cure '26 — Real-time Clinic Queue

Two synchronized screens for a clinic waiting room:

- **Receptionist console** (`/receptionist`) — add patients, set the average consultation
  time, and call the next patient.
- **Patient waiting room** (`/patient`) — shows who's being served now and the estimated
  wait for everyone waiting.

Both screens update **live, the instant the receptionist acts — no page refresh.**

---

## What it does

The receptionist adds patients (each gets a sequential token), optionally sets the average
consultation time, and presses **Call Next** to move the front of the queue into "now
serving." Every connected screen — receptionist or patient, in any tab or device — reflects
the change immediately over a WebSocket. The patient screen shows each waiting patient's
estimated wait, **computed live** from the queue position and the receptionist-set average.

---

## Tech stack

| Layer       | Choice                          |
| ----------- | ------------------------------- |
| Backend     | Node.js + Express               |
| Real-time   | Socket.IO (WebSockets)          |
| State       | In-memory object on the server  |
| Frontend    | React + socket.io-client        |
| Styling     | Tailwind CSS                    |
| Build tool  | Vite                            |

No database, no auth, no Redis — deliberately. At this scope they would add bugs and time
without improving correctness.

---

## Architecture note

**The server holds the queue as the single source of truth. Clients are dumb renderers.**

- There is exactly **one** broadcast event, `queueUpdated`, carrying the **entire** state
  object, emitted to **all** clients on **every** change. No partial diffs.
- On connect/reconnect the server immediately emits the full current state, so a rejoining
  screen resyncs instantly without a refresh (Socket.IO auto-reconnects).
- Wait time is never stored or hardcoded — it's derived on the client from broadcast state:
  `estimatedWait(position) = position × avgConsultMin`. Change the average on the
  receptionist screen and every patient's wait recomputes live.

The whole frontend and the Socket.IO server are served from **one Express server on one
origin**, so there are no CORS issues and it deploys as a single service.

Broadcast state object:

```js
{
  nowServing: { token, name } | null,
  queue: [ { token, name } ],   // patients waiting, in order
  avgConsultMin: 10,            // default
  lastUpdated: <timestamp>
}
```

Socket events the server handles: `addPatient`, `setAvgTime`, `callNext`, `undoCall`, and
full-state resync on `connection`. See [`SOCKET_DIAGRAM.md`](./SOCKET_DIAGRAM.md) for the
event flow and [`THOUGHT_PROCESS.md`](./THOUGHT_PROCESS.md) for the concurrency/edge-case
writeup.

---

## Run locally

Prereqs: Node.js 18+.

### Option A — production-style (one server, one origin)

```bash
npm install          # install server deps
npm run build        # installs client deps and builds the React app into client/dist
npm start            # serves the app + Socket.IO on http://localhost:3001
```

Then open two windows side by side:

- http://localhost:3001/receptionist
- http://localhost:3001/patient

### Option B — dev with hot reload (two processes)

```bash
npm install
npm run dev:server   # Express + Socket.IO on :3001
npm run dev:client   # Vite dev server on :5173 (proxies /socket.io to :3001)
```

Open http://localhost:5173/receptionist and http://localhost:5173/patient.

---

## Deploy to Render (single service)

1. Push this repo to GitHub.
2. Create a new **Web Service** on Render pointing at the repo.
3. Settings:
   - **Build Command:** `npm install && npm run build`
   - **Start Command:** `npm start`
   - Render provides `PORT` automatically; the server reads `process.env.PORT`.
4. Render supports persistent WebSockets, so Socket.IO works out of the box. The single
   origin means no CORS configuration.

> Free-tier note: Render spins the service down after idle and cold-starts in ~30–50s.
> Wake it just before a demo. A short screen recording of two windows syncing is good
> insurance against a cold start during review.

---

## Verify the two make-or-break behaviors

1. **Live sync (no refresh):** open `/receptionist` and `/patient` side by side. Add
   patients and click **Call Next** on the receptionist — the patient screen updates
   instantly. Open a third tab; it resyncs to the current state on load.
2. **Computed wait:** change the average consultation time on the receptionist screen —
   every patient's estimated wait recomputes immediately (it's derived, not a static label).
