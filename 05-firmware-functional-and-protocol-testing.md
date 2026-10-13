# Phase 5, 6, 7 — Firmware Functional Testing, Protocol Testing, State-Machine Testing

---

# Part 1 — Firmware Functional Testing

## 1.1 Definitions you must be able to give in 20 seconds each

**Firmware functional testing** = verifying that firmware running on real hardware produces the required *externally observable behaviour* when stimulated through its real interfaces, under the conditions the spec defines. You test the device as a black box through its actual ports, timing included.

| | Scope | Runs on | Stimulus | Oracle | Catches |
|---|---|---|---|---|---|
| **Unit test** | one function/module | host PC or simulator | direct function call | assert on return value/state | logic errors |
| **Integration test** | 2+ modules together | target or simulator | internal API | assert on inter-module contract | interface mismatches |
| **Firmware functional test** | whole firmware image | **real target hardware** | **real external interface** (CC line, UART, I2C, radio) | spec-defined observable behaviour **and timing** | state-machine bugs, timeouts, race conditions, spec non-compliance |
| **Hardware test** | the board itself | real hardware | electrical stimulus | electrical limits (V, I, impedance, eye diagram) | component/layout/tolerance faults |
| **System test** | product + environment + user | real system | end-user scenarios | user requirements | integration & usability |

Key extra distinction for embedded: **functional tests are timing-sensitive and stateful.** A functional test can fail because a response arrived 2 ms late even though the content was perfect. Unit tests almost never have that property. That is why the tester needs FPGA/RTOS-grade timestamping and the PC cannot be the timing reference.

## 1.2 How automated firmware functional testing actually works

```
1 PRECONDITION   Is this test applicable to this device? (declared capabilities / VIF / config)
2 BRING-UP       Put the device into a known, spec-defined starting state
3 STIMULUS       Drive the external interface — including deliberately invalid stimuli
4 CAPTURE        Record everything: traffic, timestamps, analog rails, device state
5 ORACLE         Compare against the spec model, not a recorded golden log
6 VERDICT        PASS / FAIL / NA / INCOMPLETE with a human-readable reason + evidence index
7 CLEANUP        Return the device and the tester to a known state for the next test
8 EVIDENCE       Persist the raw capture so a failure can be re-analysed offline
```

Your framework does all eight. Map them out loud:

| Step | Your code |
|---|---|
| 1 | `PreConfigure()`, `CheckEPRSupportField_VIF`, `DUT_Type_C_State_Machine == SRC` gate in `TD_4_2_1` |
| 2 | `Bring_Up_Procedures` / `BringUpProcedure.COMMON_PROC_BU_1_Bring_up_Source_UUT` |
| 3 | `Configure()` building `FwMsgCfg` message groups; `IterateForCableCondition()` |
| 4 | `DataStreamService` streamer thread; LSADC/HSADC channels; `.grltrace` |
| 5 | `PDStateMachine.ExpectedMsgSequence`, `PDSpecLimits`, `Common_Check_PD_1..15` |
| 6 | `PDTestResult.UpdateTestResult/UpdateTestDesc(desc, packetIndex)` |
| 7 | `StopStreamer_Prerequisite()`, `StopStreamer()`, `FW_FPGA_FullReset` per MOI |
| 8 | `.grltrace` + JSON packet export + HTML/CSV/PDF reports; replayable by `XUnit_TestFramework` |

## 1.3 How a C# framework controls firmware — the four levers

1. **Configuration commands** — set the tester's role, capabilities, Rp/Rd, VBUS/VCONN, E-Load current. `PD_Controller.SetControllerConfiguration(ControllerConfigEnum.X)`.
2. **Pre-compiled behaviour** — upload a *message group* that the firmware replays with hardware timing. This is how you get µs-accurate stimulus from a non-real-time host (see [04](04-two-repos-connected.md)).
3. **Fault injection** — deliberately wrong behaviour: `DisableGoodCrcTestScript`, `CRC_BitManipulationResponseCheck`, `ObjectBitManipulationResponseCheck`, `Config_PS_RDY_TimeDelay`, `ConfigPacketTimeDelay`, `NoRequestResponeForSourceCapMessage`, `WaitMessageResponse`, plus `MaskPacketErrors(MaskAllErrors)`.
4. **Observation** — streamer capture, `InsertTimestamp()` assertion markers injected by firmware into the capture stream, LSADC/HSADC analog capture, `Get_Tester_Temperature_Error()`.

## 1.4 Diagnosing a functional failure

The framework is built for this, and you should say so:

- Every result carries a **packet index**: `UpdateTestDesc($"...Packet index#{respPdMsg.ActualGlobalIndex}")`. You jump straight to the offending frame in the trace.
- Every result carries **expected vs obtained**: *"Expected time is within tSenderResponse Min 30 ms, Obtained time: 41.2 ms"*.
- Every byte on the wire is in the debug log as hex **with the API description**.
- Per-test **timing breakdown** (overall / config / validate) in a separate status log.
- The `.grltrace` can be reloaded offline (second app instance on port 5002, `LoadSignalCaptureFile`, `ExecuteTestCasesOffline`) and re-validated **without the hardware** — so you can change the validator and re-run the analysis on the original evidence.

## 1.5 Interview answer — "Tell me about your experience with firmware functional testing"

> "Most of my day is firmware functional testing. I work on the C#/.NET automation for a USB Power Delivery compliance tester. The device under test is a real charger, laptop, hub or cable, and we exercise it through the physical Type-C interface — CC lines, VBUS, VCONN, USB data lines.
>
> A test is a C# class with three hooks: `PreConfigure` decides whether the test even applies to this device based on its declared capabilities; `Configure` programs our instrument's firmware with the exact stimulus — what messages to send, with what timing, and deliberately what to get wrong; `Validate` runs after the exchange and asserts against everything we captured.
>
> The capture is the important part. We stream the decoded protocol traffic with hardware timestamps plus the analog rails, and we group the messages into Atomic Message Sequences. Then instead of comparing against a recorded golden log, we compare against a model of what the spec *allows* — for a Source_Capabilities message the legal endings are Accept+PS_RDY, or Reject, or Wait — and we report which one we got and why it was or wasn't valid.
>
> Concretely, the functional suite is the USB Type-C functional test spec: about 63 `TD.4.x` tests covering attach and detach for Source, Sink, DRP, Try.SRC, Try.SNK and accessory modes, role swaps, current advertisement, dead-battery behaviour, and hub port types. A test like TD.4.2.1 runs the same connection scenario across three different emulated cable conditions and validates both the protocol sequence and the USB data-line enumeration.
>
> On top of that I maintain the framework itself: test discovery by reflection so adding a test is one enum plus one class, shared bring-up procedures so 200 tests reuse one attach sequence, a common-checks layer that validates every capture against the spec-mandated checks automatically, and an offline replay layer in xUnit that re-runs the validation engine against recorded captures so I can regression-test my validation changes without hardware."

*(Trim to the first three paragraphs if you want a 60-second version.)*

---

# Part 2 — Communication Protocol Testing

## 2.1 Protocols actually present in your repos

| Protocol | Where | How you use it | How it's tested |
|---|---|---|---|
| **USB Power Delivery (BMC over CC)** | Whole stack. `GrlPdLib`, `UsbPDPhyLayer.c`, `UsbPDProtocolLayer.c` | The protocol under test | ~700 compliance + functional tests |
| **USB Type-C (CC pull-ups/downs, Rp/Rd/Ra)** | `FunctionalTestCases\TD_4_*`, `Common_Check_PD_13_Check_Correct_Use_of_Rp` | Connection state machine | 63 functional tests |
| **TCP/IP (Ethernet)** | `GrlEthernetLink_C2.cs`, `TCPIPServer\FWTCPIPServer.cs`, `FWTCPIPAsyncServer.cs` | Host ↔ instrument control + data | Implicitly, via retry/reconnect/timeout logic; `EthernetPhyReset` between repeats |
| **USB (libusbK bulk)** | `GrlUsbLink_C2.cs`, `GrlUSBLinkC2\UsbStream.cs`, `libusbK.cs` | Alternative host link; SD-card mass storage during FW update | |
| **UART / RS-232 serial** | `System.IO.Ports` in `USB_PD_Engine.ConnectToSeraiPort`, `EquipmentLib\SerialPort.cs`, `SerialPortComm.cs`, `FirmwareUpdateManager` (1200-baud erase + bossac) | 5-port switch control, E-Load flashing, instrument console | COM port discovery by VID/PID via WMI |
| **I2C** | `UsbI2C.c` (firmware master), `ccs/SSBL` (slave 0x0E / 0x1E), tunnelled host commands `0xE1,0xD2` | Zynq ↔ C2000 MCUs, FRAM, sensors | Version read, mode get/set, flash program |
| **SPI** | `UsbSPIGpio.c`, `XSpiPs` in `prvSetupHardware`, `GrlUsbLink.WriteSPI` | FPGA/peripheral access | |
| **DisplayPort AUX** | `DPAuxDecoder` project, `UsbDpAux.c`, `DP_Alt_Mode.c` | DP Alt Mode over Type-C | `DisplayPortTests\`, `DP_PDValidationHelper` |
| **Thunderbolt / USB4** | `TBTClassFactory`, `USBPD_Rev2\USB4\` | Enter_USB, Data_Reset | `TEST_PD_USB4_*` |
| **BC 1.2 / Quick Charge (D+/D- signalling)** | `BC_1_2 Tests\`, `QCLegacy.c`, `QC_Legacy_Factory` | Legacy charging protocols | dedicated MOIs |
| **HTTP/REST + JSON** | ASP.NET Core controllers + Swagger | Host API | Selenium UI regression |
| **Named pipes** | `UICommServer.cs`, `TBTFactory` | IPC | |
| **VISA (GPIB/LAN instruments)** | `GRLVisaIO.cs`, `visa32/64.cs`, `EquipmentLib\GenericIO.cs` | Scope/DMM/e-load control | |

**Not present:** BLE, CAN, sub-GHz, MQTT, Zigbee.

## 2.2 The seven axes of protocol testing (use this structure for any protocol)

For each axis: *what can go wrong* → *how you test it* → *your evidence*.

| Axis | What goes wrong | How you test it | Your evidence |
|---|---|---|---|
| **1. Framing / encoding** | Bad preamble, wrong SOP*, bit-rate drift, malformed EOP | Transmit valid and deliberately invalid framing; measure the eye and bit rate | `TEST_PD_PHY_ALL_1_Transmit_Bit_Rate_and_the_Drift`, `TEST_PD_PHY_ALL_6_Invalid_SOP_ALL`, `TEST_PD_PHY_ALL_7_Valid_SOP_ALL`, `Common_Check_PD_1_Check_Preamble` |
| **2. Integrity** | CRC not checked, or checked wrongly | Send a frame with a corrupted CRC; the DUT must **not** GoodCRC it | `TEST_PD_PHY_ALL_8_Incorrect_CRC`, `CRC_BitManipulationResponseCheck.cs`, `TG7_TS2_CRC_Bit_Manipulation_Response_Check` |
| **3. Field validity** | Reserved bits set, illegal enum values, wrong revision | Set every reserved/invalid field in turn; expect `Not_Supported`/`Reject`/ignore | `Common_Check_PD_2_Check_Message_Header`, `Common_Check_PD_10_Extended_Message_Header`, `ReservedPayloadTest.cs`, `TG3_TS2_ControlInvalidMessage`, `SendGroup_of_NotValidExtendedMessages` |
| **4. Sequence / AMS** | Correct messages in the wrong order; interrupting an uninterruptible sequence | Model all legal sequences and diff; interrupt deliberately | `PDStateMachine.ExpectedMsgSequence`, `Common_Check_PD_4_Check_Atomic_Message_Sequence`, `TG6_TS1_AMS_Interruptible_and_Uninterruptible_Test` |
| **5. Timing** | Response too slow/too fast; timer not started/stopped | Measure every response interval against the spec limit; force timer expiry by *not* responding | `APIValidationHelper.tSenderResponse/tProtErrSoftHardReset`, `PDSpecLimits`, `TEST_PD_PROT_SRC3_2_SenderResponseTimer_Timeout`, `TEST_PD_PROT_SNK_4_SinkWaitCapTimer_Timeout`, `Config_PS_RDY_TimeDelay.cs` |
| **6. Error recovery** | Doesn't Soft/Hard Reset when it should; MessageID not reset | Force a protocol error and check the reset path and the MessageID counter | `TEST_PD_PROT_ALL_2_Soft_Reset_and_Hard_Reset`, `TEST_PD_PROT_ALL_4_Reset_Signals_and_MessageID`, `TEST_PD_PROT_ALL_3_SOFTRESET_RESPONSE`, `Common_Check_PD_14_Check_Hard_Reset` |
| **7. Robustness / stress** | Breaks under repetition, collisions, random input | Fuzz-ish random tests, N-times repetition, multi-port stress | `TG5_TS1_Random_Test`, `TG8_TS1_Initiate same Message for N time`, `TG8_TS2_FourPortStressTest`, `TG4_WrongBehaviorTest`, `TG2_TS1_Glitch_After_Attach_Message` |

## 2.3 Standard protocol questions and your answers

**Q: How do you test a protocol timeout?**
> Two directions. *Deadline* — the DUT must respond within the timer; I measure the interval from the GoodCRC of my message to the first byte of its response and compare with the spec limit. *Timeout* — the DUT must give up after the timer; I deliberately don't respond and assert that it takes the correct recovery action within the window. We have both as separate tests, e.g. `SenderResponseTimer_Deadline` and `SenderResponseTimer_Timeout`. The measurement reference is the hardware timestamp from the instrument, never a `DateTime.Now` on the PC — a Windows host can't time 30 ms reliably.

**Q: How do you inject a corrupted frame?**
> The instrument firmware exposes bit-manipulation hooks. The test says "send this message but flip bit N of the header / CRC / payload". Then I assert the DUT does *not* acknowledge it and that it recovers correctly. Header, CRC and payload are three separate test scripts in our robustness suite.

**Q: How do you know the failure is in the DUT and not your tester?**
> Four checks in order. (1) The framework reports whether the *tester* successfully initiated every configured message — `PacketInitStatus` — so a tester-side failure is `INCOMPLETE`, not `FAIL`. (2) The common-check layer validates the tester's own messages in the capture too, so a malformed tester frame shows up. (3) There's a controller self-test suite (`ControllerSelfTestEngine`, VBUS/VCONN voltage and current min/max, PDC check). (4) I replay the same capture offline through the validation engine — if the verdict changes, the bug is in my validator, not the DUT.

**Q: How do you handle a protocol that's chunked?**
> `UpdateChunkDetails()` in `PDStateMachine` walks the chunk exchange, tracking each chunk, its GoodCRC index and retries, and attaches them to the AMS element — so a chunked extended message is validated as one logical message with its chunking behaviour checked separately (`TEST_PD_PROT_ALL3_5_Chunked_Extended_Message_Response`, `ChunkSenderResponseTimer_Timeout`).

**Q: A response is correct but 5 ms late. Pass or fail?**
> Fail, and the report says exactly that: expected within X ms, obtained Y ms, packet index N. Timing is part of the contract. But before filing it I'd re-run with the capture, check whether the delay is consistent or a one-off, and check whether our own stimulus was delivered on time.

---

# Part 3 — State-Machine Testing (Phase 7)

## 3.1 Concepts

- **State machine** = a finite set of states, a set of events, and a transition function `(state, event[, guard]) → state`, optionally with entry/exit actions.
- **Why firmware uses them:** deterministic, auditable behaviour with bounded memory; the spec is itself written as a state machine; and it makes "what should happen if X arrives now?" answerable rather than emergent.
- **Mealy vs Moore:** Mealy output depends on state *and* event (typical for protocol layers); Moore output depends on state only (typical for connection/LED/power states).

## 3.2 The three state machines you can talk about

### A. Protocol Layer (firmware) — `UsbPDProtocolLayer.h`
See [03](03-repo2-firmware.md) §5.1 for the full enum lists and the transition diagram. Headline: `PRL_TX_*` (10 states), `PRL_RX_*` (5), `PRL_HR_*` (7), BIST TX/RX, and the struct keeps `Pre / Current / Next`.

### B. Type-C connection (DUT behaviour under test)
`Unattached.SRC → AttachWait.SRC → Attached.SRC`, DRP toggling, `Try.SRC`/`TryWait.SNK`, accessory states. Tested by the 63 `TD_4_x` functional tests.

### C. Atomic Message Sequence model (host, C#) — `PDStateMachine.cs`
Not a classic FSM but a **legal-sequence model**: for each initiating message type it enumerates every valid continuation. This is the oracle.

## 3.3 How to test a state machine — the six test classes

| Class | Question | Your example |
|---|---|---|
| **Valid transitions** | Does each legal event move us to the right state with the right side-effect? | `TD_4_2_1 Source Connect Sink` — Rd detected → AttachWait → Attached, VBUS turns on |
| **Invalid / unexpected events** | Does an event that's illegal in this state get ignored, rejected, or cause a defined error — and *not* corrupt state? | `TEST_PD_PROT_SRC_11_Unexpected_Message_Received_In_Ready_State`, `TEST_PD_PROT_ALL_5_Unrecognized_Message`, `SendGroup_Of_InvalidControlMessages` |
| **Timeouts** | Does the state machine leave a waiting state when the timer expires, and take the right recovery action? | `SourceCapabilityTimer_TimeOut`, `SinkWaitCapTimer_Timeout`, `SenderResponseTimer_Timeout`, `PSSourceOffTimer_Timeout`, `tEnterEPR_Timer_Timeout`, `tSourceEPRKeepAlive_Timeout` |
| **Recovery / reset** | After an error, does it return to a well-defined state with counters cleared? | `TEST_PD_PROT_ALL_2_Soft_Reset_and_Hard_Reset`, `TEST_PD_PROT_ALL_4_Reset_Signals_and_MessageID`, `TEST_PD_PS_SRC_5_Source_Hard_Reset_Test` |
| **Concurrency / interruption** | What if two events arrive at once, or a sequence is interrupted mid-way? | `Common_Check_PD_4_Check_Atomic_Message_Sequence`, `TG6_*` interruptible/uninterruptible AMS tests, `TEST_PD_VDM_SNK_3_Interruption_by_PD_Message`, collision avoidance tests |
| **Coverage** | Have we visited every state and every transition at least once? | Argued from the CTS matrix (the spec defines the required coverage); plus MOI/test-group structure |

## 3.4 "How would you automate state-machine testing in C#?" — the answer

> "Three layers.
>
> **Model the machine as data, not code.** I'd express states, events, guards and expected next-state as a table — a `Dictionary<(State, Event), Transition>` or a JSON/CSV file. That makes the model reviewable by the firmware owner and lets me generate tests from it instead of hand-writing them.
>
> **Generate the test cases.** From that table: one test per valid transition (drive the machine to the source state, fire the event, assert the destination state and side effects); one test per *invalid* pair — every event that has no transition from a state should be a defined no-op or error; plus timeout tests for every state that has a timer.
>
> **Observe state, don't guess it.** The hard part in firmware is knowing what state it's actually in. Three options: a debug/telemetry command that reports the current state; inference from externally observable behaviour; or markers injected into the capture. Our firmware does the third — it calls `InsertTimestamp()` to put assertion markers into the capture stream, and the C# side reads them back as `MsgType.Test_Execution_State` packets to slice the capture per test condition. That's what `TD_4_2_1` uses to separate three cable conditions inside one run.
>
> Practically, that's roughly what we already do — our `PDStateMachine` class holds the legal sequence model and the validator diffs the capture against it, so adding a new legal response is a data change in one place rather than an edit to 200 tests."

**Follow-ups to be ready for:**
- *How do you get the machine into a deep state reliably?* → shared bring-up procedures, and if bring-up fails the test is `INCOMPLETE`, never `FAIL`.
- *What if the DUT has hidden internal states?* → you can only test observable equivalence; ask the firmware owner for a state-report command, or use W-method/transition-tour thinking to distinguish states by their response to a distinguishing sequence.
- *How do you test a transition that needs a rare condition (dead battery, brown-out)?* → the tester emulates the condition — we have `TD_4_11_2_Sink_Dead_Battery_Test` and E-Load/PPS control to create exact voltage/current conditions.
- *Coverage metric?* → transition coverage (every arc fired at least once), plus negative coverage (every illegal state/event pair exercised). Say honestly that we track it against the CTS matrix rather than computing it automatically — and that adding a generated coverage report is something you'd like to do.
