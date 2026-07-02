# CODESYS FB Templates

## Template 1: Data Acquisition Function Block

```iecst
FUNCTION_BLOCK FB_DataAcquisition

VAR_INPUT
    bEnable       : BOOL;          // Start/stop control (edge-triggered)
    bReset        : BOOL;          // Reset all data
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
END_VAR

VAR
    StartValue    : LREAL;
    NextTarget    : LREAL;
    LastEnable    : BOOL;
    IsFirstPoint  : BOOL;
    MoveDir       : LREAL;
    LastTrigger   : LREAL;
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

// Reset
IF bReset THEN
    bRunning := FALSE; bDone := FALSE; bError := FALSE;
    ErrorCode := ERR_OK; ActualLength := 0;
    StartValue := 0; NextTarget := 0;
    LastEnable := FALSE; IsFirstPoint := FALSE;
    MoveDir := 1.0; LastTrigger := 0;
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
    IsFirstPoint := TRUE; MoveDir := 1.0;
    LastTrigger := TriggerSource;
    IF bUseCurrentStart THEN
        StartValue := TriggerSource;
    ELSE
        StartValue := ManualStart;
    END_IF
    NextTarget := StartValue;
ELSIF NOT bEnable AND LastEnable THEN
    IF bRunning THEN
        bRunning := FALSE; bDone := TRUE;
    END_IF
END_IF
LastEnable := bEnable;

// Capture
IF bRunning THEN
    // Detect direction
    IF TriggerSource > LastTrigger THEN
        MoveDir := 1.0;
    ELSIF TriggerSource < LastTrigger THEN
        MoveDir := -1.0;
    END_IF
    LastTrigger := TriggerSource;

    // First point
    IF IsFirstPoint AND (ActualLength < MaxArrayLength) THEN
        DataArray[ActualLength] := SensorValue;
        ActualLength := ActualLength + 1;
        NextTarget := StartValue + StepInterval * MoveDir;
        IsFirstPoint := FALSE;
    END_IF

    // Subsequent points
    IF NOT IsFirstPoint AND (ActualLength < MaxArrayLength) THEN
        IF (MoveDir > 0 AND TriggerSource >= NextTarget) OR
           (MoveDir < 0 AND TriggerSource <= NextTarget) THEN
            DataArray[ActualLength] := SensorValue;
            ActualLength := ActualLength + 1;
            NextTarget := NextTarget + StepInterval * MoveDir;
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
