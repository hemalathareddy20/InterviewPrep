# Phase 3 + Phase 7 — Repository 2: `D:\GitHub\fw_c2_gen2`

> **Important framing:** this is the firmware of the **tester** (the instrument), not of the DUT. When you talk about it, say *"the firmware of the instrument I drive from C#"*. That is still firmware you read, debug against, and whose version/behaviour your automation depends on.

---

## 1. Layout

```
fw_c2_gen2/
├─ docs/                      (empty — .gitkeep only)
└─ sources/
   ├─ xsdk/                   Xilinx SDK projects — Zynq-7000 (ARM Cortex-A9, ps7_cortexa9_0)
   │  ├─ FSBL/                First Stage Boot Loader
   │  ├─ FSBL_BSP/            Board support package (standalone_v5_5, scugic, xilffs, AXI_LITE, AXI4_BURST)
   │  ├─ usb_pd_c2_gen2/      ★ the main FreeRTOS USB-PD application
   │  ├─ ethernetapp/         Ethernet/host-command application
   │  ├─ emulatorapp/         device-emulation application
   │  ├─ broadcastapp/        multi-port broadcast application
   │  └─ displayapp/          front-panel display + touch (gslX680) + XADC
   ├─ ccs/SSBL/               TI Code Composer Studio — C2000 F2837xD secondary bootloaders
   │  ├─ PPS/                 Programmable Power Supply MCU bootloader
   │  └─ ELOAD/               Electronic Load MCU bootloader   (same source, ELOAD macro)
   └─ petalinux/              (empty placeholder)
```

Multiple processors in one instrument:

| Processor | Role | Firmware |
|---|---|---|
| **Zynq-7000 PS** (Cortex-A9) | USB-PD protocol engine, host command handling, timers | `xsdk/usb_pd_c2_gen2` on **FreeRTOS** |
| **Zynq-7000 PL** (FPGA fabric) | BMC PHY, packet framing/CRC, sampling, DMA | RTL (not in this repo); accessed by C via AXI-Lite / AXI4-Burst registers |
| **TI C2000 F2837xD ×2** | E-Load and PPS analog control loops | application + **SSBL** in `ccs/SSBL` |

---

## 2. Main application — `xsdk/usb_pd_c2_gen2/src`

Largest files: `UsbPDMain.c` (712 KB), `usb_pdm_protocol_tests.c` (583 KB), `USBPDFunctional2.c` (481 KB), `UsbPDProtocolLayer.c` (417 KB), `UsbPDPhyLayer.c` (320 KB), `DP_Alt_Mode.c` (276 KB), `UsbPd_HWControl.c` (273 KB), `USB_PD_Interrupts.c` (257 KB), `USB_PD_EPR.c` (195 KB), `UsbPd_FWControl.c` (193 KB), `usb_pdm_common.c` (178 KB), `VDM_Handler.c` (151 KB), `USB_PD_Timers.c` (98 KB), `QCLegacy.c` (222 KB).

| Module | Responsibility |
|---|---|
| `UsbPDMain.c` | `main()`, hardware setup, FreeRTOS objects, tasks, ISR bodies, command dispatch |
| `UsbPDPhyLayer.c/.h` | BMC physical layer: preamble, SOP*, framing, CRC, bit timing, BIST |
| `UsbPDProtocolLayer.c/.h` | PRL state machines (TX/RX/Hard-Reset/BIST), message IDs, retries |
| `USB_PD_Timers.c/.h` | All spec timers (`Timer_tNoResponse`, `Timer_tSenderResponse`, `Timer_tSinkWaitCap`, `Timer_tTypeCSendSourceCap`, …) |
| `USB_PD_Interrupts.c` | GPIO / FPGA IRQ handling, packet RX/TX completion |
| `UsbPd_HWControl.c/.h`, `UsbPDRegister.c/.h`, `UsbPDAxiRegister.h` | FPGA register map, Rp/Rd, VBUS/VCONN switching, eLoad control |
| `UsbPDDMA.h` | DMA descriptors for the sample streamer |
| `UsbI2C.c/.h` | I2C master — talks to E-Load/PPS MCUs, FRAM, sensors |
| `UsbSPIGpio.c/.h` | SPI + GPIO |
| `VDM_Handler.c` | Vendor Defined Messages, Discover Identity/SVIDs/Modes |
| `DP_Alt_Mode.c`, `UsbDpAux.c` | DisplayPort Alt Mode + AUX |
| `USB_PD_EPR.c` | Extended Power Range (28–48 V) mode |
| `QCLegacy.c` | Qualcomm Quick Charge legacy emulation |
| `usb_pdm_protocol_tests.c`, `USBPDTest_Auto.c` | **Firmware-side implementations of CTS test behaviours** (the "FW implementation" path your C# selects via `PDMCurrentTestFWImpl`) |
| `USBPDEmulator.c` | Charger/device emulation |
| `PDDataStruct.c/.h` (68 KB header) | The global per-port state structure `gUsbPDStruct` |
| `RTOSWrapper.c/.h` | FreeRTOS includes, delay helpers, hand-written `memcpy`/`XMemSet`/`XMemCopy` (loop-unrolled) |

---

## 3. RTOS — FreeRTOS, confirmed

`RTOSWrapper.h`:
```c
#include "FreeRTOS.h"
#include "task.h"
#include "timers.h"
#include "semphr.h"
#include "event_groups.h"
#include "queue.h"
```

### 3.1 Startup sequence — `UsbPDMain.c:14992`

```c
int main(void)
{
    prvSetupHardware();      // GIC (XScuGic), SPI (XSpiPs), GPIO, clocks
    InitFwInstances();       // zero/initialise all per-port structures
    InitPDApplication();     // PD application state
    ApplicationDefine();     // CreateSoftwareTimers(); CreateSemaphores(); CreateTasks();
    vTaskStartScheduler();   // never returns
    for(;;);                 // only reached if the FreeRTOS heap was too small
}
```

### 3.2 Tasks — `CreateTasks()` at `UsbPDMain.c:14891`

| Task | Entry | Priority | Stack | Purpose |
|---|---|---|---|---|
| `"GpioInterrupt"` | `vInterruptHandlerTask` | `tskIDLE_PRIORITY + 7` | `0x8000` | Deferred handling of PD/GPIO/QC interrupts |
| `"CommandThread"` | `vAppCmdHandleTask` | `tskIDLE_PRIORITY + 6` | `0x4000` | Handles host commands arriving over UART |
| `"PdTimerTask"` | `vPdTimerTask` | (conditional `TIMER0_TASK`) | `0x8000` | PD protocol timer expiry |
| `"Timer1Thread"` | `vApplicationTimerHandle1Task` | (conditional `TIMER1_TASK`) | `0x8000` | Secondary timer |
| `"RpMsg"` | `vProcessingTask` | `#if 0` — disabled | | OpenAMP inter-processor messaging (superseded by shared OCM memory) |

### 3.3 ISR → task signalling: **task notifications**, not semaphores

This is the interesting engineering choice. The semaphore path exists but is commented out; the live path is direct-to-task notification, which is the fastest and lightest FreeRTOS signalling primitive.

```c
/* ISR (UsbPDMain.c:14440) */
xTaskNotifyFromISR( xIntrHandle, GPIOINTR_PORT0, eSetBits, &xHigherPriorityTaskWoken );
/* … also xTaskNotifyFromISR(xAppCmdHandle, UARTINTR, …) at 14374/14664,
   and xTaskNotifyFromISR(xPdTimerHandle, …) at 13098/13133 */

/* Task (UsbPDMain.c:13884) */
for (;;)
{
    xResult = xTaskNotifyWait( pdFALSE,        /* don't clear bits on entry  */
                               ULONG_MAX,      /* clear all bits on exit     */
                               &ulNotifiedValue,
                               pdMS_TO_TICKS(500) );   /* bounded wait       */
    if (xResult == pdPASS)
    {
        if (ulNotifiedValue & GPIOINTR_PORT0) { … Interrupt_Handler(PORT0); }
        else if (ulNotifiedValue & GPIOINTR_PORT1) { … Interrupt_Handler(PORT1); }
        if (ulNotifiedValue & GPIO_QC_INTR)  { … QuickCharge handling … }
    }
}
```

Notification bits defined in `RTOSWrapper.h`:
```c
#define TIMER0_PORT0   0x01
#define TIMER0_PORT1   0x02
#define GPIOINTR_PORT0 0x01
#define GPIOINTR_PORT1 0x02
#define GPIO_QC_INTR   0x04
#define UARTINTR       0x04
```

**Interview gold:** this is the textbook *deferred interrupt handling* pattern — ISR does the minimum and defers to a high-priority task. And the `xMaxBlockTime = 500 ms` means the task also wakes periodically even without an event, so it can't hang forever on a lost notification.

### 3.4 Mutexes — `CreateSemaphores()` at `UsbPDMain.c:14800`

```c
SemaphoreHandle_t xsrcPwrupSrcCapMutex;   // source power-up / Source_Capabilities
SemaphoreHandle_t xprSwapMutex;           // power-role swap
SemaphoreHandle_t xmsgTimerMutexPort0/1;  // per-port message timer
SemaphoreHandle_t xmsgTimerMutex0/1;
SemaphoreHandle_t xPdTxnTimerMutex0/1;    // PD transaction timer
SemaphoreHandle_t xAMSstatus;             // AMS state
SemaphoreHandle_t xbistTestMutex;         // (#if 0)
```
All created with `xSemaphoreCreateMutex()` — i.e. **mutexes with priority inheritance**, not binary semaphores. Binary semaphores are declared (`xGpioIntrSemaBinary`, `xUartIntrSemaBinary`, `xTimer0/1IntrSemaBinary`) but their use is commented out in favour of notifications.

### 3.5 Event groups
```c
EventGroupHandle_t xglFxGpioAppEventGroup, xglAppTimerEventGroup, xglAppTimerEventGroup1;
```
Declared for multi-source waits; the active code uses notification bits instead.

### 3.6 Software timers — `CreateSoftwareTimers()` at `UsbPDMain.c:14684`

Ten `xTimerCreate(...)` calls, two per concern, one per port:
`xglMsgTimerPort0/1`, `xglMsgTimer0/1`, `xglPDTxnMsgTimer0/1`, `xglMiscTimer0/1`, `xglQcTimer0/1`.
These implement the PD spec timers in software; there is also an `FPGA_TIMER` path for hard-real-time deadlines.

### 3.7 Delays
`RTOSWrapper.c`:
- `xThreadSleep(ms)` → `vTaskDelay(ms / portTICK_PERIOD_MS)` — **blocking, yields the CPU**.
- `uSDelay()`, `UserSetDelay()`, `UserSetuSDelay()`, `xBusyWait()` → **empty spin loops on a `volatile` counter** — busy-wait, does not yield.

> **Excellent interview point:** "Why two kinds of delay?" — `vTaskDelay` has tick-granularity (≈1 ms) and reschedules, which is useless for a microsecond-level protocol deadline; the busy-wait gives sub-tick precision at the cost of burning CPU and being clock-frequency dependent. That's a real embedded trade-off you can discuss, plus its consequence: a busy-wait inside a high-priority task can starve everything below it.

---

## 4. Boot chain and firmware update

### 4.1 Zynq boot
`BootROM → FSBL (xsdk/FSBL) → application`

`FSBL/src/` contains `main.c` (38 KB), `image_mover.c` (37 KB), `qspi.c`, `nand.c`, `nor.c`, `sd.c`, `pcap.c` (FPGA bitstream load via PCAP), `md5.c`, `rsa.c`, `fsbl_hooks.c`.
So the FSBL supports **multiple boot devices (QSPI / NAND / NOR / SD)**, **bitstream programming**, and **image integrity (MD5) and authentication (RSA)** hooks. That is the hardware basis for a golden-image / fallback story even though the higher-level rollback policy isn't implemented in this repo.

Field update from the host (`FirmwareUpdateManager.UpdatedFWtoSDcard()`):
1. C# sends `ControllerConfigEnum.Access_SD_Card` → the instrument exposes its SD card as a USB mass-storage device.
2. Host polls `DriveInfo.GetDrives()` up to 30× (re-issuing the command every 5th attempt) until the removable drive is `IsReady`.
3. `File.Copy` of **`BOOT.BIN`** (FSBL + bitstream + app), **`image.ub`** (kernel/rootfs image) and **`start.sh`**.
4. `Disable_SD_Card` → `FW_FPGA_FullReset` → `Reset_Controller()`; the Ethernet link is deliberately marked errored; wait for reboot.

### 4.2 TI C2000 SSBL (secondary bootloader) — the real "OTA-shaped" mechanism

`ccs/SSBL/{PPS,ELOAD}/SSBoot.h` — one source, two builds selected by `#define ELOAD`:

```c
#define BOOT_MODE   1
#define PGM_MODE    0

#define APP_DATA        0x08     /* firmware payload chunk   */
#define WRITE_FLASH     0x09
#define ERASE_SECTOR    0x0A
#define BOOT_CPU2       0x0B
#define SET             0x01
#define GET             0x55
#define SET_BOOT        0x01     /* switch to boot (run app) */
#define SET_PGM         0x02     /* switch to program mode   */
#define SW_RESET        0x03
#define GET_FW_VERSION  0x01
#define GET_MODE        0x02

#ifdef ELOAD
  #define SLAVE_ADDRESS_1  0x1E   /* I2C slave address, e-Load */
#else
  #define SLAVE_ADDRESS_1  0x0E   /* PPS */
#endif
```

`main.c` implements an **I2C-slave FIFO ISR** (`i2cFIFOISR`) that buffers 16-byte frames (2 command bytes + 14 data bytes) and a `Command_Handler()` that dispatches `SET`/`GET`; flash operations use TI's **F021 Flash API** (`Flash_sector_erase`, `Flash_data_program`) with `SeizeFlashPump`/`ReleaseFlashPump`. `copy_data()` verifies a **key value `0x08AA`** before writing. A `Soft_reset()` uses the **watchdog** (`WdRegs.WDCR = 0x28`) to force a reset. Boot mode is a flag persisted in flash (`bootmode_update` / `bootmode_check`).

`ccs/readme.txt` records the update-robustness history:
- **V1.0** — minimal SSBL footprint, can load the application above it.
- **V2.0** — **write protection of the SSBL flash addresses so the bootloader cannot be corrupted**; SSBL version readable over I2C.
- **V3.0** — **success/error status byte on the version-read API: `byte[5]` = success, `byte[6]` = write error, retry the firmware write until success**; shortened version strings (`E_3_0` / `P_3_0`).

That is: a protected bootloader, an explicit mode state machine, verified chunked transfer, version read-back, retry on failure, and a watchdog-based reset. Everything an OTA question is really about.

### 4.3 The host-side update state machine — `FirmwareUpdateManager.Initialize()`

```
GET_MODE ──┬─ Boot ──► SET_PGM ──► Firmware_Update_Reset ──► Flash_Erase ──► re-read GET_MODE
           │                                                        └─ must now be Program, else FAIL
           └─ Program ─► Firmware_Update_Reset ──► Flash_Erase ──► ready
then  DataUpdate(): 256-byte chunks via Firmware_Update(), progress % reported to the UI,
      abort flag StopFWUpdate checked each chunk, break on first failed chunk
then  Firmware_SSBL_Version() ×2 (read-back verify)
then  SET_BOOT  (activate)
```
On a failed `DataUpdate`, the host still reads the version and forces `SET_BOOT` — i.e. an explicit "leave the device in a known state" recovery path.

E-Load path (older C2): erase via a 1200-baud serial port open/close (`EraseSAM()` — the classic Arduino/SAM-BA magic), then shell out to **`bossac.exe`** with `-e -w -b ELOADFW_PORT1.cpp.bin -R`, capturing stdout into the debug log.

---

## 5. State machines — Phase 7

### 5.1 The firmware protocol-layer state machines (`UsbPDProtocolLayer.h:109-159`)

```c
typedef enum USBPDProTxEnum {
    PRL_TX_PHY_LAYER_RESET,
    PRL_TX_WAIT_FOR_MESSAGE_REQUEST,
    PRL_TX_LAYER_RESET_FOR_TRANSMIT,
    PRL_TX_CONSTRUCT_MESSAGE,
    PRL_TX_WAIT_FOR_PHY_RESPONSE,
    PRL_TX_MATCH_MESSAGEID,
    PRL_TX_MESSAGE_SENT,
    PRL_TX_CHECK_RETRYCOUNTER,
    PRL_TX_TRANSMISSION_ERROR,
    PRL_TX_DISCARD_MESSAGE,
} USBPDProTx;

typedef enum USBPDProRxEnum {
    PRL_RX_WAIT_FOR_PHY_MESSAGE,
    PRL_RX_LAYER_RESET_FOR_RECEIVE,
    PRL_RX_SEND_GOODCRC,
    PRL_RX_CHECK_MESSAGEID,
    PRL_RX_STORE_MESSAGEID,
} USBPDProRx;

typedef enum USBPDProHardResetEnum {
    PRL_HR_RESET_LAYER, PRL_HR_INDICATE_HARD_RESET, PRL_HR_REQUEST_HARD_RESET,
    PRL_HR_WAIT_FOR_PHY_HARD_RESET_COMPLETE, PRL_HR_HARD_RESET_REQUESTED,
    PRL_HR_WAIT_FOR_PE_HARD_RESET_COMPLETE, PRL_HR_PE_HARD_RESET_COMPLETE
} USBPDProHardReset;

typedef enum USBPDProBISTTxEnum { PRL_BIST_TX_INIT_PRBS, PRL_BIST_TX_WAIT_TEST_FRAME_REQUEST,
                                  PRL_BIST_TX_TEST_FRAME, PRL_BIST_TX_PHY_RESPONSE } USBPDProBISTTx;
typedef enum USBPDProBISTRxEnum { PRL_BIST_RX_RESET_COUNTER, PRL_BIST_RX_TEST_FRAME,
                                  PRL_BIST_RX_ERROR_COUNT, PRL_BIST_RX_INFORM_POLICY } USBPDProBISTRx;

typedef struct protoStateMachineStruct {
    int USBProtocPreState;
    int USBProtocCurrentState;
    int USBProtocNextState;
} USBPDProtoState;
```

**Note the `Pre / Current / Next` triple** — the firmware keeps the previous state, which is precisely what you need to debug an illegal transition or to recover. Call that out; it shows you read the code, not a textbook.

Text representation of the TX machine:

```
PRL_TX_PHY_LAYER_RESET
  → (reset done)
PRL_TX_WAIT_FOR_MESSAGE_REQUEST
  → (policy engine requests a message)
PRL_TX_CONSTRUCT_MESSAGE        [entry: stamp MessageID, build header+payload, CRC]
  → (handed to PHY)
PRL_TX_WAIT_FOR_PHY_RESPONSE    [timer: tReceive / CRCReceiveTimer]
  ├─ GoodCRC received  → PRL_TX_MATCH_MESSAGEID → PRL_TX_MESSAGE_SENT → (back to WAIT_FOR_MESSAGE_REQUEST)
  ├─ no GoodCRC        → PRL_TX_CHECK_RETRYCOUNTER
  │                         ├─ retries left → PRL_TX_CONSTRUCT_MESSAGE (same MessageID)
  │                         └─ exhausted    → PRL_TX_TRANSMISSION_ERROR → Policy Engine → Soft/Hard Reset
  └─ incoming message  → PRL_TX_DISCARD_MESSAGE   (collision)
Any state + Hard Reset → PRL_HR_RESET_LAYER  (asynchronous, highest precedence)
```

RX machine:
```
PRL_RX_WAIT_FOR_PHY_MESSAGE
  → message received
PRL_RX_SEND_GOODCRC            [must be within tReceive]
  → PRL_RX_CHECK_MESSAGEID
       ├─ same as stored MessageID → duplicate, discard payload (GoodCRC still sent)
       └─ new                       → PRL_RX_STORE_MESSAGEID → pass up to Policy Engine
  → back to WAIT_FOR_PHY_MESSAGE
Soft_Reset received → PRL_RX_LAYER_RESET_FOR_RECEIVE  (MessageIDs cleared)
```

### 5.2 Type-C connection state machine (what the functional tests exercise)

Modelled on the host side by `Type_C_State_Machine` (`SRC` / `SNK` / `DRP` / …) and `DUTConnState` (`Attached` / `Detached`) in `PDDataStruct.h:119-120` (`PreviousDUTConnState`, `DUTConnState`).

```
Disabled / Unattached.SRC
   → (Rd detected on one CC, debounce tCCDebounce)
AttachWait.SRC
   → (stable, VBUS off)                 → Attached.SRC    [entry: turn VBUS on, apply Rp, source VCONN if Ra]
   → (Rd removed)                       → Unattached.SRC
Attached.SRC
   → (Rd removed, tSRCDisconnect)       → Unattached.SRC  [exit: VBUS off, VCONN off]
   → (PR_Swap accepted)                 → Attached.SNK

Unattached.SNK → AttachWait.SNK → Attached.SNK
DRP: Unattached.SRC ⇄ Unattached.SNK toggling with tDRP duty cycle
Try.SRC / TryWait.SNK / Try.SNK / TryWait.SRC — preference states
Accessory: AudioAccessory (Ra/Ra), DebugAccessory (Rd/Rd), PoweredAccessory
```
The `TD_4_x_*` tests are literally a matrix over this: `TD_4_2_1 Source Connect Sink`, `TD_4_5_4 DRP Connect DRP`, `TD_4_6_3 Try SRC DRP Connect Try SRC DRP`, `TD_4_8_1 DRP Connect Audio Accessory`, `TD_4_11_2 Sink Dead Battery`.

### 5.3 The C#-side mirror — `GrlPdLib\AMS validation\PDStateMachine.cs`

Rather than hard-coding one golden sequence, the host models **every legal continuation** of an Atomic Message Sequence:

```csharp
case MsgType.SourceCap:
    // legal ending 1
    amsSeq = { SourceCap(tester), Request(uut), Accept(tester), PS_RDY(tester) };
    // legal ending 2
    amsSeq = { SourceCap(tester), Request(uut), Reject(tester) };
    // legal ending 3
    amsSeq = { SourceCap(tester), Request(uut), Wait(tester) };
```
…and the same for `PR_Swap`/`FR_Swap`, `VCONN_Swap`, `DR_Swap`, `Get_Sink_Cap`, `Get_Source_Cap`, `SoftReset`, `Hard_Reset`, `BIST`, `Data_Reset`, `Enter_USB`, all EPR variants, chunked extended messages, and VDMs (ACK / NAK / BUSY / Not_Supported / no-response).

`GetPDMessageSequence(...)` then:
1. takes the first observed message,
2. picks the candidate sequence whose first element matches,
3. walks the capture matching each subsequent element (handling **retries**, **GoodCRC indices**, **chunking**, and **interruptions**),
4. sets `AMSStatus` (complete / incomplete / unknown) and an error description such as *"First packet of sequence is missing"*.

> **This is your single best "I test firmware state machines" story:** the host builds the spec's legal state graph in C#, the firmware/DUT produces reality, and the validator diffs them. It is model-based testing, not assertion-by-assertion scripting.

---

## 6. Other firmware facts worth knowing

| Topic | Evidence |
|---|---|
| **Memory management** | Static/global allocation. One big per-port `gUsbPDStruct` accessed via `GetStructPtr(PORT0/PORT1)`; buffers declared `__attribute__((aligned(32)))` for DMA/cache. FreeRTOS heap sized for task stacks (`0x8000`, `0x4000` words). Custom `memcpy`, `XMemSet`, `XMemCopy` with 8-way loop unrolling in `RTOSWrapper.c`. No `malloc` in the hot path. |
| **Hardware abstraction** | `UsbPd_HWControl.c/.h` + `UsbPDRegister.c/.h` + `UsbPDAxiRegister.h` wrap AXI reads/writes (`ReadAxiReg(QC_SWITCH_CONTROL_REG, &val, 4, PORT1)`); `Peripheral_handler.c`, `UsbI2C.c`, `UsbSPIGpio.c`, `platform_info.c`. |
| **Interrupts** | Xilinx `XScuGic` GIC configured in `prvSetupHardware`; ISRs for GPIO (FPGA IRQ line), UART, timers; each ends with `xTaskNotifyFromISR`. IRQ source read from an FPGA register (`REG_RX_AUX1_IRQ_BIT`, `REG_IRQ_TXIRQ_BIT`, `REG_RX_VBUS_HIGH_IRQ_BIT`). |
| **Multi-port** | Everything is parameterised by `PORT0`/`PORT1`; host-side `ModifyAPItoSuppPortB` sets bit 7 of the command byte to target Port B. |
| **Inter-processor comm** | OpenAMP/RPMsg code present but disabled (`#if 0`); the comment says shared OCM memory is used for IPC between CPU0 and CPU1 instead. |
| **Debug** | `xil_printf` over UART; GPIO toggling for scope timing (`DelayDebug()` toggles `GPIO15_DEBUG_OUT` around a delay); `InsertTimestamp(...)` writes markers into the capture stream that the host later reads as assertion packets. |

---

## 7. Honest limits to state in the interview

- You **read and debug** this firmware; there is no evidence you authored the FreeRTOS task structure (`RTOSWrapper.c` header says *Author: GRL-PRASANNA, Sep 2016*; SSBL says *Basweshwar, Feb 2021*).
- The repo contains **no unit tests** for the firmware and **no CI** configuration.
- `petalinux/` and `docs/` are empty placeholders — don't claim a Linux build.
- Say: *"I don't own the firmware, but I read it whenever a test fails, because I have to prove whether the defect is in the DUT, in our instrument firmware, or in my automation."* That sentence alone answers half the embedded questions in this JD.
