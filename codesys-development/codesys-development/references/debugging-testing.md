# CODESYS Debugging and Testing Guide

## 1. Common Errors and Solutions

### 1.1 Compilation Errors

| Error Message | Cause | Solution |
|--------------|-------|----------|
| `Type mismatch in assignment` | Assigning wrong type (e.g., INT to LREAL) | Use explicit conversion: `rVal := INT_TO_LREAL(nVal)` |
| `Array index out of range` | Index exceeds declared bounds | Add bounds check: `IF i >= 0 AND i <= MAX THEN` |
| `VAR_IN_OUT parameter mismatch` | Array size in caller < FB declaration | Ensure caller array size >= FB declaration |
| `Cannot convert ...` | Implicit type conversion not allowed | Use `TO_` functions or type cast `LREAL#` |
| `Unknown identifier: ...` | Variable not declared or wrong scope | Check spelling, check if declared in correct VAR section |
| `Circular reference detected` | FB A calls FB B which calls FB A | Break cycle with intermediate VAR or restructure |
| `ENDIF expected` | Missing `END_IF`, `END_FOR`, `END_CASE` | Check nested block matching |
| `Duplicate identifier` | Same variable name declared twice | Rename one instance |

### 1.2 Runtime Errors

| Error | Symptoms | Cause | Solution |
|-------|----------|-------|----------|
| Watchdog triggered | PLC stops, error in log | Scan time > task cycle time | Optimize code, move heavy logic to slower task |
| Array out of bounds | PLC crash or data corruption | Index exceeds array size without check | Always validate index before access |
| Stack overflow | PLC crash | Deep recursion or large local arrays | Avoid recursion, use VAR_IN_OUT for large arrays |
| Null pointer access | PLC exception | Accessing uninitialized POINTER | Check `pVar <> 0` before dereferencing |
| Division by zero | PLC exception or NaN | Divisor is 0 | Check `IF divisor <> 0 THEN` |
| Type overflow | Unexpected negative values | UINT/INT wrap-around | Use DINT for large counters, check before increment |

### 1.3 Logic Errors

#### Edge Trigger Not Working

```iecst
// PROBLEM: Action fires every scan cycle
IF bSignal THEN
    // This runs EVERY scan while bSignal is TRUE
END_IF

// SOLUTION: Use edge detection
IF bSignal AND NOT bLastSignal THEN
    // This runs only ONCE on rising edge
END_IF
bLastSignal := bSignal;  // MUST update after check
```

#### VAR_IN_OUT Not Updating

```iecst
// PROBLEM: FB doesn't update caller's array
// CAUSE: Array sizes don't match

// FB declaration
VAR_IN_OUT
    DataArray : ARRAY[0..9999] OF LREAL;  // Expects 10000 elements
END_VAR

// Caller declaration (WRONG - too small)
VAR
    MyData : ARRAY[0..499] OF LREAL;  // Only 500 elements
END_VAR
// FIX: Declare matching size
VAR
    MyData : ARRAY[0..9999] OF LREAL;
END_VAR
```

#### Timer Not Resetting

```iecst
// PROBLEM: Timer never starts or never resets
tonDelay(IN := TRUE, PT := T#1S);
// Timer Q goes TRUE after 1s but never resets because IN stays TRUE

// SOLUTION: Use variable input, not constant TRUE
tonDelay(IN := bCondition, PT := T#1S);
// When bCondition goes FALSE, timer resets automatically
```

#### State Machine Stuck

```iecst
// PROBLEM: State machine gets stuck in one state
// CAUSE: Missing state transition condition or unreachable condition

// DEBUGGING: Add state logging
CASE nState OF
    0: // IDLE
        nState := 10;

    10: // WAIT_FOR_SENSOR
        // BUG: bSensorReady is never set because sensor address is wrong
        IF bSensorReady THEN
            nState := 20;
        END_IF

        // DEBUG: Add timeout
        tTimeout(IN := TRUE, PT := T#10S);
        IF tTimeout.Q THEN
            tTimeout(IN := FALSE);
            nState := 99; // ERROR
            nErrorCode := ERR_TIMEOUT;
        END_IF

    20: // PROCESSING
        // ...

    99: // ERROR
        IF bReset THEN
            nState := 0;
        END_IF
END_CASE
```

---

## 2. Online Debugging Techniques

### 2.1 Online Change

Modify code while PLC is running without full download:

**When to use**: Small logic changes, parameter tuning, bug fixes

**Limitations**:
- Cannot change variable declarations (type, size, VAR section)
- Cannot add/remove VAR_IN_OUT parameters
- Cannot change task assignment
- Large changes may exceed memory

**Procedure**:
1. Make code change in editor
2. Press F8 (Online Change) or "Login with Change"
3. CODESYS compiles diff and downloads only changed code
4. PLC continues running with new code

### 2.2 Write and Force Values

```
// Write value (one-time override):
//   Right-click variable → "Write value" → Enter value
//   Value is written once, PLC continues from there

// Force value (continuous override):
//   Right-click variable → "Force value" → Enter value
//   Variable is overridden every scan until force is removed
//   Use "Unforce all" to release

// When to use:
//   Write: Test specific scenario, simulate input
//   Force: Bypass faulty sensor, lock output for testing
```

### 2.3 Breakpoints

```iecst
// Set breakpoint in code editor:
//   Click in gutter next to line number, or F9

// Breakpoint behavior:
//   - PLC halts at breakpoint line (before execution)
//   - Can inspect all variables in current scope
//   - Can step: F10 (step over), F11 (step into), Shift+F11 (step out)
//   - Continue: F5

// Limitations:
//   - Only works in ST (not LD/FBD graphical editors)
//   - PLC scan is halted — outputs freeze, communication stops
//   - Watchdog may trigger if held too long
//   - Not available on all CODESYS target platforms
```

### 2.4 Trace (Oscilloscope-like Recording)

```
// Configure trace in CODESYS:
// 1. Add "Trace" object to project
// 2. Add variables to trace (up to 20 typically)
// 3. Configure trigger:
//    - None (continuous recording)
//    - Variable value trigger (rising/falling/level)
//    - Time-based (record for N seconds)
// 4. Set sample interval (must be >= task cycle time)
// 5. Start trace, perform action, stop trace
// 6. Analyze waveform: zoom, measure, export to CSV

// Typical trace configuration for motor startup:
//   Variables: bStartCmd, bMotorRunning, rActualSpeed, rCurrent, nState
//   Trigger: bStartCmd rising edge
//   Sample: 10ms
//   Duration: 5 seconds
```

### 2.5 Call Stack

```
// When breakpoint is hit, view call stack:
//   Shows: PRG_Main → FB_MotorControl.M_Start → FC_CheckInterlock
//
// Use to trace:
//   - Which code path led to current location
//   - What parameters were passed
//   - Where unexpected entry occurred
```

### 2.6 Logic Analyzer (Digital Signals)

```
// For analyzing digital signal timing:
// 1. Add "Logic Analyzer" object
// 2. Select BOOL variables to monitor
// 3. Configure sample rate (faster than trace)
// 4. View timing diagram similar to oscilloscope
//
// Use cases:
//   - Verify interlock timing
//   - Check edge detection working
//   - Measure response delay
//   - Debug communication protocol timing
```

---

## 3. Simulation and Testing

### 3.1 CODESYS Control Win V3 (SoftPLC)

Run PLC code without hardware:

**Setup**:
1. Install "CODESYS Control Win V3" (included in CODESYS installation)
2. In project, set device to "CODESYS Control Win V3"
3. Login and run — code executes on Windows PC

**Capabilities**:
- Full ST/LD/FBD execution
- All standard libraries (timers, counters, math)
- Modbus TCP communication (via localhost)
- OPC UA server (accessible from other PC applications)
- WebVisu (accessible via browser)

**Limitations**:
- No real I/O hardware (use simulated inputs)
- No EtherCAT/PROFINET (use simulated fieldbus)
- No real-time guarantees (Windows is not RTOS)
- Scan time may vary (use for logic testing, not timing)

### 3.2 Creating a Test Program

```iecst
PROGRAM PRG_Test

VAR
    // FB under test
    fbUnderTest : FB_CamTableGenerator;

    // Test data
    arrTestData : ARRAY[0..9999] OF LREAL;
    rSimAngle   : LREAL := 0.0;
    rSimEncoder : LREAL := 0.0;

    // Test control
    bStartTest  : BOOL;
    bTestDone   : BOOL;
    bTestFail   : BOOL;
    nTestStep   : UINT := 0;
    nPassCount  : UINT := 0;
    nFailCount  : UINT := 0;

    // Timing
    tonStep     : TON;
END_VAR

// --- Test Sequence ---
CASE nTestStep OF
    0: // INIT
        bStartTest := FALSE;
        bTestDone := FALSE;
        bTestFail := FALSE;
        nPassCount := 0;
        nFailCount := 0;
        IF bStartTest THEN
            nTestStep := 10;
        END_IF

    10: // TEST 1: Basic acquisition
        // Simulate angle increment
        rSimAngle := rSimAngle + 1.0;
        rSimEncoder := rSimEncoder + 0.5;

        fbUnderTest.bEnable := TRUE;
        fbUnderTest.SensorValue := rSimEncoder;
        fbUnderTest.TriggerSource := rSimAngle;
        fbUnderTest.MaxArrayLength := 100;
        fbUnderTest.StepInterval := 10.0;
        fbUnderTest(DataArray := arrTestData);

        IF fbUnderTest.ActualLength >= 10 THEN
            // Verify data
            IF arrTestData[0] = 0.0 THEN
                nPassCount := nPassCount + 1;
            ELSE
                nFailCount := nFailCount + 1;
            END_IF
            nTestStep := 20;
        END_IF

    20: // TEST 2: Reset functionality
        fbUnderTest.bReset := TRUE;
        fbUnderTest(DataArray := arrTestData);
        fbUnderTest.bReset := FALSE;

        IF fbUnderTest.ActualLength = 0 AND NOT fbUnderTest.bRunning THEN
            nPassCount := nPassCount + 1;
        ELSE
            nFailCount := nFailCount + 1;
        END_IF
        nTestStep := 30;

    30: // TEST 3: Invalid parameter
        fbUnderTest.StepInterval := -1.0;
        fbUnderTest.bEnable := TRUE;
        fbUnderTest(DataArray := arrTestData);

        IF fbUnderTest.bError AND fbUnderTest.ErrorCode = 2 THEN
            nPassCount := nPassCount + 1;
        ELSE
            nFailCount := nFailCount + 1;
        END_IF
        nTestStep := 90;

    90: // DONE
        bTestDone := TRUE;
        bStartTest := FALSE;
        nTestStep := 0;

END_CASE

// Output test results
IF bTestDone THEN
    IF nFailCount = 0 THEN
        // ALL TESTS PASSED
    ELSE
        // SOME TESTS FAILED
    END_IF
END_IF

END_PROGRAM
```

### 3.3 Unit Testing with CfUnit

CfUnit is an open-source unit testing framework for CODESYS:

```iecst
// Install CfUnit from CODESYS Store
// Create test FB extending TcUnit.FB_TestSuite

{attribute 'enable_testing'}
FUNCTION_BLOCK FB_CamTableTests EXTENDS TcUnit.FB_TestSuite

VAR
    fbCamGen : FB_CamTableGenerator;
    arrData  : ARRAY[0..9999] OF LREAL;
END_VAR

// Test method: Test basic acquisition
METHOD M_TestBasicAcquisition : BOOL
VAR
    i : UINT;
    rSimAngle : LREAL;
    rSimEnc   : LREAL;
END_VAR

// Setup
fbCamGen.bEnable := FALSE;
fbCamGen.bReset := TRUE;
fbCamGen(DataArray := arrData);
fbCamGen.bReset := FALSE;

// Run
fbCamGen.MaxArrayLength := 10;
fbCamGen.StepInterval := 10.0;
fbCamGen.bEnable := TRUE;

rSimAngle := 0.0;
rSimEnc := 100.0;

FOR i := 0 TO 200 DO
    rSimAngle := rSimAngle + 1.0;
    rSimEnc := rSimEnc + 5.0;
    fbCamGen.SensorValue := rSimEnc;
    fbCamGen.TriggerSource := rSimAngle;
    fbCamGen(DataArray := arrData);
END_FOR

// Assert
THIS.AssertTrue(fbCamGen.ActualLength > 0, 'Should have captured data');
THIS.AssertEquals(UINT#10, fbCamGen.ActualLength, 'Should capture exactly 10 points');
THIS.AssertEquals(TRUE, fbCamGen.bDone, 'Should be done');
THIS.AssertEquals(FALSE, fbCamGen.bError, 'Should not have error');
END_METHOD

// Test method: Test invalid parameter
METHOD M_TestInvalidParam : BOOL
VAR
END_VAR

fbCamGen.bReset := TRUE;
fbCamGen(DataArray := arrData);
fbCamGen.bReset := FALSE;

fbCamGen.StepInterval := -1.0;
fbCamGen.bEnable := TRUE;
fbCamGen(DataArray := arrData);

THIS.AssertTrue(fbCamGen.bError, 'Should set error flag');
THIS.AssertEquals(UINT#2, fbCamGen.ErrorCode, 'Should be ERR_PARAM');
END_METHOD

// Main test runner
METHOD M_RunTests
THIS.M_TestBasicAcquisition();
THIS.M_TestInvalidParam();
END_METHOD
```

### 3.4 Simulation Patterns

#### Simulating Sensor Inputs

```iecst
FUNCTION_BLOCK FB_SensorSimulator

VAR_INPUT
    bEnable     : BOOL;
    rBaseValue  : LREAL := 50.0;
    rNoiseAmp   : LREAL := 2.0;     // Noise amplitude
    rDriftRate  : LREAL := 0.1;     // Drift per second
END_VAR

VAR_OUTPUT
    rValue      : LREAL;
END_VAR

VAR
    rNoise      : LREAL;
    rDrift      : LREAL;
    tonUpdate   : TON;
    nNoiseSeed  : UINT := 12345;
END_VAR

// Simple pseudo-random noise generator
nNoiseSeed := (nNoiseSeed * 1103515245 + 12345) MOD 2147483648;
rNoise := (LREAL#nNoiseSeed / 2147483648.0 - 0.5) * rNoiseAmp;

// Drift accumulation
tonUpdate(IN := bEnable, PT := T#1S);
IF tonUpdate.Q THEN
    tonUpdate(IN := FALSE);
    rDrift := rDrift + rDriftRate;
END_IF

// Output = base + drift + noise
IF bEnable THEN
    rValue := rBaseValue + rDrift + rNoise;
ELSE
    rValue := 0.0;
    rDrift := 0.0;
END_IF

END_FUNCTION_BLOCK
```

#### Simulating Communication Devices

```iecst
FUNCTION_BLOCK FB_ModbusSlaveSim

VAR_INPUT
    bEnable     : BOOL;
    nPort       : UINT := 5020;  // Use non-standard port for simulation
END_VAR

VAR
    arrHoldingRegs : ARRAY[0..99] OF WORD;  // Simulated registers
    arrCoils       : ARRAY[0..99] OF BOOL;
    nRegIdx        : UINT;
END_VAR

// Initialize with test data
IF bEnable THEN
    arrHoldingRegs[0] := 100;  // Simulated temperature * 10
    arrHoldingRegs[1] := 200;  // Simulated pressure * 10
    arrHoldingRegs[2] := 300;  // Simulated flow * 10
    arrCoils[0] := TRUE;       // Motor running
    arrCoils[1] := FALSE;      // Valve closed
END_IF

// Run Modbus TCP slave (using CODESYS Modbus Slave device)
// Variables are accessible via Modbus client at:
//   Holding Register 40001 = arrHoldingRegs[0]
//   Holding Register 40002 = arrHoldingRegs[1]
//   Coil 00001 = arrCoils[0]

END_FUNCTION_BLOCK
```

---

## 4. Debugging Workflow

### 4.1 Systematic Debugging Approach

```
1. REPRODUCE
   - Can the issue be reproduced consistently?
   - What are the exact steps to trigger it?
   - Is it timing-dependent?

2. ISOLATE
   - Which FB/FC is producing the wrong result?
   - Which variable has the unexpected value?
   - When does the value become wrong? (use trace)

3. ANALYZE
   - Trace the variable back to its source
   - Check all assignments and calculations
   - Look for edge cases, overflow, type conversion

4. FIX
   - Make minimal change to fix the issue
   - Add bounds checks or validation
   - Add comments explaining the fix

5. VERIFY
   - Test the fix with the reproduction steps
   - Run existing tests to check for regressions
   - Use trace to confirm correct behavior
```

### 4.2 Debug Checklist

- [ ] Is the FB being called? (check call stack or add nCallCount increment)
- [ ] Is the trigger condition met? (trace bExecute / bEnable)
- [ ] Are input values correct? (trace or watch input variables)
- [ ] Is there an error flag set? (check bError, nErrorCode)
- [ ] Is the array index within bounds? (add IF check with logging)
- [ ] Is the edge detection working? (trace bSignal and bLastSignal)
- [ ] Is the timer configured correctly? (check PT value, IN condition)
- [ ] Is the type conversion correct? (check for truncation, sign issues)
- [ ] Is the task cycle time sufficient? (check PLC_TASK_CYCLIC_EXCEED in log)
- [ ] Is there a circular reference? (check FB call hierarchy)

### 4.3 Log Output Pattern

```iecst
// Simple logging FB for debugging
FUNCTION_BLOCK FB_Logger

VAR_INPUT
    bLog        : BOOL;
    sMessage    : STRING(80);
    nLevel      : UINT := 0;  // 0=INFO, 1=WARN, 2=ERROR
END_VAR

VAR_IN_OUT
    arrLogMsg   : ARRAY[0..99] OF STRING(80);
    arrLogLevel : ARRAY[0..99] OF UINT;
    arrLogTime  : ARRAY[0..99] OF DT;
    nLogCount   : UINT;
    nLogIdx     : UINT;
END_VAR

VAR
    LastLog : BOOL;
END_VAR

IF bLog AND NOT LastLog THEN
    arrLogMsg[nLogIdx]   := sMessage;
    arrLogLevel[nLogIdx] := nLevel;
    arrLogTime[nLogIdx]  := DT#2024-01-01-00:00:00;  // Use system clock
    nLogIdx := (nLogIdx + 1) MOD 100;
    IF nLogCount < 100 THEN
        nLogCount := nLogCount + 1;
    END_IF
END_IF

LastLog := bLog;

END_FUNCTION_BLOCK
```

---

## 5. Common Pitfalls and Solutions

### 5.1 Task Watchdog Timeout

```iecst
// PROBLEM: PLC stops with "Task watchdog exceeded" error
// CAUSE: Scan time > task cycle time

// DIAGNOSIS:
// 1. Check task cycle time in task configuration
// 2. Add scan time measurement:
VAR
    tScanStart : TIME;
    tScanDuration : TIME;
END_VAR

tScanStart := TIME();  // Or use system time
// ... main logic ...
tScanDuration := TIME() - tScanStart;
// Monitor tScanDuration in watch window

// SOLUTION:
// 1. Move heavy computation to slower task
// 2. Split computation across multiple scans (see best-practices.md)
// 3. Optimize loops — reduce iterations or simplify calculations
// 4. Use VAR_IN_OUT instead of copying large arrays
```

### 5.2 Unexpected Boolean Behavior

```iecst
// PROBLEM: BOOL variable is TRUE when it should be FALSE
// CAUSE 1: Multiple assignments to same variable
// In FB_A: bMotorRun := TRUE;
// In FB_B: bMotorRun := FALSE;  // Overrides FB_A
// SOLUTION: Use dedicated command/status variables

// CAUSE 2: Latch not reset
bError := TRUE;  // Set on error
// ... error condition clears ...
// bError is still TRUE because nothing resets it
// SOLUTION: Explicit reset
IF bReset THEN
    bError := FALSE;
END_IF

// CAUSE 3: VAR_RETAIN holding old value
// After download, retained variables keep previous values
// SOLUTION: Clear retain or explicitly initialize
```

### 5.3 Communication Issues

```iecst
// PROBLEM: Modbus communication not working
// CHECKLIST:
// 1. IP address correct? (ping the device)
// 2. Port correct? (502 for Modbus TCP)
// 3. Unit ID correct? (check device documentation)
// 4. Function code correct? (03 for read holding, 16 for write multiple)
// 5. Register address correct? (check device register map)
// 6. Byte order correct? (big-endian for Modbus)
// 7. Firewall blocking? (check Windows/network firewall)
// 8. Cable connected? (check physical connection)

// Debug: Log raw Modbus frames
VAR
    arrDebugTx : ARRAY[0..259] OF BYTE;
    arrDebugRx : ARRAY[0..259] OF BYTE;
    nDebugTxLen : UINT;
    nDebugRxLen : UINT;
    bLogFrame   : BOOL;
END_VAR

// Copy TX/RX buffers to debug arrays for inspection
IF bLogFrame THEN
    arrDebugTx := arrTxBuffer;
    nDebugTxLen := nTxLength;
    // After receive:
    arrDebugRx := arrRxBuffer;
    nDebugRxLen := nRxLength;
    bLogFrame := FALSE;
END_IF
```

### 5.4 Memory Issues

```iecst
// PROBLEM: PLC crashes or behaves erratically after running for a while
// CAUSE: Memory leak or excessive memory usage

// CHECKLIST:
// 1. Check available memory: SystemInfo.fbSysMemInfo
// 2. Avoid STRING concatenation in cyclic code (creates garbage)
// 3. Use fixed-size arrays, not dynamic allocation
// 4. Check for unbounded array growth

// Memory monitoring
VAR
    fbMemInfo : SysMemInfo;
    nFreeMem  : UDINT;
END_VAR

fbMemInfo();
nFreeMem := fbMemInfo.uliAvailUserMem;
// Log or display nFreeMem to track memory consumption over time
```
