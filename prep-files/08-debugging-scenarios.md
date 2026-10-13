# Phase 10 — Debugging Scenarios

Debugging questions are the best predictor of seniority. The pattern that scores: **isolate the variable, don't guess.** Always state (1) what you'd observe first, (2) how you'd bisect, (3) what would confirm each hypothesis, (4) how you'd prevent it recurring.

Open every answer with the same sentence:

> "First I'd want to know whether the defect is in the DUT, in our instrument firmware, in the automation, or in the environment — and I'd pick the cheapest observation that splits those four."

---

## Scenario 1 — "It passes manually but fails through automation"

**What this really means:** the automated run differs from the manual run in some way nobody has written down.

**Structured answer:**

> "This is almost always a difference in state, timing or configuration that the manual run does implicitly.
>
> **Step 1 — get both traces.** We save a `.grltrace` per test plus a hex log of every byte sent to the instrument. I'd capture the same test manually and automatically and diff the two conversations, not the two verdicts. Usually the difference is visible within ten packets.
>
> **Step 2 — the five usual causes, in order of likelihood:**
> - **Starting state.** Manually, the operator plugs the device in fresh. In the suite it's the 40th test and the device has just come out of a Hard Reset or an EPR mode exit. Our fix for this is a full FPGA/firmware reset between MOIs (`ControllerConfigEnum.FW_FPGA_FullReset`) and a bring-up procedure per test — if a test fails only in the suite, my first suspicion is a missing reset.
> - **Timing.** A human takes seconds between steps; automation takes milliseconds. Devices frequently have undocumented settling requirements. I'd add an instrumented delay and bisect: if a 200 ms wait after attach fixes it, the question becomes *is that a real device requirement or are we violating a spec timer?* — and if it's a real requirement, it belongs in the shared bring-up procedure with a comment saying why, not sprinkled in one test.
> - **Configuration.** The manual run uses the UI defaults; the automated run uses whatever the previous test left behind. Anything a test toggles in `Configure()` must be restored — a missing restore is a cross-test dependency.
> - **Environment.** Different cable, different port, a different tester serial number, a different firmware version. We read and log the instrument serial, firmware, E-Load and PPS versions at connect, so I can compare those between the two runs immediately.
> - **The assertion itself.** Sometimes the manual 'pass' is an operator eyeballing a waveform and the automation is checking a tighter limit. Then automation is right and the manual procedure was lenient — worth confirming with the spec clause before filing a device bug.
>
> **Step 3 — confirm.** Replay the automated capture offline through the validation engine. If the verdict is the same offline, the disagreement is about what happened on the wire, not about the analysis. If it differs, my validator is the bug.
>
> **Prevention:** make the automated run *more* explicit than the manual one — always bring up from a known state, always log the full configuration applied, and never rely on a previous test leaving something set."

**Follow-up: "What if adding a delay fixes it — do you ship the delay?"**
> Only with a justification. A `Thread.Sleep` that fixes a bug you don't understand is a time bomb. I'd first check whether the spec allows that settling time; if it does, it becomes a named constant in the shared procedure with the spec reference. If it doesn't, we've found a device bug and the delay is masking it — I'd file it and keep the test failing.

---

## Scenario 2 — "Firmware responds correctly 9 times out of 10"

**What this really means:** a race, a timing margin, or a resource that isn't reset.

**Structured answer:**

> "A 90% pass rate is data, not noise — I'd treat the 10% as the finding.
>
> **Step 1 — quantify.** Run it 100 times and log, per iteration, the verdict *and* the measured quantity — not just pass/fail. If the response time is normally 22 ms and failures are at 31 ms against a 30 ms limit, that's a margin problem, not a random one, and the distribution tells you so immediately. Our reports already print the obtained value for every timing assertion, so a hundred runs give you a histogram for free.
>
> **Step 2 — correlate.** Does failure correlate with:
> - **Position in the run** → resource leak, thermal, or state left by a previous test. We log instrument temperature errors (`Get_Tester_Temperature_Error`) precisely for this.
> - **The first iteration** → cold start / initialisation ordering.
> - **A specific DUT unit** → hardware variance, not firmware.
> - **A specific packet index in the capture** → look at what precedes it every time.
>
> **Step 3 — hypotheses in order:**
> - **Race in the firmware** — two events arriving close together. In a FreeRTOS system that's an ISR and a task touching shared state, or a message arriving during a state transition. This is the kind of bug where the firmware's `Pre/Current/Next` state triple is gold: the failing trace tells you which transition it was in.
> - **Timing margin** — the device is at the edge of a spec timer. Confirm by varying our own timing deliberately: if pulling our stimulus 2 ms earlier moves the failure rate, it's a margin.
> - **Our stimulus is the variable** — check whether *we* sent the same thing every time. Our framework reports whether all configured messages were initiated (`PacketInitStatus`); if that's ever false, the tester is the flake.
> - **Buffer/queue depth** — the failure appears only when a burst arrives.
>
> **Step 4 — narrow the observation window.** Rather than a 5-minute test, build a minimal reproduction that does only the suspect exchange in a tight loop. Faster iterations, cleaner traces.
>
> **Deliverable:** the bug report says 'fails 8 times in 100; obtained values are bimodal at 22 ms and 31 ms; failures always follow a Soft Reset in the previous test; trace attached at packet index N.' That's actionable. 'It's flaky' is not."

**Follow-up: "The firmware team says they can't reproduce it."**
> Give them the capture and the exact stimulus sequence — our message groups are deterministic and uploadable, so the same test can be re-run byte-identically. If they still can't, the variable is their environment: different build flags, different board revision, different optimisation level. And I'd ask whether they can add a debug counter or a state-report command; an intermittent bug in a state machine is far easier to find with the firmware's own state history than from outside.

---

## Scenario 3 — "Communication timeouts occur randomly"

**This is the diagnostic-thinking question.** Answer it as a decision tree.

> "There are five candidates and I'd separate them in this order, cheapest first.
>
> **1. Is it the physical layer / environment?**
> Symptoms: bursty, correlates with cable movement or a specific port, affects all tests. Check: does the link reconnect cleanly? Any NIC errors? Swap the cable and the port. Our code logs every socket failure and attempts reinitialisation (`RetryWriteOperation` closes and reopens the TCP connection), so the log tells me whether it's one failure retried successfully or a storm.
>
> **2. Is it the C# automation?**
> Symptoms: correlates with a code change; only under concurrency; the timeout is *our* timeout, not the device's. Check: the socket timeouts are set to 5 s (`ReceiveTimeout`/`SendTimeout`) and the read loop retries up to 100 times with a 26 ms wait when no data arrives — so a 'timeout' can mean 'our retry budget ran out', which is a very different thing from 'the device didn't answer'. I'd instrument exactly which of those happened. I'd also look hard at the locking: `Write` and `Read` share `m_ReadAPILock` so a command is atomic, but if any path ever calls them without the lock, two callers could read each other's responses and one would appear to time out.
>
> **3. Is it the communication layer / protocol framing?**
> Symptoms: always after a particular command; the response is the wrong length. Check: our read loop is `while (streamReadCount < buffer.Length && readRetry > 0)` — it's expecting a fixed byte count. If a command's response is shorter than `m_databuffersize`, that loop burns its whole retry budget and reports a timeout. That's a genuine class of bug worth naming: **a length mismatch looks exactly like a timeout.** The hex log of the partial read tells you which.
>
> **4. Is it the instrument firmware?**
> Symptoms: the device stops responding entirely until reset; correlates with a firmware version. Check: can I still reach it with a trivial command like the version read (`0xE1,0x02`)? If that fails too, the firmware or the FreeRTOS task is stuck. Given the firmware's structure, the candidates are a blocked high-priority task, a busy-wait starving the command task (`UserSetDelay` spins without yielding), or a missed interrupt. Also check whether it's port-specific — Port B commands go through a different encoding path (`ModifyAPItoSuppPortB` sets bit 7).
>
> **5. Is it timing / load?**
> Symptoms: only during heavy streaming, or only after hours. Check: the data socket and the command socket use different locks, so a long bulk read shouldn't block a command — but the *device* may be busy. We deliberately vary the read delay by buffer size (10 ms for ≤1 KB, up to 80 ms for 62 KB) which tells me somebody already found a load-dependent problem here.
>
> **The bisection that settles it:** run the same command in a tight loop with no test logic. If it never times out, the problem is load or sequencing, not the link. Then add the streamer. Then add the test. The step that reintroduces the failure is the answer."

**Follow-up: "How would you make this more diagnosable?"**
> Three things. A correlation ID per command so a request and its response can be matched in the log. Explicit distinction in the error between 'socket error', 'no data within timeout' and 'short read' — right now they all surface as a failed read. And a periodic heartbeat command so we know whether the device went away between tests rather than discovering it mid-test.

---

## Scenario 4 — "A regression test suddenly fails after a firmware update"

> "Three possibilities and they need different responses: the firmware has a new bug; the firmware fixed a bug and the test was encoding the old behaviour; or the firmware changed something legitimate that the test was over-specifying.
>
> **Step 1 — bisect the variable.** Re-flash the previous firmware and re-run. If it passes, the firmware is the variable — that's the single most valuable data point and it takes ten minutes. We can do this from C#: the firmware update is fully automated (`FirmwareUpdateManager`), so 'run the suite on version N and N-1' is a scriptable comparison.
>
> **Step 2 — diff the traces, not the verdicts.** Compare the passing and failing captures packet by packet. In practice the answer is visible: a message that used to arrive no longer does, a field changed value, or a response got slower.
>
> **Step 3 — classify.**
> - *New defect* → file it with both traces and the exact packet index.
> - *Intentional behaviour change* → check the firmware release notes and the spec. If the new behaviour is spec-legal and the test asserted one specific legal outcome, **the test is wrong** — it should accept the set of legal outcomes. This is exactly why our AMS model enumerates every legal continuation of a sequence rather than a golden path; tests that hard-code one path are the ones that break on every firmware release.
> - *Spec revision* → update the limits/model, and version the change so old reports remain interpretable.
>
> **Step 4 — check it's not us.** Did the software version change at the same time? We compare the instrument firmware version against the software's expected version at run start (`ModuleHelper.FWversionObt` vs `FWversion`) and block or warn on a mismatch, precisely so we never debug a compatibility problem as if it were a device bug.
>
> **Prevention:** run the full regression on every firmware release candidate before it ships, keep a reference trace per test so the diff is one command, and write tests against the *set* of legal behaviours rather than the observed one."

**Follow-up: "20 tests failed, not one. Where do you start?"**
> Look for the common factor before looking at any individual test. Twenty failures almost always have one cause — a bring-up procedure that no longer works, a capability read that returns different values, or the device not entering the right mode. Our common-check layer helps here: if the same `Common_Check` fails across all twenty, that's the root cause and the twenty are symptoms. I'd fix the one and re-run before reading any individual test.

---

## Scenario 5 — "Tests start failing after running continuously for several hours"

**This is a resource-exhaustion question.** They want to hear: memory, handles, thermal, state accumulation.

> "Time-dependent failure means something is accumulating. Five candidates:
>
> **1. Memory.** We stream large sample buffers and accumulate packet lists and result trees for the whole run. Check: watch private bytes and gen-2 heap across the run; take two snapshots and diff by type. Specific suspects in our code — a `new byte[1024]` per socket write, large sample arrays landing on the Large Object Heap (not compacted by default, so it fragments), and static state in `ModuleHelper` that persists across tests. Fix: pool the buffers (`ArrayPool<byte>`), clear per-run static state explicitly, and stream results to disk rather than holding the whole run in memory.
>
> **2. Handles / sockets / file handles.** We open three `TcpClient`s per controller and reinitialise them on error. If a reconnect path ever leaks, handle count climbs steadily. Check: handle count in Process Explorer over time. Our `Dispose()` closes the clients and streams, but the class doesn't implement `IDisposable` and there's no `GC.SuppressFinalize`, so cleanup depends on either an explicit call or finalization — worth auditing every error path to confirm `Dispose()` is actually called.
>
> **3. Threads.** Each test run creates a thread; the streamer creates another. If an abort leaves a thread alive, they accumulate. Check: thread count over time. Our abort path does spin until `!IsAlive`, but `Thread.Abort` can fail to terminate a thread stuck in unmanaged code.
>
> **4. Device state accumulation.** The instrument and the DUT also accumulate: FRAM writes, internal counters, temperature. We log temperature errors explicitly (`Get_Tester_Temperature_Error`), and there's an E-Load duty-cycle consideration — continuous high-current loading heats the hardware. If failures correlate with thermal rather than with iteration count, cooling and duty cycle are the answer, not software.
>
> **5. Disk.** Every test writes a trace plus HTML/CSV; every repeat writes a new run folder. Hours of running fills a disk, and the failure mode is confusing because the test fails at report generation, not at execution.
>
> **How I'd isolate it:** run the *same* test in a loop for hours, rather than the suite. If it degrades, it's per-iteration accumulation. If it only degrades with the full suite, it's cross-test state. And plot the metric — memory, handles, temperature, response time — against iteration number. A monotonic line names the cause without any guesswork.
>
> **Prevention:** a long-run soak as part of the release criteria, with resource counters recorded, so accumulation shows up as a trend before it shows up as a failure."

**Follow-up: "How would you prove it's a leak and not just GC not having run?"**
> Force a full blocking collection and re-measure — if the number doesn't come down, it's rooted. Then find the root: a snapshot diff will show which object is holding it, and in this codebase my first two guesses would be a static collection and a delegate/event subscription that's never removed.

---

## The universal closing line

Whatever the scenario, end with prevention:

> "And then I'd make the failure cheaper to diagnose next time — usually that means adding the one piece of evidence I wished I'd had: a correlation ID, a resource counter, a state marker in the capture, or a distinction between two error conditions that currently look identical."
