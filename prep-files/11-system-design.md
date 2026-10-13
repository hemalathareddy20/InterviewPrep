# Phase 18 — System Design for Test Automation

Five designs. For each: the structure, then the trade-offs an interviewer will push on.

**Universal opening move — always do this first:**
> "Before I design, three questions: how many devices and in what physical topology? Is a test conversation exclusive to one device or can devices share an instrument? And what's the required turnaround — does a regression need to finish in an hour or overnight? Those three answers change the architecture."

---

# Q1. Design a C# automation framework for testing 100 firmware devices

## Clarify first
- 100 devices **in parallel**, or 100 device *types* tested one at a time? (Very different.)
- Do they share an instrument, or does each need its own?
- Are they physically co-located, or distributed across sites?

Assume: 100 physical devices, tested in parallel on pooled instruments, in one or more labs.

## Architecture

```
┌────────────────────────────────────────────────────────────────┐
│ CONTROL PLANE                                                  │
│  Scheduler / Orchestrator (ASP.NET Core service)               │
│    - job queue, priorities, retries                            │
│    - device+instrument reservation (lease with TTL)            │
│    - result ingestion                                          │
│  Inventory / Asset DB                                          │
│    - station id, instrument serial+FW, fixture set,            │
│      DUT id+FW+capabilities, health, last-used, calibration due │
│  Results store (SQL) + artefact store (blob/file share)        │
│  Dashboard + REST API                                          │
└──────────┬─────────────────────────────────────────────────────┘
           │ jobs (queue: RabbitMQ / Azure Service Bus / SQL table)
┌──────────▼─────────────────────────────────────────────────────┐
│ EXECUTION PLANE  — N Test Agents (one process per station)     │
│  Agent                                                         │
│   ├─ heartbeat + capability advertisement                      │
│   ├─ provision: flash FW, self-test, power-cycle DUT           │
│   ├─ TestRunner  (the existing engine, headless)               │
│   │    ├─ discovery (cached Dictionary<TestId, Type>)          │
│   │    ├─ TestCase lifecycle: IsApplicable/Arrange/Act/Assert  │
│   │    └─ ITesterLink → Ethernet | USB | Serial | Simulator    │
│   └─ publish results + artefacts, release the lease            │
└──────────┬─────────────────────────────────────────────────────┘
           │
   Instruments  ──  DUTs  (1..N per station, switch box for multiplexing)
```

## Key decisions and why

| Decision | Choice | Why |
|---|---|---|
| **Unit of parallelism** | The **station** (instrument + DUT + fixture), not the test | A protocol test is one exclusive conversation with one device over one physical link. Parallelising *within* a station is impossible; parallelising *across* stations is embarrassingly parallel. |
| **Process isolation** | One agent **process** per station | A hung driver, a leaked handle or a `Thread.Abort` gone wrong kills one station, not the fleet. Also sidesteps static/singleton state — which is exactly the constraint in our current design (`ControllerHelper` singleton, static `ModuleHelper`). |
| **Resource allocation** | Lease with a TTL and a heartbeat | If an agent dies mid-run, the lease expires and the station returns to the pool automatically. Never rely on a clean shutdown to release a resource. |
| **Transport abstraction** | `ITesterLink` + a **Simulator** implementation | Lets ~70% of the framework be tested with no hardware, in CI, on every PR. |
| **Test selection** | Metadata-driven: each test declares required capabilities | The scheduler matches jobs to stations instead of humans maintaining station-specific lists. |
| **Results** | Immutable rows + artefacts; never overwrite | Trend analysis and flakiness detection need history. |
| **Idempotent provisioning** | Flash + self-test + power-cycle at the start of every job | You must never debug a dirty environment as a product bug. |

## Scale and failure handling
- **Throughput** = stations × (tests/hour per station). To go faster, add stations or cut per-test time — not threads.
- **Instrument failure** → agent marks the station unhealthy, drains, and alerts; the scheduler stops routing to it.
- **Agent crash** → lease expires, job is requeued once with a "requeued" marker so an infinite retry loop is visible.
- **Partial run** → results are published incrementally per test, so a crash at test 300 of 700 still yields 299 results plus the crash context.
- **Poison job** → a job that kills its agent twice is quarantined, not retried forever.

## Trade-offs to volunteer
- Central scheduler = single point of failure → make it stateless over a durable queue + DB, run two instances.
- A queue adds operational complexity; for 5 stations a SQL table with row locking is enough. **Don't over-engineer — say so.**
- Per-station processes mean per-station config drift; solve with declarative config from the inventory DB, not files on each PC.

---

# Q2. Design a system that executes firmware functional tests in parallel

## The central point — lead with it
> "You have to be precise about what 'parallel' means for hardware tests, because three different things get called parallel and only two of them are safe."

| Level | Safe? | Why |
|---|---|---|
| **Across stations** (different instrument + DUT) | ✅ Yes | Fully independent physical resources |
| **Across ports on one instrument** | ⚠️ Sometimes | Our instrument has Port A and Port B and the command encoding targets them separately (`ModifyAPItoSuppPortB` sets bit 7). But they **share one Ethernet link, one command lock, one streamer and one FPGA reset**. A `FW_FPGA_FullReset` for port A destroys port B's test. So only tests that need no global reset can share an instrument. |
| **Within a test** | ❌ No | A protocol conversation is inherently sequential and timing-ordered |

## Design

```
Scheduler
  partitions the selected test list into independent WORK UNITS
  a work unit = (test subset, required capabilities, estimated duration)
  bin-packs work units onto free stations by longest-first to minimise makespan

Station Agent (process)
  runs its work unit strictly sequentially
  streams results as they complete

Aggregator
  merges results, detects cross-station disagreement
  (same test, different verdict on two stations = environment problem, not a product bug)
```

## What must change in your current design to allow it
Be honest and specific — this is a strong answer:
1. **Static/singleton state.** `ControllerHelper` singleton, `FWAppObj.FWHost` static host, `ModuleHelper` static god class, `PD_Controller.m_ControllerLst` static dictionary. In-process parallelism would corrupt all of this. Process-per-station sidesteps it entirely, which is why that's my choice — it's the pragmatic route to parallelism without a rewrite.
2. **Shared report folders.** `ModuleHelper.RunDirectory` is static; two runs would collide. Needs to be per-run state.
3. **The singleton is not thread-safe** — classic check-then-create. Would need `Lazy<T>` or DI.
4. **Modal dialogs.** `ShowMessage` in the execution path blocks a headless run. Needs a non-interactive mode.

> "So my answer is: get parallelism through process isolation first, because it's achievable in weeks rather than months, then refactor the static state opportunistically. Rewriting a 1,400-file codebase to be thread-safe as a prerequisite for parallelism would be the wrong trade."

## Determinism
- Fixed random seeds for any randomised/fuzz tests, logged per run.
- Never depend on execution order; tests bring the device up themselves.
- Record which station and which instrument serial ran each test, so a station-specific failure is visible immediately.

---

# Q3. Design a system that communicates with hardware devices and collects test results

This one is really about **layering and the data model**. Give them both.

## Communication layer

```csharp
public interface ITesterLink : IDisposable
{
    bool IsOpen { get; }
    Task OpenAsync(CancellationToken ct);
    Task<byte[]> TransactAsync(byte[] command, TimeSpan timeout, CancellationToken ct);
    Task CloseAsync();
    event EventHandler<LinkStateChangedEventArgs> StateChanged;
}

// Implementations: EthernetLink, UsbLink, SerialLink, SimulatorLink, ReplayLink
```

Design rules:
- **Transact, not Write+Read.** A command/response pair must be atomic — otherwise two callers interleave and each reads the other's response. (Our current code achieves this with a shared lock across `Write` and `Read`; making it one method makes the invariant structural instead of conventional.)
- **Timeouts everywhere**, and distinguish *socket error* from *no data within timeout* from *short read*. Those three look identical today and are three different bugs.
- **Bounded retry with backoff**, and the retry count in the result — never hidden.
- **Log every frame** with direction, timestamp, hex, a decoded description and a correlation ID.
- **Cancellation** cooperative, with the device left in a defined state in `finally`.

Above it, a **typed device API** (`ITesterDevice`) that speaks capabilities, not bytes. Tests depend only on that.

## Data collection

Three distinct streams, deliberately separated:

| Stream | Rate | Storage |
|---|---|---|
| **Control** (commands/responses) | low | text log |
| **Protocol capture** (decoded messages + timestamps) | medium | structured file (`.grltrace` equivalent) + DB summary |
| **Analog/sample stream** (ADC, waveforms) | very high | binary file, referenced from the DB, never in the DB |

> "The mistake to avoid is putting high-rate sample data in the database. Store the *verdict and the measured values* in the database for querying and trending; store the raw evidence as files with a pointer. Our current system does this correctly — the report links to a `.grltrace` file rather than embedding the samples."

## Result model

```
TestRun      (id, start, end, station, instrument serial, instrument FW,
              software version, DUT id, DUT FW, config hash, verdict, counts)
 └─ TestCaseResult (id, run id, test id, spec ref, verdict, duration,
                    started, description, artefact paths)
     └─ Assertion   (id, parent, name, spec clause, verdict,
                     expected, obtained, units, evidence index)
```

Non-negotiables:
- **Verdict vocabulary with meaning**: Pass / Fail / **NotApplicable** / **Incomplete** / Warning / NotExecuted. NA and Incomplete are not failures and must not be counted as such. (Your `TestResult` enum already has exactly this.)
- **Store the measured value, not just the verdict** — that's what lets you see a test drifting toward its limit before it fails.
- **Deterministic roll-up** from assertion → test → run, with the rule written down. (Your `PDTestResult.UpdateTestResult` with `TestValidationPriority` is this.)
- **Immutable** — a new run never overwrites an old one.
- **Emit a standard format too** (JUnit/TRX) so CI can ingest it.

---

# Q4. Design a regression system for firmware validation

## Structure

```
TRIGGERS       PR  |  merge  |  nightly  |  firmware release  |  manual/release
                 │       │         │              │
SELECTION      offline  smoke    full         full + previous FW baseline
                 │       │         │              │
PROVISION      none   flash+selftest+powercycle DUT   (idempotent, fails the job early)
                 │       │         │              │
EXECUTE        xUnit   station pool, work units, results streamed
                 │
ANALYSE        compare vs baseline → NEW FAIL / FIXED / STILL FAILING / FLAKY
                 │
REPORT         standard results + artefacts + delta summary + owner assignment
```

## The parts people forget — lead with these

**1. A baseline, not just a result.** A regression system's output is a **delta**: what broke that used to work, what got fixed, what's still broken, what's newly flaky. A list of 40 failures where 38 are known is useless. Your `RepeatCondition` (re-run only failures) is a manual version of this idea.

**2. Stability, not pass/fail.** For hardware, run the suite N times and report a pass *rate* per test. A test at 92% is a finding. Store the measured values so you can see drift.

**3. Version pinning.** Every run records instrument FW, E-Load FW, PPS FW, software version, spec revision, DUT FW, cable type, station ID and config hash. Without that, comparing two runs is meaningless. Your app already reads and logs most of this at connect — it belongs in the report header.

**4. Triage routing.** Classify failures automatically where you can: infrastructure (instrument unreachable, self-test failed) → ops; NA/Incomplete → configuration; genuine FAIL → the owning engineer. The single biggest killer of regression suites is failures with no owner.

**5. Time budget.** If the nightly doesn't finish by morning it stops being a nightly. Track per-test duration — your framework already logs overall/config/validate timings — and either parallelise across stations or move slow, low-yield tests to the weekly suite.

**6. Quarantine with an expiry date.** Acceptable briefly and visibly; never permanent.

## Trade-offs
- Full regression every night gives the fastest feedback but burns hardware and may not finish. **Test impact analysis** (run only what a change could affect) is the alternative, but for firmware the blast radius of a change is hard to compute, so a time-boxed risk-based subset nightly + full weekly is usually the honest answer.
- More repeats = better flakiness data = less coverage per night. Pick per tier: nightly ×1, release ×3–5.

---

# Q5. Design an automation framework supporting multiple device types and firmware versions

This is the **variability management** question. There are exactly four axes and each needs a different mechanism — say that explicitly.

| Axis | Mechanism | Your precedent |
|---|---|---|
| **Device type / role** (Source, Sink, DRP, Cable, Hub) | **Applicability declared by the device, evaluated per test** | `PreConfigure()` reads the VIF and returns `NA` with a reason: *"Expected Type_C_State_Machine SRC, obtained DRP"* |
| **Device capability** (supports EPR? PPS? Alt Mode?) | **Capability data from the device**, not a hard-coded matrix | VIF file + `GetVIFComponentModel_CTS().GetStaticPortElement(...)` returning `Status?` with a `?? Status.NONE` default |
| **Firmware / spec version** | **Version-aware behaviour selection**, with tests asserting the version-appropriate expectation | `SupportedPDMode { PD_2_0, PD_3_0, BOTH }` per test; `PD_Controller.CurrentSpecRev`; PD2/PD3 test conditions from JSON |
| **Instrument variant** (C2 vs C2-Gen2) | **Feature detection at connect + a capability flag** | `FWAppObj.FWHost.ApplicationType == AppType.C2Gen2` branches; `Detect_Noise_Eload()` probes which ports have an E-Load |

## The design

```
DeviceProfile           ← loaded per DUT (our VIF)
   type, role, declared capabilities, firmware version, spec revision

TestDescriptor          ← metadata on every test
   id, spec reference, required capabilities, applicable spec revisions,
   required instrument features, required fixture, estimated duration

Applicability engine
   for each (test, device, instrument):
       NotApplicable(reason)  |  Applicable(with these test conditions)

Test condition expansion
   one logical test → N executions
   (PD2-chunked, PD3-chunked, PD3-unchunked; cable condition 1..3; port A/B)

Version-aware expectations
   limits and legal-behaviour models keyed by spec revision, loaded as data
```

## The three rules I'd defend

**1. Capability, not device name.** Never `if (device == "ChargerX")`. Always `if (device.Supports(Feature.EPR))`. Device names multiply forever; capabilities don't. Our VIF-driven applicability is exactly this and it's why the framework scales to arbitrary DUTs.

**2. Version differences are data, not branches.** A test that says `if (specRev == PD3) expect X else expect Y` becomes unmaintainable across four revisions. Instead the *expectation model* is keyed by revision and loaded from data — which is what our PDM JSON test-condition files do, and what the AMS legal-sequence model enables.

**3. "Not applicable" is a first-class, reported outcome.** With a reason. Otherwise you get one of two failure modes: tests that fail because they shouldn't have run (noise, and people stop reading the report), or tests silently skipped (false confidence that coverage exists). Our framework reports NA with an explicit reason string, and I'd make the *count* of NAs a tracked metric — a sudden jump in NAs usually means a capability file is wrong, not that the device changed.

## Trade-offs to volunteer
- Declared capability can lie. A device's VIF can claim EPR support it doesn't have. So some tests must verify the *declaration* itself — we have consistency tests (`TD_PD_VNDI_*`) that check declared capability against observed behaviour. Worth naming, it's a subtle point.
- Test-condition expansion multiplies run time: three cable conditions × two PD revisions × two chunk modes = 12 executions of one logical test. You need per-condition selection for smoke suites, and you need the report to collapse them sensibly — which is why the result model is a tree with one node per condition.

---

# The five sentences to close any system-design answer

1. "The unit of parallelism is the station, because a protocol conversation is exclusive and sequential."
2. "Applicability is data the device declares; the framework evaluates it, and 'not applicable' is a reported outcome with a reason, never a pass and never a failure."
3. "Store verdicts and measured values in the database, raw evidence as files, and never overwrite history."
4. "Provision the environment at the start of every job so you never debug a dirty environment as a product bug."
5. "Distinguish infrastructure failures from product failures in the result, or nobody will trust the suite."
