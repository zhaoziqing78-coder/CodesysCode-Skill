# CODESYS FB Templates

## Template 1: Data Acquisition Function Block

> **Trigger mechanism**: Uses absolute difference (`ABS(TriggerSource - LastCaptured) >= StepInterval`).
> Do NOT use direction-based triggering (`AngleDir` / `NextTarget`) — direction jitter causes duplicate captures.

```iecst
FUNCTION_BLOCK FB_DataAcquisition

VAR_INPUT
    bEnable       : BOOL;          // Start/stop control (edge-triggered)
    bReset        : BOOL;          // Reset all data + zero arrays
    SensorValue   : LREAL;        // Input signal to record
    TriggerSource : LREAL;        // Signal that triggers capture (e.g., angle)
    MaxArrayLength: UINT := 1000; // Max capture points
    StepInterval  : LREAL := 1.0; // Trigger interval
    bUseCurrentStart: BOOL := TRUE;
    ManualStart   : LREAL;        // Manual start value (if bUseCurrentStart=FALSE)
END_VAR

VAR_OUTPUT
    bRunning      : BOOL;
    bDone         : BOOL;
    bError        : BOOL;
    ErrorCode     : UINT;
    ActualLength  : UINT;
    DeltaValue    : LREAL;        // Current diff from last capture (monitoring)
END_VAR

VAR
    StartValue    : LREAL;
    LastCaptured  : LREAL;        // Last captured trigger value (for ABS diff)
    LastEnable    : BOOL;
    IsFirstPoint  : BOOL;
    AbsDelta      : LREAL;        // Internal: absolute difference
    ClearIdx      : UINT;         // Array clear loop variable
END_VAR

VAR CONSTANT
    ERR_OK     : UINT := 0;
    ERR_FULL   : UINT := 1;
    ERR_PARAM  : UINT := 2;
    MAX_SIZE   : UINT := 10000;
END_VAR

VAR_IN_OUT
    DataArray : ARRAY[0..9999] OF LREAL;
END_VAR

// Reset (clears state + zeroes array)
IF bReset THEN
    bRunning := FALSE; bDone := FALSE; bError := FALSE;
    ErrorCode := ERR_OK; ActualLength := 0;
    StartValue := 0; LastCaptured := 0;
    LastEnable := FALSE; IsFirstPoint := FALSE;
    DeltaValue := 0; AbsDelta := 0;
    FOR ClearIdx := 0 TO MAX_SIZE - 1 DO
        DataArray[ClearIdx] := 0.0;
    END_FOR;
    RETURN;
END_IF

// Param check
IF StepInterval <= 0 THEN
    bError := TRUE; ErrorCode := ERR_PARAM; RETURN;
END_IF

IF MaxArrayLength > MAX_SIZE THEN
    MaxArrayLength := MAX_SIZE;
END_IF

// Edge detection
IF bEnable AND NOT LastEnable THEN
    bRunning := TRUE; bDone := FALSE; bError := FALSE;
    ErrorCode := ERR_OK; ActualLength := 0;
    IsFirstPoint := TRUE;
    IF bUseCurrentStart THEN
        StartValue := TriggerSource;
    ELSE
        StartValue := ManualStart;
    END_IF
ELSIF NOT bEnable AND LastEnable THEN
    IF bRunning THEN
        bRunning := FALSE; bDone := TRUE;
    END_IF
END_IF
LastEnable := bEnable;

// Capture (ABS difference trigger — no direction needed)
IF bRunning THEN
    // First point: immediate
    IF IsFirstPoint AND (ActualLength < MaxArrayLength) THEN
        DataArray[ActualLength] := SensorValue;
        LastCaptured := TriggerSource;
        ActualLength := ActualLength + 1;
        IsFirstPoint := FALSE;
        DeltaValue := 0;
    END_IF

    // Subsequent points: ABS diff >= StepInterval
    IF NOT IsFirstPoint AND (ActualLength < MaxArrayLength) THEN
        AbsDelta := TriggerSource - LastCaptured;
        IF AbsDelta < 0.0 THEN
            AbsDelta := -AbsDelta;
        END_IF;
        DeltaValue := AbsDelta;

        IF AbsDelta >= StepInterval THEN
            DataArray[ActualLength] := SensorValue;
            LastCaptured := TriggerSource;
            ActualLength := ActualLength + 1;
            DeltaValue := 0;
        END_IF
    END_IF

    // Full check
    IF ActualLength >= MaxArrayLength THEN
        bRunning := FALSE; bDone := TRUE;
        bError := TRUE; ErrorCode := ERR_FULL;
    END_IF
END_IF

END_FUNCTION_BLOCK
```

## Template 2: Processing/Fitting Function Block

```iecst
FUNCTION_BLOCK FB_DataProcessor

VAR_INPUT
    bExecute      : BOOL;          // Trigger calculation
    bReset        : BOOL;
    DataLength    : UINT;          // Valid data length
    SmoothWindow  : UINT := 3;     // Moving average window (odd)
END_VAR

VAR_OUTPUT
    bDone         : BOOL;
    bValid        : BOOL;
    bError        : BOOL;
    ErrorCode     : UINT;
    Result        : LREAL;        // Calculated result (e.g., slope)
END_VAR

VAR
    LastExecute   : BOOL;
    i             : INT;
    j             : INT;
    Sum           : LREAL;
    Half          : INT;
    StartIdx       : INT;
    EndIdx         : INT;
END_VAR

VAR CONSTANT
    ERR_OK     : UINT := 0;
    ERR_DATA   : UINT := 1;  // Insufficient data
    ERR_CALC   : UINT := 2;  // Calculation error
END_VAR

VAR_IN_OUT
    SrcArray : ARRAY[0..9999] OF LREAL;
    OutArray : ARRAY[0..9999] OF LREAL;
END_VAR

// Reset
IF bReset THEN
    bDone := FALSE; bValid := FALSE; bError := FALSE;
    ErrorCode := ERR_OK; Result := 0;
    LastExecute := FALSE;
    RETURN;
END_IF

// Edge-triggered execution
IF bExecute AND NOT LastExecute THEN
    bDone := FALSE; bValid := FALSE; bError := FALSE;
    
    // Validate
    IF DataLength < 2 THEN
        bError := TRUE; ErrorCode := ERR_DATA;
        LastExecute := bExecute;
        RETURN;
    END_IF

    // --- Calculate result (e.g., slope) ---
    Result := (SrcArray[DataLength-1] - SrcArray[0]) / (DataLength - 1);
    
    // --- Generate output (e.g., moving average) ---
    Half := INT#(SmoothWindow - 1) / 2;
    FOR i := 0 TO INT#(DataLength - 1) DO
        StartIdx := i - Half;
        EndIdx := i + Half;
        IF StartIdx < 0 THEN StartIdx := 0; END_IF
        IF EndIdx > INT#(DataLength - 1) THEN EndIdx := INT#(DataLength - 1); END_IF
        Sum := 0;
        FOR j := StartIdx TO EndIdx DO
            Sum := Sum + SrcArray[j];
        END_FOR
        OutArray[i] := Sum / (EndIdx - StartIdx + 1);
    END_FOR

    bDone := TRUE; bValid := TRUE;
END_IF

LastExecute := bExecute;

END_FUNCTION_BLOCK
```

## Template 3: Test Program

```iecst
PROGRAM PRG_Test

VAR
    MyFB     : FB_DataAcquisition;
    MyProc   : FB_DataProcessor;
    
    arrData  : ARRAY[0..9999] OF LREAL;
    arrOut   : ARRAY[0..9999] OF LREAL;
    
    btnStart : BOOL;
    btnReset : BOOL;
    btnCalc  : BOOL;
    
    // Status
    bRunning : BOOL;
    bDone    : BOOL;
    nCount   : UINT;
    rResult  : LREAL;
END_VAR

// Acquisition
MyFB.bEnable := btnStart;
MyFB.bReset := btnReset;
MyFB.SensorValue := rSensorInput;
MyFB.TriggerSource := rTriggerInput;
MyFB.MaxArrayLength := 500;
MyFB.StepInterval := 10.0;
MyFB(DataArray := arrData);

bRunning := MyFB.bRunning;
bDone := MyFB.bDone;
nCount := MyFB.ActualLength;

// Processing
MyProc.bExecute := btnCalc;
MyProc.bReset := btnReset;
MyProc.DataLength := nCount;
MyProc.SmoothWindow := 3;
MyProc(SrcArray := arrData, OutArray := arrOut);

rResult := MyProc.Result;

IF btnReset THEN
    btnCalc := FALSE;
END_IF

END_PROGRAM
```

## Template 4: Communication Function Block (Modbus TCP Read)

```iecst
FUNCTION_BLOCK FB_ModbusRead

VAR_INPUT
    bExecute     : BOOL;          // Trigger read (edge-triggered)
    sIpAddr      : STRING(15);    // Target IP address
    nPort        : UINT := 502;   // Modbus TCP port
    nUnitId      : BYTE := 1;     // Slave unit ID
    nStartAddr   : UINT;          // Starting register address
    nQuantity    : UINT;          // Number of registers to read
    tTimeout     : TIME := T#3S;  // Communication timeout
END_VAR

VAR_OUTPUT
    bDone        : BOOL;
    bBusy        : BOOL;
    bError       : BOOL;
    nErrorCode   : UINT;
    nDataCount   : UINT;          // Number of valid registers read
END_VAR

VAR_IN_OUT
    arrData      : ARRAY[0..99] OF WORD;  // Output register data
END_VAR

VAR
    LastExecute  : BOOL;
    nState       : UINT := 0;
    fbSocket     : SYSSOCKET;
    hSocket      : SYSHANDLE;
    arrTxBuffer  : ARRAY[0..259] OF BYTE;
    arrRxBuffer  : ARRAY[0..259] OF BYTE;
    nTxLen       : UINT;
    nRxLen       : UINT;
    nTransId     : UINT := 1;
    tTimer       : TON;
    i            : UINT;
END_VAR

VAR CONSTANT
    ERR_OK       : UINT := 0;
    ERR_PARAM    : UINT := 2;
    ERR_TIMEOUT  : UINT := 3;
    ERR_COMM     : UINT := 4;
    ERR_CRC      : UINT := 5;

    ST_IDLE      : UINT := 0;
    ST_CONNECT   : UINT := 10;
    ST_SEND      : UINT := 20;
    ST_RECV      : UINT := 30;
    ST_PARSE     : UINT := 40;
    ST_CLOSE     : UINT := 50;
    ST_ERROR     : UINT := 99;
END_VAR

// Edge-triggered execution
IF bExecute AND NOT LastExecute THEN
    // Validate parameters
    IF nQuantity = 0 OR nQuantity > 100 OR sIpAddr = '' THEN
        bError := TRUE;
        nErrorCode := ERR_PARAM;
        LastExecute := bExecute;
        RETURN;
    END_IF

    bDone := FALSE;
    bBusy := TRUE;
    bError := FALSE;
    nErrorCode := ERR_OK;
    nDataCount := 0;
    nState := ST_CONNECT;
END_IF
LastExecute := bExecute;

// State machine
CASE nState OF
    ST_IDLE:
        // Waiting for execute

    ST_CONNECT:
        // Open TCP socket and connect
        // (Implementation depends on specific SysSocket library)
        // On success: nState := ST_SEND
        // On failure: nState := ST_ERROR; nErrorCode := ERR_COMM

        // Build Modbus TCP request
        arrTxBuffer[0] := BYTE#(nTransId / 256);
        arrTxBuffer[1] := BYTE#(nTransId MOD 256);
        arrTxBuffer[2] := 0;  // Protocol ID
        arrTxBuffer[3] := 0;
        arrTxBuffer[4] := 0;  // Length = 6
        arrTxBuffer[5] := 6;
        arrTxBuffer[6] := nUnitId;
        arrTxBuffer[7] := 3;  // Function code 03 (Read Holding)
        arrTxBuffer[8] := BYTE#(nStartAddr / 256);
        arrTxBuffer[9] := BYTE#(nStartAddr MOD 256);
        arrTxBuffer[10] := BYTE#(nQuantity / 256);
        arrTxBuffer[11] := BYTE#(nQuantity MOD 256);
        nTxLen := 12;
        nTransId := nTransId + 1;

        tTimer(IN := TRUE, PT := tTimeout);
        nState := ST_SEND;

    ST_SEND:
        // Send TCP data (SysSocket.Send)
        // On success: nState := ST_RECV
        // On failure: nState := ST_ERROR; nErrorCode := ERR_COMM
        // On timeout: nState := ST_ERROR; nErrorCode := ERR_TIMEOUT

        IF tTimer.Q THEN
            nState := ST_ERROR;
            nErrorCode := ERR_TIMEOUT;
        END_IF

    ST_RECV:
        // Receive TCP data (SysSocket.Recv)
        // On success: nState := ST_PARSE
        // On failure: nState := ST_ERROR; nErrorCode := ERR_COMM
        // On timeout: nState := ST_ERROR; nErrorCode := ERR_TIMEOUT

        IF tTimer.Q THEN
            nState := ST_ERROR;
            nErrorCode := ERR_TIMEOUT;
        END_IF

    ST_PARSE:
        // Parse Modbus TCP response
        // Response: MBAP(7) + FC(1) + ByteCount(1) + Data(n*2)

        // Verify transaction ID
        // Verify function code (no exception 0x83)
        // Extract register data
        FOR i := 0 TO nQuantity - 1 DO
            arrData[i] := WORD#(arrRxBuffer[9 + i*2]) * 256
                        + WORD#(arrRxBuffer[10 + i*2]);
        END_FOR

        nDataCount := nQuantity;
        tTimer(IN := FALSE);
        nState := ST_CLOSE;

    ST_CLOSE:
        // Close socket
        bDone := TRUE;
        bBusy := FALSE;
        nState := ST_IDLE;

    ST_ERROR:
        // Error handling
        tTimer(IN := FALSE);
        bDone := FALSE;
        bBusy := FALSE;
        bError := TRUE;
        nState := ST_IDLE;

END_CASE

END_FUNCTION_BLOCK
```

## Template 5: PID Controller Function Block

```iecst
FUNCTION_BLOCK FB_PIDController

VAR_INPUT
    rSetPoint   : LREAL;        // Desired value
    rActValue   : LREAL;        // Measured value (feedback)
    bEnable     : BOOL;         // Enable PID control
    bReset      : BOOL;         // Reset integrator
    rKp         : LREAL := 1.0; // Proportional gain
    rTi         : TIME := T#1S; // Integral time (reset time)
    rTd         : TIME := T#0S; // Derivative time (rate time)
    rOutputMin  : LREAL := 0.0; // Output lower limit
    rOutputMax  : LREAL := 100.0; // Output upper limit
    tSampleTime : TIME := T#10MS; // Sample interval
END_VAR

VAR_OUTPUT
    rOutput     : LREAL;        // Control output
    rError      : LREAL;        // Current error (SP - PV)
    rP_Part     : LREAL;        // Proportional contribution
    rI_Part     : LREAL;        // Integral contribution
    rD_Part     : LREAL;        // Derivative contribution
    bOutputLimited : BOOL;      // Output at limit
END_VAR

VAR
    rIntegral   : LREAL;        // Integral accumulator
    rLastError  : LREAL;        // Previous error for derivative
    tonSample   : TON;          // Sample timer
    bFirstRun   : BOOL := TRUE;
    rTiReal     : LREAL;        // Ti in seconds
    rTdReal     : LREAL;        // Td in seconds
    rSampleReal : LREAL;        // Sample time in seconds
END_VAR

// Reset
IF bReset OR NOT bEnable THEN
    rIntegral := 0.0;
    rLastError := 0.0;
    bFirstRun := TRUE;
    rOutput := 0.0;
    rP_Part := 0.0;
    rI_Part := 0.0;
    rD_Part := 0.0;
    rError := 0.0;
    bOutputLimited := FALSE;
    tonSample(IN := FALSE);
END_IF

// Sample timer
tonSample(IN := bEnable, PT := tSampleTime);

IF tonSample.Q AND bEnable THEN
    tonSample(IN := FALSE, PT := tSampleTime);

    // Convert times to seconds
    rTiReal := TIME_TO_LREAL(rTi) / 1000.0;
    rTdReal := TIME_TO_LREAL(rTd) / 1000.0;
    rSampleReal := TIME_TO_LREAL(tSampleTime) / 1000.0;

    // Calculate error
    rError := rSetPoint - rActValue;

    // Proportional part
    rP_Part := rKp * rError;

    // Integral part (with anti-windup)
    IF rTiReal > 0.0 THEN
        rI_Part := rKp * (rSampleReal / rTiReal) * rError;
        rIntegral := rIntegral + rI_Part;
    ELSE
        rI_Part := 0.0;
    END_IF

    // Derivative part
    IF NOT bFirstRun AND rTdReal > 0.0 THEN
        rD_Part := rKp * (rTdReal / rSampleReal) * (rError - rLastError);
    ELSE
        rD_Part := 0.0;
    END_IF
    rLastError := rError;
    bFirstRun := FALSE;

    // Calculate raw output
    rOutput := rP_Part + rIntegral + rD_Part;

    // Output limiting with anti-windup
    IF rOutput > rOutputMax THEN
        rOutput := rOutputMax;
        bOutputLimited := TRUE;
        // Anti-windup: clamp integral
        rIntegral := rOutputMax - rP_Part - rD_Part;
        IF rIntegral < 0.0 THEN rIntegral := 0.0; END_IF
    ELSIF rOutput < rOutputMin THEN
        rOutput := rOutputMin;
        bOutputLimited := TRUE;
        rIntegral := rOutputMin - rP_Part - rD_Part;
        IF rIntegral > 0.0 THEN rIntegral := 0.0; END_IF
    ELSE
        bOutputLimited := FALSE;
    END_IF
END_IF

END_FUNCTION_BLOCK
```

## Template 6: Alarm Handler Function Block

```iecst
FUNCTION_BLOCK FB_AlarmHandler

VAR_INPUT
    bEnable     : BOOL;
    bReset      : BOOL;         // Acknowledge all alarms
END_VAR

VAR_IN_OUT
    // Alarm trigger inputs (from application)
    arrAlarmTriggers : ARRAY[0..31] OF BOOL;
    arrAlarmAck      : ARRAY[0..31] OF BOOL;
    arrAlarmActive   : ARRAY[0..31] OF BOOL;
    arrAlarmLatched  : ARRAY[0..31] OF BOOL;
    nActiveCount     : UINT;
END_VAR

VAR
    i           : UINT;
    LastReset   : BOOL;
END_VAR

// Global reset (acknowledge all)
IF bReset AND NOT LastReset THEN
    FOR i := 0 TO 31 DO
        arrAlarmLatched[i] := FALSE;
    END_FOR
    nActiveCount := 0;
END_IF
LastReset := bReset;

IF bEnable THEN
    nActiveCount := 0;

    // Process each alarm
    FOR i := 0 TO 31 DO
        // Set active when trigger is TRUE
        IF arrAlarmTriggers[i] THEN
            arrAlarmActive[i] := TRUE;
            arrAlarmLatched[i] := TRUE;
        END_IF

        // Clear active when trigger is FALSE
        IF NOT arrAlarmTriggers[i] THEN
            arrAlarmActive[i] := FALSE;
        END_IF

        // Individual acknowledge
        IF arrAlarmAck[i] THEN
            arrAlarmLatched[i] := FALSE;
            arrAlarmAck[i] := FALSE;
        END_IF

        // Count active alarms
        IF arrAlarmActive[i] THEN
            nActiveCount := nActiveCount + 1;
        END_IF
    END_FOR
END_IF

END_FUNCTION_BLOCK
```

## Template 7: Generic State Machine Function Block

```iecst
FUNCTION_BLOCK FB_StateMachine

VAR_INPUT
    bStart      : BOOL;
    bStop       : BOOL;
    bReset      : BOOL;
    bError      : BOOL;        // External error condition
    tStepTimeout: TIME := T#30S; // Timeout per step
END_VAR

VAR_OUTPUT
    nState      : UINT;
    nLastState  : UINT;
    bRunning    : BOOL;
    bDone       : BOOL;
    bError      : BOOL;
    nErrorCode  : UINT;
    tStateTime  : TIME;        // Time in current state
END_VAR

VAR
    LastStart   : BOOL;
    LastStop    : BOOL;
    tonTimeout  : TON;
    tonStateTime: TON;
END_VAR

VAR CONSTANT
    ST_IDLE     : UINT := 0;
    ST_INIT     : UINT := 10;
    ST_STEP1    : UINT := 20;
    ST_STEP2    : UINT := 30;
    ST_STEP3    : UINT := 40;
    ST_COMPLETE : UINT := 50;
    ST_ERROR    : UINT := 99;

    ERR_NONE    : UINT := 0;
    ERR_TIMEOUT : UINT := 3;
    ERR_EXTERNAL: UINT := 10;
END_VAR

// Reset
IF bReset THEN
    nState := ST_IDLE;
    nLastState := ST_IDLE;
    bRunning := FALSE;
    bDone := FALSE;
    bError := FALSE;
    nErrorCode := ERR_NONE;
    tonTimeout(IN := FALSE);
    tonStateTime(IN := FALSE);
    RETURN;
END_IF

// Stop
IF bStop AND bRunning THEN
    nLastState := nState;
    nState := ST_IDLE;
    bRunning := FALSE;
    tonTimeout(IN := FALSE);
    RETURN;
END_IF

// External error
IF bError AND nState <> ST_ERROR AND nState <> ST_IDLE THEN
    nLastState := nState;
    nState := ST_ERROR;
    nErrorCode := ERR_EXTERNAL;
    bRunning := FALSE;
    bError := TRUE;
    tonTimeout(IN := FALSE);
    RETURN;
END_IF

// State time
tonStateTime(IN := (nState <> ST_IDLE));

// State machine
CASE nState OF
    ST_IDLE:
        bRunning := FALSE;
        bDone := FALSE;
        tonStateTime(IN := FALSE);
        tStateTime := T#0S;

        IF bStart AND NOT LastStart THEN
            nState := ST_INIT;
            bRunning := TRUE;
        END_IF

    ST_INIT:
        // Initialization step
        tonTimeout(IN := TRUE, PT := tStepTimeout);
        IF tonTimeout.Q THEN
            nState := ST_ERROR;
            nErrorCode := ERR_TIMEOUT;
        ELSE
            // Perform init, then advance
            nState := ST_STEP1;
        END_IF

    ST_STEP1:
        tonTimeout(IN := TRUE, PT := tStepTimeout);
        IF tonTimeout.Q THEN
            nState := ST_ERROR;
            nErrorCode := ERR_TIMEOUT;
        ELSIF bStep1Done THEN
            nState := ST_STEP2;
        END_IF

    ST_STEP2:
        tonTimeout(IN := TRUE, PT := tStepTimeout);
        IF tonTimeout.Q THEN
            nState := ST_ERROR;
            nErrorCode := ERR_TIMEOUT;
        ELSIF bStep2Done THEN
            nState := ST_STEP3;
        END_IF

    ST_STEP3:
        tonTimeout(IN := TRUE, PT := tStepTimeout);
        IF tonTimeout.Q THEN
            nState := ST_ERROR;
            nErrorCode := ERR_TIMEOUT;
        ELSIF bStep3Done THEN
            nState := ST_COMPLETE;
        END_IF

    ST_COMPLETE:
        bDone := TRUE;
        bRunning := FALSE;
        tonTimeout(IN := FALSE);
        // Auto-return to idle after completion
        nState := ST_IDLE;

    ST_ERROR:
        bError := TRUE;
        bRunning := FALSE;
        tonTimeout(IN := FALSE);
        // Wait for reset

END_CASE

LastStart := bStart;
LastStop := bStop;

END_FUNCTION_BLOCK
```

## Template 8: Visualization Data Provider FB

```iecst
FUNCTION_BLOCK FB_VisuDataProvider

VAR_INPUT
    bEnable     : BOOL;
END_VAR

VAR_IN_OUT
    // Data arrays for visualization
    arrTrendData : ARRAY[0..199] OF LREAL;  // Trend buffer
    nTrendIdx    : UINT;                     // Current write index
    nTrendCount  : UINT;                     // Valid data count

    // Status data for HMI
    stStatus     : ST_SystemStatus;
END_VAR

VAR
    tonSample    : TON;
    i            : UINT;
END_VAR

// Status structure
TYPE ST_SystemStatus :
STRUCT
    rTemperature   : LREAL;
    rPressure      : LREAL;
    rMotorSpeed    : LREAL;
    nMotorState    : UINT;
    bAlarmActive   : BOOL;
    nAlarmCode     : UINT;
    sAlarmMessage  : STRING(80);
    dtLastUpdate   : DATE_AND_TIME;
END_STRUCT
END_TYPE

IF bEnable THEN
    // Sample trend data at 500ms intervals
    tonSample(IN := TRUE, PT := T#500MS);
    IF tonSample.Q THEN
        tonSample(IN := FALSE);

        // Store in circular buffer
        arrTrendData[nTrendIdx] := stStatus.rTemperature;
        nTrendIdx := (nTrendIdx + 1) MOD 200;
        IF nTrendCount < 200 THEN
            nTrendCount := nTrendCount + 1;
        END_IF
    END_IF

    // Update status (called every scan)
    stStatus.dtLastUpdate := DATE_AND_TIME#2024-01-01-00:00:00;  // Use system clock

    // Format alarm message
    IF stStatus.bAlarmActive THEN
        CASE stStatus.nAlarmCode OF
            1:  stStatus.sAlarmMessage := 'Temperature High';
            2:  stStatus.sAlarmMessage := 'Pressure Low';
            3:  stStatus.sAlarmMessage := 'Motor Overload';
            4:  stStatus.sAlarmMessage := 'Communication Error';
            ELSE
                stStatus.sAlarmMessage := 'Unknown Alarm';
        END_CASE
    ELSE
        stStatus.sAlarmMessage := 'System OK';
    END_IF
END_IF

END_FUNCTION_BLOCK
```

## Template 9: Test Program for All FBs

```iecst
PROGRAM PRG_TestAll

VAR
    // FBs under test
    fbAcq        : FB_DataAcquisition;
    fbProc       : FB_DataProcessor;
    fbModbus     : FB_ModbusRead;
    fbPID        : FB_PIDController;
    fbAlarm      : FB_AlarmHandler;
    fbStateMachine: FB_StateMachine;
    fbVisu       : FB_VisuDataProvider;

    // Test data arrays
    arrAcqData   : ARRAY[0..9999] OF LREAL;
    arrProcOut   : ARRAY[0..9999] OF LREAL;
    arrModbusData: ARRAY[0..99] OF WORD;
    arrTrendData : ARRAY[0..199] OF LREAL;

    // Alarm arrays
    arrAlarmTrig : ARRAY[0..31] OF BOOL;
    arrAlarmAck  : ARRAY[0..31] OF BOOL;
    arrAlarmAct  : ARRAY[0..31] OF BOOL;
    arrAlarmLat  : ARRAY[0..31] OF BOOL;

    // Status struct
    stSysStatus  : ST_SystemStatus;

    // Simulated inputs
    rSimAngle    : LREAL := 0.0;
    rSimEnc      : LREAL := 100.0;
    rSimTemp     : LREAL := 25.0;
    rSimPress    : LREAL := 5.0;

    // Control buttons
    bStartAcq    : BOOL;
    bResetAcq    : BOOL;
    bStartCalc   : BOOL;
    bStartModbus : BOOL;
    bStartPID    : BOOL;
    bResetAll    : BOOL;
    bStartSM     : BOOL;
    bStopSM      : BOOL;
END_VAR

// --- Simulate sensor inputs ---
rSimAngle := rSimAngle + 0.5;
rSimEnc := rSimEnc + 2.3;
rSimTemp := rSimTemp + 0.1;
IF rSimTemp > 90.0 THEN rSimTemp := 25.0; END_IF

// --- Data Acquisition Test ---
fbAcq.bEnable := bStartAcq;
fbAcq.bReset := bResetAcq;
fbAcq.SensorValue := rSimEnc;
fbAcq.TriggerSource := rSimAngle;
fbAcq.MaxArrayLength := 500;
fbAcq.StepInterval := 10.0;
fbAcq(DataArray := arrAcqData);

// --- Data Processing Test ---
fbProc.bExecute := bStartCalc;
fbProc.bReset := bResetAcq;
fbProc.DataLength := fbAcq.ActualLength;
fbProc.SmoothWindow := 5;
fbProc(SrcArray := arrAcqData, OutArray := arrProcOut);

// --- PID Controller Test ---
fbPID.rSetPoint := 50.0;
fbPID.rActValue := rSimTemp;
fbPID.bEnable := bStartPID;
fbPID.bReset := bResetAll;
fbPID.rKp := 2.0;
fbPID.rTi := T#5S;
fbPID.rTd := T#500MS;
fbPID.rOutputMin := 0.0;
fbPID.rOutputMax := 100.0;
fbPID.tSampleTime := T#100MS;
// Note: PID output would control a heater in real application

// --- Alarm Handler Test ---
arrAlarmTrig[0] := (rSimTemp > 80.0);
arrAlarmTrig[1] := (rSimPress < 2.0);
fbAlarm.bEnable := TRUE;
fbAlarm.bReset := bResetAll;
fbAlarm(
    arrAlarmTriggers := arrAlarmTrig,
    arrAlarmAck := arrAlarmAck,
    arrAlarmActive := arrAlarmAct,
    arrAlarmLatched := arrAlarmLat,
    nActiveCount := nAlarmCount
);

// --- State Machine Test ---
fbStateMachine.bStart := bStartSM;
fbStateMachine.bStop := bStopSM;
fbStateMachine.bReset := bResetAll;
fbStateMachine.tStepTimeout := T#10S;

// --- Visualization Data Provider Test ---
stSysStatus.rTemperature := rSimTemp;
stSysStatus.rPressure := rSimPress;
stSysStatus.bAlarmActive := (fbAlarm.nActiveCount > 0);
stSysStatus.nAlarmCode := 1;

fbVisu.bEnable := TRUE;
fbVisu(
    arrTrendData := arrTrendData,
    nTrendIdx := nTrendIdx,
    nTrendCount := nTrendCount,
    stStatus := stSysStatus
);

END_PROGRAM
```
