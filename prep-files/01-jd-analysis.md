# Phase 1 — JD Analysis and Evidence Map

## How to read this

- **Strong** = large body of code you clearly worked in, with signed comments.
- **Good** = present and substantial, you can speak to it in detail.
- **Partial** = present but adjacent to what the JD literally asks.
- **None** = no repository evidence. Prepare conceptually and say so.

---

## The requirement table

| # | JD Requirement | Prep Priority | What the interviewer is really testing | Evidence in your repos | Level | What to prepare |
|---|---|---|---|---|---|---|
| 8 | **Strong programming skills in C#/.NET** | **1** | Can you write and reason about production C#? OOP, interfaces, generics, async, threads, exceptions, IDisposable, GC. | ~1,450 `.cs` files. Reflection-based dispatch, abstract/virtual hierarchies, delegates+events, `Task`/`Thread`, `lock`, `Dictionary`/`List`/LINQ, `DataContractSerializer`, `BinaryFormatter`, named pipes, `TcpClient`. | **Strong** | Be able to *criticise* your own code: `Thread.Abort()`, `BinaryFormatter`, catch-all `catch{}`, finalizer without `IDisposable`, static mutable state. See [06](06-csharp-dotnet-qa.md). |
| 9 | **Developing and maintaining automation frameworks** | **1** | Do you understand framework design, not just test writing? | `AppFrmWrk` (host, property bag, module loader, results tree, report engine) + `USB_PD_Engine` execution engine + `TestCase` base classes + per-domain factories + `GrlC2Controller` scripting API. | **Strong** | Draw the layer diagram from memory. Know where you'd add a new test in 5 minutes. |
| 7 | **Firmware Functional Testing** | **2** | Do you test *behaviour of firmware over a real interface*, not unit tests? | `FunctionalTestCases\` — 63 `TD_4_x_*` USB Type-C functional tests driving attach/detach, Rp/Rd, DRP toggling, role swaps, VBUS/VCONN, dead-battery. | **Strong** | See [05](05-firmware-functional-and-protocol-testing.md). Use TD.4.2.1 as your worked example. |
| 2 | **Testing communication protocols and firmware state machines** | **2** | Sequence, timing, negative testing, malformed frames. | USB-PD protocol tests (`Protocol tests\`, `Power tests\`, `EPR Test cases\`), AMS validation engine (`PDStateMachine.cs`, `AMSValidationEngine.cs`), timer checks (`tSenderResponse`, `tSinkWaitCap`, `tPSSourceOn`) in `APIValidationHelper.cs`, bit-manipulation/CRC-corruption tests in `C2APIScript\PD Robutness tests`. | **Strong** | This is your best differentiator. Prepare the "expected AMS vs obtained AMS" explanation. |
| 1 | **Embedded systems architecture and debugging** | **3** | Can you talk about the stack below your C#? | Repo 2: Zynq-7000 PS+PL, FSBL → application, AXI-Lite/AXI4-Burst register access, DMA streamer, I2C/SPI/GPIO/UART drivers, interrupt handling. | **Good** | Know the block diagram: PC → Ethernet → ARM app → AXI regs → FPGA PHY → CC line. |
| 6a | **Serial communication** | **3** | UART/RS-232 basics and automation over COM ports. | `System.IO.Ports.SerialPort` in `USB_PD_Engine.ConnectToSeraiPort()` (5-port switch box), `FirmwareUpdateManager` (E-Load bootloader over COM via `bossac`), `EquipmentLib\SerialPort.cs`, COM port discovery by VID/PID with WMI. Firmware: UART ISR → `UartCmdHandler()`. | **Good** | Baud/parity/stop bits, framing errors, flow control, timeouts, port enumeration. |
| 6b | **OTA firmware upgrade / rollback** | **3** | Do you understand image transfer, verify, activate, recover? | `FirmwareUpdateManager.cs` (Zynq SD image copy; TI SSBL: mode switch → erase → 256-byte chunk write → version read-back → set boot mode → reset). Firmware: `ccs\SSBL\{PPS,ELOAD}\main.c` with `SET_BOOT / SET_PGM / SW_RESET / GET_MODE / GET_FW_VERSION`, flash write-protection of the SSBL region, success/error status byte with retry (`ccs\readme.txt`). | **Good** (in-system update; not wireless OTA) | Say **"in-system / in-field firmware update"**, not OTA. Then explain how the same test matrix applies to OTA. See [09](09-ble-serial-ota-rf-rtos.md). |
| 3 | **RTOS familiarity (FreeRTOS)** | **4** | Tasks, priorities, ISR-to-task signalling, mutex vs semaphore. | **FreeRTOS confirmed**: `RTOSWrapper.h/.c`, `UsbPDMain.c` — `xTaskCreate` (GpioInterrupt prio idle+7, CommandThread idle+6, PdTimerTask), `xSemaphoreCreateMutex` ×10, `xTimerCreate` ×10, `EventGroupHandle_t`, `xTaskNotifyFromISR` → `xTaskNotifyWait`, `vTaskStartScheduler()`. | **Good (reading) / Partial (writing)** | Honest line: *"The firmware I test runs FreeRTOS and I read it to debug; I haven't authored FreeRTOS task code."* |
| 11 | **CI/CD pipelines and automated regression** | **4** | Can you get hardware tests running unattended? | Regression *exists* but is manual/in-app: rerun counts, repeat-on-fail conditions, 5-port sequential loop, Selenium UI regression (`Selenium\C2WebAppTest.py`), offline xUnit regression. **No pipeline definition file anywhere.** | **Partial** | Prepare the design, not a claim. See [10](10-cicd-azure-devops.md). |
| 12 | **Azure DevOps test management** | **4** | Test Plans/Suites/Cases, linking automation to test cases. | **None.** Zero hits for azure/vsts/TFS in source. | **None** | Learn the vocabulary and the automated-test association flow. Say you haven't used it. |
| 4 | **Radio communication systems** | **5** | Basic literacy only (panel said training provided). | None. | **None** | 20 minutes of fundamentals. |
| 5 | **RF fundamentals (RSSI, sensitivity, BW, modulation)** | **5** | Same. | None. | **None** | Memorise the definitions + the four "vs" comparisons. |
| 10 | **Sub-GHz radios / BLE / IoT devices** | **5** | Same. | None. **No BLE anywhere.** | **None** | BLE concept sheet in [09](09-ble-serial-ota-rf-rtos.md). |
| 6c | **Battery requirements** | **5** | Battery-aware behaviour. | Partial and *protocol-level only*: USB-PD `Get_Battery_Status`, `Get_Battery_Cap`, `Battery_Capabilities`, `TD_4_11_2_Sink_Dead_Battery_Test`. No battery-life/current-drain testing. | **Partial** | Be precise: "battery *messages* and dead-battery behaviour, not battery life characterisation." |

---

## Likely questions per requirement, and where the answer lives

| JD line | Questions you should expect | Answer prepared in |
|---|---|---|
| C#/.NET | "Walk me through how your framework finds and runs a test." / "Why reflection?" / "Interface vs abstract class in your code?" / "Where do you use async vs threads and why?" | [02](02-repo1-architecture.md), [06](06-csharp-dotnet-qa.md) |
| Framework maintenance | "You have 700 test cases. How do you stop the framework rotting?" / "How do you share setup between tests?" | [02](02-repo1-architecture.md) §B, [06](06-csharp-dotnet-qa.md) |
| Firmware functional testing | "What's the difference between unit, integration and functional firmware testing?" / "Give me a functional test you automated end to end." | [05](05-firmware-functional-and-protocol-testing.md) |
| Protocol testing | "How do you test a protocol timeout?" / "How do you inject a corrupted frame?" / "How do you know the DUT is wrong and not your tester?" | [05](05-firmware-functional-and-protocol-testing.md) |
| State machines | "How do you test an invalid state transition?" | [03](03-repo2-firmware.md), [05](05-firmware-functional-and-protocol-testing.md) |
| Serial/OTA | "How would you test a firmware upgrade that is interrupted by power loss?" | [09](09-ble-serial-ota-rf-rtos.md) |
| RTOS | "Mutex vs binary semaphore? Priority inversion?" | [09](09-ble-serial-ota-rf-rtos.md) |
| CI/CD | "How do you run hardware tests in a pipeline?" | [10](10-cicd-azure-devops.md) |
| Azure DevOps | "How would you connect an automated test to a test case?" | [10](10-cicd-azure-devops.md) |
| RF | "How much RF have you done?" | [09](09-ble-serial-ota-rf-rtos.md) — scripted honest answer |

---

## The one-sentence framing that makes the JD fit you

> "I build and maintain the C#/.NET automation that drives a USB-PD compliance tester: the framework discovers test classes by reflection, configures our tester firmware over TCP, captures the live protocol traffic from the DUT, validates message sequences and timing against the spec, and produces HTML/CSV/PDF reports. The radio is different, but the shape of the job — take a spec test case, turn it into a maintainable C# test that drives firmware over a link and validates timed protocol behaviour — is exactly what I do every day."
