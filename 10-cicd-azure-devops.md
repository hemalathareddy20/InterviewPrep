# Phase 14 + 15 — CI/CD, Regression, and Azure DevOps Test Management

---

## What's actually in your repo — state this accurately

**Searched for and NOT found:** `azure-pipelines.yml`, `.github/workflows/`, `Jenkinsfile`, `.gitlab-ci.yml`, `Dockerfile`, any build script beyond NuGet's own install/uninstall PowerShell. The only `.github` content is `PULL_REQUEST_TEMPLATE.md`. Zero source hits for azure/vsts/TFS/Azure DevOps.

**What DOES exist that is genuinely CI/CD-adjacent — claim these:**

| Capability | Evidence |
|---|---|
| **Branch + PR discipline** | Branch names are issue IDs (`EV1-T2521`, and code comments reference `EV1-I1249`, `AA-I6426`, `EV1-T1031`) — an issue tracker is wired into the workflow. `CODEOWNERS` enforces review. A PR template exists. |
| **Automated regression execution** | `RerunCount` + `RepeatCondition` (All / Fail / Pass / Incomplete tests) — re-run only failures, N times, each into its own `Run{n}_Rep{m}_{timestamp}` folder |
| **Unattended multi-DUT sweep** | 5-port mode: `TestExecutionEngine` loops `FivePortExecData`, drives an external switch box over serial, and runs the configured suite per port with its own configuration and repeat conditions |
| **Hardware-free regression** | `XUnit_TestFramework` — xUnit + Moq replaying recorded JSON captures through the production validation engine. **This is the part that could run in CI today.** |
| **UI regression** | `Selenium\C2WebAppTest.py` — drives the React app end to end across 5 ports (connect → product capability → test config → execute → results), with a multi-port board helper |
| **Machine-readable artefacts** | CSV and XML report generation (`C2_XML_Report_Generation.cs`, `API_CSV_Report.cs`) alongside HTML/PDF |
| **Version gating** | Instrument firmware / E-Load / PPS versions read at connect and compared to the expected version; execution blocked or warned on mismatch |
| **Automated firmware provisioning** | `FirmwareUpdateManager` can flash the instrument from code — i.e. a pipeline *could* set the device under test to a known firmware version before running |
| **Test-run telemetry** | Per-test overall/config/validate timings written to a dedicated log; run summary with total/pass/fail counts (`TestResultSummary`) |

**How to say it:**
> "We don't have a pipeline definition in the repo — regression is triggered from the application and from scripts on lab machines. What we do have is the parts a pipeline needs: unattended multi-port execution with rerun-on-failure, machine-readable CSV/XML output, a hardware-free replay suite in xUnit, Selenium UI regression, and the ability to flash a known firmware version from code. Wiring that into Azure DevOps is something I'd want to do, and I've thought about how."

---

# Part 1 — How to build CI/CD for hardware-in-the-loop test automation

## 1.1 The three-tier pipeline

```
TIER 1 — Every commit / PR       (no hardware, < 10 min)
  build all projects (msbuild / dotnet build)
  static analysis + analyzers
  unit tests (MSTest projects)
  offline replay regression (XUnit_TestFramework — validation engine vs recorded captures)
  publish results + coverage
  → gate the PR on this

TIER 2 — Merge to main / nightly  (real hardware, ~1 hr)
  self-hosted agent on a lab PC with the instrument attached
  flash the instrument to the target firmware version   ← FirmwareUpdateManager
  run the instrument self-test                          ← ControllerSelfTestEngine
  run the SMOKE suite against a reference DUT
  publish results, traces, logs as artefacts

TIER 3 — Release candidate / weekly  (full regression, hours)
  full MOI suite across the 5-port board and multiple reference DUTs
  N repeats to measure stability, not just pass/fail
  compare against the previous baseline; report deltas
  publish the full report bundle
```

## 1.2 "How would you integrate a C# test framework into CI/CD?"

> "Four things have to be true before hardware tests belong in a pipeline.
>
> **1. The runner must be headless and scriptable.** Today our entry point is an HTTP POST from a browser and there are `MessageBox` prompts in the execution path — `StartTestExecutionThread` shows a modal if the firmware version doesn't match, and several tests show operator prompts. Step one is a CLI entry point plus a non-interactive mode where every prompt has a configured default and a prompt with no default is a hard failure, not a hang. That's the single biggest blocker and I'd do it first.
>
> **2. Results must be machine-readable.** We already emit CSV and XML; I'd add JUnit or TRX so `PublishTestResults@2` can ingest them directly and Azure DevOps can show per-test history, flakiness and ownership without anyone parsing HTML.
>
> **3. The agent must own the hardware.** A self-hosted agent on the lab PC with a **demand/capability** tag like `usbpd-tester=c2gen2` so only jobs that need that instrument are scheduled there, and a pipeline-level lock so two runs can't grab the same instrument. Azure DevOps supports both — agent capabilities and an exclusive-lock deployment environment.
>
> **4. The device state must be established, not assumed.** Every run starts by flashing the instrument to a known firmware, running the instrument self-test, and power-cycling the DUT. If any of that fails the job fails *before* the tests, so you never debug a bad environment as if it were a product bug.
>
> Then the split: everything that doesn't need hardware runs on every PR — build, unit tests, and our offline replay suite. Hardware runs nightly and on release candidates."

## 1.3 "How do you trigger regression?"

> Four triggers, deliberately different in scope:
> - **PR** → build + hardware-free tests only. Must be under ~10 minutes or people stop caring.
> - **Merge to main** → smoke suite on hardware.
> - **Nightly scheduled** → full regression, one reference DUT of each class.
> - **Manual / release** → the full matrix: all MOIs × all reference DUTs × N repeats, plus the previous firmware version for comparison.
>
> Plus one that's specific to this domain: **triggered by a firmware release**. When the firmware team publishes a new image, the pipeline flashes it and runs the regression automatically. That closes the loop the JD cares about.

## 1.4 "How do you publish test results?"

> Emit a standard format and let the platform do the work. JUnit XML or TRX → `PublishTestResults@2` in Azure DevOps. That gives per-test history, automatic flaky-test detection, and failure trends for free.
>
> Alongside that, publish the *evidence* as build artefacts: the HTML report bundle, the `.grltrace` capture per test, and the debug log. The rule I'd enforce: **a failing build must be diagnosable from its artefacts alone**, without anyone going to the lab machine. Our reports already carry expected-vs-obtained and the packet index, so that's mostly satisfied — it's the plumbing that's missing.
>
> For trends I'd also push the run summary (total/pass/fail/incomplete, duration per MOI) to a dashboard, because for hardware suites the interesting signal is drift over time, not a single run.

## 1.5 "How do you handle hardware-dependent tests?"

> Five mechanisms.
> **Tag them.** Every test declares what it needs — instrument type, fixture, cable, DUT class. In our framework that information already exists as `SupportedDevice`, licence requirements and fixture checks; it just isn't exposed as a selection filter for a pipeline.
> **Pool and lock the hardware.** Agents advertise capabilities; jobs declare demands; an exclusive lock prevents two runs sharing an instrument.
> **Verify before testing.** Self-test the instrument, check firmware versions, confirm the fixture is present. Our tests already report `NA` with a reason when a fixture is missing — in CI that must be a *visible* NA, not a silent pass.
> **Fail fast and cleanly.** If the instrument is unreachable, abort the job with a distinct exit code so it's triaged as infrastructure, not as a product regression. That distinction is the difference between a team that trusts its pipeline and one that ignores it.
> **Have a hardware-free path.** Offline capture replay means the majority of validation-logic changes can be regression-tested with no instrument at all. That's what keeps the PR gate fast.

## 1.6 "How do you separate smoke tests from regression?"

| | Smoke | Regression |
|---|---|---|
| Purpose | Is the build/device fundamentally working? | Has anything changed anywhere? |
| Size | 10–30 tests | Everything |
| Duration | < 10 min | Hours |
| Selection | Highest-signal, most-used paths: connect, read capabilities, one bring-up per role, one test per MOI | Full CTS matrix |
| Trigger | Every merge | Nightly / release |
| On failure | Block the merge | Triage and file |

> Our MOI/test-group structure already gives a natural selection axis — a smoke suite is "one representative test per `TestGroup`". I'd derive it from the existing metadata rather than maintaining a separate list, so it can't drift.

## 1.7 "How do you manage test environments?"

> Treat the lab station as a deployable environment, not as someone's desk.
> - **Inventory**: each station has an ID, an instrument serial number, a firmware version, a fixture set and a DUT set — all recorded. We already read the instrument serial and all firmware versions at connect and put them in the property bag, so the run *knows* which station it ran on; that belongs in the report header.
> - **Provisioning**: firmware flashing is automated, so "set this station to firmware X" is a pipeline step.
> - **Configuration as data**: per-station config files, not code.
> - **Health checks**: self-test before every run; fail the job on a health-check failure with a distinct signal.
> - **Isolation**: one run per station at a time, enforced by the agent pool.

## 1.8 "How do you prevent flaky tests?"

> Prevention first, then measurement, then policy.
>
> **Prevention:** every test brings the device up from a known state; full instrument reset between MOIs; no test depends on another; waits are on conditions with timeouts rather than fixed sleeps; timing measured from hardware timestamps, never from the PC clock; anything a test changes it restores.
>
> **Measurement:** run the suite N times and record the pass rate per test, not just the last verdict. A test at 92% is flaky even though it 'passed' last night. Azure DevOps has built-in flaky detection if you publish standard results; I'd also keep the measured value, not just the verdict, so you can see a test drifting toward its limit before it starts failing.
>
> **Policy:** a newly flaky test gets an owner and a deadline. Quarantine — still runs, doesn't block the gate — is acceptable for a short, tracked period; permanent quarantine is how suites die. And the hard rule: **never make a test pass by retrying it silently.** If it needs three attempts, that's the finding.
>
> One domain-specific point: in hardware testing some variance is physical. A timing measurement will have a distribution. The right response is to assert against the spec limit with the *measured value reported*, so a test that passes at 29 ms against a 30 ms limit is visible as at-risk rather than green."

---

# Part 2 — Azure DevOps Test Management

**Say this up front:** *"I haven't used Azure DevOps. We use Git with an issue tracker referenced in branch names and commits, and code review through pull requests with CODEOWNERS. I've read up on Azure DevOps Test Plans for this conversation and I'm comfortable with the model — it's the same shape as other test-management tools."*

## 2.1 The object model — learn these

| Object | What it is |
|---|---|
| **Organization → Project** | Top-level container; a project holds boards, repos, pipelines, test plans, artifacts |
| **Work item** | The unit of tracking. Types: Epic, Feature, User Story / PBI, Task, Bug, **Test Case**, **Test Suite**, **Test Plan**, Shared Steps, Shared Parameters |
| **Test Plan** | A container for a testing effort, usually tied to a release or iteration |
| **Test Suite** | A grouping inside a plan. Three kinds: **static** (hand-picked cases), **requirement-based** (all cases linked to a given user story — this is what gives you requirement traceability), **query-based** (a saved work-item query, so the suite updates itself) |
| **Test Case** | A work item with *steps*, each step having an action and an expected result; supports **parameters** for data-driven cases and **shared steps** for reuse. Can be linked to requirements, to bugs, and to an **automated test method** |
| **Test Point** | A Test Case × a configuration (e.g. "Windows 11 / Chrome"). The thing that actually has a pass/fail outcome |
| **Test Configuration / Variable** | The environment dimensions (OS, browser, device, firmware version). A case runs once per configuration |
| **Test Run** | An execution instance containing results for a set of test points, with attachments, duration, and the build it ran against |
| **Test Result** | The per-point outcome: Passed / Failed / Blocked / Not applicable / Paused / In progress |
| **Pipeline** | Build/release automation (YAML or classic). Tasks like `VSBuild`, `DotNetCoreCLI`, `VSTest`, `PublishTestResults`, `PublishBuildArtifacts` |
| **Environment** | A deployment target with approvals, checks and **exclusive locks** — how you stop two runs sharing one instrument |
| **Agent pool / Agent capabilities & demands** | Self-hosted agents advertise capabilities; jobs declare demands — this is how a job is routed to the lab PC that has the tester attached |

## 2.2 "How do you connect a manual test case to an automated test?"

This is the question they're most likely to ask, because it's literally the job.

> "In Azure DevOps you **associate** an automated test method with a Test Case work item. The classic route is from Visual Studio: open the test method, right-click → *Associate to Test Case*, and it writes three fields onto the work item — the **fully qualified method name**, the **assembly/container**, and the **automation type**. After that, running the test plan with `VSTest` executes the real test and the outcome flows back to that Test Case's test points.
>
> The important consequence is **traceability**: because the Test Case is linked upward to a requirement or user story and downward to an automated method, you can answer 'which requirements are covered by automation?' and 'which requirement is this failure affecting?' from one view. That's the actual value of test management — not the execution, which a pipeline does anyway.
>
> There's also an API route — `PublishTestResults` with a results file where each result carries the Test Case ID — which is what I'd use for a framework like ours that doesn't run under the VSTest adapter. Our tests are our own classes discovered by reflection, not `[TestMethod]`s, so rather than pretending to be MSTest I'd emit JUnit/TRX with the Azure DevOps Test Case ID carried as the test identifier. Our test catalogue already has a unique ID per test — the enum name and the spec ID like `TD.4.2.1` — so mapping that to a Test Case ID is a one-time exercise."

## 2.3 "How would you convert your manual test cases into Azure DevOps?"

> "Three stages.
>
> **Stage 1 — model the catalogue.** Our master list already exists as an enum with a description per test (`TD.4.2.1 Source Connect Sink Test`), grouped by `TestGroup` and by MOI. That maps directly: MOI → Test Plan or top-level Suite, TestGroup → Suite, each test → Test Case. I'd generate the work items from the enum via the REST API rather than hand-entering 700 of them, and keep the enum as the source of truth so they can't drift.
>
> **Stage 2 — link.** Each Test Case gets linked to the requirement it verifies — for us that's a CTS spec clause — and to the automated implementation. Configurations become the dimensions we already vary: DUT type, cable type, PD revision, port.
>
> **Stage 3 — publish.** The pipeline runs the suite and publishes results with the Test Case IDs, so the plan shows real execution history rather than someone ticking boxes.
>
> The part I'd be careful about: **not every test should be automated.** Tests that need a human judgement, or a physical action we can't perform, stay manual and are marked as such — and the plan should show that honestly rather than reporting 100% automation."

## 2.4 Terms to have ready

- **Test Plan vs Test Suite vs Test Case** — plan = effort, suite = grouping, case = the test itself.
- **Requirement-based suite** — auto-populated from cases linked to a work item; gives requirement→test traceability.
- **Query-based suite** — auto-populated from a work-item query; self-maintaining.
- **Shared steps / shared parameters** — the test-management equivalent of a shared bring-up procedure; reuse across cases.
- **Configuration** — case × environment = test point.
- **Outcomes** — Passed, Failed, **Blocked** (couldn't run — environment), **Not applicable** (doesn't apply). Note this maps almost exactly onto your `TestResult` enum (`PASS`, `FAIL`, `INCOMPLETE`, `NA`) — **say that, it's a great connection.**
- **Traceability matrix** — requirements vs tests vs results.
- **Test Impact Analysis** — run only the tests affected by a change.
- **Flaky test detection** — built in when you publish standard results.

## 2.5 The honest, strong closing line

> "I haven't used Azure DevOps Test Plans, but I've been maintaining the equivalent concepts by hand: a catalogue of 700 test cases with stable IDs and spec references, grouped into suites, with per-run results that distinguish pass, fail, not-applicable and incomplete, and reports linked back to the evidence. Putting that into Azure DevOps is mostly a matter of generating the work items from the catalogue we already have and emitting results in a format the platform understands. I'd expect to be productive on it quickly, and I'd rather say that than claim experience I don't have."
