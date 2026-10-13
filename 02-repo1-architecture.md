# Phase 2 — Repository 1 Deep Analysis: `D:\GitHub\C2BrowserApp`

This is the highest-value document in the pack. The panel said C#/.NET automation is the primary focus; this is your evidence.

---

# A. Architecture

## A.1 Solution and projects

Solution: `C2WebApp\C2WebApp\C2BrowserApp.sln` — 16 projects, all **.NET Framework 4.7.2** (one legacy at v4.6, one MSTest project at `net5.0`).

| Project | Assembly / namespace | Role |
|---|---|---|
| `C2WebApp\C2WebApp\C2WebApp\C2BrowserApp.csproj` | `C2BrowserApp` | **ASP.NET Core Web API host** (`Microsoft.NET.Sdk.Web`, targets `net472`). Kestrel + libuv, port 5001 (5002 = offline instance), Swagger, serves the React build from `wwwroot`. |
| `AppFrmWrk` | `AppFrameWork` | **Framework core.** `Host`, `PropertyBag`, `ModuleManager`/`IModule`, `ResultsKeeper`/`ResultsTable`, `ReportHTML`, `ReportModule_PDF`, `RTDataManager`, `TestResult` enum, `ErrorHandler`, delegates, `UICommClient`, `GRLVisaIO`/`visa32`/`visa64`, waveform format readers. |
| `USB_PD_TesterLib` → `GrlUsbPdControllerLibC2` | `GrlUsbPdControllerLib` | **Device / hardware abstraction.** `PD_Controller` (~793 KB — the whole tester command API), `PD_Controller_C2Gen2`, `GrlEthernetLink_C2`, `GrlUsbLink_C2`/`libusbK`, `SerialPortComm`, `DataStreamService`, `PacketLogBulkLP`, `FlashFPGA`, `SysData` (FRAM calibration), `TCPIPServer`, `RunScript` (visual step engine), `TestImplementation` (`ITestImpl`/`TestWrapper`). |
| `GrlPdLib` | `GrlPdLib` | **USB-PD protocol model.** `PDMessage`, `DecodePayload`, `PDmessageDecodeEngine`, `CalculateCRC`, `SourceCapability`/`SinkCapability`/`RequestMessage`/`EPR*`, capture file read/write, and the **AMS validation** subsystem (`AMSGenerator`, `AMSValidationEngine`, `PDStateMachine`, `AtomicMessageSequence`, `ValidateMsgSeq`, `MergeAMS`). |
| `GRLSharedModule` | `GRLSharedModule` | Shared enums/DTOs used by everything (`PDMessage`, `MessageHeader`, `PDOData`, `UsbPDHelper`, report helpers). |
| `USBPDModule\SampleModule` → `USB_PDModuleC2` | `USB_PDModule` | **The test content + execution engine.** 632 `.cs` files. `USB_PD_Engine`, `ModuleHelper`, `TestCase`, `USBPD_2_TestCase`, `FunctionalTests`, all the per-domain factories, `PDValidationHelper`, `Bring_Up_Procedures`, `Generate_GRL_Report_Module`, `FirmwareUpdateManager`, `VIFInfo`. |
| `TestCaseHelper` | `TestCaseHelper` | **Result + metadata model**: `PDTestResult`, `PDSecTest`, `TestCaseDetails`, `ComplianceTestEnum` (the master test catalogue), `TestGroup`, `ReturnStatus`. |
| `GrlC2ApiLib` | `GrlPdApiLib`/`GrlC2ApiLib` | **Public scripting API** — `partial class GrlC2Controller`, split across `GrlC2Controller*.cs` (Set/Get/Config/Send/Read/Reset/EPR/Eload/GroupCommand/ReportHelper). This is what external C# scripts and the Python API tool call. |
| `Grl.APIValidation` | `Grl.APIValidation` | Validation helpers for the scripting API: message look-up (`GetPDMsgFromList`), spec timing checks (`tSenderResponse`, `tProtErrSoftHardReset`), CSV/HTML API reports. |
| `C2APIScript` | `C2APIScript` | **Console test runner** for API-level scripts. `abstract TestScript`, `TestFactory`, `TBTExecutionEngine`/`TBTFactory` (talks to the web app over a **named pipe**), ~100 robustness/emulation scripts. |
| `XUnit_TestFramework` | `XUnit_TestFramework` | **Offline unit/regression tests** — xUnit 2.9 + Moq 4.20, replaying captured PD traffic from JSON fixtures through the real validation engine. |
| `UnitTest_NetFrmWrk`, `UnitTest` | — | MSTest projects (`net472` and `net5.0`). |
| `DPAuxDecoder` | | DisplayPort AUX channel decoder. |
| `Grl.Core.FilesManager` | | File writers with a factory (`IWriteFile`, `CSVWriteFileFactory`). |
| `HWDataSet` → `GRLHWDataSet` | | Large waveform/sample data store (`DataStore`, `DataStream`, `FileBuffer`, `MemoryBuffer`). |
| `C2UnitTestTool_CapFile` | | WinExe tool for working with capture files. |

Outside the solution but in the repo: `reactui` (React front end), `Selenium` (Python Selenium UI regression), `BuildFiles` (installer payload, Doxygen API docs, Python API tool).

## A.2 Layer diagram (actual, not generic)

```
┌──────────────────────────────────────────────────────────────────────┐
│ PRESENTATION                                                         │
│  reactui (React SPA)         Swagger UI          Python API tool     │
└───────┬──────────────────────────┬────────────────────┬──────────────┘
        │ HTTP :5001               │                    │
┌───────▼──────────────────────────▼────────────────────▼──────────────┐
│ API LAYER                                                            │
│  C2BrowserApp  — ASP.NET Core controllers                            │
│   ConnectionSetupController / ProductCapabilityController /          │
│   TestConfigurationController / ResultsController /                  │
│   ReportsGenerationController / C2ApiController / DebugModeController │
│  Helpers\ControllerHelper  (SINGLETON — owns the engine + controller) │
│  GrlC2ApiLib.GrlC2Controller (partial) — public scripting API         │
└───────┬──────────────────────────────────────────────────────────────┘
        │
┌───────▼──────────────────────────────────────────────────────────────┐
│ TEST ORCHESTRATION (USB_PDModule)                                    │
│  USB_PD_Engine                                                       │
│    StartTestExecutionThread → TestExecutionEngine →                  │
│    RunSelectedTestCases (repeat loop) → ExecuteTestCase_AllMOIs       │
│  Per-MOI factories:                                                  │
│    USB_2_0_Engine (PD-merged CTS)   PD3_CTS_Factory                   │
│    PD2_CTS_Factory_C2               DeterministicTestEngine           │
│    FunctionalTestFactory            DPAltModeFactory                  │
│    QC_Legacy_Factory / QC4_0_Factory / QC3Plus_Factory                │
│    TBTClassFactory / CB_TestEngine / BC_1_2_TestEngine / IEC_Engine   │
│    PD3_PPS_Factory / MFiTestFactory                                   │
└───────┬──────────────────────────────────────────────────────────────┘
        │  reflection: enum name == class name → Activator.CreateInstance
┌───────▼──────────────────────────────────────────────────────────────┐
│ TEST CASES                                                           │
│  abstract TestCase { PreConfigure(); Configure(); Validate(); }       │
│    ├── abstract USBPD_2_TestCase     (PD merged CTS, ~200 tests)      │
│    ├── abstract FunctionalTests      (TD_4_x, 63 tests)               │
│    └── domain bases (TBT, QC, DP-Alt, IEC, BC1.2 …)                   │
│  Shared building blocks:                                             │
│    Bring_Up_Procedures, Common_Check_PD_1..15, COMMON_PROC_PD_1..18   │
│    USBPDConfigHelper (configure tester), PDValidationHelper (assert)  │
└───────┬──────────────────────────────────────────────────────────────┘
        │
┌───────▼──────────────────────────────────────────────────────────────┐
│ PROTOCOL LIBRARY (GrlPdLib / GRLSharedModule)                        │
│  PDmessageDecodeEngine → PDMessage                                   │
│  AMSGenerator → AtomicMessageSequence                                │
│  PDStateMachine.ExpectedMsgSequence(msg) → List<ExpectedAMSSequence>  │
│  AMSValidationEngine, CalculateCRC, capability/PDO models             │
└───────┬──────────────────────────────────────────────────────────────┘
        │
┌───────▼──────────────────────────────────────────────────────────────┐
│ DEVICE / TRANSPORT (GrlUsbPdControllerLib)                           │
│  PD_Controller  (build byte[] command frames; WriteReg/ReadReg)      │
│  PacketLogBulkLP (WriteFWCommand / ReadFWCommand)                    │
│  GrlEthernetLink_C2   ── TCP :5002  write / read / data-stream       │
│  GrlUsbLink_C2 (libusbK)   SerialPortComm (System.IO.Ports)          │
│  DataStreamService  ── DMA streamer thread → CC-line + LSADC samples  │
└───────┬──────────────────────────────────────────────────────────────┘
        │  Ethernet
┌───────▼──────────────────────────────────────────────────────────────┐
│ TESTER HARDWARE (repo 2)  Zynq-7000: FreeRTOS app + FPGA PD PHY      │
└───────┬──────────────────────────────────────────────────────────────┘
        │  CC1/CC2, VBUS, VCONN, D+/D-, SBU
┌───────▼──────────────────────────────────────────────────────────────┐
│ UUT / DUT — charger, laptop, hub, cable, phone                       │
└──────────────────────────────────────────────────────────────────────┘

Results path (reverse):
 sampled CC line + ADC → DataStreamService → CClinestreamer
   → PDmessageDecodeEngine → List<PDMessage>
   → AMSGenerator → List<AtomicMessageSequence>
   → test Validate() → PDTestResult tree
   → Generate_GRL_Report_Module → HTML / CSV / PDF / XML + .grltrace
   → UICommClient (named pipe) → React UI live status
```

## A.3 Cross-cutting services

| Concern | Where | Notes |
|---|---|---|
| **Configuration** | `AppFrmWrk\PropertyBag.cs` + `AppProperty` + `PropertyItem` enum; `C2MoiConfigurations\*Configuration.cs` per test domain; `appsettings.json` | `PropertyBag._LoadProperyFile()` / `SavePropertyFile()` — persisted key/value store loaded at startup in `USB_PD_Engine`'s constructor. Web layer reads/writes via `PUT Put*Configuration` endpoints. |
| **Logging** | `Host.WriteToDebugLogger(DebugModeType, string[, Exception])`, `WriteToDebugLogger_TestStatus`, `WriteToDebugLogger_ErrorCase`, `ClearDebugLogFile` | Own file logger, not `ILogger`. Every byte written to/read from the tester is logged as hex + API description (`GrlEthernetLink_C2.Write`). Separate timestamped file per test run. |
| **Reporting** | `AppFrmWrk\ReportHTML.cs`, `ReportModule_PDF.cs`; `USB_PDModule\Generate_GRL_Report_Module.cs`, `GRL_Report_Module.cs`, `C2_XML_Report_Generation.cs`; `Grl.APIValidation\API_html_Report.cs` | Per-MOI folder, per-run folder, plus a parent summary. Served back as static files at `/Report`. |
| **Live UI events** | `AppFrmWrk\UICommServer.cs` (`UICommClient`) over `NamedPipeWrapper` → JSON `UIEventMessage` | Connection change, toast, status log, test-case result, run summary. |
| **Test data** | `C2WebApp\Resource\PDM Resource\PDM JSON\*` (bring-up + test-condition definitions per CTS section), VIF files, `XUnit_TestFramework\**\*.json` capture fixtures | The bring-up procedure and test conditions for PD-merged tests are **data-driven from JSON**, parsed by `PDMBUJsonParser*` in `PDValidationHelper`. |
| **Serialization** | `Newtonsoft.Json` (config, API, UI events), `DataContractSerializer` (`TestCaseDetails.TransferTestResultsToFile`), `BinaryFormatter` (`ExtensionMethods.DeepCopy<T>`, capture files) | |
| **Licensing** | `AppFrmWrk\LicenseManager.cs`, `AppLicense`, `LicenseValidator`; `C2LicenseManager` reading FRAM | Every MOI is gated: `ModuleHelper.UpdateMOI_BasedOn_License(C2_hardwareLicense.X)`. |

---

# B. The Automation Framework

## B.1 How a test is created

### The base class — Template Method

`USBPDModule\SampleModule\TestCase.cs`

```csharp
public abstract class TestCase
{
    public TestCaseDetails ObjTestCaseDetails { get; set; }   // metadata + result object
    public PD_TesterCommCfg ObjCommonConfi   { get; set; }    // tester comms/config
    public USBPDConfigHelper ObjTstrUtil     { get; set; }    // tester behaviour builder
    public PDValidationHelper ObjValHelp     { get; set; }    // assertion library
    public PDTestResult ObjTestResults => ObjTestCaseDetails?.TestResultObject;

    public bool StartStreamer(List<LSADC_Channel> channelList, bool bEnQD = false,
                              bool bEnHSADC = false, bool bEnablePortB = false) { … }
    public virtual bool StopStreamer() { … }

    public virtual bool PreConfigure() => true;   // applicability / precondition gate
    public abstract bool Configure();             // program the tester for this test
    public abstract PDTestResult Validate();      // assert against the capture
}
```

Three fixed hooks. That is the whole contract a new test must satisfy.

### Domain base classes add shared behaviour

`USBPD_Rev2\USBPD_2_TestCase.cs`

```csharp
public abstract class USBPD_2_TestCase : TestCase
{
    public PDValidationHelper   m_ObjPD2Helper;
    public USBPDConfigHelper    m_ObjConfHelp;
    public Bring_Up_Procedures  objBringUp;
    public EPRSrc_Helper objEprSrcHelper; public EPRSnk_Helper objEPRSnkHelper;
    public ProtocolHelper objProtocolHelper; …

    public abstract SupportedPDMode m_SupportedPDVersion { get; set; }  // PD2 / PD3 / BOTH

    public override bool PreConfigure() { /* EPR fixture check, bring-up selection,
        sub-sampling ratio, tester src/snk PDOs, default properties,
        test-specific tester behaviour */ }
    public override bool Configure()     { /* build per-PD-mode message buffers */ }
    public override PDTestResult Validate() { /* common-check / common-procedure passes */ }
    public virtual bool ConstructAfterPDCMessageGroup() => false; // runtime message group
}
```

`FunctionalTestCases\FunctionalTests.cs` does the same for the Type-C functional suite (adds `bEnDataLinesValidation`, `bEnPortB`, `ValidateHighSpeedLine()`, `Read_All_Packets_Bef_Validation()`).

### A concrete test

`USBPD_Rev2\EPR Test cases\EPR Snk Test cases\TEST_PD_EPR_SNK3_9_EPR_Entry_failed_due_to_SourceCap.cs`

```csharp
class TEST_PD_EPR_SNK3_9_EPR_Entry_failed_due_to_SourceCap : USBPD_2_TestCase
{
    public override SupportedPDMode m_SupportedPDVersion { get; set; }

    public TEST_PD_EPR_SNK3_9_...(TestCaseDetails d, USBPDConfigHelper cfg, PDValidationHelper val)
        : base(d, cfg, val)
    { m_SupportedPDVersion = SupportedPDMode.PD_3_0; IsFWImple = FWAppObj.FWHost.PDMCurrentTestFWImpl; }

    public override bool PreConfigure()   // 1. is this test even applicable?
    {
        bool bEPRSnkSupp = m_ObjPD2Helper.CheckEPRSupportField_VIF(ObjTestResults, false);
        return bEPRSnkSupp ? base.PreConfigure() : false;
    }

    public override bool Configure()      // 2. set the tester up as an EPR Source
    {
        objBringUp.SetBringUp_ForConfiguration = BringUpProcedure.COMMON_PROC_BU_2_Bring_up_Sink_UUT;
        m_ObjPD2Helper.TestConditionCount = 1;
        ConstructAfterPDCMessageGroup();
        return base.Configure();
    }

    public override bool ConstructAfterPDCMessageGroup()   // 3. runtime message injection
    {
        FwMsgCfg VDMCfg = new FwMsgCfg(msgRetry: false, bNonPDmsg: true);
        m_ObjConfHelp.m_objPDCtrl.SetPDMMode(true, ModuleHelper.EnableEPRTestFlag(...));
        VDMCfg.MsgHdrPld.AddRange(m_ObjConfHelp.m_objPDCtrl.m_databuffer_PDC);
        VDMCfg.TxTimer = 50; VDMCfg.RxExpTimer = 50;
        m_ObjConfHelp.MessageGroup_AfterPDC.msgList.Add(VDMCfg);
        int TestId = m_ObjPD2Helper.SetTestId(32, ObjTestCaseDetails.TestCaseEnum);
        m_ObjConfHelp.AddTestInitToGrpCmdPhyTest(TestId, (Status)m_ObjPD2Helper.IscaptiveCable);
        return false;
    }

    public override PDTestResult Validate()  // 4. assert against captured traffic
    {
        PDTestResult testResult = new PDTestResult("TEST_PD_EPR_SNK3_9_...");
        base.Validate();

        var ObjtestCase_Check1 = new PDTestResult("EPR mode capable",            "TEST.PD.EPR.SNK3.9#1");
        var ObjtestCase_Check2 = new PDTestResult("UUT initiates EPR_Mode enter","TEST.PD.EPR.SNK3.9#2");
        var ObjtestCase_Check3 = new PDTestResult("Sourcecheck",                 "TEST.PD.EPR.SNK3.9#3");
        var ObjtestCase_Check4 = new PDTestResult("UUT sends Hardreset",         "TEST.PD.EPR.SNK3.9#4");

        List<AtomicMessageSequence> ams = m_ObjConfHelp.StrmServ.GetAMSList_PortA;
        List<PDMessage> MsgLst = m_ObjConfHelp.StrmServ.ObjCClinestreamer_PortA.TotalmsgList;

        bool status = objEPRSnkHelper.EPR_Mode_Request(ObjtestCase_Check1, iStAMS, iSpAMS, MsgLst.Count);
        if (!status) { /* mark 2,3,4 as NA — don't cascade a false failure */ }
        else {
            var EPREnter = m_ObjPD2Helper.GetDesiredAMS(
                new GetDesiredAMS_Param(iStAMS, iSpAMS, AMSName.EPR_Mode_Enter), false, false);
            if (objEPRSnkHelper.EPR_Mode_Enter(ObjtestCase_Check2, EPREnter, …))
                objEPRSnkHelper.SourceCap_HardReset_Validation(iStAMS, iSpAMS,
                                    ObjtestCase_Check3, ObjtestCase_Check4, EPREnter, true);
        }
        TestCondRes.AddTest(Check1..4);  testResult.AddTest(TestCondRes);
        return testResult;
    }
}
```

**Talking point:** note the `NA` cascade. When an early assertion can't be evaluated, the later ones are marked `NA` rather than `FAIL`. That's a deliberate anti-noise design — a failing report should point at the first real defect, not produce four red lines from one root cause.

### Test metadata and discovery

`TestCaseHelper\PD test results\PDTestResult.cs` holds `ComplianceTestEnum` — the master catalogue, with a strict contract written into the file as a comment:

```
//1. TestNameEnum shall be same as enum description except the special characters
//2. TestNameEnum shall have two parts, Test ID followed by test name, separated by a single space
//3. Test ID shall not contain any blank space
```

Each entry carries `[Description("TD.4.2.1 Source Connect Sink Test")]`. Ranges are delimited by sentinel members (`START_FTS`/`STOP_FTS`, `START_PD3`/`STOP_PD3`, `START_DET`/`STOP_DET`, `START_PD_MERGED`/`STOP_PD_MERGED`, `START_EPR_SNK`/`STOP_EPR_SNK`…), which the engines use to filter a flat selection list into a MOI:

```csharp
if (testObj.TestCaseEnum < ComplianceTestEnum.START_FTS ||
    testObj.TestCaseEnum > ComplianceTestEnum.STOP_FTS) continue;
```

`TestCaseDetails` is the per-test runtime record: `TestCaseEnum`, `DisplayName`, `TestGroup`, `TestId`, `IsSelected`, `IsSupported`, `IsExecuted`, `TestExecutionStatus`, `TestResultObject`, `GRLPlotTraceLoc`, `LsadcFileInfo`. It is `[Serializable]` and round-trips via `DataContractSerializer`.

**Reflection is the discovery mechanism** — `USB_2_0_Engine.cs:1139`:

```csharp
private static Type GetTestCaseObj(Assembly assembly, TestCaseDetails testObj)
    => assembly.GetTypes()
               .First(t => t.Name.ToUpper() == testObj.TestCaseEnum.ToString().ToUpper());
…
TestCase testScriptObject = (TestCase)Activator.CreateInstance(
        testScriptType, testObj, m_ObjCommhelper, m_objValidationhelper);
```

Identical pattern in `FunctionalTestFactory.cs:102`.

> **Adding a test is therefore: (1) add an enum member with a `[Description]`, (2) add a class with the same name deriving from the right base, (3) implement `Configure()`/`Validate()`.** No registry, no switch statement to edit.

## B.2 How a test is executed — the 12 steps

1. **Trigger.** `POST /api/TestConfiguration/PostTestListToExecute/{RerunCount}` with the selected test IDs (`TestConfigurationController.cs:479`). Python/CLI paths call `GrlC2Controller` directly; `C2APIScript` runs as its own process.
2. **Guard + thread start.** `USB_PD_Engine.StartTestExecutionThread()` — checks firmware version compatibility (prompts if mismatched), checks cable IR-drop calibration, sets `HostState.BUSY`, creates a dedicated `Thread` running `TestExecutionEngine(...)`, sets `TestExeState = Active`.
3. **Group by MOI and order.** `UpdateTestSelectionAndReorder` → `GetMOIOrder` builds `Dictionary<int, PDAppMode>` from the selection.
4. **Repeat loop.** `RunSelectedTestCases(moiorder, rerunCount)` — for each repeat: reset Ethernet PHY and reconnect (Gen2), create `Run{n}_Rep{m}_{timestamp}` directory, clear previous results, set `TestResultSummary.TotalTestCaseCount`.
5. **Per-MOI setup.** `ExecuteTestCase_AllMOIs` — per MOI: create the MOI folder, `ResetBeforeExecuting_MOI()` (`ControllerConfigEnum.FW_FPGA_FullReset`), check the hardware licence, set `PD_Controller.CurrentSpecRev`, then construct the right factory.
6. **Per-test discovery.** Factory loops the test list, filters by enum range, checks `IsSelected && IsSupported`, resolves the `Type` by reflection, `Activator.CreateInstance`.
7. **Precondition.** `testScriptObject.PreConfigure()` — VIF field applicability, fixture presence, device-type match. If not applicable the result is set to `NA` with a description and the test is skipped. **This is why the reports have very few false failures.**
8. **Device setup.** `StartOfTestCaseExecution()` — detach CC lines, reset streamer data, configure LSADC channels, start the DMA streamer thread.
9. **Configure + execute.** `testScriptObject.Configure()` builds the tester's behaviour as a *message group* (a sequence of `FwMsgCfg` frames with Tx/Rx timers), then `Execute()` pushes it to firmware and `PacketInitStatus(...)` waits for firmware to confirm all commands were initiated.
10. **Capture.** The streamer thread continuously pulls CC-line packets and ADC samples; `PDmessageDecodeEngine` decodes to `PDMessage`; `AMSGenerator` groups them into `AtomicMessageSequence`s.
11. **Validate.** `testScriptObject.Validate()` builds the `PDTestResult` tree; then `ValidateCommonCheckandCommonProc()` runs the 15 shared common checks and 18 common procedures against the same capture; `PostValidate()` folds in the firmware command-init status.
12. **Finish.** `EndOfTestcaseExecution()` stops the streamer, saves the `.grltrace`, writes packets to the results table; `PopulateResultHtmlFile()` emits the per-test HTML; after the MOI, `GenerateReportData_InsideMoiFolder()`; after the run, `GenerateReport_ParentFolder()` plus the Get-Caps CSV; debug log is rotated per repeat.

**Timing instrumentation is built in** — `ExecuteTestCaseOldImplementation` records overall / config / validate start-stop times into a dedicated test-status log. Good answer to "how do you find slow tests?".

**Stopping** — `StopTestExecution(bool commErrOccured)`:
```csharp
Task forceStop = Task.Run(() => Forcestop(commErrOccured, true));
Task.Run(async () => { await forceStop; AbortTestExecThread(); });
```
`Forcestop` stops the streamer thread and marks in-flight tests `INCOMPLETE` with a reason ("User stopped the execution" / "Tester ethernet error occured…"); `AbortTestExecThread` calls `Thread.Abort()` and spins until `!IsAlive`, then `finally { HostState = READY; ResetAppVariables(); }`.
There is a re-entrancy guard `m_TestAbortStatus` with the comment *"to avoid calling Forcestop() multiple times in single test execution which will cause race condition in ethernet communication class"*.

> **Be ready to criticise this.** `Thread.Abort()` is unsupported on .NET Core/5+ (throws `PlatformNotSupportedException`) and can leave locks and device state corrupted. The modern design is a `CancellationToken` checked at safe points, plus a hard device reset in `finally`. Saying this unprompted is a strong senior signal.

## B.3 How tests are maintained

| Mechanism | Where | What it buys |
|---|---|---|
| **Three-hook contract** | `TestCase` | A new engineer only has to learn `PreConfigure/Configure/Validate`. |
| **Shared bring-up procedures** | `Bring_Up_Procedures.cs` + `BringUpProcedure` enum (`COMMON_PROC_BU_1_Bring_up_Source_UUT`, `..._2_Bring_up_Sink_UUT`, PPS variants) | ~200 tests reuse the same attach/negotiate sequence; a spec change is one edit. |
| **Common checks / common procedures** | `common check\Common_Check_PD_1..15`, `Common procedure\COMMON_PROC_PD_1..18` | Spec-mandated checks (message header, GoodCRC, AMS, Rp usage, hard reset, source/sink caps) run against *every* capture automatically — tests don't re-implement them. |
| **Assertion library** | `PDValidationHelper.cs` (752 KB), `EPRSnk_Helper`, `EPRSrc_Helper`, `ProtocolHelper`, `PowerHelper`, `USB4_Helper`, `FunctionalHelper` | Domain assertions (`GetDesiredAMS`, `GetPDMsgFromAMS`, timing checks) live once. |
| **Data-driven test conditions** | `Resource\PDM Resource\PDM JSON\Section *.json` parsed by `PDMBUJsonParser*` | Bring-up + PD2/PD3 test-condition matrices are configuration, not code. |
| **Central catalogue with a naming contract** | `ComplianceTestEnum` | Enum name = class name = report ID. Rename discipline is enforced by the reflection lookup failing loudly. |
| **Config objects per domain** | `AppFrmWrk\C2MoiConfigurations\*Configuration.cs` + `PropertyBag` | User-tunable behaviour is not hard-coded in tests. |
| **Result priority rules** | `PDTestResult.UpdateTestResult`, `SetTestResultUpdatePriority(TestValidationPriority)` | A parent result derives deterministically from children; `TYPE2` used by functional tests treats INCOMPLETE differently. |
| **Offline replay regression** | `XUnit_TestFramework` | Validation-engine changes are regression-tested from JSON captures without hardware. |
| **Versioning** | Firmware/E-Load/PPS versions read at connect and compared to `ModuleHelper.FWversion`; execution blocks or warns on mismatch | Prevents "test failed because firmware is old". |
| **Rerun/repeat conditions** | `RepeatCondition` = All / Fail / Pass / Incomplete tests, `RerunCount` | Built-in flake triage: re-run only the failures. |

---

# C. C#/.NET concepts → actual file → why → interview question

| Concept | File / member | Why it's used | Likely question |
|---|---|---|---|
| **Abstract class + Template Method** | `SampleModule\TestCase.cs` — `abstract bool Configure()`, `abstract PDTestResult Validate()`, `virtual bool PreConfigure()` | Fixed lifecycle, variable body | *Why an abstract class here and not an interface?* |
| **Interface** | `AppFrmWrk\IModule.cs`, `ITestImpl.cs`, `IBusElement`, `ISignal`, `IResultsTable`, `Grl.Core.FilesManager\IWriteFile.cs`, `IChannelTemperatureChecker.cs` | Plugin contract across assembly boundaries | *When would you switch `TestCase` to an interface?* |
| **Inheritance depth** | `TestCase` → `USBPD_2_TestCase` → `TEST_PD_EPR_SNK3_9…` | Share domain behaviour | *What's the risk of a 3-level hierarchy? Would composition be better?* |
| **Polymorphism** | Engine holds `TestCase testScriptObject` and calls `Configure()`/`Validate()` without knowing the concrete type | Open/closed | *How does the engine avoid a giant switch?* |
| **Encapsulation** | `TestCaseDetails` private fields + read-only properties; `PDTestResult.UpdateTestResult()` instead of a public setter | Enforce the merge rule | *Why can't a test just assign `TestResult = PASS`?* |
| **Generics** | `List<PDMessage>`, `Dictionary<PDAppMode, List<TestCaseDetails>>`, `Dictionary<int, GrlEthernetLink_C2>`; `public static T DeepCopy<T>(this T self)` in `ControllerHelper.cs` | Type-safe containers, reusable clone | *What constraint should `DeepCopy<T>` have? (`where T : class` / needs `[Serializable]`)* |
| **Collections** | `Dictionary<PDAppMode, List<TestCaseDetails>> AllMoiTestList`; `Dictionary<C2_hardwareLicense, LicenseType>` | O(1) lookup by MOI/licence | *Dictionary vs List — when and why?* |
| **LINQ** | `assembly.GetTypes().First(t => t.Name.ToUpper() == …)`; `msgList.FindAll(x => x.msgTypeSec == MsgTypeSec.Test_Condition)`; `drvInfo.FirstOrDefault(x => x.DriveType == DriveType.Removable && !x.IsReady)`; `allProcesses.ToList().FindAll(...)` | Declarative filtering | *`First` vs `FirstOrDefault` vs `Single` — which does your code use and what happens when the test class is missing?* (It throws; the `catch` logs and returns `null`.) |
| **Delegates** | `AppFrmWrk\Host.cs`: `DelegateForceStopEvents`, `DelegateAbortDataReadThread`, `DelegateResultLogger`, `DelegateStatusLogger`, `DelegateLinkSignal` — wired in `USB_PD_Engine` ctor | Break the dependency from the low-level comm layer back up to the engine | *Why a delegate instead of calling the engine directly?* (layering — `GrlUsbPdControllerLib` must not reference `USB_PDModule`) |
| **Events / pub-sub** | `UICommClient.PublishStatusMessage/PublishTestCaseStatusEvent/...` over named pipes; `FileWatcher.Changed` | Decoupled UI updates | *How does the React UI know a test finished?* |
| **Attributes** | `[Description("TD.4.2.1 …")]` on every `ComplianceTestEnum`; `[Serializable]`; `[DesignerSerializationVisibility(Hidden)]`; `[HttpGet]/[HttpPost]/[FromBody]`; `[Fact]` | Metadata read back by `EnumHelper.GetEnumDescription` | *Where do you read attributes at runtime, and what does it cost?* |
| **Reflection** | `ModuleManager.GetInsalledModuleDetails()` (`Assembly.LoadFrom` every DLL, find the type implementing `IModule`, `Activator.CreateInstance`); `GetTestCaseObj`; `MethodInfo.GetCurrentMethod().Name` in log statements | Plugin loading + convention-based test discovery | *What are the downsides of reflection here?* (no compile-time safety, slow, renames break silently at runtime, `GetTypes()` can throw `ReflectionTypeLoadException`) |
| **async/await + Task** | `Task.Run(() => C2Engine.Get_DUT_Capabilities(port)).Wait()` in `ControllerHelper`; `Task forceStop = Task.Run(...); Task.Run(async () => { await forceStop; AbortTestExecThread(); })` | Offload blocking device I/O from the request thread | *`Task.Run(...).Wait()` — what's wrong with that?* (blocks a thread-pool thread, deadlock risk with a sync context, hides exceptions in `AggregateException`) |
| **Thread** | `_testExecutionThread = new Thread(delegate () { TestExecutionEngine(...); }); .Start()`; streamer `m_LSADCthread` | Long-running, abortable, non-pool work | *Task vs Thread — why a raw Thread for the runner?* |
| **CancellationToken** | **Not used.** Cancellation is `Thread.Abort()` + `bool` flags (`StopRunSequence`, `IsThreadForceStopped`). | — | *How would you modernise cancellation?* — your best "what I'd improve" answer |
| **lock / synchronization** | `GrlEthernetLink_C2`: `private readonly object m_ReadWriteLock`, `m_ReadAPILock`; `lock(m_ReadAPILock){…}` around every socket write/read; `volatile bool isIvoked` | One socket, many callers | *Why two lock objects? Any deadlock risk?* |
| **Exception handling** | `try/catch(Exception ex){ WriteToDebugLogger(...) }` nearly everywhere; `PDValidationHelper.LogExceptionIntoDebugLogger(ex)`; `throw ex;` in `ModuleManager.LoadModule` | Keep a long run alive when one test blows up | *Catching `Exception` everywhere — defend it.* (Answer: acceptable at the *test boundary* so one bad test doesn't kill a 4-hour regression, wrong everywhere else; and `throw ex;` resets the stack trace — should be `throw;`) |
| **IDisposable / finalizer** | `GrlEthernetLink_C2` has `public void Dispose()` **and** `~GrlEthernetLink_C2()` but does **not** implement `IDisposable`; `using(var client = new NamedPipeClientStream(...))` in `TBTFactory`; `using(StreamReader)` in `FirmwareUpdateManager.GetDataFromFile` | | *Spot the bug.* — no `IDisposable`, so `using` can't be applied; finalizer touches managed objects; no `GC.SuppressFinalize`. Great senior discussion. |
| **Extension methods** | `ControllerHelper.ExtensionMethods.DeepCopy<T>`; `AppFrmWrk\Extension.cs`; `GrlPdLib\LibExtension.cs`; `ModuleExtension.cs` | Add helpers without touching sealed/shared types | *When is an extension method the wrong choice?* |
| **Configuration** | `PropertyBag` + `PropertyItem` enum + per-MOI config classes; `appsettings.json`; `AppConfig.cfg` read by `ModuleManager.Initialize` to relocate the module directory | | *How does a tester change a timeout without a rebuild?* |
| **Logging** | `Host.WriteToDebugLogger(...)` + per-run log files + hex dump of every frame | | *What do you log on a failure so you can debug it a week later?* |
| **Serialization** | `Newtonsoft.Json` (`JsonConvert.SerializeObject` for UI events and IPC), `DataContractSerializer` (`TestCaseDetails`), `BinaryFormatter` (`DeepCopy<T>`, offline result files) | | *`BinaryFormatter` is obsolete and a security risk — what would you replace it with?* (System.Text.Json / protobuf / an explicit `Clone()`) |
| **File handling** | `Directory.CreateDirectory(RunDirectory)`, `File.Copy(BOOT.BIN → SD card)`, `DirectoryCopy(...)` recursive, `FileStream` + `BinaryFormatter` for captures, `DriveInfo.GetDrives()` | | *How are run artefacts organised?* Project → Run_Rep_timestamp → MOI → test. |
| **Threading + producer/consumer** | `DataStreamService` streamer thread producing CC-line packets and ADC samples consumed by the validation code | | *How do you avoid tearing when the test reads the packet list while the streamer writes it?* |
| **State machine (in C#)** | `GrlPdLib\AMS validation\PDStateMachine.cs` — `ExpectedMsgSequence(PDMessage)` returns every legal continuation of an AMS; `AtomicMessageSequence` carries `AMSStatus`; `USB_PD_TesterLib\RunScript\BaseClass\CStateOperation.cs` for the visual step engine | Model the *spec*, then diff reality against it | *How do you validate a protocol sequence without hard-coding one golden path?* |
| **Interop / P-Invoke** | `visa32.cs`, `visa64.cs`, `libusbK.cs` | Instrument control (VISA) and USB driver access | *Have you done native interop?* — yes, consumed. |
| **WMI** | `ManagementObjectSearcher("SELECT * FROM Win32_PnPEntity")` in `FirmwareUpdateManager.GetComportfromVID` — finds the E-Load COM port by VID `0x2341` / PID `0x003D` | Identify the right COM port deterministically | *How do you avoid hard-coding COM5?* Strong, concrete answer. |
| **IPC** | `NamedPipeWrapper` for UI events; `NamedPipeClientStream("FourPortPipe")` in `TBTFactory` with JSON over a `StreamReader`/`StreamWriter` | Run a test in a separate process and stream results back | *Why a separate process for TBT four-port tests?* (separate controller instances / isolation) |
| **DI** | Only ASP.NET Core's built-in container in `Startup.ConfigureServices` (`AddMvc`, `AddCors`, `AddSwaggerGen`). Application objects use a **manual singleton** (`ControllerHelper.GetControllerHelper`) and static state (`ModuleHelper`, `FWAppObj.FWHost`). | | *There's no DI in the core — is that a problem?* (Yes for testability; but there is exactly one physical tester so a singleton is defensible. The fix: an `ITesterLink` interface + constructor injection so tests can be mocked — `Moq` is already referenced in `XUnit_TestFramework`.) |

---

# D. Design patterns actually present

## 1. Template Method — `TestCase`
- **Where:** `SampleModule\TestCase.cs`, `USBPD_2_TestCase`, `FunctionalTests`.
- **Why:** every test has the same lifecycle (gate → configure device → run → validate capture → clean up) but different content.
- **How:** base declares `abstract Configure()`/`Validate()`, provides `virtual PreConfigure()` and concrete `StartStreamer/StopStreamer`; subclasses call `base.Configure()` / `base.Validate()` to layer on shared work.
- **Advantages:** ~700 tests with one lifecycle; the engine is trivially generic.
- **Disadvantages:** deep inheritance; base-class changes ripple; a subclass that forgets `base.Validate()` silently skips the common checks.
- **Alternative:** composition — inject an `ITestStep` pipeline, or a strategy object per phase. Worth saying you'd move to composition if the hierarchy grew a fourth level.
- **Questions:** *Why not an interface? What happens if `base.PreConfigure()` isn't called? How do you enforce the contract?*

## 2. Abstract Factory / Factory — per-MOI test factories
- **Where:** `ExecuteTestCase_AllMOIs` constructs `PD3_CTS_Factory`, `DeterministicTestEngine`, `FunctionalTestFactory`, `DPAltModeFactory`, `QC_Legacy_Factory`, `TBTClassFactory`, `USB_2_0_Engine`, … each exposing `Execute*()`.
- **Why:** each compliance domain has genuinely different setup, licensing and reporting needs.
- **Disadvantage:** the selection is a long `if/else if` chain on `PDAppMode` — the one place a new MOI requires editing existing code. **Honest improvement:** a `Dictionary<PDAppMode, Func<PD_TesterCommCfg, ITestFactory>>` registry, or attribute-driven discovery like the test classes already use.
- **Questions:** *That if/else chain violates open/closed — how would you fix it?*

## 3. Reflection-based Service Locator / Convention over Configuration
- **Where:** `GetTestCaseObj` + `Activator.CreateInstance`; `ModuleManager` loading `IModule` plugins from `C:\GRL\Libraries`.
- **Why:** adding a test requires no edit to the engine.
- **Disadvantages:** renames break at runtime not compile time; `Assembly.GetTypes()` cost and `ReflectionTypeLoadException`; constructor signature is an implicit contract; `First(...)` throws if two classes match.
- **Alternatives:** source generators, a `[TestCase(ComplianceTestEnum.X)]` attribute scanned once and cached in a `Dictionary<enum, Type>`, or an xUnit/NUnit adapter.
- **Questions:** *What's the startup cost? How would you cache it? How do you catch a typo at build time?*

## 4. Singleton
- **Where:** `ControllerHelper.GetControllerHelper` (classic lazy, **not thread-safe**); `PD_TesterCommCfg.GetTesterCommCfgObject(true)`; `USBPDConfigHelper.GetPDConfigHelper`; `FWAppObj.FWHost` static host.
- **Why:** one physical tester, one engine, one property bag.
- **Disadvantages:** global mutable state, untestable, and the null-check singleton has a race if two HTTP requests hit it concurrently.
- **Fix to mention:** `Lazy<T>` with `LazyThreadSafetyMode.ExecutionAndPublication`, or register as a DI singleton.
- **Questions:** *Show me a thread-safe singleton. Why is `ControllerHelper`'s version risky?*

## 5. Facade
- **Where:** `GrlC2Controller` (partial class, ~9 files) presents one flat, documented API over `PD_Controller`, `PD_Controller_C2Gen2`, `DataStreamService`, `APIValidationHelper`.
- **Why:** external scripts (C# and Python) get a stable surface while the internals churn.
- **Questions:** *How do you keep a public API stable across firmware changes?*

## 6. Adapter
- **Where:** `GrlEthernetLink_C2`, `GrlUsbLink_C2`, `SerialPortComm` all expose the same conceptual `Write(byte[])` / `Read(ref byte[])`; `PD_Controller.ControllerComm` / `ControllerCommSelect` chooses the transport.
- **Why:** the same command API over Ethernet, USB or serial.
- **Honest weakness:** they do **not** implement a common interface — the switch is by enum/static. That's the single highest-value refactor to propose: `interface ITesterLink { bool Write(byte[]); bool Read(ref byte[]); }`, which immediately makes the whole stack mockable.

## 7. Strategy
- **Where:** `WriteFileFactory` / `CSVWriteFileFactory` returning `IWriteFile`; `TemperatureCheckerFactory` returning `IChannelTemperatureChecker`; `SignalClassFactory` picking a waveform format reader (Tek/Lecroy/Keysight/R&S/CSV/GRL).
- **Questions:** *Where would you add a new export format?*

## 8. Composite
- **Where:** `PDTestResult` contains `List<PDTestResult> TestList` and `List<PDSecTest> SecTestList`; `AddTest()` recursively rolls the child verdict up via `UpdateTestResult`.
- **Why:** a test has conditions, which have assertions, which have sub-assertions — and the report needs every level.
- **Questions:** *How is the parent verdict computed? What if a child is INCOMPLETE?*

## 9. Observer / Publish-Subscribe
- **Where:** `UICommClient` named-pipe events; `Host` delegates (`DelegateForceStopEvent`, `DelegateAbortDataRead`) invoked from the comm layer.

## 10. Builder (loosely)
- **Where:** `USBPDConfigHelper` accumulating `MessageGroup_AfterPDC` / `MessageGroup_Consolidated` as a list of `FwMsgCfg` before a single `Execute()`; `SourceCapability.AddFixedSupply(...).AddPpsApdo(...)` style accumulation.

## 11. State
- **Where:** `AppFrameWork.HostState` (READY/BUSY), `TestExecutionState`, `TestRunningStatus`, `GRLExecState`, `EthernetConnectionState`; `RunScript\BaseClass\CStateOperation.cs` for the step engine.

> **Patterns that are NOT here — don't claim them:** Repository, Unit of Work, Mediator, CQRS, DI container for domain objects, proper IoC.

---

# E. Test case structure — worked examples

## E.1 Compliance test — `TEST_PD_EPR_SNK3_9_EPR_Entry_failed_due_to_SourceCap`

| Aspect | Reality |
|---|---|
| **Objective** | UUT is an EPR-capable **Sink**. The tester (as Source) offers a Source_Capabilities set that must cause EPR entry to fail; the UUT must detect this and issue a **Hard Reset**. |
| **Precondition** | `CheckEPRSupportField_VIF` — the UUT's VIF must declare EPR-as-Sink support. Otherwise → `NA`. Also the EPR test fixture must be connected (or the app type must be C2Gen2). |
| **Setup** | Bring-up procedure `COMMON_PROC_BU_2_Bring_up_Sink_UUT`; tester source/sink PDOs set; sub-sampling ratio set; streamer started on the required LSADC channels. |
| **Input / stimulus** | A runtime message group: PDM-mode enable frame + test-ID init + the crafted EPR Source_Capabilities, each with `TxTimer`/`RxExpTimer`. |
| **Device interaction** | `m_objPDCtrl.SetPDMMode(...)` builds the byte buffer; the group is pushed to firmware which replays it on CC at the right times. |
| **Firmware interaction** | Tester firmware drives the CC line, auto-responds GoodCRC, applies configured delays, and timestamps every packet. |
| **Expected result** | 4 assertions: EPR-capable bit set in the RDO; UUT initiates `EPR_Mode Enter`; Source_Capabilities check; UUT sends Hard Reset. |
| **Actual result** | Read from `StrmServ.ObjCClinestreamer_PortA.TotalmsgList` and `GetAMSList_PortA`. |
| **Validation** | `GetDesiredAMS(AMSName.EPR_Mode_Enter)` then helper assertions that write PASS/FAIL + a human description into each `PDTestResult`. |
| **Cleanup** | `StopStreamer_Prerequisite()` → `StopStreamer()`; MOI-level `FW_FPGA_FullReset` before the next test. |
| **Reporting** | Result tree → per-test HTML, MOI folder, run summary; `.grltrace` capture saved and linked. |

## E.2 Functional test — `TD_4_2_1_Source_Connect_Sink_Test`

- **Objective:** a Type-C **Source** UUT must correctly detect and power a Sink across three cable conditions (PD2.0 5A active, PD3.0 5A active, USB4 re-driver) — or one condition for a captive-cable product.
- **Applicability gate (`Configure`)**: `ModuleHelper.DUT_Type_C_State_Machine == Type_C_State_Machine.SRC`. If the VIF isn't loaded → `INCOMPLETE` with the VIF error text; if the state machine is wrong → `NA` with *"Expected state: SRC. Obtained state: X"*. **This is exactly what an interviewer means by "good test hygiene".**
- **Iteration:** `IterateForCableCondition()` emulates each cable in turn; each iteration is bracketed by an injected `MsgType.Test_Execution_State` assertion marker so `Validate()` can slice the capture per condition via `Get_Assertion_Message(...)` and global packet indices.
- **Extra validation:** `ValidateHighSpeedLine()` checks USB 2.0/3.x enumeration (`bEnDataLinesValidation`) when the VIF says the port carries data.
- **Result shape:** one `PDTestResult` per cable condition added to the test's result object.

## E.3 Offline/unit test — `XUnit_TestFramework`

`TEST.PD.EPR.SNK3.10\ValidateGetPDMsgFromAMS.cs`

```csharp
// Arrange — load a recorded capture (JSON) into the real streamer/validation objects
UpdateTestDetails(@"Json_Data\Power_Negotiation_EPR_Source_Capabilities_Hard_Reset_AMS.json");
UpdatedAMS(AMSName.Power_Negotiation, amsStatusComplete: true, isTestInitiated: true);

// Act — run the production API under test
for (int i = 0; i < currentAMS.ExpAMS.AmsElement.Count; i++)
    expectedMsgTypeList.Add(helper.pdValidationHelper.GetPDMsgFromAMS(
        baseTestResult, passingMessageList[i], currentAMS, iMsgInd));

// Assert
Assert.Equal(MsgType.SourceCap, expectedMsgTypeList[0].msgType);
Assert.Equal(MsgType.Request,   expectedMsgTypeList[1].msgType);
Assert.Equal(MsgType.Accept,    expectedMsgTypeList[2].msgType);
Assert.Equal(MsgType.PS_RDY,    expectedMsgTypeList[3].msgType);
```

There are negative fixtures too — `EPR_Entry_Process_Should_Fail.json`, `PR_SWAP_NoResponse.json`, `UUT_Not_sent_EPR_Mode_Enter.json`, `Power_PARTIAL_TESTER_PACKET_MISSING.json` — and the corresponding assertion is `Assert.Equal(TestResult.FAIL, baseTestResult.TestResult)`. **Golden-file regression testing of the validation engine.** Use this whenever asked "how do you test your test framework?".

Worth noting honestly: `ExecuteTestsSequentially` runs nine logical checks inside a **single** `[Fact]`, so a failure reports one red test. The improvement is `[Theory]` + `[MemberData]`, giving one result per case. Volunteer that.

---

## F. Quick file map for the interview

| If they ask about… | Open this |
|---|---|
| Framework core / plugin loading | `AppFrmWrk\AppFrmWrk\ModuleManager.cs`, `IModule.cs` |
| Test base class | `USBPDModule\SampleModule\TestCase.cs` |
| Test discovery + execution | `USBPD_Rev2\USB_2_0_Engine.cs` (lines 41–330, 1139) |
| Run orchestration, repeat, force-stop | `SampleModule\USB_PD_Engine.cs` (210–520, 774–1330) |
| Transport / retries / locking | `USB_PD_TesterLib\GrlEthernetLink_C2.cs` |
| Command framing | `USB_PD_TesterLib\PD_Controller.cs` (`WriteReg` 2825, `ReadReg` 2913, `ModifyAPItoSuppPortB` 2934) |
| Protocol state model | `GrlPdLib\AMS validation\PDStateMachine.cs` |
| Timing/spec assertions | `Grl.APIValidation\APIValidationHelper.cs` (`tSenderResponse`, `tProtErrSoftHardReset`) |
| Result model | `TestCaseHelper\PD test results\PDTestResult.cs`, `TestCaseDetails.cs` |
| Firmware update | `SampleModule\FirmwareUpdateManager.cs` |
| REST surface | `C2WebApp\...\Controllers\TestConfigurationController.cs` |
| Offline regression | `XUnit_TestFramework\**` |
| UI regression | `Selenium\C2WebAppTest.py` |
