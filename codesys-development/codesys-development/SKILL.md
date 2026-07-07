---
name: codesys-development
description: "CODESYS IEC 61131-3 development skill covering ST, LD, and FBD programming. Use when: (1) writing PLC programs in CODESYS, (2) creating function blocks (FB) for motion control, cam tables, encoder processing, winding/tension control, (3) implementing communication via Modbus TCP/RTU, EtherCAT, PROFINET, OPC UA, (4) developing WebVisu/HMI visualization interfaces, (5) debugging, testing, and simulating PLC code, (6) any mention of \"CODESYS\", \"PLC\", \"ST\", \"LD\", \"FBD\", \"凸轮表\", \"编码器\", \"卷绕\", \"功能块\", \"FB\", \"Modbus\", \"EtherCAT\", \"OPC UA\", \"WebVisu\", \"梯形图\", \"功能块图\". Covers FB design patterns, VAR_IN_OUT array binding, edge-triggered control, state machines, communication FBs, visualization element patterns, naming conventions, project structure standards, and error handling."
agent_created: true
---

# CODESYS Development Skill

## Overview

This skill provides structured knowledge for writing CODESYS IEC 61131-3 code across three programming languages (ST, LD, FBD) and multiple application domains:

- **Structured Text (ST)**: Complex logic, algorithms, data processing, state machines
- **Ladder Diagram (LD)**: Discrete logic, interlocks, electrical-style control
- **Function Block Diagram (FBD)**: Signal flow, analog processing, closed-loop control
- **Motion Control**: Cam tables, encoder processing, winding/tension control, servo synchronization
- **Communication Protocols**: Modbus TCP/RTU, EtherCAT, PROFINET, OPC UA
- **Visualization**: WebVisu HMI development, dashboard elements, alarm displays
- **Best Practices**: Naming conventions, code structure, project organization, comment standards
- **Debugging & Testing**: Common error troubleshooting, online debugging, simulation methods

## Core Knowledge Areas

### 1. IEC 61131-3 Programming Languages

#### 1.1 Structured Text (ST)

ST is the primary language for complex logic and algorithms. Key syntax rules:

- Variable declarations: `VAR_INPUT`, `VAR_OUTPUT`, `VAR_IN_OUT`, `VAR`, `VAR_CONSTANT`
- Data types: `BOOL`, `INT`, `UINT`, `DINT`, `LREAL`, `STRING`, `TIME`, arrays
- `ARRAY[0..N] OF LREAL` — fixed-size arrays (CODESYS does not support dynamic arrays)
- `VAR_IN_OUT` for passing large arrays by reference (avoid memory copy)
- `FUNCTION_BLOCK ... END_FUNCTION_BLOCK` structure
- `PROGRAM ... END_PROGRAM` structure
- Edge detection pattern: `IF bEnable AND NOT LastEnable THEN ... LastEnable := bEnable;`
- `RETURN` statement for early exit
- `CASE ... OF ... END_CASE` for state machines
- Comments: `//` single line, `(* ... *)` block
- Type conversion: `TO_LREAL()`, `TO_INT()`, or `LREAL#nValue` cast syntax

For common ST code snippets (edge detection, timers, state machines, circular buffers, type conversion), refer to `references/st-code-snippets.md`.

#### 1.2 Ladder Diagram (LD)

LD uses relay-logic symbolism for discrete control. Key elements:

- **Contacts**: Normally open `---| |---`, normally closed `---|/|---`
- **Coils**: `---( )---` standard, `---(S)---` set, `---(R)---` reset
- **Timers**: TON (on-delay), TOF (off-delay), TP (pulse)
- **Counters**: CTU (up), CTD (down), CTUD (up/down)
- **Comparison**: `---[>=]---`, `---[=]---`, `---[<>]---` inline comparison blocks
- **Jump/Label**: `---(JMP)---` with `Label:` targets
- **Return**: `---(RET)---` for early POU exit

LD is best for: safety interlocks, motor start/stop circuits, sequence logic visible to electrical engineers.

For LD programming patterns and best practices, refer to `references/ld-fbd-patterns.md`.

#### 1.3 Function Block Diagram (FBD)

FBD uses graphical block connections for signal-flow logic. Key elements:

- **Blocks**: AND, OR, NOT, XOR, ADD, SUB, MUL, DIV, GT, LT, GE, LE, EQ, NE
- **Selection blocks**: MUX (multiplexer), SEL (selector), LIMIT (clamper)
- **Type conversion blocks**: INT_TO_LREAL, BOOL_TO_INT, etc.
- **Timer/Counter blocks**: TON, TOF, TP, CTU as graphical blocks
- **Feedback loops**: Use `VAR` intermediate variables to break cyclic connections
- **User FBs**: Custom FBs appear as graphical blocks with named pins

FBD is best for: analog signal processing, PID control chains, signal conditioning, multi-input logic where signal flow matters more than sequence.

For FBD programming patterns, refer to `references/ld-fbd-patterns.md`.

### 2. Function Block Design Patterns

#### Pattern A: Data Acquisition FB
- Trigger source: external signal (angle, position, or time)
- Array fill pattern: index increment with overflow check
- Edge-triggered start/stop
- Auto-read current value as start point
- Bidirectional movement detection (use ABS difference, not direction-based)

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

#### Pattern D: Communication FB
- Connection management (connect/disconnect/reconnect)
- Request-response cycle with timeout
- Error handling with retry counter
- Status output for HMI display

#### Pattern E: Visualization Data Provider FB
- Collects and formats data for HMI display
- Provides alarm/status arrays
- Handles user input from visualization elements
- Manages page/screen navigation state

For ready-to-use FB code templates, refer to `references/fb-templates.md`.

### 3. Motion Control

Cam table = array of (X, Y) pairs mapping master axis position to slave axis position:
- X-axis: typically encoder position or master axis position
- Y-axis: typically slave axis position or angle
- Trigger by angle interval: every N degrees of winding axis, capture encoder position
- Average slope = (X[end] - X[0]) / (Y[end] - Y[0])
- Linear fit: FitX[i] = X[0] + AvgSlope * (Y[i] - Y[0])
- Moving average smoothing: average of N neighboring points (N must be odd)
- Array max size: 10000 (compiled), runtime configurable via MaxArrayLength

For cam table design patterns, curve fitting algorithms, and bidirectional capture details, refer to `references/cam-table-patterns.md`.

### 4. Communication Protocols

CODESYS supports multiple industrial communication protocols. Key patterns:

#### Modbus TCP/RTU
- Use `NVL.ModbusTCP_Server` / `NVL.ModbusRTU_Server` for device emulation
- Use `SysSocket` or Modbus FB library for custom client/server
- Register mapping: holding registers (40001+), input registers (30001+), coils (00001+), discrete inputs (10001+)
- Function codes: 03 (read holding), 06 (write single), 16 (write multiple)

#### EtherCAT
- Configured via EtherCAT Device in device tree
- Use `EtherCAT slaves` with ESI/EEPROM files
- PDO mapping for cyclic data exchange
- SDO access for parameter read/write (acyclic)

#### PROFINET
- Configured via PROFINET IO Device in device tree
- GSDML files describe device capabilities
- Slot/subslot configuration for module mapping
- Real-time (RT) and isochronous (IRT) modes

#### OPC UA
- Use `OpcUaServer` device for built-in OPC UA server
- Symbol configuration: select variables to expose
- Namespace mapping: PLC variables automatically appear under namespace
- Client access via `FB_OpcUaClient` from OPC UA library

For detailed communication FB templates, register mapping examples, and protocol-specific code patterns, refer to `references/communication-protocols.md`.

### 5. Visualization (WebVisu)

CODESYS WebVisu enables browser-based HMI development. Key concepts:

#### Visualization Manager
- Configure in device tree under "Visualization Manager"
- Set web server port (default 8080)
- Select target: HTML5 (modern browsers), native (CODESYS apps)

#### Visualization Elements
- **Basic shapes**: Rectangle, RoundRect, Ellipse, Polygon, Line, Bitmap
- **Controls**: Button, Text Input, Combo Box, Table, Slider, Gauge
- **Display**: Text field, Bar, Lamp (LED indicator)
- **Special**: Alarm table, Trend control, Recipe management

#### Variable Binding
- Elements bind to PLC variables via "Variable" property
- Use `%s`, `%d`, `%f` format strings for text display
- Dynamic properties (color, visibility, position) via variable expressions
- Toggle inputs: "Tap" (momentary), "Tap on / Tap off" (alternating)

#### Common Patterns
- Motor control: Button with feedback lamp, green=running, red=stopped
- Analog display: Bar gauge with high/low limit coloring
- Alarm display: Alarm table with acknowledge buttons
- Trend: Real-time data plot with configurable time window
- Navigation: Page selector with menu buttons

For detailed WebVisu development patterns, element configuration examples, and HMI design best practices, refer to `references/visualization-guide.md`.

### 6. Best Practices and Conventions

#### Variable Naming Conventions

| Prefix | Meaning | Example |
|--------|---------|---------|
| `b` | BOOL | `bEnable`, `bDone`, `bError` |
| `n` | INT/UINT | `nState`, `nCount` |
| `r` | REAL/LREAL | `rPosition`, `rAngle` |
| `s` | STRING | `sMessage`, `sAlarmText` |
| `t` | TIME | `tDelay`, `tTimeout` |
| `arr` | ARRAY | `arrCamTableX` |
| `st` | STRUCT | `stConfig`, `stStatus` |
| `e` | ENUM | `eMode`, `eState` |
| `fb` | FB instance | `fbTimer`, `fbModbus` |
| `Step` | Configurable step | `StepInterval` |
| `Max` | Maximum limit | `MaxArrayLength` |
| `Last` | Previous cycle value | `LastEnable`, `LastCapturedY` |
| `Actual` | Current valid | `ActualLength` |
| `Fit` | Fitted result | `FitLineX`, `FitSmoothY` |
| `Delta` | Difference value | `DeltaAngle`, `AbsDelta` |

#### POU Naming Conventions

| Prefix | Type | Example |
|--------|------|---------|
| `FB_` | Function Block | `FB_CamTableGenerator` |
| `FC_` | Function | `FC_CalculateSlope` |
| `PRG_` | Program | `PRG_Main` |
| `M_` | Method | `M_Reset`, `M_Start` |
| `P_` | Property | `P_IsRunning` |

#### Error Handling Pattern

```iecst
VAR CONSTANT
    ERR_OK    : UINT := 0;
    ERR_FULL  : UINT := 1;  // Array full
    ERR_PARAM : UINT := 2;  // Invalid parameter
    ERR_TIMEOUT : UINT := 3; // Timeout
    ERR_COMM  : UINT := 4;  // Communication error
END_VAR

// At start of logic:
IF StepInterval <= 0 THEN
    bError := TRUE;
    ErrorCode := ERR_PARAM;
    RETURN;
END_IF;
```

For full best practices guide (project structure, comment standards, code organization, safety considerations), refer to `references/best-practices.md`.

### 7. Debugging and Testing

#### Common Error Categories

| Category | Symptoms | Typical Cause |
|----------|----------|---------------|
| Array overflow | `ActualLength` exceeds array bounds | Missing bounds check before write |
| Edge trigger failure | Action fires every scan or never | Missing `LastXxx` assignment or wrong order |
| VAR_IN_OUT mismatch | Compile error on FB call | Array size in caller smaller than FB declaration |
| Type conversion | Unexpected values | Implicit conversion truncation (LREAL to INT) |
| Communication timeout | No response from device | Wrong IP/port, network issue, wrong function code |
| Task watchdog | PLC stops, watchdog error | Scan time exceeds task cycle time |
| Memory overflow | PLC crash or erratic behavior | Too many large arrays in one task |

#### Online Debugging Techniques
- **Online change**: Modify code while PLC running (without download)
- **Write/force values**: Override variable values for testing
- **Breakpoint**: Halt execution at a specific line (limited in some runtimes)
- **Trace**: Record variable values over time with sampling
- **Call stack**: View POU call hierarchy during breakpoint
- **Logic analyzer**: Multi-channel signal recording for digital signals

#### Simulation and Testing
- **CODESYS Control Win V3**: SoftPLC for Windows-based simulation
- **Unit testing**: Use `CfUnit` library for automated FB testing
- **Mock inputs**: Create a `PRG_Test` that simulates sensor inputs
- **Trace comparison**: Record expected vs actual behavior

For detailed debugging workflows, testing strategies, and common pitfall solutions, refer to `references/debugging-testing.md`.

## Reference Files

| File | Content |
|------|---------|
| `references/st-code-snippets.md` | Common ST code snippets: edge detection, timers, state machines, circular buffers, type conversion, error patterns |
| `references/fb-templates.md` | Ready-to-use FB code templates: data acquisition, processing, state machine, communication, and test programs |
| `references/cam-table-patterns.md` | Cam table design patterns: data acquisition, curve fitting, bidirectional capture, SoftMotion integration |
| `references/ld-fbd-patterns.md` | LD and FBD programming patterns: motor control, interlocks, analog processing, signal conditioning |
| `references/communication-protocols.md` | Communication FBs: Modbus TCP/RTU, EtherCAT, PROFINET, OPC UA code examples and register mapping |
| `references/visualization-guide.md` | WebVisu development: element configuration, variable binding, HMI design patterns, alarm tables |
| `references/best-practices.md` | Naming conventions, project structure, comment standards, code organization, safety guidelines |
| `references/debugging-testing.md` | Debugging techniques, simulation methods, common errors, unit testing with CfUnit |

## Usage Workflow

1. **Identify the task**: User describes the PLC function needed
2. **Determine language**: Choose ST (logic/algorithm), LD (discrete/interlock), or FBD (signal flow/analog)
3. **Select pattern**: Match to appropriate design pattern (acquisition, processing, state machine, communication, visualization)
4. **Generate code**: Use the matching template from reference files as starting point
5. **Apply conventions**: Follow naming conventions and error handling patterns
6. **Generate test program**: Create a `PRG_Test` that simulates inputs and verifies outputs
7. **Provide guidance**: Include parameter tuning, debugging tips, and integration instructions
8. **Recommend debugging**: Suggest appropriate debugging technique (trace, breakpoint, online change)

## Limitations

- Cannot directly compile or test CODESYS code (no CODESYS runtime access)
- Code must be copied manually into CODESYS Development System
- Array sizes are fixed at compile time (max 10000 elements)
- FBD and LD are graphical languages — can only provide textual descriptions and ST equivalents
- Visualization configuration is GUI-based — can only provide configuration guidance and variable binding patterns, not visual layouts
- EtherCAT/PROFINET configuration is device-tree based — code alone cannot configure bus topology
- For complex motion control, SoftMotion configuration in device tree is required before FB code works
