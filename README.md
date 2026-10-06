# Windfall

**A multi-agent AI pipeline that rebuilds abandoned travel carts instead of
discounting them.**

COMPFEST 18 AIC Team 6 Eyes · Zhillan Baniaksa · Micguel Katili · Tania Ju

When a traveler abandons a booking, three agents run in sequence: a
**Classifier** reasons about who they are, a **Searcher** walks a rebuild
ladder looking for a cheaper comparable trip, and a **Notification Curator**
drafts the message. An analyst approves before anything is sent.

The thesis is restraint. The system is judged not on how many carts it
discounts, but on how many it recovers *without* discounting and on
deliberately sending no discount to travelers whom price was never blocking.

---

## Quick start

```bash
git clone https://github.com/ZhillanTLE/6EyesCOMPFEST.git
cd 6EyesCOMPFEST
docker compose up --build
```

The first build takes a few minutes. It is finished when the log shows the
backend listening on `:8000` and Next.js ready on `:3000`.

Needs **Docker Compose v2.24 or newer** (`docker compose version`). The backend
service declares its env file with the long form:

```yaml
env_file:
  - path: ./backend/.env
    required: false
```

`required:` arrived in v2.24. It is what lets a clean clone start with no
`backend/.env` at all; on an older Compose the same block is a parse error.
Docker Desktop 4.27+ ships a new enough Compose.

Then open **http://localhost:3000** for the landing page that explains the
decision logic, or **http://localhost:3000/recovery** for the console.

That is the whole setup. No API keys, no database, no accounts. The demo ships
with `WINDFALL_FIXTURES=1`, replaying a captured run from local files, and the
console labels replay mode visibly in the header.

To stop: `Ctrl-C`, then `docker compose down`.

### Three ways to run it

Prices and reasoning are **separate axes**, so they can be turned on
independently. That matters here: replaying prices needs no account at all,
while live prices need a Duffel signup and a paid RapidAPI subscription.

| Mode | Env | Keys needed | What is real |
|---|---|---|---|
| **Replay** (default) | `WINDFALL_FIXTURES=1` | none | nothing captured prices, captured reasoning. Fully offline. |
| **Live agents** | `WINDFALL_FIXTURES=1` + `WINDFALL_LIVE_INFERENCE=1` | `GEMINI_API_KEY` | **the three agents actually call Gemini**; prices still replay |
| **Fully live** | `WINDFALL_FIXTURES=0` | `GEMINI_API_KEY` + `DUFFEL_API_KEY` + `RAPIDAPI_KEY` | prices re-queried from Duffel and RapidAPI, agents call Gemini |

**Live agents is the interesting middle setting.** One free Gemini key is
enough to watch real inference run over the seeded carts, with no travel-API
credentials at all:

```bash
cp backend/.env.example backend/.env     # then set GEMINI_API_KEY
WINDFALL_LIVE_INFERENCE=1 docker compose up --build
```

Fully live calls Duffel for flights and RapidAPI for hotels. Prices and
therefore outcomes will differ from the captured run, which is the point:
the paper promises bookable prices rather than a simulation.

```bash
WINDFALL_FIXTURES=0 docker compose up --build
```

> **If a run returns HTTP 503, this is why.** Any mode that enables live
> reasoning refuses to start without `GEMINI_API_KEY`, rather than filling the
> trace with deterministic templates that look exactly like model output in a
> screenshot. The error message names the three ways out. It is a refusal, not
> a crash.

Whichever mode is active, the console header says so the run is labelled
with where its prices came from and where its reasoning came from, separately.

### Enabling email

Sending is off by default because a clean clone has no SMTP credentials, and a
silent failure is worse than an explicit suppression. To turn it on, set
`SEND_ENABLED=true`, `DEMO_RECIPIENT` and the `SMTP_*` values in
`backend/.env`.

Every send routes to `DEMO_RECIPIENT` whatever the traveler's stored address.
The seeded travelers are synthetic and their addresses are invented; delivering
to them would bounce. The intended traveler is recorded in an
`X-Windfall-Traveler` header so the routing stays visible.

**Fixtures do not disable sending.** They replace *inference*, not *delivery*.

---

## What you are looking at

Ten abandoned carts, taken from the design bundle's own seed table. Between
them they exercise every decision path.

| Traveler | Tier | Campaign share | Outcome |
|---|---|---|---|
| Ria Lavenia | Value | 46% | **rebuild** hotel down one star |
| Zhillan Baniaksa | Comfort | 48% | **rebuild** hotel down one star |
| Nasywa Namira | Premium | 9% | **reminder** no discount |
| Adriano Goran | cold start | | **alternative** different trip |
| Zayyan Ramadzaki | Comfort | 47% | **lateral** same-star swap |
| Christiano Hosea | Comfort | 31% | **error** carrier inventory down |
| Micguel Katili | Value | 45% | **rebuild** hotel down one star |
| Salsabilla Hasan | Premium | 11% | **reminder** no discount |
| Darius Sagala | Comfort | 43% | **alternative** different trip |
| Tania Ju | Value | 50% | **rebuild** hotel down one star |

**Compare the two Premium travelers with the seven high-share ones.** Nasywa
Namira and Salsabilla Hasan sit at 9% and 11%: their carts run over their usual
spend, and they are still sent nothing but a reminder, because price is not what
stopped them. Every traveler at or above 25% gets the ladder. One axis is never
enough on its own that is the product.

Every card shows a *provisional* tier estimate, labelled "Likely:". It is a
deterministic percentile lookup, not the Classifier's verdict, and the two can
differ. It is shown so you have something to test the model's reasoning
against.

---

## The decision

A price intervention requires **both** signals to agree:

```
c_i = campaignShare           share of past spend on discounted product
g_i = (p_0 - s_i) / s_i       how far the cart sits above usual spend

  rebuild ladder      if  c_i >= 0.25  AND  g_i > 0
  reminder            otherwise
```

A high campaign share alone never triggers a discount. Neither does a cart
merely being expensive.

When the ladder does run, it tries the smallest change first and stops at the
first attempt clearing the tier threshold (Value 5%, Comfort 10%,
Premium 15%):

1. **Re-price** the same cart
2. **Lateral** a comparable hotel at the same star rating, same dates, same area
3. **Tier-down** one star lower, destination and dates unchanged

The flight never changes. A Duffel Hold Order is held against one specific
flight offer, so swapping it would void the price guarantee.

**No outcome concedes partner margin.** A rebuild is cheaper because the *trip*
composition changed, not because anyone discounted. A test asserts this across
every path.

---

## Architecture

```
frontend (Next.js 16)  :3000   the console a client that calls Flask
backend  (Flask)       :8000   the synchronous recovery pipeline
```

The console is three screens: **browse** the open carts, watch the **pipeline**
reason over the one you picked, then read the **previews** of what the traveler
will receive. Approval lives on that third screen, so nothing is sent by
scrolling past a finished trace.

The whole pipeline runs inside **one request/response cycle**. No background
jobs, no queue, no scheduler, no distributed database the seed is a local
JSON file and the entire reasoning trace comes back in the response body.

The browser talks to `/api/recovery/*` on its own origin; Next forwards to
Flask, so there is no CORS and no backend hostname in the client bundle.

### Endpoints

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/recovery/queue` | abandoned carts awaiting a decision |
| `POST` | `/api/recovery/run` | run the pipeline, return the whole trace |
| `POST` | `/api/recovery/send` | deliver the approved notification |
| `GET` | `/api/recovery/health` | mode and cart count |

Running the pipeline never sends anything. Delivery needs its own explicit
approval click clicking through ten travelers to read their traces must not
put six emails in anyone's inbox.

---

## Running without Docker

Needs **Python 3.11+** and **Node 20+** (developed on Python 3.13 and Node 24).
Run both commands from the repository root, in two shells.

```bash
# shell 1 backend on :8000
pip install -r backend/requirements.txt
WINDFALL_FIXTURES=1 AUTH_DISABLED=true PORT=8000 python -m backend.app

# shell 2 frontend on :3000
cd frontend && npm install && npm run dev
```

On **Windows PowerShell** the env-var prefix above is a syntax error. Set them
first instead:

```powershell
# shell 1 backend on :8000
pip install -r backend/requirements.txt
$env:WINDFALL_FIXTURES = "1"; $env:AUTH_DISABLED = "true"; $env:PORT = "8000"
python -m backend.app

# shell 2 frontend on :3000
cd frontend; npm install; npm run dev
```

### Checks

```bash
python -m unittest discover -s backend/tests -t .   # 123 tests, stdlib only
python scripts/scope_check.py                       # competition scope gate
cd frontend && npm run build
```

The test suite needs no keys, no network and no running server; it neutralises
`backend/.env` so a real `GEMINI_API_KEY` on the machine cannot turn a unit
test into a billed API call.

`scope_check.py` is a real pre-commit gate, not documentation: it fails the
build on a background thread, a write path in a runtime module, a second
datastore, auth on the recovery route, a bulk runner over the seed, a tracked
secret, or a hex literal outside the design tokens.

### Capturing fixtures

One cart at a time, deliberately:

```bash
WINDFALL_FIXTURES=1 python -m backend.tools.build_golden --cart wf-02
```

---

## Documentation

- **`docs/windfall-paper.md`** the specification
- **`docs/HANDOFF.md`** current state, settled decisions, what is left
- **`frontend/CLAUDE.md`** resolved decisions that post-date the paper
