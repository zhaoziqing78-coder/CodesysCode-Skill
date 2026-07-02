---
name: codesys-development
description: CODESYS IEC 61131-3 Structured Text (ST) development skill. Use when: (1) writing PLC programs in CODESYS, (2) creating function blocks (FB) for motion control, cam tables, encoder processing, (3) implementing winding/tension control logic, (4) generating cam table data acquisition and curve fitting algorithms, (5) any mention of "CODESYS", "PLC", "ST", "凸轮表", "编码器", "卷绕", "功能块", "FB". Covers FB design patterns, VAR_IN_OUT array binding, edge-triggered control, bidirectional detection, and error handling.
---

# CODESYS Development Skill

## Overview

This skill provides structured knowledge for writing CODESYS IEC 61131-3 ST code, focusing on:
- Function Block (FB) design patterns for industrial motion control
- Cam table data acquisition and curve fitting
- Encoder/winding axis processing
- Edge-triggered control logic
- Error handling and state machines

## Core Knowledge Areas

### 1. IEC 61131-3 ST Syntax Rules

- Variable declarations: `VAR_INPUT`, `VAR_OUTPUT`, `VAR_IN_OUT`, `VAR`, `VAR_CONSTANT`
- Data types: `BOOL`, `INT`, `UINT`, `DINT`, `LREAL`, `STRING`, arrays
- `ARRAY[0..N] OF LREAL` — fixed-size arrays (CODESYS does not support dynamic arrays)
- `VAR_IN_OUT` for passing large arrays by reference (avoid memory copy)
- `FUNCTION_BLOCK ... END_FUNCTION_BLOCK` structure
- `PROGRAM ... END_PROGRAM` structure
- Edge detection pattern: `IF bEnable AND NOT LastEnable THEN ... LastEnable := bEnable;`
- `RETURN` statement for early exit
- Comments: `//` single line, `(* ... *)` block

### 2. Function Block Design Patterns

#### Pattern A: Data Acquisition FB
- Trigger source: external signal (angle, position, or time)
- Array fill pattern: index increment with overflow check
- Edge-triggered start/stop
- Auto-read current value as start point
- Bidirectional movement detection

#### Pattern B: Data Processing FB
- Takes source arrays via `VAR_IN_OUT`
- Outputs result arrays via `VAR_IN_OUT`
- Execute trigger with `bExecute` rising edge
- Single-cycle or multi-cycle calculation
- Error codes for invalid input

#### Pattern C: State Machine FB
- `CASE state OF` pattern
- States: IDLE, RUNNING, DONE, ERROR
- Explicit state transitions
- Reset capability

### 3. Cam Table Specific Knowledge

- Cam table = array of (X, Y) pairs mapping master axis position to slave axis position
- X-axis: typically encoder position or master axis position
- Y-axis: typically slave axis position or angle
- Trigger by angle interval: every N degrees of winding axis, capture encoder position
- Average slope = (X[end] - X[0]) / (Y[end] - Y[0])
- Linear fit: FitX[i] = X[0] + AvgSlope * (Y[i] - Y[0])
- Moving average smoothing: average of N neighboring points (N must be odd)
- Array max size: 10000 (compiled), runtime configurable via MaxArrayLength

### 4. Common Variable Naming Conventions

| Prefix | Meaning | Example |
|--------|---------|---------|
| `b` | BOOL | `bEnable`, `bDone`, `bError` |
| `n` | INT/UINT | `nState`, `nCount` |
| `r` | REAL/LREAL | `rPosition`, `rAngle` |
| `arr` | ARRAY | `arrCamTableX` |
| `Step` | Configurable step | `StepInterval` |
| `Max` | Maximum limit | `MaxArrayLength` |
| `Last` | Previous cycle value | `LastEnable`, `LastAngle` |
| `Next` | Next target | `NextTargetAngle` |
| `Actual` | Current valid | `ActualLength` |
| `Fit` | Fitted result | `FitLineX`, `FitSmoothY` |

### 5. Error Handling Pattern

```iecst
VAR CONSTANT
    ERR_OK    : UINT := 0;
    ERR_FULL  : UINT := 1;  // Array full
    ERR_PARAM : UINT := 2;  // Invalid parameter
END_VAR

// At start of logic:
IF StepInterval <= 0 THEN
    bError := TRUE;
    ErrorCode := ERR_PARAM;
    RETURN;
END_IF;
```

### 6. FB Instantiation and Calling

```iecst
// Declaration
VAR
    MyFB : FB_CamTableGenerator;
    arrX : ARRAY[0..9999] OF LREAL;
    arrY : ARRAY[0..9999] OF LREAL;
END_VAR

// Calling with VAR_IN_OUT arrays
MyFB.bEnable := TRUE;
MyFB.EncoderPosition := rEncPos;
MyFB(CamTableX := arrX, CamTableY := arrY);
```

## Reference Files

- `references/fb-templates.md` — Ready-to-use FB code templates
- `references/cam-table-patterns.md` — Cam table design patterns and algorithms
- `references/st-code-snippets.md` — Common ST code snippets and utilities

## Usage Workflow

1. User describes the PLC function needed
2. Identify which pattern applies (acquisition, processing, state machine)
3. Generate FB code using the appropriate template
4. Generate a test PRG program
5. Provide usage instructions and parameter tuning guidance

## Limitations

- Cannot directly compile or test CODESYS code (no CODESYS runtime access)
- Code must be copied manually into CODESYS Development System
- Array sizes are fixed at compile time (max 10000 elements)
- Does not support visualization (FBD/LD) — ST only
