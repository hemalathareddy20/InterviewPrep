# Phase 25 — Final Cheat Sheet

Read this the morning of the interview. Nothing else.

---

## C# / .NET — 20 things

1. **Interface vs abstract class** — abstract when you share *state* and behaviour in one is-a hierarchy (`TestCase`); interface for a contract across unrelated types or assemblies (`IModule`).
2. **Reflection** — `Assembly.GetTypes()` + `Activator.CreateInstance(type, args)`. Convention over configuration. Costs: no compile-time safety, `GetTypes()` can throw `ReflectionTypeLoadException`, renames break at runtime.
3. **Generics** — reified in .NET, no boxing for value types. Constraints (`where T : class`) make `T` usable.
4. **`First` vs `FirstOrDefault` vs `Single`** — throws / returns default / throws unless exactly one.
5. **Delegates vs events** — delegate = typed method reference (inverts a dependency without an interface); event = delegate with subscription control.
6. **`Task` vs `Thread`** — Task for short composable work on the pool; dedicated Thread for long-running blocking work (your test runner and streamer).
7. **async/await** — compiler state machine; returns to the caller at an incomplete await; resumes on the captured context unless `ConfigureAwait(false)`. About **not blocking a thread**, not about parallelism.
8. **`Task.Run(...).Wait()` is wrong** — burns two threads, deadlocks with a sync context, wraps exceptions in `AggregateException`.
9. **`async void`** — only for event handlers; exceptions escape and can crash the process.
10. **CancellationToken** — cooperative: `IsCancellationRequested` / `ThrowIfCancellationRequested()` / `Register`. Replaces `Thread.Abort`, which is unsupported on .NET 5+.
11. **`lock`** — on a `private readonly object`, never on `this` or a public object. Deadlock needs nested/ordered locks; keep locks leaf-level.
12. **`Mutex` (cross-process) vs `SemaphoreSlim` (in-process, counting, has `WaitAsync`) vs `lock` (cheapest) vs `Interlocked` (lock-free)**.
13. **`volatile`** — prevents caching/reordering of a field; **not** a substitute for a lock on read-modify-write.
14. **IDisposable pattern** — `Dispose()` → `Dispose(bool)` → `GC.SuppressFinalize(this)`. Finalizer only for raw unmanaged handles, and must not touch managed objects.
15. **Value vs reference types; boxing** — boxing allocates; hides in `object` APIs and non-generic collections (`ArrayList`).
16. **GC** — generational, LOH (≥85 KB) not compacted by default. `IDisposable` is for unmanaged resources, GC handles memory.
17. **`throw;` not `throw ex;`** — `throw ex` resets the stack trace.
18. **DI lifetimes** — Transient / Scoped / Singleton; the classic bug is a captive dependency (Scoped inside a Singleton).
19. **.NET Framework vs .NET 5+** — Windows-only/in-place vs cross-platform/side-by-side. Gone in Core: AppDomains, remoting, `Thread.Abort`, `BinaryFormatter`.
20. **`==` vs `Equals` vs `GetHashCode`** — override `Equals` *and* `GetHashCode` together, or hashing breaks. (`TestCaseDetails` overrides only `Equals` — a real bug worth volunteering.)

---

## Test Automation — 20 things

1. **Layers**: test case → test base/helpers → device API → transport abstraction → hardware. One-way dependencies.
2. **Test lifecycle**: `IsApplicable → Arrange → Act → Assert → Cleanup`. Cleanup in `finally`, always.
3. **Discovery by convention** — adding a test must not require editing the engine.
4. **A test contains only its differences.** Everything shared goes in the shared layer.
5. **Applicability is a first-class outcome** — `NotApplicable` with a reason, never pass, never fail.
6. **`NA` ≠ `INCOMPLETE` ≠ `FAIL`** — doesn't apply / couldn't get evidence / device is wrong.
7. **Model what the spec allows, not one golden path.** Otherwise every firmware release breaks your tests.
8. **Record the measured value, not just the verdict** — so you see a test drifting toward its limit.
9. **A failure report should be a complete bug report**: expected, obtained, spec clause, evidence index.
10. **Save raw evidence per test** so a failure can be re-analysed offline without the hardware.
11. **Replay recorded captures** through the production validator in unit tests — regression-test the test framework.
12. **Never retry silently.** If it needs three attempts, that's the finding.
13. **Parallelism unit = the station**, not the test. A protocol conversation is exclusive and sequential.
14. **Provision the environment at the start of every job** — flash, self-test, power-cycle. Never debug a dirty environment as a product bug.
15. **Distinguish infrastructure failures from product failures** in the result, or nobody trusts the suite.
16. **Regression output is a delta**, not a list: new fail / fixed / still failing / newly flaky.
17. **Version-pin everything in the report header**: instrument FW, software, spec revision, DUT FW, station ID, config hash.
18. **Smoke vs regression** — 10 min high-signal on every merge; everything, nightly.
19. **Flakiness is measured as a pass rate over N runs**, not observed once.
20. **Quarantine with an expiry date.** Permanent quarantine is how suites die.

---

## Firmware — 10 things

1. **Functional testing** = whole image, real hardware, real external interface, spec-defined observable behaviour **including timing**.
2. Unit → integration → **functional** → hardware → system. Functional is where state-machine and timing bugs live.
3. **The host can't be the timing reference.** Compile the stimulus, upload it, let the firmware/FPGA replay it.
4. **Firmware state machines**: states, events, guards, entry/exit actions, and a *previous* state for diagnosis.
5. **Test classes for a state machine**: valid transitions, invalid events, timeouts, recovery, concurrency/interruption, coverage.
6. **Observing state is the hard part** — a debug/state-report command, behavioural inference, or markers injected into the capture.
7. **Bring the device up from a known state every test.** Reset between groups.
8. **Fault injection is the point** of most protocol tests — deliberately don't respond, deliberately corrupt a field.
9. **Is it the DUT, the instrument, or my automation?** Four checks: did we initiate correctly, are our own frames valid, does the instrument self-test pass, does offline replay agree.
10. **Read the firmware.** Being able to say "here's the trace, here's the spec clause, here's the firmware path" is what makes an automation engineer trusted.

---

## Protocol Testing — 10 axes and 10 checks

**The seven axes:** framing/encoding · integrity (CRC) · field validity · sequence/AMS · timing · error recovery · robustness/stress.

1. **Deadline vs timeout** — must respond *within* T; must give up *after* T. Two separate tests.
2. **Measure from a defined reference edge** (end of the ack to the start of the response), with hardware timestamps.
3. **Corrupt the CRC** → the peer must not acknowledge it.
4. **Set every reserved/invalid field** → expect Not_Supported / Reject / ignore.
5. **Interrupt an uninterruptible sequence** → expect the defined error.
6. **Force a protocol error** → check the reset path *and* that sequence counters/IDs were cleared.
7. **Chunked messages** are one logical message with chunking behaviour validated separately.
8. **Count retries** — they're legal behaviour your model must account for, not noise.
9. **Stress**: same message N times, random sequences, multi-port concurrently.
10. **Negative assertions matter** — "must not respond" is often the real point of the test.

---

## Serial / BLE / OTA

**Serial:** UART = async, agreed baud, start/data/parity/stop (`8N1`). No message boundaries — you need framing on top. Never hard-code COM ports: enumerate and identify by **VID/PID**. Always set read/write timeouts and close in `finally`.

**BLE:** PHY (2.4 GHz, 40 ch, 3 advertising) → Link Layer (Central/Peripheral, AFH, connection interval 7.5 ms–4 s) → L2CAP → **ATT** → **GATT** (Service → Characteristic → Descriptor; Read/Write/**Notify** unacked/**Indicate** acked; CCCD enables notifications) → GAP (advertising, pairing, bonding). **ATT_MTU default 23 bytes**, negotiable. Test with: device console for control, a BLE central API or dongle for interaction, and an **air sniffer** for timing/retransmission truth.

**OTA pipeline:** announce → download → **validate (CRC + signature + compatibility)** → install → activate → **verify version** → rollback.
- **Power loss** → A/B dual-bank with an atomic boot-pointer flip, or a **write-protected immutable bootloader** (what your SSBL V2.0 does).
- **Corrupted image** → must be rejected *before* activation.
- **Verify** → read the version back after reboot (yours reads it twice).
- **Battery** → refuse below threshold.
- Your line: *"in-system firmware update, not wireless OTA — but the test matrix is identical."*

---

## RF — definitions only

- **RSSI** = received power now (dBm). **Sensitivity** = the minimum power for a target error rate. **Link margin = RSSI − sensitivity.**
- **Bandwidth (Hz) ≠ data rate (bit/s).** Wider bandwidth admits more noise → worse sensitivity. `C = B·log₂(1+SNR)`.
- **λ = c/f.** Lower frequency → bigger antenna, better penetration and range. Higher → smaller antenna, more bandwidth, worse penetration.
- **Modulation** — GFSK (BLE, most sub-GHz: constant envelope → efficient cheap PA), PSK/QAM (more bits/symbol, needs better SNR), OFDM (Wi-Fi), CSS (LoRa).
- **BLE vs sub-GHz** — 2.4 GHz global / short range / 1–2 Mbps / phone-compatible / crowded versus 868–915 MHz regional / long range / low rate / better penetration / bigger antenna.
- **Range vs TX power** — logarithmic; **+6 dB ≈ 2× free-space range**. Sensitivity, antenna and lower data rate usually buy more range than power, which is regulation-capped anyway.
- **SNR ≠ RSSI** — RSSI can be strong with terrible SNR if there's an interferer.
- **Antenna** — gain (dBi), pattern, polarisation, **VSWR/return loss** (want <2:1 / >10 dB), size ~λ/4.
- **PER** only means something with its test condition stated.
- **Your honest line, rehearsed:** *"None hands-on. My physical-layer work is wired — eye diagrams, bit-rate drift, receiver interference rejection — so I'm comfortable that the PHY is measured, not asserted. I know the RF vocabulary; the domain I'd need to learn."*

---

## RTOS / FreeRTOS

- **Real-time = deterministic**, not fast. Hard = a missed deadline is a failure.
- **RTOS vs bare-metal** — choose an RTOS when you have several independent activities with different deadlines. Costs: a stack per task + context-switch time.
- **FreeRTOS scheduler** = fixed-priority preemptive, round-robin among equals. **Higher number = higher priority.**
- **★ Mutex vs binary semaphore** — mutex has **ownership + priority inheritance** → protect a resource. Binary semaphore has neither → signal between ISR and task. Wrong choice = unbounded priority inversion.
- **Task notification** — a 32-bit value per task, fastest and lightest, single recipient. What your firmware actually uses.
- **ISR rules** — short, no blocking, `...FromISR` API only, `portYIELD_FROM_ISR(&woken)` at the end.
- **Deferred interrupt handling** — ISR does the minimum, notifies a high-priority task.
- **Priority inversion** — high waits on low which is preempted by medium. Fix: priority inheritance / ceiling.
- **`vTaskDelay` yields (tick resolution); a busy-wait spins** (sub-tick precision, burns CPU, starves lower priorities). Both exist in your firmware — a real trade-off you can discuss.
- **Stack overflow** — `configCHECK_FOR_STACK_OVERFLOW`, `uxTaskGetStackHighWaterMark`.

---

## CI/CD

- **Three tiers**: PR (no hardware, <10 min) → merge/nightly (hardware smoke/regression) → release (full matrix × N repeats).
- **Four preconditions for hardware in a pipeline**: headless scriptable runner with no modal prompts; machine-readable results (JUnit/TRX); a self-hosted agent owning the hardware via capabilities/demands + an exclusive lock; and idempotent provisioning that fails the job *before* the tests.
- **Trigger on firmware release**, not just on code — flash and regress automatically.
- **Publish evidence as artefacts** — a failing build must be diagnosable from its artefacts alone.
- **Distinct exit code for infrastructure failure** vs product failure.
- **Flakiness**: measure the pass rate over N runs; never retry silently; quarantine with an expiry.

---

## Azure DevOps

Organization → Project → **Test Plan** → **Test Suite** (static / requirement-based / query-based) → **Test Case** (steps, parameters, shared steps) → × **Configuration** = **Test Point** → **Test Run** → **Result** (Passed / Failed / **Blocked** / **Not applicable**).

- **Associate an automated test** to a Test Case: writes the fully-qualified method name + assembly + automation type onto the work item; `VSTest` then reports back to that case. Or publish results carrying the Test Case ID.
- **Requirement-based suite** = requirement→test traceability.
- Pipeline tasks: `VSBuild` / `DotNetCoreCLI`, `VSTest`, `PublishTestResults@2`, `PublishBuildArtifacts`.
- **Agent capabilities / job demands** route a job to the lab PC with the instrument; **Environment exclusive lock** stops two runs sharing it.
- **Your connection:** Azure DevOps outcomes (Passed / Failed / Blocked / Not applicable) map almost exactly onto your `TestResult` enum (PASS / FAIL / INCOMPLETE / NA). Say that.
- **Your honest line:** *"Haven't used it. I've been maintaining the equivalent concepts by hand — a 700-case catalogue with stable IDs and spec references, grouped into suites, with results distinguishing pass/fail/NA/incomplete. Putting that into Azure DevOps is mostly generating work items from the catalogue we already have."*

---

## Your project — 10 facts to never get wrong

1. **GRL USB-PD C2 / C2-EPR (Gen2)** — a USB Type-C / USB Power Delivery **compliance and functional tester**. The device under test is called the **UUT/DUT**.
2. **C# on .NET Framework 4.7.2**, ASP.NET Core Web API on Kestrel/libuv, **port 5001** (5002 for the offline instance), Swagger, **React** front end. 16 projects.
3. **A test is a class** with `PreConfigure / Configure / Validate`. Base: `abstract class TestCase`.
4. **Discovery is by reflection** — the test-case enum name *is* the class name → `Activator.CreateInstance`.
5. **~700 test cases**, grouped into MOIs: PD2/PD3 compliance, PD-merged, Deterministic (Type-C), Functional (63 × `TD.4.x`), DP Alt Mode, QC Legacy / QC3+ / QC4, Thunderbolt, BC 1.2, IEC, Source Power.
6. **Transport is TCP to the instrument** (`GrlEthernetLink_C2`, three sockets on port 5002) — plus USB (libusbK) and serial paths.
7. **The host never does real-time work.** It composes a message group (`IsTesterCommEn = false` captures instead of sending), uploads it, and the firmware replays it with hardware timing.
8. **Validation models what the spec allows** — `PDStateMachine.ExpectedMsgSequence` enumerates every legal continuation of an Atomic Message Sequence; the validator diffs the capture against it, including timing.
9. **Results are a tree** (`PDTestResult` composite) with a deterministic priority merge (FAIL > INCOMPLETE > WARNING > NA > PASS), rendered to HTML/CSV/PDF/XML, with a `.grltrace` capture per test.
10. **The instrument firmware (repo 2) runs FreeRTOS on a Zynq-7000** with the PD physical layer in FPGA fabric; protocol-layer state machines are `PRL_TX_*` / `PRL_RX_*` / `PRL_HR_*`; firmware update from C# covers the Zynq SD image and the TI C2000 E-Load/PPS MCUs over an I2C secondary bootloader.

**Your own biggest contributions (by comment footprint):** the deterministic Type-C test engine, the functional test suite, the PD message decode engine, the instrument communication layer, and the module helper / configuration layer.

---

## The last thing to read before you walk in

> "The radio domain is new to me, and I'll say so. The job — read a spec test case, turn it into a maintainable C# test that drives firmware over a link, validate timed protocol behaviour, and make the failure diagnosable by someone else — is exactly what I do every day."
