# Phase 4 — How the Two Repositories Connect

## The full path

```
C# TEST CASE                          USBPDModule\...\TEST_PD_*.cs / TD_4_*.cs
   │  Configure()
   ▼
TEST/CONFIG HELPER                    USBPDConfigHelper, Bring_Up_Procedures
   │  builds List<FwMsgCfg> (header+payload, TxTimer, RxExpTimer)
   ▼
DEVICE API                            GrlUsbPdControllerLib.PD_Controller
   │  SetPDMMode / SendCommand / SetControllerConfiguration / ConfigureResponseDelay
   │  → WriteReg(reqCode 0xD5 = config, 0xDC = runtime, byte[] dataBuffer)
   │  → ModifyAPItoSuppPortB()  : bit7 of cmd byte selects Port A / Port B
   ▼
FRAME LOGGER                          PacketLogBulkLP.WriteFWCommand / ReadFWCommand
   │  (also writes the hex dump + API description to the debug log)
   ▼
TRANSPORT                             GrlEthernetLink_C2
   │  3 × TcpClient to 192.168.255.1 : 5002
   │    m_WriteClient/m_WriteStream   → commands out
   │    m_ReadClient/m_ReadStream     → command responses
   │    m_ReadData/m_ReadDataStream   → bulk sample stream (DMA)
   │  lock(m_ReadAPILock) / lock(m_ReadWriteLock); NoDelay = true; 5 s timeouts
   │  1024-byte command frames; read loop with 100 retries; retry+reconnect on failure
   ▼
═══════════════ Ethernet ═══════════════
   ▼
INSTRUMENT FIRMWARE (repo 2)          xsdk/ethernetapp  →  xsdk/usb_pd_c2_gen2 (FreeRTOS)
   │  UART/socket byte stream → xTaskNotifyFromISR(xAppCmdHandle, UARTINTR)
   │  vAppCmdHandleTask → UartCmdHandler() → command dispatch
   │  UsbPd_FWControl.c / UsbPDMain.c apply the configuration
   ▼
FPGA (PL)                             AXI-Lite / AXI4-Burst registers
   │  UsbPDRegister.c: ReadAxiReg/WriteAxiReg
   │  BMC encoder/decoder, CRC, preamble, SOP*, sample capture, DMA
   ▼
ANALOG FRONT END                      Rp/Rd switching, VBUS/VCONN sourcing,
   │                                  E-Load (TI C2000), PPS (TI C2000) over I2C
   ▼
══════ USB Type-C cable: CC1/CC2, VBUS, VCONN, D+/D-, SBU ══════
   ▼
UUT / DUT                             charger, laptop, hub, phone, cable
   │  responds per USB-PD / Type-C spec
   ▼  (CC line activity + VBUS/VCONN analog)
FPGA SAMPLER + DMA                    packets + LSADC/HSADC samples into DMA buffers
   ▼  (host polls the data socket)
STREAMER                              DataStreamService (m_LSADCthread)
   │  ObjCClinestreamer_PortA / PortB, StreamerData, CClineStreamerData
   ▼
DECODER                               GrlPdLib.PDmessageDecodeEngine + DecodePayload
   │  → List<PDMessage> (Header, PdoByteList, sopType, msgType, IsTesterMsg,
   │     dStartTime, dStopTime, ActualGlobalIndex, MsgDesc)
   ▼
SEQUENCE BUILDER                      GrlPdLib.AMS_validation.AMSGenerator
   │  → List<AtomicMessageSequence>  (via PDStateMachine.ExpectedMsgSequence)
   ▼
VALIDATION                            testScriptObject.Validate()
   │  + PDValidationHelper / EPRSnk_Helper / ProtocolHelper / PowerHelper
   │  + Common_Check_PD_1..15, COMMON_PROC_PD_1..18
   │  + timing limits from PDSpecLimits / APIValidationHelper
   ▼
RESULT TREE                           PDTestResult (Composite) → TestCaseDetails
   ▼
REPORTING                             Generate_GRL_Report_Module → HTML / CSV / PDF / XML
   │                                  + .grltrace capture file per test
   ▼
UI / CONSUMER                         UICommClient (named pipe, JSON) → React UI
                                      /Report static files, ResultsController, ReportsGenerationController
```

### Command framing detail (the bit interviewers like)

`PD_Controller.WriteReg(byte reqCode, byte devAddress, byte[] dataBuffer, int bytesCount, string sApiDesc)`

- `reqCode == 0xD5` → **configuration** group ("BlkLpAC")
- `reqCode == 0xDC` → **runtime** group ("BlkLpRT"), passed through `UpdateRuntimeCommandBuffer()` first
- otherwise → `GrlUsbLink.WriteSPI(...)` (the USB/SPI path)

`ModifyAPItoSuppPortB(ref byte[] data)` rewrites the first byte to select the port:

```csharp
if (data[0] >= 0x00 && data[0] <= 0x7B)
    data[0] = PortB ? (byte)(data[0] | 0x80)      // set bit 7  → Port B
                    : (byte)(data[0] & 0x7F);     // clear bit 7 → Port A
else if (data[0] >= 0xE0 && data[0] <= 0xEF)      // extended/I2C-tunnelled commands
    prepend { 0xFC, PortB ? 0x80 : 0x00 }          // wrap in an envelope
else if (data[0] == 0xF0)
    prepend { 0xFE, PortB ? 0x80 : 0x00 }
```

And the key flag `IsTesterCommEn`:

```csharp
if (IsTesterCommEn == true)  operationStatus = objpacketlog.WriteFWCommand(dataBuffer, …);
else                         CopyBufferforExternalcall(dataBuffer, sApiDesc);   // capture, don't send
```

> **Why this matters:** when `IsTesterCommEn == false`, the same API that would send a command instead **captures the byte buffer** into `m_databuffer_PDC`. Test cases use this to *compose* a command sequence (e.g. `SetPDMMode(...)` then `VDMCfg.MsgHdrPld.AddRange(m_objPDCtrl.m_databuffer_PDC)`) which is then uploaded as a **message group** and replayed by firmware with hardware-accurate timing. This is how microsecond-level protocol timing is achieved from a Windows host that has no real-time guarantees. **Say this in the interview** — it is the single most senior-sounding thing in the codebase.

---

# Three real end-to-end flows

## Flow 1 — Connect and identify the instrument

**Trigger:** user clicks *Connect* in the React UI → `ConnectionSetupController` → `USB_PD_Engine.InitializeC2HWComm(ipAddress)`.

```
1  PD_Controller.UpdateGRLControllerConfig(idx, ip)
       → m_ControllerLst[idx] = new GrlEthernetLink_C2(ip)     // Dictionary<int, link>: multi-controller
2  ModuleHelper.InitHWCommunication(ControllerCommSelect.C2_Ethernet_Comm, ip)
       → GrlEthernetLink_C2.InitilizePort()
             EthernetDiscovery() → enumerate NICs, skip TAP/VMware/Virtual, find the real IP
             new TcpClient(ip, 5002) × 3;  NoDelay = true;  Receive/SendTimeout = 5000
3  objTester.SetStreamerBufferSize(DMA_Mem_Size62k)
4  objTester.ReadControllerVersion()
       byte[] cmd = { 0xE1, 0x02 };  ReadReg(ref cmd)
       → PacketLogBulkLP.ReadFWCommand → Ethernet Write then Read
       → firmware returns an ASCII version string → PD_Controller.FwVersion
5  objTester.GetApplicationStatus()          // C2 vs C2Gen2
6  (Gen2) objTesterGen2.SystemOnTimeUpdate() // push host date/time to the instrument
7  objTester.Reset_Controller_DefaultState_WithoutPowerCycle()
8  objTester.Detect_Noise_Eload()            // which ports have an E-Load, is the noise board fitted
9  objTester.GetControllerSerialNo()  /  GetSerialNumber() if FRAMRev >= 8
10 objTester.ReadEloadVersion()   { 0xE1, 0x04 }
   objTester.ReadPPSFWVersion()   { 0xE1, 0xD2 }   ← tunnelled over I2C to the C2000 MCUs
11 PropertyBag.UpdateProperty(CS_TesterStatus / CS_FirmwareVersion / CS_EloadVersion /
                              CS_SerialNumber / CS_C2IpAddress / CS_LicenseList)
12 ModuleHelper.FWversionObt compared with ModuleHelper.FWversion at execution start;
   mismatch → modal "You are using incompatible firmware… proceed?"
```

**Firmware side:** the `0xE1,0xD2` read is a host command that the Zynq app turns into an **I2C transaction with the PPS C2000 at slave address 0x0E**, whose SSBL answers `GET / GET_FW_VERSION` with `I2C_TX_Buffer[0]=0x05, [1]=0x01,` then the version bytes. Only 9 bytes are read *"as per the I2C limitations"*.

**What this flow demonstrates in interview terms:** device discovery, version handshaking, capability probing, transport initialisation, and a firmware-compatibility gate before any test runs.

---

## Flow 2 — Run one compliance test (PD-merged CTS)

**Example:** `TEST_PD_EPR_SNK3_9_EPR_Entry_failed_due_to_SourceCap`

```
HOST                                                    FIRMWARE / HARDWARE            UUT
────────────────────────────────────────────────────────────────────────────────────────────
POST /PostTestListToExecute/{n}
  TestConfigurationController → USB_PD_Engine.StartTestExecutionThread
  new Thread(TestExecutionEngine) .Start()
  → RunSelectedTestCases → ExecuteTestCase_AllMOIs
  → moi == PD_Merged → new USB_2_0_Engine().ExecuteTestCases(report)

  CTSStart():
    ConfigureMessageFilter(msgCategories) Port A + Port B  ──► firmware sets capture filter
    UpdateDUTDetails()  → do a PDC (PD contract) and read the UUT's PDOs/caps ─────► negotiates

  per selected test:
    Type t = assembly.GetTypes().First(name == enum)
    TestCase tc = (TestCase)Activator.CreateInstance(t, testObj, cfgHelper, valHelper)

    tc.PreConfigure()
      CheckEPRSupportField_VIF → if UUT VIF says no EPR-Sink → result NA, skip
      SetSamplingRatio / SetDefaultProperties(bringUp, srcPowerProfile)
      SetTester_Source_Sink_CapabilitiesPDO / ConfigureDefaultProperties
      ConfigureTestSpecificTesterBehaviour(enum)

    StartOfTestCaseExecution(testObj)
      detach CC lines ─────────────────────────────────────────► CC detach            sees detach
      ResetLSADC_CCLine_Data(); Start_Service(channels…) ──────► start DMA streamer

    tc.Configure()
      IsTesterCommEn = false                       // compose, don't send
      SetPDMMode(true, enableEPR) → m_databuffer_PDC
      FwMsgCfg{ MsgHdrPld, TxTimer=50, RxExpTimer=50 } → MessageGroup_AfterPDC
      AddTestInitToGrpCmdPhyTest(TestId=32, captiveCable)
      base.Configure(): build per-PD-mode buffers (PD2/PD3, chunked/unchunked)
      → upload MessageGroup_Consolidated (reqCode 0xDC) ───────► firmware stores the group

    Execute()  ──────────────────────────────────────────────► firmware replays the group
                                                                on CC with exact timing  ──► UUT
                                                                ◄── UUT responses           ◄──
    PacketInitStatus(msgCount, iMaxWaitTimer_Sec) ◄──────────── firmware reports "all
                                                                commands initiated"
    WaitforHSADCCapture(testObj, expectedCaptureCount)

    tc.Validate()
      TotalmsgList  ← decoded PDMessage list from the streamer
      GetAMSList_PortA ← AtomicMessageSequences built by AMSGenerator/PDStateMachine
      GetDesiredAMS(AMSName.EPR_Mode_Enter) → the AMS to assert on
      4 × PDTestResult assertions (PASS/FAIL/NA + description + packet index)

    ValidateCommonCheckandCommonProc(testObj)     // the 15 common checks + 18 procedures
    PostValidate(testObj, results, bFWExecAllCmds)
    EndOfTestcaseExecution → StopStreamer, save .grltrace, add packets to results table
    PopulateResultHtmlFile(testObj)
    log: "Overall time X-Y / Config time / Validate time"

  CTSStop(); GenerateReportData_InsideMoiFolder(); GenerateReport_ParentFolder()
  finally: SetPDMMode(false,false); IsTesterCommEn = true; HostState = READY
```

**Point to make:** the host never does real-time work. It *compiles* a behaviour, uploads it, and the FreeRTOS + FPGA firmware executes it deterministically. The host then does the (non-real-time) heavy lifting of decoding and validating. Correct division of responsibility between a PC and an embedded target.

---

## Flow 3 — Update the E-Load / PPS firmware from C#

```
HOST (FirmwareUpdateManager)                    I2C via Zynq              TI C2000 SSBL
──────────────────────────────────────────────────────────────────────────────────────
Update(filePath, Firmware_Selection.PPS)
  GetDataFromFile()  → List<byte>  (ASCII-hex text file, whitespace separated)
  if empty → log "Data read from file is incorrect", return false

  Initialize(PPS):
    Get_SSBL_Program_Mode(Check_Program_Mode.PPS) ──► 0x55,0x02 GET_MODE ──► bootmode_check()
                                                   ◄── 0xFF/0xEE ◄── flash boot-mode flag
    if Boot:
       Firmware_Program_Selection(PPS, Program)   ──► 0x01,0x02 SET_PGM  ──► bootmode_update(PGM)
       Firmware_Update_Reset(PPS)                 ──► 0x01,0x03 SW_RESET ──► SysCtl_resetDevice()
       wait 2000 ms
       Firmware_Flash_Erase(PPS)                  ──► ERASE_SECTOR       ──► Flash_sector_erase()
       wait 2000 ms
       re-read GET_MODE; must be Program, else FAIL
    if Program:
       reset → erase → ready

  DataUpdate(data, PPS):                          // 256-byte chunks
    do {
       if (StopFWUpdate) break;                   // user abort
       tempByte = data[0..255]
       retVal = Firmware_Update(tempByte, PPS)    ──► APP_DATA frames    ──► key check 0x08AA
                                                                            → copy_data()
                                                                            → Flash_data_program()
       report (DataSentBytes/TotalBytes)% to the UI spinner
       wait 100 ms
       data.RemoveRange(0,256)
       if (!retVal) break;                        // stop on first failed chunk
       DataSentBytes += 256
    } while (data.Count > 256)
    send the tail chunk

  verify:  Firmware_SSBL_Version(PPS) ×2  ──► 0x55,0x01 GET_FW_VERSION ──► "P_3_0" + status byte
           (V3.0: byte[5]=success, byte[6]=write error → retry the write)
  activate: Firmware_Program_Selection(PPS, Boot) ──► 0x01,0x01 SET_BOOT ──► bootmode_update(BOOT)
  on failure path: still read version, still force SET_BOOT → leave the device bootable
```

**Zynq image path (same class, different mechanism):**
`Access_SD_Card` → instrument exposes SD over USB → poll `DriveInfo` ≤30× → copy `BOOT.BIN` + `image.ub` + `start.sh` → `Disable_SD_Card` → `FW_FPGA_FullReset` → `Reset_Controller()` → mark Ethernet as errored → wait → user informed to wait for reboot.

**What to say:** *"That's an in-system firmware update driven end-to-end from C#: mode transition, erase, chunked transfer with per-chunk failure handling and progress reporting, version read-back verification, activation, and a defined recovery state. Swap I2C for a BLE GATT characteristic and it's the same test matrix."*

---

## Where the two repos touch, precisely

| Host-side C# | Wire | Firmware-side C |
|---|---|---|
| `GrlEthernetLink_C2.Write/Read/ReadDataStreamer` | TCP :5002 | `ethernetapp` socket handling → `vAppCmdHandleTask` |
| `PD_Controller.ReadReg({0xE1,0x02})` | | version register read |
| `PD_Controller.SetControllerConfiguration(ControllerConfigEnum.FW_FPGA_FullReset)` | | FPGA + FW reset path |
| `SetPDMMode`, `ConfigureResponseDelay`, message groups (`0xD5` config, `0xDC` runtime) | | `UsbPd_FWControl.c`, `usb_pdm_protocol_tests.c` — firmware-side CTS behaviours |
| `ConfigurePDInterrupts(bool)` | | GIC/GPIO interrupt enable |
| `ReadPPSFWVersion({0xE1,0xD2})`, `Firmware_Update`, `Get_SSBL_Program_Mode` | I2C tunnel | `ccs/SSBL/*/main.c` `Command_Handler()` |
| `DataStreamService` polling the data socket | DMA | `UsbPDDMA`, FPGA sample FIFO |
| `PDmessageDecodeEngine` | | mirrors `UsbPDPhyLayer.c` framing/CRC |
| `PDStateMachine` expected AMS | | mirrors `UsbPDProtocolLayer.c` PRL state machines |
| `ModuleHelper.FWversion` compatibility gate | | `Bootloader_ver` / FW version strings |

> The two repos are **two halves of one protocol implementation**: the firmware implements it for real; the C# implements the spec's *expectations* and diffs them against reality.
