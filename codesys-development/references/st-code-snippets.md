# ST Code Snippets and Utilities

## 1. Edge Detection (Rising/Falling Edge)

```iecst
// Rising edge
IF bSignal AND NOT bLastSignal THEN
    // Action on rising edge
END_IF
bLastSignal := bSignal;

// Falling edge
IF NOT bSignal AND bLastSignal THEN
    // Action on falling edge
END_IF
```

## 2. Array Initialization

```iecst
// Clear array
FOR i := 0 TO 9999 DO
    arrData[i] := 0.0;
END_FOR

// Initialize with specific value
FOR i := 0 TO 9999 DO
    arrData[i] := 1.0;
END_FOR
```

## 3. Moving Average (Generic)

```iecst
FUNCTION_BLOCK FB_MovingAverage
VAR_INPUT
    rInput   : LREAL;
    nWindow  : UINT := 5;
END_VAR
VAR_OUTPUT
    rOutput  : LREAL;
END_VAR
VAR
    arrBuf   : ARRAY[0..99] OF LREAL;
    nIdx     : UINT;
    nCount   : UINT;
    rSum     : LREAL;
    i        : UINT;
END_VAR

// Add new value
arrBuf[nIdx] := rInput;
nIdx := (nIdx + 1) MOD nWindow;
IF nCount < nWindow THEN
    nCount := nCount + 1;
END_IF

// Calculate average
rSum := 0;
FOR i := 0 TO nCount - 1 DO
    rSum := rSum + arrBuf[i];
END_FOR
rOutput := rSum / nCount;
```

## 4. Clamp / Limit

```iecst
// Clamp value between min and max
rClamped := LIMIT(rMin, rValue, rMax);

// Or manual
IF rValue < rMin THEN
    rClamped := rMin;
ELSIF rValue > rMax THEN
    rClamped := rMax;
ELSE
    rClamped := rValue;
END_IF
```

## 5. State Machine Pattern

```iecst
VAR
    nState : INT := 0;
END_VAR

CASE nState OF
    0: // IDLE
        IF bStart THEN
            nState := 1;
        END_IF
    
    1: // RUNNING
        // Do work
        IF bComplete THEN
            nState := 2;
        ELSIF bError THEN
            nState := 99;
        END_IF
    
    2: // DONE
        // Wait for reset
        IF bReset THEN
            nState := 0;
        END_IF
    
    99: // ERROR
        IF bReset THEN
            nState := 0;
        END_IF
END_CASE
```

## 6. Timer / Debounce

```iecst
VAR
    tonDelay : TON;
    bDelayed : BOOL;
END_VAR

// 100ms delay
tonDelay(IN := bSignal, PT := T#100MS);
bDelayed := tonDelay.Q;
```

## 7. Type Conversion

```iecst
// INT to LREAL
rValue := LREAL#nIntValue;

// LREAL to INT (truncates)
nValue := INT#rValue;

// Using TO_ functions
rValue := TO_LREAL(nIntValue);
nValue := TO_INT(rValue);
```

## 8. Modular Arithmetic for Circular Buffers

```iecst
// Circular buffer index
nNextIdx := (nCurrentIdx + 1) MOD nBufferSize;

// Wrap angle to 0..360
IF rAngle >= 360.0 THEN
    rAngle := rAngle - 360.0;
ELSIF rAngle < 0.0 THEN
    rAngle := rAngle + 360.0;
END_IF
```

## 9. Error Code Pattern

```iecst
VAR CONSTANT
    ERR_OK         : UINT := 0;
    ERR_PARAM      : UINT := 1;
    ERR_OVERFLOW   : UINT := 2;
    ERR_TIMEOUT    : UINT := 3;
    ERR_COMM       : UINT := 4;
END_VAR

// Set error
bError := TRUE;
ErrorCode := ERR_PARAM;

// Clear error
bError := FALSE;
ErrorCode := ERR_OK;
```

## 10. VAR_IN_OUT Best Practices

- Use for large arrays to avoid memory copy
- Caller must declare the array and pass it
- FB reads and writes directly to the caller's array
- Array size in FB declaration must match or exceed caller's

```iecst
// FB declaration
VAR_IN_OUT
    DataArray : ARRAY[0..9999] OF LREAL;
END_VAR

// Caller declaration
VAR
    MyData : ARRAY[0..9999] OF LREAL;
END_VAR

// Caller invocation
MyFB(DataArray := MyData);
```
