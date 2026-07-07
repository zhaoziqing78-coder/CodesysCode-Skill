# CODESYS Best Practices and Conventions

## 1. Naming Conventions

### 1.1 Variable Naming (Hungarian-style Prefixes)

| Prefix | Data Type | Example | Notes |
|--------|-----------|---------|-------|
| `b` | BOOL | `bEnable`, `bError`, `bDone` | On/off, status flags |
| `n` | INT/UINT/DINT | `nState`, `nCount`, `nErrorCode` | Integer counters, states |
| `r` | REAL/LREAL | `rPosition`, `rTemperature`, `rSpeed` | Floating-point values |
| `s` | STRING | `sMessage`, `sAlarmText`, `sIpAddr` | Text strings |
| `t` | TIME | `tDelay`, `tTimeout`, `tSample` | Time durations |
| `dt` | DATE_AND_TIME | `dtStartTime`, `dtLogTime` | Timestamps |
| `arr` | ARRAY | `arrData`, `arrCamTableX`, `arrTrendBuffer` | Arrays |
| `st` | STRUCT | `stConfig`, `stStatus`, `stRecipe` | Structure instances |
| `e` | ENUM | `eMode`, `eState`, `ePage` | Enumeration values |
| `w` | WORD | `wControlWord`, `wStatusWord` | 16-bit bit fields |
| `dw` | DWORD | `dwFlags`, `dwErrorCode` | 32-bit bit fields |
| `by` | BYTE | `byModbusUnit`, `byRegister` | 8-bit values |
| `fb` | FB instance | `fbTimer`, `fbModbusClient`, `fbCommMgr` | Function Block instances |
| `p` | POINTER | `pBuffer`, `pData` | Pointer references |
| `x` | REFERENCE | `xValue`, `xResult` | Reference (VAR_IN_OUT) |

### 1.2 Suffix Conventions

| Suffix | Meaning | Example |
|--------|---------|---------|
| `Enable` | Activation control | `bEnable`, `bCommEnable` |
| `Start` / `Stop` | Action controls | `bStart`, `bStopCmd` |
| `Reset` | Reset action | `bReset`, `bFaultReset` |
| `Done` | Completion flag | `bDone`, `bAcqDone` |
| `Error` | Error flag | `bError`, `bCommError` |
| `Running` | Active state | `bRunning`, `bMotorRunning` |
| `Ready` | Prepared state | `bReady`, `bAxisReady` |
| `Actual` | Current/measured value | `rActualTemp`, `nActualLength` |
| `Target` | Setpoint value | `rTargetTemp`, `rTargetPos` |
| `Cmd` | Command value | `bMotorCmd`, `rSpeedCmd` |
| `Fb` | Feedback value | `bMotorFb`, `rPositionFb` |
| `Max` / `Min` | Limits | `rMaxTemp`, `nMinCount` |
| `Last` | Previous cycle value | `LastEnable`, `rLastValue` |
| `Delta` | Difference | `rDeltaTemp`, `nDeltaCount` |
| `Config` | Configuration data | `stCommConfig`, `rConfigValue` |
| `Status` | Status data | `stAxisStatus`, `nCommStatus` |

### 1.3 POU Naming

| Prefix | Type | Example | Rules |
|--------|------|---------|-------|
| `FB_` | Function Block | `FB_CamTableGenerator`, `FB_ModbusClient` | PascalCase, descriptive noun |
| `FC_` | Function | `FC_CalculateSlope`, `FC_Crc16` | PascalCase, verb-noun |
| `PRG_` | Program | `PRG_Main`, `PRG_MotorControl` | PascalCase, domain name |
| `M_` | Method | `M_Start`, `M_Reset`, `M_GetStatus` | PascalCase, action verb |
| `P_` | Property | `P_IsRunning`, `P_ActualValue` | PascalCase, boolean or noun |
| `ITF_` | Interface | `ITF_ICommDevice` | PascalCase, I-prefix convention |
| `E_` | Enumeration | `E_State`, `E_VisuPage`, `E_CommMode` | PascalCase |

### 1.4 Constant Naming

| Pattern | Example | Notes |
|---------|---------|-------|
| `ERR_` prefix | `ERR_OK`, `ERR_PARAM`, `ERR_TIMEOUT` | Error codes |
| `ST_` prefix | `ST_IDLE`, `ST_RUNNING`, `ST_ERROR` | State constants |
| `MAX_` prefix | `MAX_ARRAY_SIZE`, `MAX_RETRIES` | Upper limits |
| `MIN_` prefix | `MIN_TEMP`, `MIN_PRESSURE` | Lower limits |
| `DEF_` prefix | `DEF_SAMPLE_TIME`, `DEF_TIMEOUT` | Default values |
| `CMD_` prefix | `CMD_START`, `CMD_STOP`, `CMD_RESET` | Command constants |

---

## 2. Project Structure

### 2.1 Recommended Folder Structure

```
ProjectName/
├── PLC_PRG                    // Main program (task entry)
├── Program/
│   ├── PRG_MotorControl       // Motor control logic
│   ├── PRG_TempControl        // Temperature control logic
│   ├── PRG_CommHandler        // Communication handler
│   └── PRG_AlarmHandler       // Alarm management
├── FunctionBlocks/
│   ├── FB_CamTableGenerator   // Cam table FB
│   ├── FB_ModbusClient        // Modbus client FB
│   ├── FB_PIDController       // PID controller FB
│   └── FB_StateMachine        // Generic state machine FB
├── Functions/
│   ├── FC_Crc16               // CRC calculation
│   ├── FC_LinearInterpolation // Interpolation utility
│   └── FC_Clamp               // Value limiting
├── DataTypes/
│   ├── ST_MotorConfig         // Motor configuration struct
│   ├── ST_AlarmEntry          // Alarm entry struct
│   ├── E_OperationMode        // Operation mode enum
│   └── E_CommState            // Communication state enum
├── Globals/
│   ├── VAR_GLOBAL_Main        // Global variables
│   └── VAR_GLOBAL_Constants   // Global constants
├── Visualization/
│   ├── Visu_Main              // Main HMI page
│   ├── Visu_Motor             // Motor control page
│   ├── Visu_Alarm             // Alarm display page
│   └── Visu_Settings          // Settings page
└── AlarmConfig                // Alarm configuration
```

### 2.2 Task Configuration

| Task | Priority | Cycle | Content |
|------|----------|-------|---------|
| MainTask | 1 (high) | 10ms | Motion control, safety logic |
| CommTask | 2 | 50ms | Communication handlers |
| VisuTask | 3 (low) | 100ms | Visualization updates |
| SlowTask | 4 (lowest) | 500ms | Data logging, diagnostics |

### 2.3 Variable Declaration Order

Within each POU, declare variables in this order:

```iecst
FUNCTION_BLOCK FB_Example

VAR_INPUT          // 1. External inputs (from caller)
VAR_OUTPUT         // 2. External outputs (to caller)
VAR_IN_OUT         // 3. Reference parameters (arrays, large data)
VAR                // 4. Internal state variables
VAR CONSTANT       // 5. Constants (error codes, state IDs)
VAR RETAIN         // 6. Retained variables (power-fail safe)

END_FUNCTION_BLOCK
```

---

## 3. Comment Standards

### 3.1 POU Header Comment

```iecst
(*
    FB_CamTableGenerator

    Description:
        Generates a cam table by capturing encoder positions at regular
        angle intervals. Supports bidirectional movement detection.

    Author:        [Name]
    Created:       2024-01-15
    Modified:      2024-03-20 - Added auto-reconnect (v1.2)
    Version:       1.2

    Inputs:
        bEnable        - Start/stop acquisition (edge-triggered)
        TriggerSource  - Signal for trigger interval (e.g., angle)
        StepInterval   - Trigger interval in engineering units

    Outputs:
        bDone          - TRUE when acquisition complete
        ActualLength   - Number of captured points
        ErrorCode      - 0=OK, 1=Full, 2=InvalidParam

    Usage:
        MyFB.bEnable := TRUE;
        MyFB.TriggerSource := rAngle;
        MyFB.StepInterval := 5.0;
        MyFB(DataArray := arrData);

    Dependencies:
        - None (standalone FB)

    Notes:
        - Array must be declared with size >= MaxArrayLength
        - Uses ABS difference trigger (not direction-based)
*)
FUNCTION_BLOCK FB_CamTableGenerator
```

### 3.2 Inline Comments

```iecst
// --- Parameter validation ---
IF StepInterval <= 0 THEN
    bError := TRUE;
    ErrorCode := ERR_PARAM;
    RETURN;
END_IF

// --- Edge detection: start on rising edge of bEnable ---
IF bEnable AND NOT LastEnable THEN
    bRunning := TRUE;
    IsFirstPoint := TRUE;
END_IF
LastEnable := bEnable;

// --- Data capture: use ABS difference to avoid direction jitter ---
AbsDelta := TriggerSource - LastCaptured;
IF AbsDelta < 0.0 THEN
    AbsDelta := -AbsDelta;  // Manual ABS (some compilers don't optimize)
END_IF
```

### 3.3 Section Dividers

Use consistent section markers within large POUs:

```iecst
// ============================================================
// SECTION: Initialization
// ============================================================

// ------------------------------------------------------------
// Sub-section: Parameter validation
// ------------------------------------------------------------

// --- Individual step: Check array bounds ---
```

---

## 4. Code Organization Principles

### 4.1 Single Responsibility

Each FB should have ONE primary function:

| Good | Bad |
|------|-----|
| `FB_CamTableGenerator` (only captures data) | `FB_MotorAndTempAndComm` (does everything) |
| `FB_ModbusClient` (only Modbus communication) | `FB_CommAndAlarm` (communication + alarms) |
| `FB_PIDController` (only PID calculation) | `FB_TempControl` (PID + filtering + logging) |

### 4.2 Encapsulation

Hide internal state, expose only necessary interfaces:

```iecst
FUNCTION_BLOCK FB_MotorControl

VAR_INPUT
    bStart         : BOOL;
    bStop          : BOOL;
    rSpeedSetpoint : LREAL;
END_VAR

VAR_OUTPUT
    bRunning       : BOOL;
    rActualSpeed   : LREAL;
    bError         : BOOL;
    nErrorCode     : UINT;
END_VAR

VAR
    // Internal state — not accessible from outside
    nInternalState : UINT;
    rInternalSpeed : LREAL;
    tStartupTimer  : TON;
    fbFilter       : FB_MovingAverage;
END_VAR

// Internal logic hidden from caller
// Caller only sees inputs and outputs

END_FUNCTION_BLOCK
```

### 4.3 Reusability

Design FBs to be reusable across projects:

```iecst
// Generic, reusable FB
FUNCTION_BLOCK FB_MovingAverage
// Works with any LREAL input, no project-specific logic

// Project-specific wrapper
FUNCTION_BLOCK FB_TempFilter
VAR
    fbAvg : FB_MovingAverage;
END_VAR
// Adds project-specific configuration and alarm logic
```

---

## 5. Error Handling Strategy

### 5.1 Error Code System

```iecst
VAR CONSTANT
    // General errors (0-99)
    ERR_OK           : UINT := 0;
    ERR_UNKNOWN      : UINT := 1;
    ERR_PARAM        : UINT := 2;
    ERR_TIMEOUT      : UINT := 3;
    ERR_NOT_READY    : UINT := 4;
    ERR_BUSY         : UINT := 5;

    // Data errors (100-199)
    ERR_DATA_EMPTY   : UINT := 100;
    ERR_DATA_FULL    : UINT := 101;
    ERR_DATA_INVALID : UINT := 102;
    ERR_DATA_RANGE   : UINT := 103;

    // Communication errors (200-299)
    ERR_COMM_CONNECT : UINT := 200;
    ERR_COMM_SEND    : UINT := 201;
    ERR_COMM_RECV    : UINT := 202;
    ERR_COMM_CRC     : UINT := 203;

    // Motion errors (300-399)
    ERR_AXIS_NOT_READY : UINT := 300;
    ERR_AXIS_MOVING    : UINT := 301;
    ERR_AXIS_LIMIT     : UINT := 302;
END_VAR
```

### 5.2 Error Handling Pattern

```iecst
// Standard error handling pattern in every FB

// 1. Clear previous error on new execution
IF bExecute AND NOT LastExecute THEN
    bError := FALSE;
    nErrorCode := ERR_OK;
END_IF

// 2. Check preconditions
IF NOT bInitialized THEN
    bError := TRUE;
    nErrorCode := ERR_NOT_READY;
    RETURN;
END_IF

IF rSetpoint < 0.0 THEN
    bError := TRUE;
    nErrorCode := ERR_PARAM;
    RETURN;
END_IF

// 3. Try operation with timeout
tTimer(IN := TRUE, PT := tTimeout);
IF tTimer.Q THEN
    bError := TRUE;
    nErrorCode := ERR_TIMEOUT;
    tTimer(IN := FALSE);
    RETURN;
END_IF

// 4. Success
bDone := TRUE;
bError := FALSE;
```

### 5.3 Error Recovery

```iecst
// Auto-recovery pattern for transient errors
VAR
    nRetryCount : UINT;
    tRetryDelay : TON;
END_VAR

IF bError AND nErrorCode = ERR_COMM_TIMEOUT THEN
    // Transient error — attempt auto-recovery
    tRetryDelay(IN := TRUE, PT := T#5S);
    IF tRetryDelay.Q THEN
        tRetryDelay(IN := FALSE);
        IF nRetryCount < 3 THEN
            nRetryCount := nRetryCount + 1;
            bError := FALSE;
            nErrorCode := ERR_OK;
            nState := ST_RECONNECT;
        END_IF
    END_IF
END_IF

// Permanent errors require manual reset
IF bError AND (nErrorCode = ERR_PARAM OR nErrorCode = ERR_DATA_INVALID) THEN
    // Cannot auto-recover — requires bReset
    nRetryCount := 0;
END_IF
```

---

## 6. Safety Considerations

### 6.1 Fail-Safe Design

```iecst
// Outputs should default to safe state on error
IF bError OR NOT bInitialized THEN
    bMotorEnable := FALSE;    // Motor off
    bValveOpen := FALSE;      // Valve closed
    rHeaterOutput := 0.0;     // Heater off
END_IF
```

### 6.2 Watchdog Pattern

```iecst
VAR
    tWatchdog : TON;
    bWatchdogOK : BOOL;
END_VAR

// Reset watchdog each cycle in main program
tWatchdog(IN := TRUE, PT := T#500MS);
bWatchdogOK := NOT tWatchdog.Q;

// In main task
IF NOT bWatchdogOK THEN
    // System is stuck — enter safe state
    bEmergencyStop := TRUE;
END_IF

// Watchdog reset (must be called at end of healthy scan)
tWatchdog(IN := FALSE);
```

### 6.3 Interlock Implementation

```iecst
// Hardware interlocks (highest priority)
IF NOT bSafetyDoorClosed OR bEmergencyStopPressed THEN
    bMotorEnable := FALSE;
    bSystemReady := FALSE;
    RETURN;  // Exit immediately — safety takes priority
END_IF

// Software interlocks (second priority)
IF rTemperature > rMaxTempLimit THEN
    bHeaterEnable := FALSE;
    bOverTempAlarm := TRUE;
END_IF

IF rPressure > rMaxPressLimit THEN
    bPumpEnable := FALSE;
    bOverPressAlarm := TRUE;
END_IF

// Normal operation (lowest priority)
IF bSystemReady AND bStartCmd THEN
    bMotorEnable := TRUE;
END_IF
```

### 6.4 Emergency Stop Sequence

```iecst
CASE nEStopState OF
    0: // MONITORING
        IF bEmergencyStop THEN
            nEStopState := 10;
        END_IF

    10: // STOP_ALL
        // Immediately de-energize all outputs
        bMotorEnable := FALSE;
        bHeaterEnable := FALSE;
        bValveOpen := FALSE;
        rAnalogOutput := 0.0;
        nEStopState := 20;

    20: // WAIT_FOR_RESET
        // Wait for physical reset + no E-stop signal
        IF NOT bEmergencyStop AND bResetCmd THEN
            nEStopState := 30;
        END_IF

    30: // RESET_COMPLETE
        bEStopActive := FALSE;
        nEStopState := 0;

END_CASE

bEStopActive := (nEStopState > 0);
```

---

## 7. Performance Optimization

### 7.1 Task Cycle Optimization

```iecst
// BAD: Heavy computation every scan
FOR i := 0 TO 9999 DO
    arrResult[i] := arrData[i] * rScale + rOffset;
END_FOR

// GOOD: Distribute across multiple scans
IF nProcessIdx < nDataLength THEN
    // Process N items per scan
    FOR i := 0 TO 99 DO
        IF nProcessIdx < nDataLength THEN
            arrResult[nProcessIdx] := arrData[nProcessIdx] * rScale + rOffset;
            nProcessIdx := nProcessIdx + 1;
        END_IF
    END_FOR
ELSE
    bProcessingDone := TRUE;
END_IF
```

### 7.2 Memory Efficiency

```iecst
// Use appropriate data types
bFlag   : BOOL;    // 1 bit, not INT
nCount  : UINT;    // 16 bits, not DINT if < 65535
rValue  : REAL;    // 32 bits, not LREAL if precision allows

// Use VAR_IN_OUT for large arrays (pass by reference)
VAR_IN_OUT
    arrLargeData : ARRAY[0..9999] OF LREAL;  // No copy
END_VAR
```

### 7.3 String Handling

```iecst
// BAD: Dynamic string operations (slow, fragmented memory)
sMessage := CONCAT('Temperature: ', CONCAT(REAL_TO_STRING(rTemp), ' C'));

// GOOD: Use fixed format strings
sMessage := 'Temperature: %.1f C';
// In visualization: bind sMessage as format, rTemp as variable

// For logging, use fixed-size arrays of strings
VAR
    arrLogMessages : ARRAY[0..99] OF STRING(80);
    arrLogValues   : ARRAY[0..99] OF LREAL;
END_VAR
```
