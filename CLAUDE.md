# CLAUDE.md — Interview Preparation Workspace

## What this workspace is

Preparation for a technical interview for a **C#/.NET firmware test automation** role.
This folder holds the prep material. It is **not** a codebase — the source of truth for every claim is two read-only repositories:

| Repo | Path | What it is |
|---|---|---|
| **Repo 1** (primary evidence) | `D:\GitHub\C2BrowserApp` | GRL USB-PD **C2 / C2-EPR (Gen2)** tester host software. C#/.NET, ~1,450 `.cs` files, 16 projects. The automation framework. |
| **Repo 2** | `D:\GitHub\fw_c2_gen2` | Instrument firmware. Xilinx Zynq-7000 + **FreeRTOS**, plus TI C2000 secondary bootloaders. |

Both are readable and **write-denied** via `.claude/settings.json`. Never modify them.

---

## Ground rules — these override any instinct to be helpful

1. **Never claim the user implemented something without repository evidence.** If you can't verify it, say *"No repository evidence found."*
2. **Always separate** hands-on experience / conceptual knowledge / no evidence.
3. **Cite file paths** when explaining anything technical — repo, project, file, class, method.
4. **Three things that must never be claimed in the first person** (verified absent, zero occurrences in either repo):
   - **BLE / Bluetooth**
   - **RF, RSSI, sub-GHz, modulation** — no RF work of any kind
   - **Azure DevOps, or any CI/CD pipeline definition** (no `azure-pipelines.yml`, no `.github/workflows`, no Jenkinsfile)
5. **Effort split when producing new material:** ~50% C#/.NET + automation, 20% firmware/functional/protocol testing, 10% embedded/RTOS, 10% CI/CD + Azure DevOps, 10% BLE/Serial/OTA + RF.
6. **Answers must sound like the user, not a textbook.** Structure: What → Why → How → Real project example → Result/Trade-off.
7. **Never invent metrics, achievements, team sizes or percentages.** Approximate and honest beats precise and fabricated.

---

## Folder layout

```
interview-prep/
├─ CLAUDE.md                  ← this file
├─ 00-INDEX.md                ← navigation + 12 must-know project facts
├─ 01 … 14 *.md               ← the prep pack (see index)
├─ .claude/
│   ├─ settings.json          ← read access to both source repos, write denied
│   └─ commands/mock.md       ← /mock — resume the mock interview
├─ convo-clc/                 ← exported conversation logs, one file per session
├─ context/                   ← (empty — reserved by the user)
└─ prep-files/                ← (empty — reserved by the user)
```

> **Note:** the numbered prep files are currently at the root, not inside `prep-files/`.
> `00-INDEX.md` links to them with root-relative names. If they are moved, the links in `00-INDEX.md` and the cross-links inside files 01–14 must be repaired in the same change.

### The prep pack

| File | Covers |
|---|---|
| `00-INDEX.md` | Navigation, 12 key project facts, the three never-claims |
| `01-jd-analysis.md` | JD requirement → evidence → experience level → gap |
| `02-repo1-architecture.md` | **Largest and most important.** Solution/layers, how a test is created/run/maintained, C# concepts mapped to real files, design patterns, test-case anatomy |
| `03-repo2-firmware.md` | Zynq + FreeRTOS architecture, tasks/ISRs/timers, protocol state machines, bootloaders |
| `04-two-repos-connected.md` | Full SW↔FW↔HW path + 3 end-to-end flows |
| `05-firmware-functional-and-protocol-testing.md` | Functional testing, the 7 axes of protocol testing, state-machine testing |
| `06-csharp-dotnet-qa.md` | The big Q&A bank with follow-up chains |
| `07-manual-to-automated.md` | The 18-step conversion method + worked examples |
| `08-debugging-scenarios.md` | 5 failure scenarios with structured answers |
| `09-ble-serial-ota-rf-rtos.md` | Serial, BLE, OTA/rollback, battery, RF, RTOS |
| `10-cicd-azure-devops.md` | Hardware-in-the-loop CI/CD, Azure DevOps Test Plans |
| `11-system-design.md` | 5 system-design questions |
| `12-project-pitch-and-deep-dive.md` | 30s/1min/2min pitch, day-to-day, repo-specific grilling |
| `13-honesty-map-and-priorities.md` | Experience level per skill; MUST/SHOULD/BASIC prep list |
| `14-cheat-sheet.md` | Final morning-of revision sheet |

---

## Verified facts — don't re-derive these

**Repo 1**
- ASP.NET Core Web API hosted on **.NET Framework 4.7.2**, Kestrel + libuv, port **5001** (5002 = offline instance), Swagger, **React** front end. Solution: `C2WebApp\C2WebApp\C2BrowserApp.sln`, 16 projects.
- A test is a **class**: `abstract class TestCase` → `PreConfigure()` / `Configure()` / `Validate()` (Template Method) — `USBPDModule\SampleModule\TestCase.cs`.
- **Test discovery is by reflection**: the `ComplianceTestEnum` member name *is* the class name →
  `assembly.GetTypes().First(t => t.Name.ToUpper() == testObj.TestCaseEnum.ToString().ToUpper())` then `Activator.CreateInstance` — `USB_2_0_Engine.cs:1143`, `FunctionalTestFactory.cs:102`.
- Execution engine: `USB_PD_Engine.cs` — dedicated `Thread`, grouped by MOI, repeat/rerun conditions, per-run output folders, `Thread.Abort`-based force-stop.
- Transport: `GrlEthernetLink_C2.cs` — 3 × `TcpClient` on port 5002 (command-write, command-read, data-stream), two lock objects, retry + reconnect, hex logging of every frame. Also USB (libusbK) and serial paths, with **no common interface** — that missing `ITesterLink` is the headline refactor.
- **The key design idea:** `IsTesterCommEn = false` makes the device API *capture* a command buffer instead of sending it, so a test **composes** a stimulus sequence which is uploaded and replayed by firmware with hardware timing. The host never does real-time work.
- Validation models **what the spec allows**, not a golden path — `GrlPdLib\AMS validation\PDStateMachine.cs` enumerates every legal continuation of an Atomic Message Sequence.
- Results: `PDTestResult` composite tree, priority merge FAIL > INCOMPLETE > WARNING > NA > PASS. HTML/CSV/PDF/XML + a `.grltrace` capture per test.
- Hardware-free regression: `XUnit_TestFramework` replays recorded JSON captures through the production validator (xUnit + Moq), including deliberately-bad fixtures.
- In-system firmware update automated from C#: `FirmwareUpdateManager.cs`.
- ~700 test cases across MOIs: PD2/PD3 compliance, PD-merged, Deterministic (Type-C), Functional (63 × `TD.4.x`), DP Alt Mode, QC Legacy/QC3+/QC4, Thunderbolt, BC 1.2, IEC, Source Power.

**Repo 2**
- **FreeRTOS confirmed**: `RTOSWrapper.h` includes `FreeRTOS.h`/`task.h`/`timers.h`/`semphr.h`/`event_groups.h`/`queue.h`; `UsbPDMain.c` has `CreateTasks` / `CreateSemaphores` / `CreateSoftwareTimers` and `vTaskStartScheduler()`.
- 2 live tasks: `"GpioInterrupt"` (prio idle+7) and `"CommandThread"` (idle+6). 10 mutexes, 10 software timers, 3 event groups declared. ISR → `xTaskNotifyFromISR` → `xTaskNotifyWait(500 ms)` — deferred interrupt handling via task notifications, not semaphores.
- Protocol state machines in `UsbPDProtocolLayer.h`: `PRL_TX_*` (10), `PRL_RX_*` (5), `PRL_HR_*` (7), BIST TX/RX; `USBPDProtoState { Pre, Current, Next }`.
- TI C2000 SSBL (`ccs/SSBL/{PPS,ELOAD}`): I2C slave 0x0E/0x1E, `SET_BOOT`/`SET_PGM`/`SW_RESET`/`GET_MODE`/`GET_FW_VERSION`, F021 flash API. `ccs/readme.txt` records V2.0 adding **SSBL flash write protection** and V3.0 adding a **success/error status byte with retry**.
- `docs/` and `sources/petalinux/` are empty placeholders. No firmware unit tests, no CI.

**Authorship signal** (for "what did I actually own"): comments signed `Bala`/`Balaji` appear 1,178 times, heaviest in `DeterministicTestEngine.cs` (305), `PDmessageDecodeEngine.cs` (83), `PD_Controller.cs` (42), `FunctionalHelper.cs` (39), `ModuleHelper.cs` (27).

---

## Current state

| | |
|---|---|
| Prep phases 1–25 | **Complete** — files `00`–`14` |
| Phase 26, mock interview | **In progress.** Question 1 has been asked; the first answer has not been given yet. |
| Question progression | Project → C# → .NET → Test Automation → Framework Architecture → Firmware Testing → Protocol Testing → Debugging → State Machines → Serial/BLE → OTA → RTOS → RF → CI/CD → Azure DevOps → System Design |

**Question 1 (outstanding):** *"Tell me about your current project and your role in it."*

---

## How to run the mock interview

Use `/mock` — or follow this loop directly:

1. Ask **one** question. Never a list.
2. Wait for the answer.
3. Then, in this order:
   - **Score** it out of 10 with a one-line reason.
   - **What was missing** — technically, not stylistically.
   - **What was inaccurate** — check it against the repos; correct it with a file path.
   - **The senior-level version** of the answer, in the user's voice, using their real project.
   - **The follow-up an interviewer would actually ask next**, and ask it.
4. Increase difficulty gradually. Stay on the progression above unless the user redirects.
5. If the user claims BLE, RF or Azure DevOps experience in an answer, **stop and correct it** — that is a credibility risk, not a style note.

---

## Backlog

`prep-tips.md` at the root holds the user's own rough notes for where to take this next. As of
2026-09-23 it is a draft, not yet actioned, and asks for:

1. **Per-concept questioning** across every concept / technology / tool — with analogies, sample
   apps, and open-source repo references; plus enterprise-grade coding-standard guidance.
2. **A mindmap in XLSX** linking high-level topics down to the low-level sections inside the prep
   `.md` files.
3. **Re-reading the prep files from multiple points of view** — learning, building, interview prep,
   and others — not just interview prep.

Treat it as a backlog, not as standing instructions. Ask before acting on it, and if the user
asks for it to be "fine tuned and executed", clarify which of the three they want first — they
are three separate pieces of work.

## Conversation logging

Each working session gets one Markdown file in `convo-clc/`, named
`YYYY-MM-DD-session-NN-<short-topic>.md`.

Format: user messages verbatim, assistant chat messages verbatim, tool activity summarised in a table, and a **State at end of session** block recording what phase the mock interview reached. See `convo-clc/README.md`.

When asked to export a conversation, append a new session file — never overwrite an existing one.
