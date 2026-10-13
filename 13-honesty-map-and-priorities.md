# Phase 23 + 24 — Honest Experience Map and Preparation Priority

---

# Part 1 — Experience map

Levels: **Strong hands-on** · **Good hands-on** · **Some exposure** · **Conceptual only** · **No evidence**

## C# / .NET

| Skill | Evidence in repo | Level | What you can say | What to learn |
|---|---|---|---|---|
| C# language (OOP, generics, collections, LINQ, delegates, attributes) | ~1,450 `.cs` files across 16 projects | **Strong hands-on** | "Daily, for years, in a large production codebase" | Modern C# 8–12: pattern matching, records, `init`, nullable reference types, `Span<T>` |
| Abstract classes / inheritance / polymorphism | `TestCase` → `USBPD_2_TestCase` → concrete tests | **Strong hands-on** | Design rationale, not just syntax | — |
| Interfaces | `IModule`, `ITestImpl`, `IWriteFile`, `IChannelTemperatureChecker` | **Good hands-on** | Plus the honest gap: no interface at the transport boundary | Interface segregation in practice |
| Reflection + `Activator.CreateInstance` | Test discovery, plugin loading | **Strong hands-on** | This is a genuine differentiator | Caching, source generators as an alternative |
| Attributes | `[Description]`, `[Serializable]`, `[Http*]`, `[Fact]` | **Good hands-on** | Read back at runtime via `EnumHelper` | Writing custom attributes |
| Threading (`Thread`, `lock`, `volatile`) | Runner thread, streamer thread, socket locks | **Good hands-on** | Can explain why a dedicated thread, why two locks | `Interlocked`, `ReaderWriterLockSlim`, concurrent collections |
| `async`/`await`/`Task` | Sparse: `Task.Run(...).Wait()`, one `await` chain | **Some exposure** | Be honest: "the codebase is mostly synchronous blocking I/O with dedicated threads; I understand async and can explain why `Task.Run().Wait()` is wrong" | **Study properly — this will be asked.** Deadlocks, `ConfigureAwait`, `async void`, exception flow |
| `CancellationToken` | **None** — `Thread.Abort` + bool flags | **Conceptual only** | Turn it into your best "what I'd improve" answer | `CancellationTokenSource`, linked tokens, `Register`, cancelling blocking I/O |
| `IDisposable` / `using` | `using` used correctly in several places; `GrlEthernetLink_C2` has `Dispose()` + a finalizer but no `IDisposable` | **Good hands-on** | Volunteer the bug — it's a great senior signal | Full dispose pattern, `IAsyncDisposable` |
| Exception handling | Everywhere, including over-broad catches | **Strong hands-on** | Know where broad catching is right (test boundary) and wrong | Custom exception hierarchies, `throw` vs `throw ex` |
| Serialization | Newtonsoft.Json, `DataContractSerializer`, `BinaryFormatter` | **Good hands-on** | `BinaryFormatter` is obsolete — say so | `System.Text.Json` |
| Extension methods / generics | `DeepCopy<T>`, `Extension.cs`, `LibExtension.cs` | **Good hands-on** | Know why `DeepCopy<T>` needs a constraint | Generic constraints |
| File / stream / path handling | Run folders, capture files, `DriveInfo`, recursive copy | **Strong hands-on** | | |
| Interop / P-Invoke | `visa32/64.cs`, `libusbK.cs` consumed | **Some exposure** | "Consumed, not authored" | `DllImport`, marshalling |
| WMI | `Win32_PnPEntity` VID/PID COM port discovery | **Good hands-on** | Concrete and memorable — use it | |
| IPC (named pipes) | `UICommClient`, `TBTFactory` JSON over a pipe | **Good hands-on** | | |
| Dependency injection | Only ASP.NET Core's container for MVC/CORS/Swagger; domain uses manual singletons | **Some exposure** | Explain the gap and the fix | Lifetimes, captive dependencies, constructor injection discipline |
| Design patterns | Template Method, Factory, Singleton, Facade, Adapter, Strategy, Composite, Observer, Builder, State | **Strong hands-on** | Name where you follow *and* break them | Repository/UoW/Mediator (not present — don't claim) |

## .NET platform

| Skill | Evidence | Level | Notes |
|---|---|---|---|
| ASP.NET Core Web API | `C2BrowserApp` — controllers, routing, `[FromBody]`, CORS, static files, Swagger | **Good hands-on** | It's ASP.NET Core 2.1 hosted on .NET Framework 4.7.2 — unusual, explain it |
| Middleware | `ApiExecutionTimeMiddleware` written but commented out | **Some exposure** | Good concrete answer for "explain the pipeline" |
| Configuration | `PropertyBag`, `appsettings.json`, typed config classes | **Good hands-on** | Hand-rolled rather than `IOptions<T>` |
| Logging | Custom `WriteToDebugLogger` family | **Good hands-on** | Say you'd move to `ILogger` with a scope per test |
| Hosted / background services | Not used — raw `Thread` | **Conceptual only** | `BackgroundService` is the modern home for the runner |
| .NET Framework vs .NET 5+ | Targets `net472`; one `net5.0` test project | **Good hands-on (Framework)** / **Some exposure (Core)** | Have the migration blocker list ready: `Thread.Abort`, `BinaryFormatter`, WMI, WinForms in library projects |
| Kestrel / hosting | `WebHost.CreateDefaultBuilder`, `UseLibuv`, `UseUrls`, `ConfigureKestrel` | **Some exposure** | |

## Test automation

| Skill | Evidence | Level | Notes |
|---|---|---|---|
| Building/maintaining a framework | The whole `AppFrmWrk` + engine + base classes | **Strong hands-on** | Your headline |
| Test discovery / execution engine | Reflection dispatch, MOI factories, `USB_PD_Engine` | **Strong hands-on** | |
| Converting spec test cases to automation | ~700 tests mapping to CTS clauses | **Strong hands-on** | **The exact thing the JD asks for** |
| Reporting | HTML/CSV/PDF/XML, result tree, evidence links | **Strong hands-on** | |
| Debugging automation failures | Per-packet evidence, hex logs, offline replay | **Strong hands-on** | |
| Regression execution | Rerun count, repeat conditions, 5-port sweep | **Good hands-on** | Manual trigger, not pipeline |
| Unit testing (xUnit / MSTest / Moq) | `XUnit_TestFramework` with JSON fixtures; MSTest projects | **Good hands-on** | Small suite; be proportionate |
| UI automation (Selenium) | `Selenium\C2WebAppTest.py` — Python, 5-port loop | **Some exposure** | Python not C#; say so |
| Test data management | VIF files, PDM JSON matrices, recorded captures | **Good hands-on** | |
| Mocking | Moq referenced; used lightly | **Some exposure** | Practise a Moq example — likely to be asked |

## Firmware / embedded

| Skill | Evidence | Level | Notes |
|---|---|---|---|
| Firmware functional testing | 63 `TD_4_x` Type-C functional tests + the whole compliance suite | **Strong hands-on** | |
| Protocol testing | 7 axes, all present | **Strong hands-on** | Your strongest technical story |
| State-machine testing | `PDStateMachine` AMS model; firmware PRL state machines | **Strong hands-on (host side)** | |
| Reading embedded C | Repo 2 — you can navigate and explain it | **Good hands-on (reading)** | Never claim you wrote it |
| Writing embedded C | No evidence | **No evidence** | "I read firmware; I don't write it" |
| Embedded architecture (Zynq PS/PL, AXI, DMA, boot chain) | Repo 2 structure | **Good (conceptual + reading)** | You can draw the block diagram |
| Interrupts / ISR design | `xTaskNotifyFromISR`, deferred handling | **Conceptual + reading** | |
| FreeRTOS | Confirmed in repo 2; tasks, mutexes, timers, notifications | **Good reading / Conceptual** | "The firmware I test runs FreeRTOS; I read it to debug. I haven't authored task code." |
| In-system firmware update | `FirmwareUpdateManager` + SSBL, end to end | **Good hands-on** | Say "in-system update", not "OTA" |
| Debugging embedded (JTAG, trace tools) | No evidence | **No evidence** | Don't claim |

## Communication interfaces

| Skill | Evidence | Level |
|---|---|---|
| TCP/IP sockets | `GrlEthernetLink_C2`, `TCPIPServer` | **Strong hands-on** |
| USB (host side, libusbK) | `GrlUsbLink_C2`, `UsbStream` | **Some exposure** |
| USB-PD / USB Type-C | Everything | **Strong hands-on** |
| Serial / UART | Switch box, E-Load flashing, WMI port discovery | **Good hands-on** |
| I2C | Host side tunnelled; firmware side read | **Some exposure** |
| SPI | Firmware side read | **Conceptual** |
| DisplayPort Alt Mode / AUX | `DPAuxDecoder`, DP test suite | **Some exposure** |
| Thunderbolt / USB4 | `TBTClassFactory`, USB4 tests | **Some exposure** |
| BC 1.2 / Quick Charge | Dedicated MOIs | **Some exposure** |
| HTTP/REST/JSON | ASP.NET Core API | **Good hands-on** |
| **BLE / Bluetooth** | **None** | **No evidence** |
| **CAN, MQTT, Zigbee, LoRa** | **None** | **No evidence** |

## Radio / RF

| Skill | Evidence | Level |
|---|---|---|
| RF fundamentals (RSSI, sensitivity, BW, modulation) | **None** | **Conceptual only — learn the definitions** |
| Sub-GHz radio | **None** | **No evidence** |
| Antennas, link budget, spectrum analysis | **None** | **No evidence** |
| **Adjacent strength you DO have** | PHY-layer testing: eye diagrams, bit-rate drift, receiver interference rejection, receiver input impedance, BIST PRBS (`TEST_PD_PHY_ALL_*`, `BISTEyeMask.cs`, `EyeDiagramMaker.cs`, `SignalMask.cs`) | **Good hands-on** — *use this bridge* |

## CI/CD and tooling

| Skill | Evidence | Level |
|---|---|---|
| Git (branching, PRs, code review) | Issue-ID branches, `CODEOWNERS`, PR template | **Good hands-on** |
| Issue tracking | Issue IDs throughout comments and branches | **Good hands-on** |
| CI/CD pipelines | **No pipeline file anywhere** | **Conceptual only** |
| Azure DevOps | **None** | **No evidence** |
| Regression automation | In-app rerun/repeat, 5-port sweep, offline replay, Selenium | **Good hands-on (not pipeline-driven)** |
| Docker / containers | **None** | **No evidence** |

---

# Part 2 — Preparation priority

## MUST PREPARE — do these first, they are most of the interview

1. **The 12-step execution flow**, spoken in 90 seconds without notes. ([02](02-repo1-architecture.md) §B.2)
2. **Reflection-based test discovery** — how, why, costs, alternatives. ([02](02-repo1-architecture.md) §C, [06](06-csharp-dotnet-qa.md) §B7)
3. **The `PreConfigure / Configure / Validate` contract** and why a test contains only its differences.
4. **The manual → automated conversion method**, plus one worked example you can write on a whiteboard. ([07](07-manual-to-automated.md))
5. **`Thread.Abort` vs `CancellationToken`** — your best "what I'd improve". ([06](06-csharp-dotnet-qa.md) §C3)
6. **async/await fundamentals**, including why `Task.Run(...).Wait()` is wrong. **This is your weakest strong-area — study it properly.**
7. **Interface vs abstract class**, with your code as the example.
8. **The `ITesterLink` refactor** — the single change that unlocks mockability, CI and parallelism.
9. **Firmware functional testing** — the definition table and the five-paragraph answer. ([05](05-firmware-functional-and-protocol-testing.md))
10. **Protocol testing — the seven axes**, with a test name for each. ([05](05-firmware-functional-and-protocol-testing.md) §2.2)
11. **The AMS legal-sequence model** — modelling what the spec allows instead of a golden path. This is your most distinctive idea.
12. **Result model**: tree, priority merge, and why NA ≠ INCOMPLETE ≠ FAIL.
13. **Debugging scenarios 1 and 3** — passes-manually-fails-automated, and random timeouts. ([08](08-debugging-scenarios.md))
14. **Your project pitch** at 30 s / 1 min / 2 min. ([12](12-project-pitch-and-deep-dive.md))
15. **How you know whether it's the DUT, the instrument or your automation.**

## SHOULD PREPARE

16. FreeRTOS concepts — **mutex vs binary semaphore**, priority inversion, ISR→task signalling, deferred interrupt handling. ([09](09-ble-serial-ota-rf-rtos.md) §5)
17. Firmware architecture of repo 2 — draw the block diagram: PC → Ethernet → ARM/FreeRTOS → AXI → FPGA PHY → CC line.
18. The PRL state machines and how you'd test invalid transitions and timeouts. ([03](03-repo2-firmware.md) §5, [05](05-firmware-functional-and-protocol-testing.md) §3)
19. Serial communication and COM port discovery by VID/PID.
20. In-system firmware update flow + the OTA test matrix, including power loss and corrupted image. ([09](09-ble-serial-ota-rf-rtos.md) §3)
21. CI/CD design for hardware-in-the-loop — the three tiers, and the four preconditions. ([10](10-cicd-azure-devops.md))
22. Azure DevOps vocabulary — Plan / Suite / Case / Point / Configuration / Run, and associating an automated test. ([10](10-cicd-azure-devops.md) §2)
23. System design Q1 and Q5 (100 devices; multiple device types and firmware versions). ([11](11-system-design.md))
24. SOLID with your code as evidence, including where you break it.
25. Design patterns present — and the ones that aren't.
26. `IDisposable` and the `GrlEthernetLink_C2` bug.
27. Debugging scenarios 2, 4, 5.
28. Moq — write a small mock example by hand.

## BASIC PREPARATION — 30–45 minutes total, no more

29. RF definitions and the five "vs" comparisons. ([09](09-ble-serial-ota-rf-rtos.md) §4)
30. BLE architecture: GAP/GATT, services/characteristics, notify vs indicate, MTU, connection interval.
31. Sub-GHz vs 2.4 GHz trade-off.
32. The scripted honest RF answer — rehearse it once out loud so it sounds calm, not apologetic.
33. Battery: average current determines life; low-battery behaviour is a functional test.

---

# Part 3 — Phrases to use, and phrases to avoid

## Use these

- "I read the firmware to debug, I don't author it."
- "There's no repository evidence of that — I haven't done it."
- "Here's what I'd change about my own code, and why it hasn't been changed yet."
- "The unit of parallelism is the station, because a protocol conversation is exclusive and sequential."
- "Not-applicable is a reported outcome with a reason, never a pass and never a failure."
- "A failure report should be a complete bug report without anyone re-running the test."
- "I model what the spec allows, not one golden path."
- "The host never does real-time work — it compiles the stimulus and the firmware replays it."

## Avoid these

- Any claim about **BLE, sub-GHz, RSSI, modulation, or Azure DevOps** in the first person.
- "We have a full CI/CD pipeline." (You don't.)
- "I wrote the firmware." (You didn't.)
- "Our framework is really well designed." (Say what's good *and* what isn't — much stronger.)
- Precise invented numbers (defect counts, percentage improvements, team sizes). Approximate and honest beats precise and invented.
- Bluffing on async. If you don't know, say "I'd have to check" — that's a better answer than a wrong one.

## The single most important framing

> "The radio domain is new to me. The job — take a spec test case, turn it into a maintainable C# test that drives firmware over a link, validate timed protocol behaviour, and make the failure diagnosable — is exactly what I do every day."
