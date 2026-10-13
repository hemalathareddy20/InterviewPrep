# Phase 11, 12, 13 — Serial, BLE, OTA, Battery, RF, RTOS

---

# 1. Serial communication — **you have real evidence**

## Fundamentals
- **UART** = asynchronous serial: no clock line; both ends agree on **baud rate**. Frame = start bit + 5–9 data bits + optional parity + 1/1.5/2 stop bits. `8N1` = 8 data, no parity, 1 stop.
- **RS-232 / TTL / RS-485** are the electrical layers; UART is the framing. USB-serial adapters (CP210x, FTDI, CDC-ACM) present a virtual COM port.
- **Flow control:** hardware RTS/CTS, software XON/XOFF, or none. Without it, a slow receiver overruns.
- **Common failures:** baud mismatch (garbage, often consistent garbage); framing error (wrong stop bits/baud); parity error; buffer overrun; the port held open by another process; COM number changing between boots.

## In your repo
- `System.IO.Ports.SerialPort` used in `USB_PD_Engine.ConnectToSeraiPort(comPort, port)` / `SerialWriteData(...)` to drive an **external 5-port switch box**, so one run can iterate five DUT ports.
- `USB_PD_TesterLib\SerialPortComm.cs`, `EquipmentLib\SerialPort.cs` — instrument serial I/O.
- `FirmwareUpdateManager.EraseSAM()` — opens the port at **1200 baud** then closes it. That's the Atmel SAM / Arduino bootloader trigger: a 1200-baud open/close puts the MCU into its bootloader. Then `bossac.exe -i -d --port=COMx -U false -e -w -b ELOADFW_PORT1.cpp.bin -R` flashes it, with stdout captured into the debug log.
- **COM port discovery by VID/PID** using WMI — `GetComportfromVID()` queries `Win32_PnPEntity` (and falls back to `WIN32_SerialPort`), parses `VID_xxxx`/`PID_xxxx` from the device ID, and matches `VID 0x2341 / PID 0x003D`. If it can't identify the port, it asks the user to pick from `SerialPort.GetPortNames()`.
- Firmware side: UART ISR → `xTaskNotifyFromISR(xAppCmdHandle, UARTINTR)` → `vAppCmdHandleTask` → `UartCmdHandler()`.

## Questions and answers

**"How do you automate against a serial device?"**
> Never hard-code the COM port — enumerate and identify by VID/PID. We do that with WMI: the E-Load's USB-serial bridge has a known VID/PID, so we find its port deterministically, and only fall back to asking the user if identification fails. Then: open with explicit settings, set read/write timeouts (never block forever), read into a buffer until a framing delimiter or a timeout rather than assuming one read returns one message, and always close in `finally` — a leaked open port blocks the next run.

**"What's the hardest part of serial automation?"**
> Framing. A serial port is a byte stream with no message boundaries, so `Read()` can return half a message or two messages. You need a protocol on top — a length prefix, a delimiter, or a fixed frame — and a reader that accumulates until a complete frame is available. The second hardest is the port disappearing when the device resets (common during firmware update) and coming back as a different COM number.

**"How do you test a serial protocol?"**
> Same seven axes as any protocol ([05](05-firmware-functional-and-protocol-testing.md) §2.2): framing, integrity (checksum), field validity, sequence, timing, error recovery, stress. Plus serial-specific negative tests: wrong baud, truncated frame, byte-by-byte trickle to test the reassembler, and a flood to test overrun handling.

---

# 2. BLE — **no repository evidence. Conceptual only.**

**What to say:** *"I haven't worked with BLE — there's no BLE in anything I've built. I know the architecture from reading, and I'd want to be honest about that rather than bluff. What I can say is how I'd approach testing it, because the shape is the same as the protocols I do test."*

Then have this ready.

## The model in one screen

| Layer | What it does |
|---|---|
| **PHY** | 2.4 GHz ISM, 40 channels × 2 MHz. 3 advertising channels (37/38/39, chosen to dodge Wi-Fi), 37 data channels. GFSK. LE 1M / LE 2M / LE Coded (S=2, S=8 for long range). |
| **Link Layer** | Roles: Advertiser / Scanner → Peripheral (slave) / Central (master). Adaptive frequency hopping. Connection interval (7.5 ms–4 s), slave latency, supervision timeout. |
| **HCI** | Host↔controller interface. |
| **L2CAP** | Multiplexing/segmentation. |
| **ATT** | Attribute protocol — a table of handles, UUIDs, values, permissions. |
| **GATT** | Services → Characteristics → Descriptors. Operations: Read, Write, Write-Without-Response, **Notify** (no ack), **Indicate** (acked). CCCD descriptor enables notifications. |
| **GAP** | Discovery, connection, security roles; advertising data (Flags, Local Name, Service UUIDs, Manufacturer Data). |
| **Security** | Pairing (Just Works / Passkey / Numeric Comparison / OOB), bonding (keys stored), LE Secure Connections (ECDH) vs Legacy. |
| **Power** | Advertising interval and connection interval dominate current draw; slave latency lets a peripheral skip connection events. |

## Key numbers
- Connection interval: 7.5 ms – 4 s (multiples of 1.25 ms).
- Advertising interval: 20 ms – 10.24 s.
- ATT_MTU default 23 bytes (20 bytes payload); negotiable, typically up to 247. **The #1 BLE throughput question.**
- Data Length Extension: up to 251-byte PDUs.
- Supervision timeout: 100 ms – 32 s; must exceed `(1 + slaveLatency) × connInterval × 2`.

## "How would you test BLE with C#/.NET?"
> Three layers.
> **Device-side control** — you need a way to command the DUT into a test state. Usually a serial/debug console or a vendor test mode.
> **BLE central** — on Windows, `Windows.Devices.Bluetooth` (WinRT, callable from .NET) gives you `BluetoothLEAdvertisementWatcher` for scanning and `GattDeviceService`/`GattCharacteristic` for read/write/notify. For anything timing-sensitive or deterministic I'd prefer a dedicated BLE tester or a dev-kit dongle driven over serial, exactly as we use dedicated instrument firmware rather than the PC for our timing-critical stimulus.
> **Air sniffer** — a BLE sniffer with timestamps is the equivalent of my CC-line capture. Without it you can only test at the API level and you can't prove timing or count retransmissions.
>
> Then the same test taxonomy: advertising content and interval; connection establishment and parameter negotiation; GATT discovery matching the spec; read/write/notify/indicate on every characteristic including permission and length violations; MTU negotiation; pairing and bonding, including re-connection with stored keys; disconnection and supervision timeout; reconnection after an outage; and the negative cases — writing to a read-only characteristic, over-length writes, unauthenticated access, malformed ATT PDUs.

**Likely follow-up: "What's different about testing BLE vs a wired protocol?"**
> Three things I'd expect to bite. The medium isn't deterministic — interference and retransmission mean you must assert on statistics or a bounded retry count, not a single exchange. The timing reference is harder — you need a sniffer to know when something actually went out over the air. And power is a first-class requirement, so many BLE tests are really current-draw measurements correlated with radio events.

---

# 3. OTA firmware upgrade and rollback — **you have strong adjacent evidence**

**Framing sentence:** *"I've automated in-system firmware update end to end from C#; I haven't done wireless OTA specifically, but the test matrix is the same and I can walk you through it."*

## 3.1 The canonical OTA pipeline, and what to test at each stage

| Stage | What happens | What must be tested |
|---|---|---|
| **1. Announce / discover** | Device learns an update exists; version comparison | Correct version comparison incl. equal, older, malformed; device refuses a downgrade if policy says so |
| **2. Download / transfer** | Image moved to the device in chunks | Correct reassembly; out-of-order/duplicate/missing chunks; transfer interrupted and resumed; transfer while the device is busy; wrong chunk size |
| **3. Validate** | Checksum/CRC, signature, version/compatibility, image size vs partition | Corrupted image rejected **before** activation; unsigned/wrongly-signed image rejected; image for the wrong product rejected; truncated image rejected |
| **4. Install / write** | Erase + program flash | Power loss mid-erase; power loss mid-write; flash write failure reported not ignored; bootloader region protected from being overwritten |
| **5. Activate** | Set boot pointer, reboot | Device reboots into the new image; boot flag is atomic (never half-written) |
| **6. Verify** | Read back the version; self-test | Reported version matches what was flashed; app actually runs (not just boots) |
| **7. Failure / rollback** | Revert to previous or golden image | Failed boot N times → rollback; explicit rollback command; rollback preserves user data/config; rollback of the rollback |
| **8. Power/battery** | | Update refused below a battery threshold; brown-out mid-write; battery removed and reinserted |

## 3.2 The specific failure cases they will ask about

**Power interruption.** The only safe designs are: (a) **A/B (dual-bank)** — write the inactive bank, then flip a boot pointer with a single atomic write, so a power loss at any moment leaves a bootable image; or (b) a **protected immutable bootloader** that can always re-enter recovery mode. Your firmware has (b): `ccs/readme.txt` records that **SSBL V2.0 added write protection of the SSBL flash addresses to prevent corruption of the bootloader**. That's exactly the right answer and it's in your repo.

**Corrupted image.** Must be detected *before* activation, by checksum and ideally signature. Test by flipping a byte in the image and asserting it's rejected. Your evidence: `copy_data()` verifies a key value `0x08AA` before writing, and **SSBL V3.0 returns a success/error status byte (`byte[5]` success, `byte[6]` write error) so the host retries the write until it succeeds**.

**Rollback.** The honest statement: *"our mechanism has a protected bootloader and an explicit mode state machine — `SET_PGM` / `SET_BOOT` / `GET_MODE` — so a failed update leaves the device in programming mode where it can be re-flashed, rather than bricked. We don't keep a previous image to revert to; a dual-bank design with a boot counter would be the next step, and on the Zynq side the FSBL already supports multiple boot devices and image authentication, so a golden-image fallback is achievable."* That is precise and creditable.

**Version verification.** Yours: `Firmware_SSBL_Version()` is called **twice** after the update, then `SET_BOOT`. And on the host, `ModuleHelper.FWversionObt` is compared against the expected `FWversion` at run start, blocking or warning on mismatch.

## 3.3 "How would you automate OTA testing in C#?"

> "I'd build it as a state machine with fault injection at every edge, because that's where the bugs are.
>
> A `FirmwareUpdateSession` class with `Prepare → Transfer → Verify → Activate → Confirm`, each step returning a typed result, and a test harness that can abort at any step. Then a parameterised test: for each step S, for each fault F in {power cut, link drop, corrupted chunk, device busy, low battery}, inject F at S and assert the device ends in a **defined, recoverable state** — either the old firmware running, or recovery mode reachable.
>
> Power cut needs hardware: a programmable supply or relay the test can switch, so 'power loss during erase' is a real, repeatable test rather than someone pulling a plug. We already control programmable supplies and electronic loads from C#, so that's familiar.
>
> On top of that, the boring but essential ones: upgrade from every supported version to the new one (N-1, N-2, and the oldest supported), downgrade if allowed, same-version re-flash, and an interrupted-then-resumed transfer.
>
> Concretely I've automated this shape already: mode transition, flash erase, 256-byte chunked transfer with a per-chunk failure check and progress reporting, version read-back verification, activation, and a defined recovery path even when the transfer fails. Swap I2C for a BLE GATT characteristic or an HTTP endpoint and the harness is the same."

## 3.4 Battery requirements

**Honest scope:** your repo has USB-PD **battery messages** — `Get_Battery_Status`, `Get_Battery_Cap`, `Battery_Capabilities`, `BatteryStatusMesaage.cs`, `SetBatteryCapabiliyExtended.cs`, and `TD_4_11_2_Sink_Dead_Battery_Test` (a device with a dead battery must still negotiate enough power to charge). There is **no** battery-life or current-drain characterisation.

**What to know conceptually:**
- **Capacity** (mAh / Wh), **C-rate**, **state of charge (SoC)** vs **state of health (SoH)**, **self-discharge**, **cut-off voltage**.
- **Average current** is what determines battery life for a duty-cycled device: `life ≈ capacity / average current`, where average = (active current × active time + sleep current × sleep time) / total. For a BLE sensor the sleep current and the advertising interval dominate — which is why BLE tests are so often current measurements.
- **Test approach:** a high-dynamic-range current probe or source-measure unit, synchronised with the radio/protocol capture so you can attribute current to events; then long-duration soak with logging.
- **Low-battery behaviour** is a functional test: does the device refuse an OTA below threshold, does it degrade gracefully, does it report SoC correctly, does it recover from a brown-out.

---

# 4. RF fundamentals — **no evidence. Learn the definitions, time-box it.**

Panel said limited RF is acceptable. Spend 30 minutes here, not three hours.

## The scripted honest answer

> **"How much experience do you have with RF?"**
>
> "None hands-on — I want to be straight about that. My protocol work is wired: USB Type-C and Power Delivery, where I do work at the physical layer with eye diagrams, bit-rate drift, receiver interference rejection and input impedance, so I'm comfortable with signal-integrity concepts and with the idea that the physical layer is something you measure, not something you assert on.
>
> I understand the RF vocabulary — RSSI, sensitivity, bandwidth, modulation, link budget — at a definitional level, and I've read up on it for this conversation. What I'd bring immediately is the automation side: driving instruments, structuring a test framework, and turning a spec test case into a repeatable measurement. The RF domain knowledge I'd need to learn, and I understood from the description that the team is happy to train on that."

That is exactly the right answer: honest, shows adjacent depth, shows you prepared.

## Definitions — simple → technical → example → how it's tested → likely question

| Term | Simple | Technical | Example | Testing | Question |
|---|---|---|---|---|---|
| **RF** | Radio-frequency electromagnetic energy used to carry information without wires | ~3 kHz–300 GHz | Wi-Fi 2.4/5 GHz, BLE 2.4 GHz, LoRa 868/915 MHz | Spectrum analyser, vector signal analyser | *What frequency bands have you worked with?* |
| **Radio** | The transmitter+receiver that turns bits into RF and back | TX chain: encode → modulate → up-convert → PA → antenna. RX: antenna → LNA → down-convert → demodulate → decode | An nRF52 or CC1352 SoC | TX power, spectral mask, EVM; RX sensitivity, selectivity, blocking | *Describe a radio's TX and RX chain.* |
| **Sub-GHz** | Radio below 1 GHz | 169/315/433/868/915 MHz ISM bands | Smart meters, industrial sensors, alarms | Same as any radio, at those bands | *Why sub-GHz rather than 2.4 GHz?* |
| **BLE** | Low-power short-range 2.4 GHz radio | See §2 | Wearables, sensors, beacons | Conformance + interoperability + power | *BLE vs classic Bluetooth?* |
| **RSSI** | "How loud is the signal I'm receiving" | Received Signal Strength Indicator, in dBm. −40 dBm strong, −90 dBm weak. Vendor-specific accuracy, typically ±6 dB | Wi-Fi bars on your phone | Measure at known distances/attenuations in a shielded box; sweep attenuation and plot RSSI | *Is RSSI a reliable distance measure?* → No: multipath, antenna orientation, body loss, and vendor variation make it a rough indicator, fine for link quality, poor for ranging. |
| **Sensitivity** | "How quiet a signal can I still receive correctly" | The minimum received power for a specified error rate (e.g. PER ≤ 1% or BER 10⁻³). More negative = better. BLE 1M typically ≈ −95 dBm; LoRa can reach ≈ −137 dBm | | Attenuate the TX until the error rate crosses the threshold; that power is the sensitivity | *RSSI vs sensitivity?* → RSSI is what you're getting now; sensitivity is the floor of what the receiver can still decode. |
| **Bandwidth** | How much frequency the signal occupies | Hz. Wider = more data possible, but more noise admitted (noise power ∝ bandwidth) and worse sensitivity | BLE 2 MHz/channel; LoRa 125/250/500 kHz | Occupied bandwidth / spectral mask on a spectrum analyser | *Bandwidth vs data rate?* → Bandwidth is spectrum occupied (Hz); data rate is information carried (bit/s). Related by Shannon `C = B·log₂(1+SNR)` and by the modulation order, but not the same thing. |
| **Frequency / wavelength** | How fast it oscillates / how long one cycle is in space | `λ = c / f`. 868 MHz ≈ 34.6 cm; 2.4 GHz ≈ 12.5 cm | | | *Frequency vs wavelength?* → Inversely proportional. Practical consequence: lower frequency = longer wavelength = **bigger antenna** but **better penetration and range**; higher frequency = smaller antenna, more bandwidth available, worse wall penetration. |
| **Modulation** | How bits are impressed on the carrier | ASK/OOK, FSK/GFSK (BLE, most sub-GHz), PSK/BPSK/QPSK, QAM (16/64/256), OFDM (Wi-Fi/LTE), spread spectrum (DSSS, FHSS, LoRa CSS) | BLE = GFSK; LoRa = chirp spread spectrum | EVM, constellation, spectral mask | *Why GFSK for BLE?* → Constant envelope, so a cheap non-linear PA can be used efficiently — critical for battery devices. Higher-order modulation carries more bits/symbol but needs better SNR. |
| **SNR** | Wanted signal vs background noise | `SNR = P_signal / P_noise`, in dB. Thermal noise floor ≈ −174 dBm/Hz + 10log₁₀(BW) + NF | | Measured by the receiver or an analyser | *SNR vs RSSI?* → RSSI can be high while SNR is terrible if there's an interferer. SNR is what determines whether you can decode. |
| **Range** | How far it works | Link budget: `P_rx = P_tx + G_tx − path_loss + G_rx`. Works while `P_rx > sensitivity + margin`. Free-space path loss rises with frequency and distance; real environments are far worse | | Measured in a chamber (repeatable) and in the field (realistic) | *How do you increase range?* → More TX power (regulatory limit), better antenna gain, better sensitivity (lower data rate / narrower bandwidth / spreading), lower frequency. |
| **Interference** | Other signals in your band | Co-channel and adjacent-channel; 2.4 GHz is crowded with Wi-Fi, BLE, Zigbee, microwaves | | Blocking/selectivity tests: inject an interferer at a defined offset and power, measure the error rate | *How does BLE cope?* → Adaptive frequency hopping plus advertising channels placed between Wi-Fi channels. |
| **Transmit power** | How hard you shout | dBm. +10 dBm = 10 mW, +20 dBm = 100 mW. Capped by regulation (ETSI/FCC) and by battery | | Conducted power on a power meter; radiated (EIRP) in a chamber | *Range vs TX power?* → +6 dB roughly doubles free-space range, but it's a logarithmic, diminishing relationship and costs current linearly-ish; improving sensitivity or lowering the data rate is often cheaper. |
| **Packet loss / PER** | Fraction of packets not received correctly | PER; BER at the bit level | | Send N known packets, count correct receptions; sweep attenuation to get a PER-vs-power curve | *What's an acceptable PER?* → Depends: BLE sensitivity is specified at 30.8% PER for dirty-transmitter tests, 1% for others — the number only means something with its test condition. |
| **Antenna basics** | The transducer between circuit and air | Gain (dBi), radiation pattern, polarisation, impedance matching (**VSWR / return loss**, want <2:1 / >10 dB). Typical size ~λ/4 | PCB trace, chip antenna, whip | VNA for return loss; anechoic chamber for pattern | *Why does a metal enclosure hurt?* → Detunes the antenna and blocks radiation. *What's VSWR?* → How much power is reflected instead of radiated. |

## The five "vs" comparisons — memorise these

1. **RSSI vs sensitivity** — RSSI = measured received power *right now*. Sensitivity = the *minimum* power the receiver needs for a given error rate. Link margin = RSSI − sensitivity.
2. **Bandwidth vs data rate** — Hz vs bit/s. Wider bandwidth *allows* higher data rate but also admits more noise, degrading sensitivity. LoRa trades data rate for sensitivity by spreading.
3. **Frequency vs wavelength** — `λ = c/f`, inversely proportional. Lower frequency → larger antenna, better penetration/range; higher frequency → smaller antenna, more available bandwidth, worse penetration.
4. **BLE vs sub-GHz** — BLE: 2.4 GHz, global band, short range (10–100 m), higher data rate (1–2 Mbps), phone-compatible, crowded band, small antenna. Sub-GHz: 868/915 MHz (regional!), long range (hundreds of m to km), low data rate, better penetration, less congestion, bigger antenna, no phone support.
5. **Range vs transmit power** — logarithmic, not linear: +6 dB ≈ 2× free-space range, and real-world path loss exponents make it worse. Sensitivity, antenna and data rate usually give more range per unit of effort than raw power, which is capped by regulation anyway.

---

# 5. RTOS / FreeRTOS — **you have reading evidence, not authoring evidence**

**Framing sentence:** *"The firmware I test runs FreeRTOS on a Zynq Cortex-A9, and I read it whenever I need to understand a timing or state bug. I haven't authored FreeRTOS task code — I'd call it strong reading knowledge and conceptual understanding rather than hands-on development."*

Then be genuinely good on the concepts, using your firmware as the example.

| Concept | Answer | Your example |
|---|---|---|
| **What is an RTOS?** | An OS that guarantees *bounded, predictable* response time to events. "Real-time" means deterministic, not fast. Hard real-time = a missed deadline is a system failure; soft = degraded quality. | |
| **RTOS vs bare-metal** | Bare-metal = superloop + ISRs: simplest, lowest overhead, but any long operation blocks everything and priorities are implicit in code order. RTOS = independent tasks with explicit priorities, blocking primitives, and a scheduler — costs RAM (a stack per task) and context-switch time, buys separation of concerns and analysable timing. Choose an RTOS when you have several independent activities with different deadlines. | Our firmware has exactly that: interrupt handling, host commands, and protocol timers are three concerns with very different urgency. |
| **Task** | An independent thread of execution with its own stack, priority and state (Running / Ready / Blocked / Suspended). Created with `xTaskCreate(fn, name, stackWords, param, priority, &handle)`. | `xTaskCreate(vInterruptHandlerTask, "GpioInterrupt", 0x8000, NULL, tskIDLE_PRIORITY+7, &xIntrHandle)` |
| **Scheduler** | Chooses which Ready task runs. FreeRTOS default is **fixed-priority preemptive** with round-robin time-slicing among equal priorities (`configUSE_TIME_SLICING`). Started with `vTaskStartScheduler()`. | `UsbPDMain.c:15010` |
| **Priority** | Higher number = higher priority in FreeRTOS (opposite of some RTOSes). Idle task is priority 0. | Interrupt task 7 > command task 6 — interrupts beat host commands, correctly. |
| **Preemption** | A higher-priority task becoming Ready immediately displaces a lower one. Consequence: a lower-priority task can be starved indefinitely. | A busy-wait (`UserSetDelay`) in a priority-7 task would starve everything below it — a real risk in this codebase and a good thing to point out. |
| **Context switch** | Save the current task's registers/stack pointer to its TCB, restore the next task's. Costs microseconds; triggered by a tick, a blocking call, or `portYIELD_FROM_ISR`. | |
| **Queue** | Thread-safe FIFO that **copies** data; senders/receivers block with a timeout. The primary FreeRTOS IPC. `xQueueSend`/`xQueueReceive`, `...FromISR` variants. | `queue.h` is included; the active design uses task notifications instead. |
| **Binary semaphore** | Signalling: one task waits, an ISR or another task gives. No ownership, no priority inheritance. | Declared (`xGpioIntrSemaBinary`) but the live path uses notifications. |
| **Counting semaphore** | Counts N available resources or N pending events. | |
| **Mutex** | Mutual exclusion with **ownership** and **priority inheritance** — the holder is temporarily boosted to the priority of the highest waiter, which bounds priority inversion. Must be given back by the same task that took it. **Never use a mutex from an ISR.** | Ten `xSemaphoreCreateMutex()` calls: `xsrcPwrupSrcCapMutex`, `xprSwapMutex`, `xmsgTimerMutexPort0/1`, `xAMSstatus`, … |
| **Mutex vs binary semaphore** | **The classic question.** Both are binary, but a mutex has ownership and priority inheritance → use for protecting a shared resource. A binary semaphore has neither → use for signalling between an ISR and a task. Using a semaphore where you need a mutex gives you unbounded priority inversion. | Our firmware uses mutexes for shared state and notifications for signalling — textbook correct. |
| **Event group** | A set of bits; tasks wait on any or all of them (`xEventGroupWaitBits`). Lets one task wait on several conditions. | `xglFxGpioAppEventGroup`, `xglAppTimerEventGroup(1)` declared. |
| **Task notification** | A 32-bit value per task, set directly from an ISR or task. ~45% faster and uses less RAM than a semaphore, but only supports a single recipient. **This is what our firmware actually uses.** | `xTaskNotifyFromISR(xIntrHandle, GPIOINTR_PORT0, eSetBits, &woken)` → `xTaskNotifyWait(0, ULONG_MAX, &val, pdMS_TO_TICKS(500))` |
| **Software timer** | A callback run in the timer service task after a period; one-shot or auto-reload. Cheaper than a task per timeout, but the callback must not block. | 10 × `xTimerCreate` for PD message/transaction/QC/misc timers, two per port |
| **ISR** | Runs at hardware priority above all tasks. Rules: keep it short, no blocking calls, only the `...FromISR` API, and `portYIELD_FROM_ISR(woken)` at the end so a woken higher-priority task runs immediately. | ISRs read the FPGA IRQ register and notify — nothing else |
| **Deferred interrupt handling** | ISR does the minimum and hands work to a high-priority task, so the system spends almost no time with interrupts disabled. | Exactly what `vInterruptHandlerTask` is |
| **Race condition** | Two contexts access shared state and the result depends on timing. Fix: mutex, or make the access atomic, or confine the data to one task. | |
| **Priority inversion** | A high-priority task waits on a resource held by a low-priority task, which is itself preempted by a medium-priority task — so effectively the high task runs at the low priority. **Unbounded** inversion is the Mars Pathfinder bug. Fixes: priority inheritance (FreeRTOS mutexes do this) or priority ceiling. | |
| **Deadlock** | Circular wait on mutexes. Prevent with a global lock ordering, or by never holding two. | |
| **Starvation** | A low-priority task never runs because higher ones always are Ready. | A busy-wait in a high-priority task causes exactly this |
| **Stack overflow** | Each task's stack is fixed at creation. FreeRTOS offers `configCHECK_FOR_STACK_OVERFLOW` and `uxTaskGetStackHighWaterMark()`. | Stacks here are 0x8000 / 0x4000 **words** — generously sized, which suggests someone hit this |
| **Heap** | FreeRTOS heap_1..heap_5. Most embedded systems allocate everything at startup and never free, to avoid fragmentation. | The `for(;;)` after `vTaskStartScheduler()` with the comment about insufficient heap is the standard FreeRTOS idiom |
| **Tick** | The periodic timer interrupt that drives delays and time-slicing; `configTICK_RATE_HZ` typically 1000 → 1 ms resolution. `pdMS_TO_TICKS(x)` converts. | `xThreadSleep(ms)` → `vTaskDelay(ms / portTICK_PERIOD_MS)` |
| **`vTaskDelay` vs busy-wait** | `vTaskDelay` blocks and yields — other tasks run; resolution is one tick. A busy-wait spins, giving sub-tick precision but burning CPU and starving lower-priority tasks. | Both exist in `RTOSWrapper.c` — a real trade-off you can discuss |

## RTOS questions you should be able to answer cold

**"How do you signal from an ISR to a task?"**
> Binary semaphore give-from-ISR, a queue send-from-ISR, an event-group set-from-ISR, or a task notification — and always the `...FromISR` variant with `portYIELD_FROM_ISR(&xHigherPriorityTaskWoken)` so the woken task runs on exit rather than at the next tick. Our firmware uses task notifications with bit flags, one bit per interrupt source, which is the lightest option when there's a single recipient.

**"How would you debug a task that never runs?"**
> Check its priority against what's Ready; check whether a higher-priority task is busy-waiting instead of blocking; check `uxTaskGetStackHighWaterMark` in case it overflowed and corrupted something; check whether it's blocked forever on a semaphore nobody gives (which is why our wait has a 500 ms timeout rather than `portMAX_DELAY`); and use a trace tool — Tracealyzer or SEGGER SystemView — to see the actual schedule.

**"Would you put this test logic in the firmware or in the host?"**
> Timing-critical behaviour in the firmware, analysis in the host. That's the split our system makes deliberately: the host compiles a stimulus sequence and uploads it, the firmware replays it with hardware timing, and the host does the decoding and validation afterwards. A PC can't be the timing reference for a 30 ms protocol deadline, and an MCU shouldn't be doing report generation.
