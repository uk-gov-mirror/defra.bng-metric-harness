# CDP performance testing

How to write JMeter performance tests for the BNG Metric service that run
unmodified on the **Core Delivery Platform (CDP)**. The baseline
geometry-validation endpoint is the worked example, but the pattern applies to
any endpoint.

!!! info "Why this exists"
    CDP mandates JMeter and injects the target host, port, proxy, and
    environment at run time. A plan written the "normal" way — hard-coded host,
    GUI listeners, a soak loop — will fail to run or waste the 2-hour budget.
    These conventions make a plan work first time.

## What CDP gives you, and what you write

In the CDP Portal choose **Create → Performance Test Suite**; it scaffolds a repo
from the platform template. You own **one file** — the `.jmx` — plus its test
data. Don't reinvent the runner, reporting, or Docker packaging.

| CDP owns (the scaffold) | You own (the payload) |
| --- | --- |
| JMeter runner + Docker image | The **`.jmx` test plan** |
| `entrypoint.sh` (invokes JMeter, injects properties, wires the proxy) | Test data (fixtures, a `CSV` of ids) |
| Report generation + publishing (HTML dashboard, and/or Allure) | Any `setUp`/staging logic inside the plan |
| Triggering runs from the Portal | Assertions / thresholds that define pass/fail |

**Platform facts that shape the plan** (from the CDP *Performance Testing FAQ*
and the template's `entrypoint.sh`):

- **JMeter only** — the DEFRA-approved tool. Results publish back as an HTML
  dashboard (some scaffolds add Allure — check yours).
- **Target is injected as separate properties**, never a URL: `-Jprotocol`,
  `-Jdomain`, `-Jport`, `-Jenv`. Read these; don't hard-code a host.
- **Egress is via a proxy** at `localhost:3128`, set on the JVM by the entrypoint.
  Don't configure the proxy in the `.jmx`.
- **Don't call external services** — rate limits, WAFs, and anti-bot measures
  block automated suites. Test only your own service.
- **2-hour hard cap; soak is discouraged.** Design for targeted load, stress, and
  spike tests, not endurance.

**How a run happens:** merge to `main` → a GitHub Actions **Publish** workflow
builds a versioned Docker image and pushes it to CDP → you launch that image from
the Portal, which runs it as an ECS task and publishes the report. "Editing a
test" means merging to `main`, not uploading in the Portal. You can still run the
plan locally first (see [Dry-run locally](#dry-run-locally-before-pushing)).

## Writing the `.jmx` to CDP's contract

Six rules turn an ordinary JMeter plan into a CDP-compatible one.

**1. Read the target from CDP's properties.** One **HTTP Request Defaults**
element at the top, fed from the injected properties with local fallbacks so the
same file runs on a dev machine:

```xml
<ConfigTestElement guiclass="HttpDefaultsGui" testclass="ConfigTestElement" testname="HTTP Request Defaults (backend)">
  <stringProp name="HTTPSampler.protocol">${__P(protocol,http)}</stringProp>
  <stringProp name="HTTPSampler.domain">${__P(domain,localhost)}</stringProp>
  <stringProp name="HTTPSampler.port">${__P(port,3001)}</stringProp>
</ConfigTestElement>
```

Every sampler then carries **only the path** (`/baseline/validate/${UPLOAD_ID}`),
inheriting protocol/domain/port — so CDP can point the plan at any environment.

**2. Don't touch the proxy.** The entrypoint already sets it; adding proxy config
in the plan doubles up and breaks in-platform calls.

**3. No GUI listeners.** Remove *View Results Tree*, *Summary Report*, etc. CDP
captures the `.jtl` (`-e -l "${REPORTFILE}"`) and builds the report itself.

**4. Name samplers and assertions clearly.** The report groups by name, so give
each a human name (`POST /baseline/validate (size ${PARCELS})`). Wrap any
multi-request journey (upload → poll → validate) in a **Transaction Controller**
to report it as one end-to-end number.

**5. Make pass/fail explicit.** A load test with no assertions is just traffic.
Add a **Response Assertion** for status, and a **Duration Assertion** where you
have an SLO.

**6. Use the standard load-control variables**, set from the Portal, so its knobs
work without edits:

| Variable | Meaning |
| --- | --- |
| `THREAD_COUNT` | Concurrent virtual users |
| `RAMPUP_SECONDS` | Time to ramp all users up |
| `LOOP_COUNT` | Iterations per user |
| `DURATION_SECONDS` | Max run time (a cap — keep well under 2 hours) |
| `TEST_SCENARIO` | Which scenario file under `scenarios/` to run |

!!! note "Confirm the exact bridge in your scaffold"
    Check how your generated `entrypoint.sh` maps these to JMeter (`-J` property
    vs. environment variable) and match it. Other knobs (SLO thresholds, ids)
    should be `-J` properties with sensible defaults.

**Auth: inject the token, never commit it.** The backend authenticates every
request with a **bearer JWT** (the `defra-jwt` Hapi strategy, verified against a
JWKS). Add an `Authorization: Bearer ${TOKEN}` header (once, in an **HTTP Header
Manager**) with the token read from a property:

```xml
<elementProp name="TOKEN" elementType="Argument">
  <stringProp name="Argument.name">TOKEN</stringProp>
  <stringProp name="Argument.value">${__P(token,)}</stringProp>
</elementProp>
```

Supply it at run time — `-Jtoken="$TOKEN"` from `entrypoint.sh` sourcing a **CDP
secret**, or minted by a login step in the plan. Confirm the mechanism your
scaffold supports.

## Designing tests that fit the 2-hour budget

Split the work into short, focused thread groups. Three cover the validation
endpoint:

| Thread group | Shape | What it answers |
| --- | --- | --- |
| **A — size sweep** | 1 thread, iterate a CSV of `(uploadId, parcels)` | How latency scales with **input size** (parcel count). A benchmark, not a load test. |
| **B — concurrency ramp** | N threads ramping against the largest file, short duration | **Load / stress**: where p95/p99 knees up as uploads contend for the pool and CPU. |
| **C — lock contention** | Concurrent full-pipeline requests against **one** project | That the write path degrades **gracefully** (clean `409`s, not `5xx` or hangs). |

!!! tip "Size vs. concurrency are different questions"
    Geometry work scales with the **parcels in one file**, not the number of
    users. Group A isolates size (single-threaded, varying file); B and C isolate
    concurrency (fixed file, varying load). A single mixed run answers neither
    cleanly.

Enable only Group A by default; turn B and C on for dedicated load runs. Keep
every duration well under the cap. For a realistic picture, add a **mixed
scenario** running several journeys as parallel thread groups at once — use the
isolated groups to find bottlenecks, the mixed run to check the service copes
under a traffic blend.

!!! tip "Say what the numbers mean, and compare to a target"
    These groups are **closed-loop with no think time** — they measure how many
    **simultaneous users** stay responsive, *not* the real-world arrival rate.
    State that, and compare to a **capacity target** (the BNG equivalent of an NFR
    figure, e.g. "N uploads/hour"). Watch **error rate** as a headline KPI, and
    remember JMeter throughput is **per second** — ×60 for per-minute, ×3,600 for
    per-hour.

## The BNG service specifics

**The endpoint takes an `uploadId`, not the file.**
`POST /baseline/validate/{uploadId}` (and `/post-intervention/validate/{uploadId}`)
does **not** receive the GeoPackage in the body — the **CDP Uploader** already
pushed it to S3. The handler polls the uploader's `/status/{uploadId}` for the
bucket/key, then downloads and validates. The body is just an optional
`{ "projectId": "<uuid>" }`. So:

- **Pre-stage uploads** — each fixture goes through *upload → ready* once to get an
  `uploadId`.
- **Validation with no `projectId` is read-only, idempotent, and replayable** —
  stage once, drive many iterations. Add a `projectId` only to exercise the write
  path.

**`projectId` turns on sizing, persistence, and a row lock.** With a `projectId`
present, the request also calculates habitat sizes, extracts the document, and
**persists** it — taking a `FOR UPDATE` row lock with a `lock_timeout`. Concurrent
requests for the **same** project return `409` (what Group C asserts). Use
**distinct** ids to load-test the happy write path, a **shared** id to prove the
contention behaviour.

## Staging test data

The shared library produces synthetic GeoPackages with a controllable parcel
count:

```js
import { generateSyntheticGpkg } from 'bng-library' // src/api.mjs

// One file per size in your sweep.
for (const numParcels of [100, 500, 1000, 2000]) {
  generateSyntheticGpkg({ numParcels /*, out: `baseline-${numParcels}.gpkg` */ })
}
```

Turn fixtures into `uploadId`s, in order of robustness:

1. **A `setUp` Thread Group inside the plan** (preferred). Calls the CDP Uploader
   — *initiate → PUT the fixture → poll `/status/{uploadId}` until `ready`* — and
   captures each `uploadId` into a property. Self-staging and safe to schedule.
2. **A hand-built `uploads.csv`** of `uploadId,parcels` staged out of band. Fine
   for a one-off local run; brittle against a fresh environment.

```csv
uploadId,parcels
00000000-0000-0000-0000-000000000001,100
00000000-0000-0000-0000-000000000002,2000
```

!!! warning "ids are environment-specific"
    An `uploadId` only resolves where it was staged. Don't carry a CSV from local
    into a CDP run — stage in-environment (option 1).

## Dry-run locally before pushing

The property defaults let you exercise the plan against a locally running backend
(`npm run dev` in the harness, backend on `:3001`) before it reaches CDP:

```sh
jmeter -n -t scenarios/test.jmx \
  -Jprotocol=http -Jdomain=localhost -Jport=3001 \
  -Jtoken="$LOCAL_STUB_TOKEN" \
  -JLOOP_COUNT=5 \
  -l results.jtl -e -o ./html-report
```

A green run with a stub token confirms structure, headers, and assertions before
you spend a CDP slot. For higher fidelity, run against the **full local stack**
(the backend's `compose.yml` brings up PostGIS, Redis, LocalStack, and the CDP
uploader stub) — the closest mirror of a CDP run.

## Checklist

Before committing a plan to a CDP performance test suite:

- [ ] Target read from `${__P(protocol)}` / `${__P(domain)}` / `${__P(port)}` via
      **HTTP Request Defaults** — no hard-coded host.
- [ ] Samplers carry **paths only**.
- [ ] **No** proxy configuration, **no** GUI listeners in the plan.
- [ ] `Authorization: Bearer ${__P(token)}` header; **no token committed**.
- [ ] Descriptive sampler/assertion names; multi-request journeys in a
      **Transaction Controller**.
- [ ] Every request has a **Response Assertion**; SLO'd requests a **Duration
      Assertion**.
- [ ] Load driven by the standard vars (`THREAD_COUNT`, `RAMPUP_SECONDS`,
      `LOOP_COUNT`, `DURATION_SECONDS`, `TEST_SCENARIO`); other knobs are `-J`
      properties with defaults.
- [ ] Durations **well under 2 hours**; no soak loop.
- [ ] Results read against a **capacity target** and an **error-rate** KPI.
- [ ] Only the service under test is called — **no external hosts**.
- [ ] Test data staged **in the target environment** (setUp thread group).
- [ ] `.jmx` under `scenarios/`; verified with a **local dry-run** first.

## Reference: example test plan

The skeleton below implements the three-group design against the baseline validate
endpoint. Place it under the scaffold's `scenarios/` directory (e.g.
`scenarios/test.jmx`) — `TEST_SCENARIO` selects which file runs — and match the
scaffold's JMeter version.

??? example "bng-baseline-validate.jmx (three-group skeleton)"
    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <jmeterTestPlan version="1.2" properties="5.0" jmeter="5.6.3">
      <hashTree>
        <TestPlan testname="BNG baseline validate — perf">
          <!-- User Defined Variables:
               PROTOCOL = ${__P(protocol,http)}
               DOMAIN   = ${__P(domain,localhost)}
               PORT     = ${__P(port,3001)}
               ENV      = ${__P(env,local)}
               TOKEN    = ${__P(token,)}        # inject at runtime, never commit
               SHARED_PROJECT_ID = ${__P(shared_project_id,)}
          -->

          <!-- HTTP Header Manager (applies to all):
               Authorization: Bearer ${TOKEN}
               Content-Type:  application/json
          -->

          <!-- HTTP Request Defaults:
               protocol=${PROTOCOL}  domain=${DOMAIN}  port=${PORT}
               (proxy is set by CDP's entrypoint at the JVM level — not here)
          -->

          <!-- GROUP A — SIZE SWEEP (enabled) ------------------------------
               1 thread, loops ${__P(LOOP_COUNT,20)}.
               CSVDataSet uploads.csv -> UPLOAD_ID, PARCELS  (recycle=true).
               Sampler: POST /baseline/validate/${UPLOAD_ID}  body: {}
                        (no projectId -> read-only, idempotent, replayable)
               Assertion: response code == 200
          -->

          <!-- GROUP B — CONCURRENCY RAMP (disabled by default) -----------
               ${__P(THREAD_COUNT,20)} threads, ramp ${__P(RAMPUP_SECONDS,60)}s,
               scheduler duration ${__P(DURATION_SECONDS,300)}s (<< 2h cap).
               Sampler: POST /baseline/validate/${__P(upload_id_2k,)}  body: {}
               Duration Assertion: < ${__P(slo_ms,8000)} ms
          -->

          <!-- GROUP C — LOCK CONTENTION (disabled by default) ------------
               ${__P(THREAD_COUNT,10)} threads, loops ${__P(LOOP_COUNT,10)},
               shared project id.
               Sampler: POST /baseline/validate/${__P(upload_id_persist,)}
                        body: {"projectId":"${SHARED_PROJECT_ID}"}
               Assertion: response code matches ^(200|409)$  (no 5xx)
          -->
        </TestPlan>
      </hashTree>
    </jmeterTestPlan>
    ```

    The full, GUI-openable version is maintained alongside the performance test
    suite; the comments above map one-to-one onto its thread groups.
