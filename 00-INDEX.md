# Interview Preparation Pack — C#/.NET Firmware Test Automation

Built from a direct read of:

- **Repo 1 (host software / automation):** `D:\GitHub\C2BrowserApp` — GRL USB-PD **C2 / C2-EPR (Gen2)** tester host application, C#/.NET.
- **Repo 2 (firmware):** `D:\GitHub\fw_c2_gen2` — GRL USB-PD C2 Gen2 controller firmware (Xilinx Zynq-7000 + FreeRTOS, plus TI C2000 secondary bootloaders).

Everything in this pack that claims "you did X" is backed by a file path you can open. Where there is **no evidence**, it says so explicitly.

---

## Files

| File | Covers | Interview phases |
|---|---|---|
| [01-jd-analysis.md](01-jd-analysis.md) | JD requirement → evidence → gap table | Phase 1 |
| [02-repo1-architecture.md](02-repo1-architecture.md) | Solution, layers, framework design, how a test is created / run / maintained, C# concepts with real files, design patterns, test-case anatomy | Phase 2 (A–E) |
| [03-repo2-firmware.md](03-repo2-firmware.md) | Zynq + FreeRTOS firmware architecture, tasks, ISRs, timers, protocol state machines, bootloaders | Phase 3, 7 |
| [04-two-repos-connected.md](04-two-repos-connected.md) | The full SW↔FW↔HW path, 3 real end-to-end flows | Phase 4 |
| [05-firmware-functional-and-protocol-testing.md](05-firmware-functional-and-protocol-testing.md) | Firmware functional testing, protocol testing, state-machine testing | Phase 5, 6, 7 |
| [06-csharp-dotnet-qa.md](06-csharp-dotnet-qa.md) | The big C#/.NET + framework-design Q&A bank with follow-ups | Phase 8, 16, 17, 22 |
| [07-manual-to-automated.md](07-manual-to-automated.md) | The 18-step conversion method + a worked example from your repo | Phase 9 |
| [08-debugging-scenarios.md](08-debugging-scenarios.md) | 5 realistic failure scenarios, structured answers | Phase 10 |
| [09-ble-serial-ota-rf-rtos.md](09-ble-serial-ota-rf-rtos.md) | Serial, BLE, OTA/rollback, battery, RF fundamentals, RTOS | Phase 11, 12, 13 |
| [10-cicd-azure-devops.md](10-cicd-azure-devops.md) | CI/CD for hardware-in-the-loop, Azure DevOps Test Plans | Phase 14, 15 |
| [11-system-design.md](11-system-design.md) | 5 system-design questions with answers | Phase 18 |
| [12-project-pitch-and-deep-dive.md](12-project-pitch-and-deep-dive.md) | 30s/1min/2min project intro, day-to-day, repo-specific grilling questions | Phase 19, 20, 21 |
| [13-honesty-map-and-priorities.md](13-honesty-map-and-priorities.md) | Experience level per skill, MUST/SHOULD/BASIC prep list | Phase 23, 24 |
| [14-cheat-sheet.md](14-cheat-sheet.md) | One-page-per-topic final revision | Phase 25 |

**Phase 26 (mock interview)** runs live in the chat, one question at a time.

---

## The 12 facts to burn into memory

1. Product: **GRL USB-PD C2 / C2-EPR (Gen2)** — a USB Type-C / USB Power Delivery **compliance and functional tester**. It emulates a Source, Sink, DRP or Cable and validates a DUT (called **UUT**) against the USB-IF CTS.
2. Host app is **C#/.NET Framework 4.7.2**, an **ASP.NET Core Web API** hosted in Kestrel/libuv on `localhost:5001`, with a **React** front end and **Swagger**. Entry: `C2WebApp\C2WebApp\C2WebApp\Program.cs`, `Startup.cs`.
3. **16 projects** in `C2WebApp\C2WebApp\C2BrowserApp.sln`, layered: framework → protocol library → device library → test content → API/UI.
4. Tests are **classes**, not scripts. `abstract class TestCase` defines `PreConfigure() → Configure() → Validate()` (Template Method). Base: `USBPDModule\SampleModule\TestCase.cs`.
5. **Test discovery is by reflection**: the test-case enum name *is* the class name.
   `assembly.GetTypes().First(t => t.Name.ToUpper() == testObj.TestCaseEnum.ToString().ToUpper())` then `Activator.CreateInstance(...)` — `USB_2_0_Engine.cs:1143`, `FunctionalTestFactory.cs:102`.
6. Execution engine: `USB_PD_Engine.cs` — runs selected tests on a dedicated `Thread`, grouped by MOI (Mode of Interest), with rerun/repeat, per-run output folders, force-stop and report generation.
7. Device link is **TCP/IP over Ethernet** to the tester: `GrlEthernetLink_C2.cs` (3 `TcpClient`s: command-write, command-read, data-stream), plus **USB (libusbK)** and **serial (`System.IO.Ports`)** paths.
8. Results model: `PDTestResult` — a **tree** of results (`TestList`) with a priority merge rule (FAIL wins). Reports: HTML, CSV, PDF, XML, plus `.grltrace` captures.
9. Firmware runs **FreeRTOS on a Zynq-7000 Cortex-A9** (`RTOSWrapper.h` includes `FreeRTOS.h/task.h/queue.h/semphr.h/event_groups.h/timers.h`; `UsbPDMain.c` creates tasks, mutexes, software timers and calls `vTaskStartScheduler()`).
10. Firmware has **explicit protocol state machines**: `PRL_TX_*`, `PRL_RX_*`, `PRL_HR_*`, BIST TX/RX in `UsbPDProtocolLayer.h`, with `USBPDProtoState { Pre, Current, Next }`.
11. Firmware update is real and automated from C#: `FirmwareUpdateManager.cs` — Zynq images (`BOOT.BIN`, `image.ub`, `start.sh`) copied to an SD card the device exposes over USB; TI C2000 E-Load/PPS MCUs flashed over an **I2C secondary bootloader (SSBL)** with mode switch, erase, 256-byte chunked write, version read-back and reboot.
12. There is a **regression/offline test layer**: `XUnit_TestFramework` replays captured PD traffic stored as JSON through the real validation engine and asserts with xUnit — no hardware needed.

---

## Three things you must NOT claim

- **BLE / Bluetooth** — zero occurrences in either repo.
- **Sub-GHz radio / RSSI / modulation / RF measurement** — zero occurrences.
- **Azure DevOps / any CI pipeline definition** — no `azure-pipelines.yml`, no `.github/workflows`, no Jenkinsfile. The only `.github` content is a PR template.

Say so plainly and pivot to what you *do* have (see [13-honesty-map-and-priorities.md](13-honesty-map-and-priorities.md)).
