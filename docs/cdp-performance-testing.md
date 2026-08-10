# CDP performance testing

How to write JMeter performance tests for the BNG Metric service that run
unmodified on the **Core Delivery Platform (CDP)**. It uses the baseline
geometry-validation endpoint as the worked example, but the pattern applies to
any endpoint in the estate.

!!! info "Why this exists"
    CDP mandates a specific toolchain and injects the target host, port, proxy,
    and environment into your test at run time. A plan written the "normal"
    JMeter way — hard-coded host, GUI listeners, a soak loop — will either fail
    to run or waste the platform's 2-hour budget. This page captures the
    conventions so a plan works first time.

## What CDP gives you, and what you write

CDP performance testing is **create-a-suite-in-the-Portal**, not
build-a-repo-by-hand. In the CDP Portal choose **Create → Performance Test
Suite**; it scaffolds a repository from the platform template. The split of
ownership is the single most important thing to understand:

| CDP owns (the scaffold) | You own (the payload) |
| --- | --- |
| The JMeter runner + Docker image | The **`.jmx` test plan** |
| `entrypoint.sh` (invokes JMeter, injects properties, wires the proxy) | Test data (fixtures, a `CSV` of ids) |
| Report generation and publishing (HTML dashboard, and/or Allure) | Any `setUp`/staging logic inside the plan |
| Triggering runs (on demand or scheduled) from the Portal | Assertions / thresholds that define pass/fail |

You are writing **one file** — the `.jmx` — plus its test data. Everything else
is the platform's. Do not reinvent the runner, the reporting, or the Docker
packaging.

### The platform facts that shape the plan

These come from the CDP *Performance Testing FAQ* and the template's
`entrypoint.sh`:

- **JMeter is the DEFRA-approved tool.** Results are published back to the Portal
  as an HTML dashboard (some scaffolds also add Allure — check which yours emits).
  No other load tool is supported.
- **The target service is injected as separate properties**, never as a URL:
  `-Jprotocol`, `-Jdomain`, `-Jport`, plus `-Jenv` for the environment name.
  Your plan must read these, not hard-code a host.
- **Outbound egress is via a proxy** at `localhost:3128`, set on the JVM by the
  entrypoint (`-Dhttp.proxyHost` / `-Dhttps.proxyHost`). You do **not** configure
  the proxy in the `.jmx`.
- **Avoid calling external services.** They have rate limits, WAFs, and anti-bot
  measures that will block an automated suite. Test only your own service.
- **There is a 2-hour hard cap, and soak tests are discouraged.** CDP steers you
  to *targeted load, stress, and spike* tests that produce actionable findings
  quickly. Design for peak-condition validation, not endurance.
- **Tenant teams run their own tests** from the Portal, on demand or on a
  schedule of their choosing.

### How a run actually happens

The suite behaves like any other CDP service:

1. You merge a change to `main` in the test-suite repo.
2. A GitHub Actions **Publish** workflow builds a **versioned Docker image** and
   pushes it to CDP.
3. From the Portal you launch that image; CDP runs it as an ECS task and provides
   the infrastructure.
4. The container runs your plan and publishes the report to the Portal.

So "editing a test" means merging to `main`, not uploading a file in the Portal.
There is no local setup for the CDP path — but you can run the same plan locally
first (see [Dry-run locally](#dry-run-locally-before-pushing)).

## Writing the `.jmx` to CDP's contract

Six rules turn an ordinary JMeter plan into a CDP-compatible one.

### 1. Read the target from CDP's properties

Put a single **HTTP Request Defaults** element at the top of the plan and feed it
from CDP's injected properties, with local fallbacks so the same file runs on a
developer machine:

```xml
<ConfigTestElement guiclass="HttpDefaultsGui" testclass="ConfigTestElement" testname="HTTP Request Defaults (backend)">
  <stringProp name="HTTPSampler.protocol">${__P(protocol,http)}</stringProp>
  <stringProp name="HTTPSampler.domain">${__P(domain,localhost)}</stringProp>
  <stringProp name="HTTPSampler.port">${__P(port,3001)}</stringProp>
  ...
</ConfigTestElement>
```

Every sampler then carries **only the path** (`/baseline/validate/${UPLOAD_ID}`),
inheriting protocol/domain/port. This is what lets CDP point the plan at any
environment without you touching the file.

### 2. Don't touch the proxy

The entrypoint already sets `-Dhttp(s).proxyHost=localhost -Dhttp(s).proxyPort=3128`.
Adding proxy config in the plan will double up and break in-platform calls. Leave
it out.

### 3. No GUI listeners

Remove *View Results Tree*, *Summary Report*, *Aggregate Report*, etc. CDP
captures the `.jtl` and builds the report itself; listeners only bloat the run
and the container. The entrypoint invokes JMeter with `-e -l "${REPORTFILE}"`
already.

### 4. Name samplers and assertions clearly

The report groups results by sampler and transaction name and shows assertion
results as steps. Give each sampler a human name (`POST /baseline/validate (size
${PARCELS})`) and each assertion a descriptive `testname`. Readable names here
become a readable report. Wrap any multi-request journey (e.g. upload → poll →
validate) in a **Transaction Controller** so it reports as one end-to-end number
plus a per-request breakdown.

### 5. Make pass/fail explicit with assertions

A load test with no assertions is just traffic. Add a **Response Assertion** for
the expected status and, where you have an SLO, a **Duration Assertion**. These
become the red/green in the report and the signal for a scheduled run.

### 6. Use the standard load-control variables

CDP shapes the load through a standard set of variables, set from the Portal (and
passed to the container locally). Read these in the plan rather than inventing
your own names, so the Portal's knobs work without edits:

| Variable | Meaning |
| --- | --- |
| `THREAD_COUNT` | Concurrent virtual users |
| `RAMPUP_SECONDS` | Time to ramp all users up |
| `LOOP_COUNT` | Iterations per user |
| `DURATION_SECONDS` | Max run time (a cap — keep well under 2 hours) |
| `TEST_SCENARIO` | Which scenario file under `scenarios/` to run |

!!! note "Confirm the exact bridge in your scaffold"
    Treat these as the convention to align to, not gospel — check how your
    generated `entrypoint.sh` maps them to JMeter (`-J` property vs. environment
    variable) and match it. Any other knobs (SLO thresholds, ids) should still be
    `-J` properties with sensible defaults.

### Auth: inject the token, never commit it

The BNG backend authenticates every request with a **bearer JWT** (the
`defra-jwt` Hapi strategy, verified against a JWKS). Your samplers need an
`Authorization: Bearer ${TOKEN}` header (set once in an **HTTP Header Manager**),
but the token must **never** be committed.

```xml
<elementProp name="TOKEN" elementType="Argument">
  <stringProp name="Argument.name">TOKEN</stringProp>
  <stringProp name="Argument.value">${__P(token,)}</stringProp>
</elementProp>
```

Supply it at run time — either `-Jtoken="$TOKEN"` from `entrypoint.sh` sourcing a
**CDP secret** exposed as an environment variable, or minted by a login step
inside the plan. Confirm the mechanism your scaffold supports; secret handling is
the one detail that varies by suite.

## Designing tests that fit the 2-hour budget

Because soak is discouraged, split the work into short, focused thread groups.
For the validation endpoint, three cover the ground:

| Thread group | Shape | What it answers |
| --- | --- | --- |
| **A — size sweep** | 1 thread, iterate a CSV of `(uploadId, parcels)` | How per-request latency scales with **input size** (parcel count). A benchmark, not a load test. |
| **B — concurrency ramp** | N threads ramping against the largest file, short duration | **Load / stress**: where p95/p99 knees up as concurrent uploads contend for the connection pool and CPU. |
| **C — lock contention** | Concurrent full-pipeline requests against **one** project | That the write path degrades **gracefully** (clean `409`s, not `5xx` or hung connections). |

!!! tip "Size vs. concurrency are different questions"
    Heavy geometry work scales with the **number of parcels in one file**, not
    with the number of users. Group A isolates that (single-threaded, varying
    file size); Groups B and C isolate concurrency (fixed file, varying load).
    Keep them separate — a single mixed run answers neither cleanly.

Enable only Group A by default; turn B and C on for dedicated load runs. Keep
every duration well under the 2-hour cap — a few minutes at realistic peak beats
an hour of steady state.

For a more realistic picture, you can also add a **mixed scenario** that runs
several journeys as parallel thread groups at once, rather than one journey in
isolation. Use the isolated groups above to find bottlenecks,
and a mixed run to check the service copes under a realistic traffic blend.

!!! tip "Say what the numbers mean, and compare to a target"
    These thread groups are **closed-loop with no think time** — they push as hard
    as the service will allow, so they measure how many **simultaneous users** stay
    responsive, *not* the real-world arrival rate. State that when you report
    results, and compare them to a **capacity target** (find or agree the BNG
    equivalent of an NFR figure, e.g. "N uploads/hour"). A pass/fail against a
    target is far more useful than a bare latency number. Also watch **error rate**
    as a headline KPI, and remember JMeter's throughput column is **per second** —
    multiply by 60 for per-minute, 3,600 for per-hour.

## The BNG service specifics

Two things about this service determine how the plan is built.

### The endpoint takes an `uploadId`, not the file

`POST /baseline/validate/{uploadId}` (and the sibling
`/post-intervention/validate/{uploadId}`) does **not** receive the GeoPackage in
the request body. The file was already pushed to S3 by the **CDP Uploader**; the
handler resolves it by polling the uploader's `/status/{uploadId}` for the S3
bucket/key, then downloads and validates it. The request body is just an optional
`{ "projectId": "<uuid>" }`.

Two consequences:

- **You must pre-stage uploads.** Each fixture has to go through *upload → ready*
  once to obtain an `uploadId` the plan can hit.
- **Validation with no `projectId` is read-only and idempotent.** It downloads
  from S3 and runs the geometry checks without writing anything, so a staged
  `uploadId` is **replayable** — stage once, drive many iterations. Include a
  `projectId` only when you deliberately want to exercise the write path.

### `projectId` turns on sizing, persistence, and a row lock

When a `projectId` is present the request additionally calculates habitat sizes,
extracts the document, and **persists** it — taking a `FOR UPDATE` row lock with
a `lock_timeout`. Concurrent requests for the **same** project are designed to
return `409`, which is exactly what Group C asserts. Use **distinct** project ids
to load-test the happy write path, or a **shared** id to prove the contention
behaviour.

## Staging test data

### Generate fixtures of varying size

The shared library produces synthetic GeoPackages with a controllable parcel
count:

```js
import { generateSyntheticGpkg } from 'bng-library' // src/api.mjs

// One file per size in your sweep.
for (const numParcels of [100, 500, 1000, 2000]) {
  generateSyntheticGpkg({ numParcels /*, out: `baseline-${numParcels}.gpkg` */ })
}
```

### Turn fixtures into `uploadId`s

Two options, in order of robustness:

1. **A `setUp` Thread Group inside the plan** (preferred for deployed
   environments). It calls the CDP Uploader — *initiate → PUT the fixture → poll
   `/status/{uploadId}` until `ready`* — and captures each returned `uploadId`
   into a JMeter variable/property for the main thread groups. This makes the
   suite self-staging and safe to schedule unattended.
2. **A hand-built `uploads.csv`** of `uploadId,parcels` staged out of band. Fine
   for a one-off local run; brittle against a fresh environment where those ids
   don't exist.

```csv
uploadId,parcels
00000000-0000-0000-0000-000000000001,100
00000000-0000-0000-0000-000000000002,2000
```

!!! warning "ids are environment-specific"
    An `uploadId` only resolves in the environment where it was staged. Don't
    carry a CSV from local into a CDP run — stage in-environment (option 1).

## Dry-run locally before pushing

The property defaults let you exercise the plan against a locally running backend
(`npm run dev` in the harness, backend on `:3001`) before it ever reaches CDP:

```sh
jmeter -n -t scenarios/test.jmx \
  -Jprotocol=http -Jdomain=localhost -Jport=3001 \
  -Jtoken="$LOCAL_STUB_TOKEN" \
  -JLOOP_COUNT=5 \
  -l results.jtl -e -o ./html-report
```

A green local run with a stub token confirms the plan's structure, headers, and
assertions before you spend a CDP slot.

For higher fidelity, run it against the **full local stack** (the backend's
`compose.yml` brings up PostGIS, Redis, LocalStack, and the CDP uploader stub),
so the upload → validate path exercises real infrastructure — the closest mirror
of a CDP run before you push.

## Checklist

Before committing a plan to a CDP performance test suite:

- [ ] Target read from `${__P(protocol)}` / `${__P(domain)}` / `${__P(port)}` via
      **HTTP Request Defaults** — no hard-coded host.
- [ ] Samplers carry **paths only**.
- [ ] **No** proxy configuration in the plan.
- [ ] **No** GUI listeners.
- [ ] `Authorization: Bearer ${__P(token)}` header; **no token committed**.
- [ ] Samplers and assertions have descriptive names; multi-request journeys are
      wrapped in a **Transaction Controller**.
- [ ] Every request has a **Response Assertion**; SLO'd requests have a
      **Duration Assertion**.
- [ ] Load is driven by the standard vars (`THREAD_COUNT`, `RAMPUP_SECONDS`,
      `LOOP_COUNT`, `DURATION_SECONDS`, `TEST_SCENARIO`); other knobs are `-J`
      properties with defaults.
- [ ] Durations sit **well under 2 hours**; no soak loop.
- [ ] Results are read against a **capacity target** and an **error-rate** KPI.
- [ ] Only the service under test is called — **no external hosts**.
- [ ] Test data is staged **in the target environment** (setUp thread group).
- [ ] `.jmx` lives under `scenarios/`; verified with a **local dry-run** first.

## Reference: example test plan

The following skeleton implements the three-group design above against the
baseline validate endpoint. Place it under the scaffold's `scenarios/` directory
(e.g. `scenarios/test.jmx`) — `TEST_SCENARIO` selects which file there runs — and
match the scaffold's JMeter version. Confirm the exact path against your generated
suite.

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

    The full, GUI-openable version of this plan is maintained alongside the
    performance test suite. The comments above map one-to-one onto its thread
    groups; build it out in the JMeter GUI to confirm structure before
    committing.
