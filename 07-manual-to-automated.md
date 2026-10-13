# Phase 9 — Converting a Manual Test Case into an Automated C# Test

The panel said this is literally the job. Have this method ready as a **spoken checklist**, and one worked example.

---

## The 30-second version (say this first)

> "I read the test case and extract five things: **preconditions** — is it even applicable to this device; **setup** — what state the device must be in before the stimulus; **stimulus** — what I send, with what timing, including what I deliberately get wrong; **expected observable behaviour** — what the device must do, by when, measured at which interface; and **cleanup**. Then I check which of those already exist as reusable pieces in the framework, write only what's genuinely new, and the test class becomes the difference between this test and the shared building blocks. Then I run it against a known-good device and a known-bad device, because a test that can't fail isn't a test."

---

## The full 18-step method

### Understand (steps 1–8) — before writing any code

**1. Test objective.** One sentence: *what property of the firmware does this prove?* If you can't write that sentence, you don't understand the test yet — go back to the spec clause it cites.

**2. Preconditions and applicability.** Two different things:
- *Preconditions* = state I must create (device attached, PD contract established, EPR mode entered).
- *Applicability* = whether the test applies at all (device declares EPR-Sink support; device is a Source; cable is captive).
Applicability belongs in `PreConfigure()` and produces `NA`, never `FAIL`.

**3. Inputs / stimulus.** Exact messages, fields, values, and **timing**. Manual steps say "send a Get_Source_Cap"; automation needs to know at what point in the sequence, with what header revision, and what to do if the device answers early.

**4. Device state at each step.** Map the manual steps onto the state machine: which state should the device be in before step 3? That tells you which bring-up procedure to use and where to put your assertions.

**5. Communication mechanism.** Which interface carries the stimulus and which carries the observation. Often not the same — you may stimulate over CC and observe on VBUS.

**6. Expected behaviour.** Split into:
- *Must* — a hard assertion (FAIL if violated).
- *May / optional* — the model must accept several legal outcomes (this is why our AMS model enumerates all legal endings, rather than one golden path).
- *Must not* — negative assertions, often the real point of the test.

**7. Measurable outputs.** For each expectation: what exactly is measured, at what resolution, against what limit, with what tolerance. "Responds quickly" is not automatable; "responds within tSenderResponse (24–30 ms), measured from the end of the GoodCRC to the start of the response" is.

**8. APIs / interfaces needed.** Now do a gap analysis against the framework:
- Does a bring-up procedure already exist? (`BringUpProcedure` enum)
- Does the device API already expose this stimulus? (`PD_Controller` / `GrlC2Controller`)
- Does an assertion helper already exist? (`PDValidationHelper`, `ProtocolHelper`, `EPRSnk_Helper`)
- Is there a spec limit constant? (`PDSpecLimits`)
**Anything missing goes in the shared layer, not in the test.**

### Build (steps 9–14)

**9. Implement reusable components first.** New device capability → device API. New assertion → helper. New setup → bring-up procedure. Do this *before* the test so the test stays thin.

**10. Implement the test class.**
```
- add a member to ComplianceTestEnum with [Description("<Spec ID> <Name>")]
- create class <SameName> : <appropriate base>
- constructor: set supported PD mode / flags, call base
- PreConfigure(): applicability gate, then base.PreConfigure()
- Configure(): choose bring-up, build the stimulus message group, call base.Configure()
- Validate(): build the PDTestResult tree
```

**11. Add validation.** One `PDTestResult` per spec assertion, each with a spec-derived name and ID (`"TEST.PD.EPR.SNK3.9#2"`). Cascade `NA` when a prerequisite assertion couldn't be evaluated — don't produce four failures from one root cause.

**12. Add logging.** Test start/stop, the configuration applied, and for every assertion: expected, obtained, and the packet index. The rule: *a failure report should be a complete bug report without anyone re-running the test.*

**13. Add error handling.** Wrap `Validate()` bodies; log the exception; the framework's per-test try/catch means one bad test doesn't kill the run. Distinguish three outcomes: FAIL (device is wrong), INCOMPLETE (we couldn't get the evidence), NA (doesn't apply).

**14. Add cleanup.** Stop the streamer, save the trace, restore any non-default tester configuration you changed. Anything you toggle in `Configure()` you must restore — otherwise you have created a cross-test dependency.

### Verify and integrate (steps 15–18)

**15. Execute locally, three ways.**
- **Known-good device** → must PASS.
- **Known-bad device or injected fault** → must FAIL, with the right message. *(A test that has never failed is not yet a test.)*
- **Device that doesn't support the feature** → must report NA.
Then run it **10 times** and check it's stable.

**16. Add to the regression suite.** Put it in the right MOI/test group so it's selected by the standard suite; capture a reference trace and, if the validation logic is non-trivial, add a JSON fixture to the offline xUnit suite so the validator itself is regression-tested.

**17. Add CI/CD execution.** Tag it: does it need hardware? which fixture/cable? how long does it take? Smoke suites get the short, high-signal tests; the full regression runs nightly on a reserved station.

**18. Analyse failures.** When it fails in the suite, the triage order is: reproduce in isolation → read the capture at the reported packet index → check whether the *tester's* own stimulus was correct → check firmware/software versions → compare with a previous passing trace. See [08](08-debugging-scenarios.md).

---

## Worked example 1 — a protocol test

### The manual test case (as it would arrive)

> **TD.PD.PROT.SRC.2 — Get_Source_Cap, No Request**
> **Applies to:** UUT that is a Source.
> **Purpose:** verify the Source re-sends Source_Capabilities when the Sink asks, and enters the correct recovery when no Request follows.
> **Procedure:**
> 1. Bring up the UUT as a Source with the Tester as a Sink; establish an explicit contract.
> 2. Tester sends `Get_Source_Cap`.
> 3. UUT must respond with `Source_Capabilities` within tSenderResponse.
> 4. Tester does **not** send a Request.
> 5. UUT must issue a `Hard Reset` after SenderResponseTimer expires.
> **Pass criteria:** steps 3 and 5 satisfied, all message fields valid.

### The conversion

| Step | Extracted | Framework piece |
|---|---|---|
| 1 | Applicability: UUT is a Source | `PreConfigure()` → check VIF / `Type_C_State_Machine == SRC`, else `NA` |
| 1 | Precondition: explicit contract | `objBringUp.SetBringUp_ForConfiguration = BringUpProcedure.COMMON_PROC_BU_1_Bring_up_Source_UUT` |
| 2 | Stimulus: `Get_Source_Cap` | `FwMsgCfg` added to `MessageGroup_AfterPDC` with `TxTimer`/`RxExpTimer` |
| 3 | Assertion A: response type + timing | `GetDesiredAMS(AMSName.Get_Source_Capabilities)` + `tSenderResponse` check |
| 4 | Deliberate omission: no Request | configure the tester **not** to respond — this is the fault injection |
| 5 | Assertion B: Hard Reset within the timer | search the capture for `MsgType.Hard_Reset` after the response and check the interval |
| — | Field validity | comes free from `Common_Check_PD_7_Source_Capability_Message` |
| — | Cleanup | base `StopStreamer_Prerequisite()` / `StopStreamer()` |

### The code

```csharp
// 1) Catalogue entry — TestCaseHelper\PD test results\PDTestResult.cs (ComplianceTestEnum)
[Description("TD.PD.PROT.SRC.2 Get Source Cap No Request")]
TEST_PD_PROT_SRC_2_Get_Source_Cap_No_Request,

// 2) The test class — name MUST equal the enum name (reflection discovery)
namespace USB_PDModule.USBPD_Rev2.Protocol_tests
{
    class TEST_PD_PROT_SRC_2_Get_Source_Cap_No_Request : USBPD_2_TestCase
    {
        public override SupportedPDMode m_SupportedPDVersion { get; set; }

        public TEST_PD_PROT_SRC_2_Get_Source_Cap_No_Request(
                TestCaseDetails d, USBPDConfigHelper cfg, PDValidationHelper val)
            : base(d, cfg, val)
        {
            m_SupportedPDVersion = SupportedPDMode.BOTH;   // run in PD2 and PD3 conditions
            IsFWImple = FWAppObj.FWHost.PDMCurrentTestFWImpl;
        }

        // ---- APPLICABILITY -------------------------------------------------
        public override bool PreConfigure()
        {
            try
            {
                if (ModuleHelper.DUT_Type_C_State_Machine != Type_C_State_Machine.SRC)
                {
                    ObjTestCaseDetails.TestResultObject.UpdateTestResult(TestResult.NA);
                    ObjTestCaseDetails.TestResultObject.UpdateTestDesc(
                        $"Not applicable. Expected Type-C state machine: SRC. " +
                        $"Obtained: {ModuleHelper.DUT_Type_C_State_Machine}");
                    return false;
                }
                return base.PreConfigure();   // bring-up selection, sampling, tester PDOs
            }
            catch (Exception ex) { PDValidationHelper.LogExceptionIntoDebugLogger(ex); return false; }
        }

        // ---- STIMULUS ------------------------------------------------------
        public override bool Configure()
        {
            try
            {
                objBringUp.SetBringUp_ForConfiguration =
                    BringUpProcedure.COMMON_PROC_BU_1_Bring_up_Source_UUT;

                ConstructAfterPDCMessageGroup();
                return base.Configure();
            }
            catch (Exception ex) { PDValidationHelper.LogExceptionIntoDebugLogger(ex); return false; }
        }

        public override bool ConstructAfterPDCMessageGroup()
        {
            try
            {
                // compose, don't send: IsTesterCommEn == false makes the device API
                // capture the frame into m_databuffer_PDC instead of transmitting it
                m_ObjConfHelp.m_objPDCtrl.IsTesterCommEn = false;

                var getSrcCap = new FwMsgCfg(msgRetry: false, bNonPDmsg: false);
                m_ObjConfHelp.m_objPDCtrl.SendControlMessage(MsgType.Get_Source_Cap);
                getSrcCap.MsgHdrPld.AddRange(m_ObjConfHelp.m_objPDCtrl.m_databuffer_PDC);
                getSrcCap.TxTimer    = 50;
                getSrcCap.RxExpTimer = 50;
                m_ObjConfHelp.MessageGroup_AfterPDC.msgList.Add(getSrcCap);

                // FAULT INJECTION: do not queue a Request after Source_Capabilities.
                // The tester stays silent so the UUT's SenderResponseTimer must expire.
                return true;
            }
            catch (Exception ex) { PDValidationHelper.LogExceptionIntoDebugLogger(ex); return false; }
        }

        // ---- ASSERTIONS ----------------------------------------------------
        public override PDTestResult Validate()
        {
            var result = new PDTestResult("Get_Source_Cap with no Request");
            base.Validate();
            try
            {
                var chkResponse  = new PDTestResult("UUT responds Source_Capabilities",
                                                    "TD.PD.PROT.SRC.2#1");
                var chkHardReset = new PDTestResult("UUT issues Hard Reset after timer expiry",
                                                    "TD.PD.PROT.SRC.2#2");

                List<PDMessage> msgs = m_ObjConfHelp.StrmServ.ObjCClinestreamer_PortA.TotalmsgList;

                // assertion 1 — right message, right time
                PDMessage getSrcCap = m_ObjPD2Helper.GetInitiatedMsg(
                                          msgs, MsgType.Get_Source_Cap, isTesterMsg: true);
                if (getSrcCap == null)
                {
                    chkResponse.UpdateTestResult(TestResult.INCOMPLETE);
                    chkResponse.UpdateTestDesc("Tester failed to initiate Get_Source_Cap");
                    chkHardReset.UpdateTestResult(TestResult.NA);
                }
                else
                {
                    PDMessage srcCap = m_ObjPD2Helper.GetResponseMsg(
                                           msgs, getSrcCap, MsgType.SourceCap, isTesterMsg: false);
                    if (srcCap == null)
                    {
                        chkResponse.UpdateTestResult(TestResult.FAIL);
                        chkResponse.UpdateTestDesc(
                            $"UUT did not respond to Get_Source_Cap. Packet index#{getSrcCap.ActualGlobalIndex}");
                        chkHardReset.UpdateTestResult(TestResult.NA);
                    }
                    else
                    {
                        double ms = (srcCap.dStartTime - getSrcCap.dStopTime) * 1000;
                        bool ok = ms <= PDSpecLimits.tSenderResponse_Min;
                        chkResponse.UpdateTestResult(ok ? TestResult.PASS : TestResult.FAIL);
                        chkResponse.UpdateTestDesc(
                            $"UUT responded {srcCap.msgType}. Expected within tSenderResponse " +
                            $"{PDSpecLimits.tSenderResponse_Min} ms. Obtained {Math.Round(ms,3)} ms. " +
                            $"Packet index#{srcCap.ActualGlobalIndex}");

                        // assertion 2 — recovery after the deliberate silence
                        PDMessage hr = m_ObjPD2Helper.GetInitiatedMsg(
                                           msgs, MsgType.Hard_Reset, isTesterMsg: false,
                                           startIndex: srcCap.ActualGlobalIndex + 1);
                        if (hr == null)
                        {
                            chkHardReset.UpdateTestResult(TestResult.FAIL);
                            chkHardReset.UpdateTestDesc(
                                "UUT did not issue Hard Reset after SenderResponseTimer expiry. " +
                                $"Searched from packet index#{srcCap.ActualGlobalIndex + 1}");
                        }
                        else
                        {
                            double hrMs = (hr.dStartTime - srcCap.dStopTime) * 1000;
                            bool hrOk = hrMs >= PDSpecLimits.tSenderResponse_Min &&
                                        hrMs <= PDSpecLimits.tSenderResponse_Max;
                            chkHardReset.UpdateTestResult(hrOk ? TestResult.PASS : TestResult.FAIL);
                            chkHardReset.UpdateTestDesc(
                                $"UUT issued Hard Reset {Math.Round(hrMs,3)} ms after Source_Capabilities. " +
                                $"Expected {PDSpecLimits.tSenderResponse_Min}–{PDSpecLimits.tSenderResponse_Max} ms. " +
                                $"Packet index#{hr.ActualGlobalIndex}");
                        }
                    }
                }

                result.AddTest(chkResponse);     // AddTest rolls the verdict up (FAIL wins)
                result.AddTest(chkHardReset);
            }
            catch (Exception ex) { PDValidationHelper.LogExceptionIntoDebugLogger(ex); }
            return result;
        }
    }
}
```

*(Helper names like `GetInitiatedMsg`/`GetResponseMsg` are illustrative of the shape; the real ones in the repo are `GetInitiatePDMessage` / `GetResponsePDMessage` in `Grl.APIValidation\APIValidationHelper.cs` and `GetPDMsgFromAMS` / `GetDesiredAMS` in `PDValidationHelper`. Say "the exact helper depends on the suite" if pressed — the point is the structure.)*

### How I'd verify it

1. Run against a compliant charger → both assertions PASS.
2. Run against a device known not to Hard Reset → assertion 2 FAILs with the right text.
3. Run against a Sink-only device → whole test NA with the reason.
4. Run 10×, check stability and the timing spread.
5. Save a capture, add it as a JSON fixture to the offline xUnit suite so the *validation logic* is regression-tested without hardware.

---

## Worked example 2 — a functional test (shorter)

> **TD.4.2.1 Source Connect Sink** — a Source UUT must correctly detect, power and negotiate with a Sink, across three cable conditions.

Real implementation (`FunctionalTestCases\TD_4_2_1_Source_Connect_Sink_Test.cs`):
- Constructor sets which validations apply (`bEnDataLinesValidation`, `bEnDataLineCease_Validation`, sub-sampling ratio) and builds one `PDTestResult` per condition: *"USB PD 2.0 5A Active cable"*, *"USB PD 3.0 5A Active cable"*, *"USB4 re-driver cable"* — or a single *"SNK receptacle connector"* condition when the product has a captive cable (`IsPut_C == Status.YES`).
- `Configure()` gates on `ModuleHelper.DUT_Type_C_State_Machine == Type_C_State_Machine.SRC`, distinguishing "VIF not loaded" (`INCOMPLETE`) from "wrong device type" (`NA`, with expected vs obtained printed).
- `IterateForCableCondition()` emulates each cable in turn; firmware injects a `MsgType.Test_Execution_State` marker into the capture at each boundary.
- `Validate()` finds those markers with `Get_Assertion_Message(...)` to compute `iStartGlobalIndex`/`iEndGlobalIndex` and validates each slice independently.

**The transferable ideas to name in the interview:**
1. **Markers in the capture** let one physical run be sliced into N logical test conditions.
2. **Applicability is a first-class outcome**, with different verdicts for "can't tell" vs "doesn't apply".
3. **Conditions are data** (the list of cable conditions) — adding a fourth is a list entry, not new code.

---

## If the interviewer hands you a *radio* test case on the whiteboard

They may well do this — it's the fastest way to see if your method transfers. Example:

> *"The device must re-join the network within 5 seconds after the gateway comes back from a 30-second outage, and must not transmit more than N times during the outage."*

Answer with the same five extractions, out loud:

| | |
|---|---|
| **Applicability** | Does this device role support auto-rejoin? Is it battery-powered (which may change the retry policy)? If not → NA. |
| **Precondition** | Device joined, in steady state, known RF conditions, known battery/supply state. |
| **Stimulus** | Disable the gateway radio for exactly 30 s. The *fault injection* is the test — same as "don't send a Request". |
| **Observation** | Sniff the air interface with timestamps (the equivalent of my CC-line streamer), plus device current draw for the "must not transmit more than N times" part. |
| **Assertions** | (a) rejoin observed ≤ 5 s after gateway restore — measured from a defined reference edge, not from my PC clock; (b) transmission count during the outage ≤ N; (c) nothing illegal in between — a negative assertion. |
| **Cleanup** | Restore the gateway, confirm the device is back in steady state, reset the sniffer. |
| **Reusable pieces** | A `GatewayOutage(duration)` procedure and a `CountTransmissions(from, to)` assertion — both belong in the shared layer because the next ten tests will want them. |
| **Verification** | Run against a good device, a device with rejoin disabled, and 10× for stability. |

Then add the honest line: *"I'd need to learn how you capture and timestamp the air interface, and what the reference edge is for a timing measurement — that's the part that's domain-specific. The structure is the same as what I do with a CC-line capture."*

That answer is the whole interview in miniature: method, transferability, honesty.
