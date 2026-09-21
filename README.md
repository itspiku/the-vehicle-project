# The Vehicle Project

**Vehicle movement intelligence for Nepal.**

A camera on a pole. A vehicle passes. Seconds later its number plate has been
read, its arrival logged, and, if it is on a watchlist, someone has been told.
All of it on hardware a municipality can afford, under Nepali privacy law, with
the data never leaving the country.

[![CI](https://github.com/itspiku/the-vehicle-project/actions/workflows/ci.yml/badge.svg)](https://github.com/itspiku/the-vehicle-project/actions/workflows/ci.yml)
[![Licence](https://img.shields.io/badge/licence-Apache--2.0-blue)](LICENSE)

---

## In one minute

Here is what happens when a vehicle drives past one of the project's cameras.

**1. It is noticed.** A small computer on the pole spots the vehicle and
follows it across the frame.

**2. It is read, carefully.** The camera gets ten to forty looks at the plate
as the vehicle passes. Rather than trusting any single blurry frame, the system
combines the evidence from all of them, the way you would squint at a distant
sign and let your eyes settle.

**3. One record is made, and signed.** Plate, time, camera, and how confident
the system is. Not a video stream. A single, cryptographically signed record
per vehicle, so it can later be proven that this camera produced it and nobody
has altered it since.

**4. It waits if it has to.** If the internet link is down because of load
shedding, a cut cable or a monsoon, the record sits safely on the pole and is
sent when the link returns. Nothing is lost.

**5. It is checked.** At the control room the record is verified, matched
against watchlists, and stored with an expiry date already attached.

**6. Looking it up has a cost.** An investigator who wants to see where a
vehicle has been must type a reason first. That reason is logged permanently
against their name, and only an auditor can read that log.

That is the whole system. Everything below is detail.

---

## Why Nepal needs its own

Most number plate software is built for countries with one plate design. Nepal
has **two, at the same time**, and will for years:

| | Legacy plates | Embossed plates |
|---|---|---|
| Looks like | `बा १ च १२३४` | `3 B PA 1234` |
| Script | Devanagari | Latin |
| Colour | **Says who owns it.** Red for private, black for public, green for tourist, blue for diplomatic. | Always black on white |
| On the road since | Decades | 2020 |
| Share of vehicles today | The majority | Growing |

Almost every existing Nepali plate reader handles one or the other. A system
for real Nepali roads has to read both, and must never apply one system's rules
to the other's plate. That constraint shaped everything here, from the way
plates are described to the way the reader is trained.

---

## What it promises

These are design commitments, not settings someone can switch off.

| | |
|---|---|
| **Reads both plate systems** | Devanagari and embossed Latin, in one model, with the layout rules of each built in. |
| **Keeps working offline** | Each camera site stores its own records and forwards them when it can. A power cut costs delay, never evidence. |
| **Every record is provable** | Signed at the camera and chained to the record before it. Alter or delete one and the chain breaks, visibly, at a known point. |
| **Every lookup needs a reason** | No one can browse. Searches require a written purpose, logged before the query runs and readable only by auditors. |
| **Data expires on its own** | Every record carries its own deletion date. Whether deletion is actually happening is a simple query, not a matter of trust. |
| **Faces are blurred, never recognised** | Faces are detected only in order to be blurred, at the camera, before anything is stored. Face recognition is deliberately out of scope. |
| **Nothing leaves Nepal** | No cloud services, no external map or font providers, no third party analytics. Self hosted end to end. |
| **A human decides** | The system produces evidence. No fine is issued on a machine reading alone, and low confidence reads go to a person for review. |

---

## Where it stands

**Honestly: it works end to end on synthetic data, and it has not yet seen a
real Nepali road.**

The full path is built, tested and demonstrated: synthetic plate, trained
reader, camera software, signed queue, platform, operator console. On a held
out set of 2,000 synthetic plates the reader gets **83.6%** exactly right,
identifies plate colour **97.9%** of the time, and reaches **98.5%** on plates
wider than 130 pixels.

What has *not* happened is the part that decides whether any of that is good.
No real Nepali footage has been evaluated, because the dataset to evaluate with
does not exist and has to be collected under proper legal authorisation. Until
then every number here is an **upper bound**, not a forecast.

Two known shortfalls, stated rather than hidden:

- When the system says **HIGH confidence**, it is wrong **2.2%** of the time.
  The target is 0.5%. That gap matters more than any headline accuracy, because
  a confident wrong answer is what puts the wrong person in front of a
  magistrate.
- Single row plates read at **70.2%** against two row plates at **94.9%**. The
  cause is only partly understood and is recorded as unresolved.

For calibration: the winning entry in the 2026 international competition on
low resolution plate reading scored 82.13%. Be suspicious of any system
claiming 99% on blurry footage. This one included.

Every item's status lives in [`docs/PLAN.md`](docs/PLAN.md).

---

# For developers

## Architecture

Four stages, borrowed in shape from the UK's national ANPR platform and scaled
down two orders of magnitude for Nepal's realistic volume: single digit
millions of reads per day nationally, not hundreds of millions.

```
 ┌── CAMERA SITE ──────────────────────────────────────────────────┐
 │                                                                  │
 │   RTSP ─▶ detect ─▶ track ─▶ select crops ─▶ recognise ─▶ fuse  │
 │                                                    │             │
 │              face blur ◀───────────────────────────┘             │
 │                  │                                               │
 │                  ▼                                               │
 │        zone entry / exit  ─▶  signed, hash chained queue         │
 │                                (SQLite WAL, survives restart)    │
 └──────────────────────────────────────┬───────────────────────────┘
                                        │  intermittent link
                             ┌──────────▼──────────┐
                             │  INGEST             │  verify signature, idempotent
                             ├─────────────────────┤
                             │  SCREEN             │  watchlists, cloned plate checks
                             ├─────────────────────┤
                             │  EXPLOIT            │  PostgreSQL + TimescaleDB + pgvector
                             │  API · console      │  MinIO for imagery
                             └─────────────────────┘
```

**Decisions worth knowing about**

- **One read per passage, not per frame.** The edge fuses across the whole
  track and emits once. This is what keeps national volume in the millions.
- **One database, not three.** Postgres carries relational data, the read
  time series (a TimescaleDB hypertable) and appearance vectors (pgvector).
  Three stores would be three things to operate, back up and staff.
- **The same schema runs on SQLite.** The whole test suite needs no
  infrastructure. Every Postgres only feature is a performance feature.
- **No SQL in the HTTP layer.** Every route delegates to an audited query
  layer, so an endpoint cannot read personal data unaudited. It has no way to
  query at all.
- **Permissive licences only, enforced in CI.** An AGPL dependency is a build
  failure. A copyleft obligation is a procurement blocker for a government.

Full design: [`docs/architecture.md`](docs/architecture.md).

## Repository map

```
packages/
  nepal_plate/     domain core: plate spec, layout grammars, decoder, fusion (zero deps)
  evidence/        Ed25519 signed, hash chained events (shared by edge and platform)
  synthplate/      synthetic plate renderer, physics ordered degradation, track synthesis
  scanner_models/  one trunk, three heads: CTC + colour + quality (1.9 M params)
services/
  edge/            camera agent: detect, ByteTrack, crop selection, fuse, zones, queue, uplink
  api/             FastAPI platform: ingest, screening, audited search, retention, erasure
  web/             operator console: React + TypeScript, Nepali by default
deploy/            Docker Compose + Dockerfiles for the single site tier
scripts/
  seed_demo.py     build a runnable demo through the real pipeline
docs/
  PLAN.md          delivery plan with per item status
  research/        plate specification, prior art, datasets, Phase 2 findings
```

## Quickstart

Install everything in dependency order:

```bash
pip install -e packages/nepal_plate -e packages/evidence -e packages/synthplate -e packages/scanner_models -e services/edge -e services/api
```

Run the suite (160 tests, no infrastructure needed):

```bash
python -m pytest packages services -q
```

See the whole system running with data generated through the real pipeline:
four cameras, 400 reads, zone sessions, a watchlist hit, four user roles.

```bash
python scripts/seed_demo.py --out demo --reads 400
```

Start the platform against the seeded database (the seeder prints these values):

```bash
SCANNER_DB_URL=sqlite:///demo/scanner.db SCANNER_PLATE_KEY=$(python -c "print('d'*64)") SCANNER_TOKEN_SECRET=$(python -c "print('s'*64)") scanner-api serve
```

Start the console, then open http://localhost:5173 and sign in as
`investigator` / `demo-password-123`:

```bash
npm --prefix services/web install && npm --prefix services/web run dev
```

Generate synthetic plates and look at them:

```bash
python -m synthplate.cli preview --out preview.png --rows 6 --cols 6
```

Use the domain core directly:

```python
from nepal_plate import parse

p = parse("बा १ च १२३४")
p.canonical    # 'NP-L:BA-1-CHA-1234'
p.ownership    # Ownership.PRIVATE
p.size_class   # SizeClass.LIGHT

# Every spelling of a plate must produce the same key, or watchlist
# matching silently fails. This is the most common way national ANPR
# systems break, and it is tested end to end.
parse("BA 1 CHA 1234").canonical == p.canonical   # True
```

## Measured results

`PlateNet`, 1.93 M parameters, 20 epochs on 50,000 synthetic plates, scored on
a 2,000 sample held out split.

| Metric | Result |
|---|---|
| Full plate exact match | **83.6%** |
| Plate colour (7 way) | **97.9%** |
| Crop quality MAE | **0.070** |
| Clean crops (quality ≥ 0.7) | 98.3% |
| Degraded crops (quality < 0.4) | 51.4% |
| Plates ≥ 130 px wide | 98.5% |
| Plates 40 to 60 px wide | 54.9% |
| Single row / two row | 70.2% / 94.9% |
| False positives at HIGH confidence | **2.2%** (target ≤ 0.5%, not met) |

## What the research found

The project was designed around one idea: that constraining the decoder to the
grammar of legal Nepali plates, and using plate colour as a prior, would do the
heavy lifting on degraded images. **Measured on the trained model, neither
improves accuracy.** The grammar is worth +0.001 over plain greedy decoding and
the colour prior 0.000, because a model trained only on legal plates has
already learned the grammar from the data.

Both mechanisms are kept, for reasons that survived measurement: 10% of greedy
outputs are not legal plates at all and cannot serve as a database key, and the
grammar is what makes confidence bands meaningful (HIGH 97.8%, REJECT 0%). The
full ablation, the reasoning, and the correction of earlier claims are in
[`docs/research/findings-phase2.md`](docs/research/findings-phase2.md). It is
the most important document in the repository.

## Documentation

| | |
|---|---|
| [Delivery plan](docs/PLAN.md) | Phases, deliverables, acceptance criteria, per item status |
| [Architecture](docs/architecture.md) | Edge, Ingest, Screen, Exploit, and why |
| [Security & privacy](docs/security-and-privacy.md) | Threat model, Privacy Act 2075 obligations, deliberate limits |
| [Plate specification](docs/research/plate-specification.md) | Both plate systems in full, with sources |
| [Prior art](docs/research/prior-art.md) | What exists in Nepal and globally, and where the gap is |
| [Dataset survey](docs/research/datasets.md) | Why synthetic data, and what real data still has to be collected |
| [Phase 2 findings](docs/research/findings-phase2.md) | The ablation that disproved the headline claim |
| [Deployment](deploy/README.md) | Single site Compose, secrets, edge nodes, Jetson |

Each package and service has its own README with its design positions.

## Licence

Apache-2.0. Dependencies are permissive licence only by policy.
