# Phase 19, 20, 21 — Project Pitch, Day-to-Day, and Deep-Dive Grilling

---

# Part 1 — "Tell me about your current project"

Three lengths. Learn the 1-minute one properly; the others are a subset and a superset.

## 30 seconds

> "I work on the C#/.NET software for a USB Power Delivery compliance and functional tester — it's a lab instrument that emulates a charger, a laptop or a cable and validates a device under test against the USB-IF specification.
>
> My side is the automation: a .NET test framework with around seven hundred test cases, the layer that drives the instrument's firmware over Ethernet, and the validation engine that decodes the captured protocol traffic and decides pass or fail. There's a web API and a React front end on top, and the instrument itself runs FreeRTOS on a Zynq with an FPGA doing the physical layer."

## 1 minute

> "The product is the GRL USB-PD C2 tester. It's a lab instrument that sits between a device under test and our software, emulates the other end of a USB Type-C connection, and validates the device against the USB-IF compliance test specification.
>
> The software is C# on .NET Framework 4.7.2. It's an ASP.NET Core web API with a React front end, and underneath that a layered test framework — around sixteen projects. The layers are: a framework core with configuration, logging, results and reporting; a protocol library that decodes USB-PD messages; a device library that talks to the instrument over TCP; and the test content itself, about seven hundred test cases organised into compliance suites.
>
> The design point I like is how tests work. A test is a class with three hooks — `PreConfigure` decides whether the test applies to this device, `Configure` programs the instrument with the exact stimulus including what to deliberately get wrong, and `Validate` runs afterwards against everything we captured. The engine finds test classes by reflection — the test-case enum name is the class name — so adding a test is one enum entry and one class, with no change to the engine.
>
> The validation is the interesting part. Instead of comparing against a recorded golden log, we model what the spec *allows* — for a Source_Capabilities message the legal endings are Accept then PS_RDY, or Reject, or Wait — and we diff the real capture against that model, including the timing, because most compliance failures are timing failures.
>
> On the firmware side, the instrument runs FreeRTOS on a Zynq-7000 with the protocol physical layer in FPGA fabric. I don't own that firmware, but I read it constantly, because when a test fails my first job is to work out whether the defect is in the device, in our instrument firmware, or in my automation."

## 2 minutes — senior version

Use the 1-minute version, then add:

> "A few things about how the system is put together that I think are the right calls.
>
> **The host never does real-time work.** A Windows PC can't reliably time a thirty-millisecond protocol deadline. So a test *composes* its stimulus as a message group — there's a flag in the device API that makes the same call capture the command buffer instead of transmitting it — uploads that group to the instrument, and the FreeRTOS firmware and FPGA replay it with hardware timing. The host then does the non-real-time work: decoding, validation and reporting. That split is what makes microsecond accuracy possible at all.
>
> **Shared building blocks rather than copy-paste.** There are bring-up procedures that two hundred tests reuse, fifteen common checks and eighteen common procedures that run automatically against every capture, and a helper library of assertions. A test class contains only what makes that test different. And the bring-up and test-condition matrices are loaded from JSON per specification section, so a spec revision is a data change.
>
> **Evidence over verdicts.** Every assertion records expected value, obtained value and the packet index in the capture, so a failing report is already a bug report. And every test saves its raw trace, which means we can replay a capture offline through the validation engine — we have an xUnit suite that does exactly that against recorded JSON captures, including deliberately bad ones, so I can regression-test my validation changes with no hardware.
>
> **Where I've spent most of my own time** is the deterministic Type-C test engine and the functional test suite — the TD.4.x connection tests covering Source, Sink, DRP, Try.SRC, Try.SNK, accessory modes, role swaps and dead-battery behaviour — plus the protocol message decode engine and the communication layer to the instrument.
>
> **What I'd change if I were starting again**: extract an interface at the transport boundary so the whole stack is mockable and most of it could run in CI without hardware; replace `Thread.Abort` cancellation with a `CancellationToken`, which is also what's blocking a move to modern .NET; and emit a standard test-result format so a pipeline can consume the results directly."

### Rules for delivering it
- Lead with **what the product does**, not with technology names.
- Say "around 700 test cases", "about sixteen projects" — approximate is fine and sounds natural; precise numbers sound rehearsed.
- **Do not** claim you wrote the firmware. Say "I read it" and mean it.
- End on the "what I'd change" paragraph if they seem senior — it invites the best follow-up questions and you have answers for all of them.

---

# Part 2 — "What do you do day to day?"

> "It varies, but a typical week has five things in it.
>
> **Implementing test cases.** A specification revision lands, or a customer needs a test we don't have. I read the spec clause, work out what's actually being asserted and how it's measured, check what already exists in the framework, and write the test class. Usually the test itself is small because the bring-up and the assertions already exist — if I'm writing a lot of code in a test class, that's a signal something belongs in the shared layer instead.
>
> **Debugging failures.** This is the biggest slice. A test fails and the question is always the same: is it the device, our instrument firmware, or my automation? I start from the capture — every assertion tells me the packet index — then the hex log of what we actually sent, then if needed I go into the firmware source to understand what it was doing. A lot of the value I add is being able to say confidently 'this is a device defect, here's the trace and the spec clause' rather than escalating something ambiguous.
>
> **Maintaining the framework.** Refactoring shared helpers, keeping the reflection-based discovery honest when things get renamed, keeping the report output sane, fixing the things that make the suite flaky. And reviewing other people's test code — mostly pushing back when a test reimplements something that already exists.
>
> **Running and triaging regression.** Running the suite across device types and ports, going through what failed, separating real regressions from environment problems from tests that were too strict.
>
> **Protocol and firmware-interaction work.** Extending the message decoder when the spec adds messages, working on the validation engine, and working with the firmware team when a test needs a capability the instrument doesn't expose yet.
>
> Plus the normal things — Git, branches per issue ID, pull requests with review, and keeping the offline replay tests passing so I don't break the validation engine for everyone else."

**Only claim beyond this if asked.** Do not volunteer CI/CD ownership or Azure DevOps.

---

# Part 3 — Repository-specific deep-dive questions

These are the questions an interviewer who has looked at your code would ask. Rehearse the ones marked ★.

## Architecture and design

**★ "Walk me through what happens when someone clicks Start Execution."**
→ The 12-step flow in [02](02-repo1-architecture.md) §B.2. Practise saying it in 90 seconds without notes.

**★ "Why did you discover tests by reflection instead of a registry?"**
> So adding a test requires no change to the engine — open/closed. The cost is that a rename breaks at runtime instead of compile time, and `GetTypes()` has a startup cost we don't cache. If I did it again I'd use an explicit `[TestCase(enum)]` attribute scanned once into a dictionary: same convenience, explicit intent, and a duplicate or missing mapping becomes a startup error.

**"Why is `TestCase` an abstract class and not an interface?"**
> Shared state and shared behaviour. Every test holds the same four collaborators and uses the same `StartStreamer`/`StopStreamer`, and there's exactly one lifecycle so a single is-a hierarchy is honest. An interface can't hold state. Where I *would* use an interface is the transport layer, which today has three implementations with the same shape and no common type.

**★ "You have a big `if/else if` chain dispatching MOIs. Doesn't that violate open/closed?"**
> Yes, and it's the one place where adding a capability means editing existing code — which is inconsistent with how test discovery works. A `Dictionary<PDAppMode, Func<PD_TesterCommCfg, ITestFactory>>` registry, or the same attribute-based discovery we already use for tests, would fix it. The reason it hasn't been done is that MOIs change roughly annually while tests change weekly, so the pain is low — but I wouldn't defend it as good design.

**"Why does `ControllerHelper` use a singleton?"**
> Because there's one physical instrument, so one engine. But the implementation is the classic check-then-create, which isn't thread-safe — two concurrent HTTP requests at startup could build two engines and open two sets of sockets to the same instrument. `Lazy<T>` with `ExecutionAndPublication`, or registering it in the DI container as a singleton, fixes it. Registering it in DI would also make the controllers testable, which they currently aren't because they reach out to a static.

**"There's no dependency injection in the core. Is that a problem?"**
> For testability, yes. The controllers can't be unit-tested because they reach out to a static singleton, and the device layer can't be mocked because the transports don't share an interface. The refactor I'd do is `ITesterLink` plus `ITesterEngine` registered in DI — same runtime behaviour, but then I can substitute a simulator and run most of the framework in CI without hardware. Moq is already in the solution for the offline tests, so the tooling is there.

## Concurrency and cancellation

**★ "Why do you use `Thread.Abort()` and what would you replace it with?"**
> See [06](06-csharp-dotnet-qa.md) §C3. Key beats: arbitrary injection point, can fire while holding the socket lock, leaves the instrument in an undefined state, unsupported on .NET 5+. Replace with a `CancellationToken` checked between tests and at the natural wait points in the device API, plus device recovery in `finally`. Note that the *shape* is already right — `Forcestop` marks in-flight tests INCOMPLETE with a reason before aborting.

**"What happens if a test fails halfway through execution?"**
> The engine catches it per test, logs it with the stack trace, marks that test and continues — one bad test must not end a four-hour run. Then the per-test cleanup runs: streamer stopped, trace saved. If the *instrument* is the problem rather than the test, the comm layer raises a force-stop through a delegate and the whole run stops with every remaining test marked INCOMPLETE and the reason recorded, which is the right outcome because results from an instrument in an unknown state are worthless.

**★ "How do you ensure cleanup happens after a failed test?"**
> Three layers of `finally`: per test, per MOI (a full FPGA/firmware reset so configuration can't leak forward), and per run (`HostState = READY`, module state reset, PDM mode off, tester comms re-enabled). The principle is that cleanup must not depend on the test succeeding and must not itself throw — every cleanup path is inside try/catch that logs. The gap I'd close is what happens if *cleanup* fails: today we'd keep going; we should escalate to a harder reset and, failing that, mark the whole suite as compromised.

**"How do you prevent two tests from accessing the same device simultaneously?"**
> Today, by construction: a run is a single thread on a single instrument, and the transport serialises every command/response pair behind a lock so two callers can't interleave. There's also a host-level `HostState` guard — a second execution request while one is running is rejected with a message. What we don't have is protection against two *processes* on the same PC talking to the same instrument; the app does check for an existing instance, but that's a weaker guarantee than a real lease. In a multi-station design I'd use a lease with a TTL.

**"Why two lock objects in the Ethernet link?"**
> Because there are two independent socket pairs — the command channel and the bulk data channel — so a long streamer read doesn't block a short command. `Write` and `Read` share the same lock because a command is a write-then-read transaction that has to be atomic; if they had separate locks, two callers could interleave and one would read the other's response. The invariant is conventional rather than structural though — a single `Transact()` method would make it impossible to get wrong.

## Communication

**★ "What happens if a communication call times out?"**
> It depends which kind, and that's actually a weakness. Our read loop retries up to a hundred times with a wait when no data arrives, so "timeout" can mean the socket errored, or the device didn't answer, or the response was shorter than the buffer we were waiting for — and a short read looks exactly like a timeout. All three surface the same way today. On a genuine socket failure the write path reinitialises the connection and retries once, and if that fails the comm layer raises a force-stop through a delegate, which stops the run and marks in-flight tests INCOMPLETE with "Tester ethernet error occurred". The improvement I'd make is distinguishing the three cases explicitly and adding a correlation ID so a request and its response can be matched in the log.

**"Why TCP and not UDP?"**
> Commands must not be lost or reordered, and the instrument's responses are the basis of every verdict — so reliability and ordering are non-negotiable, and TCP gives both. We set `NoDelay` to disable Nagle because our frames are small and latency matters more than efficiency. UDP would make sense for a lossy high-rate telemetry stream where you'd rather drop a sample than stall, but not for control.

**"Why three separate TCP connections?"**
> Separation of concerns and head-of-line blocking. A bulk sample read can take tens of milliseconds; a command shouldn't queue behind it. Separate sockets with separate locks means a streamer read and a command can be in flight at once.

## Testing and results

**★ "How do you determine whether a failure is caused by firmware or automation?"**
> Four checks in order: the framework reports whether *we* initiated every configured message, so a tester-side failure is INCOMPLETE not FAIL; the common-check layer validates our own messages in the capture too, so a malformed tester frame shows up; there's an instrument self-test with VBUS/VCONN voltage and current limits; and I can replay the same capture offline through the validation engine — if the verdict changes, the bug is mine.

**"Why is a test result a tree?"**
> Because a test has test conditions — PD2 chunked, PD3 chunked, PD3 unchunked, or three cable conditions — and each condition has multiple spec assertions. The report needs every level, and the top-level verdict has to be derived deterministically. `AddTest` rolls a child's verdict up through a priority rule where FAIL beats INCOMPLETE beats WARNING beats NA beats PASS, with a strategy hook so the functional suite can use a slightly different rule.

**★ "Why do you have NA and INCOMPLETE as separate outcomes?"**
> They mean completely different things and conflating them destroys a report. NA means the test doesn't apply to this device — it declared no EPR support, or it's a Sink and this is a Source test. INCOMPLETE means it should apply but we couldn't get the evidence — the fixture was missing, the user stopped the run, the instrument errored. NA is expected and fine; INCOMPLETE is a gap in coverage you need to close. If both were FAIL, the report would be mostly red and people would stop reading it.

**"How do you test the test framework itself?"**
> Recorded captures replayed through the production validation engine, asserted with xUnit. The fixture set includes deliberately bad captures — an EPR entry that should fail, a power-role swap with no response, a capture with a tester packet missing — and those assert that the verdict is FAIL. That's golden-file regression of the validator with no hardware. Honestly, it's thinner than it should be: one of the test classes runs nine logical checks inside a single `[Fact]`, so one failure reports one red test; `[Theory]` with `[MemberData]` would give one result per case.

**"How do you handle firmware version differences?"**
> Two mechanisms. The instrument's firmware version is read at connect and compared against what the software expects — a mismatch blocks or warns before any test runs, so we never debug a compatibility problem as a product bug. And for the *device's* spec revision, each test declares which PD revisions it applies to and the test-condition matrix expands it into the right set of executions, with the expectations keyed by revision and loaded from JSON rather than branched in code.

## Scale and maintainability

**★ "How would you make this framework scalable?"**
> The honest answer is process-level parallelism across stations rather than threads, because the framework has a lot of static state — a static host object, a static module helper, a static controller dictionary, static report paths — that would need a very large refactor to be thread-safe. One agent process per instrument gets me linear scaling in weeks instead of months, and I can refactor the static state opportunistically afterwards. Within a station, parallelism isn't available anyway: a protocol conversation is one exclusive, sequential exchange over one physical link.

**★ "How do you keep this maintainable as hundreds of tests are added?"**
> Convention-based discovery so the engine never changes; one lifecycle so there's one thing to learn; shared bring-up and assertion layers so a test contains only its differences; automatic cross-cutting checks so spec-mandated validation is never re-implemented; and test conditions as data.
>
> And I'd name the actual debt rather than pretend: some helper classes have grown to several hundred kilobytes in a single file. That's the real maintainability problem — not the number of tests, but helpers becoming god objects. I'd split them along seams that already exist, by PD revision and by message category, before adding more to them.

**"What's the worst code in the repository?"**
> Pick two and be specific — this question rewards candour:
> - The static god classes (`ModuleHelper`, and the largest validation helper) — too many responsibilities in one file, and static mutable state that blocks parallelism and testability.
> - `Thread.Abort()` for cancellation, plus the empty `catch (Exception) { }` blocks that hide real failures.
> Then say what you'd do and in what order, and why you haven't: *"the reason these survive is that they're load-bearing and there's no safety net to refactor behind — which is exactly why I'd extend the offline replay suite first."*

## Curveballs

**"If I gave you our codebase tomorrow, how would you get productive?"**
> Read the entry point and follow one test all the way through: how it's selected, how it's discovered, how it talks to the device, how the verdict is produced, how it's reported. Once I can trace one test end to end I can place anything else. Then I'd write one new test, deliberately break it, and watch how the failure surfaces — that tells me more about a framework's real quality than reading it does. Then I'd ask which tests people don't trust, because that's where the debt is.

**"What would you do in your first month?"**
> Learn the framework by converting manual test cases — which is the job anyway and is also the fastest way to find the rough edges. Shadow a regression triage session to learn how failures are actually diagnosed here. And keep a list of the friction I hit as a newcomer, because that list is worth more in month one than it will ever be again.

**"What don't you know that worries you?"**
> The RF and radio domain — I've been straight about that. Also, I'd want to understand early how your team decides whether a failure is a product bug or a test bug, because in my experience that judgement is where an automation engineer either earns trust or loses it.
