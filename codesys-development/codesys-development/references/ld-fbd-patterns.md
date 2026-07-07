# LD and FBD Programming Patterns in CODESYS

## 1. Ladder Diagram (LD) Fundamentals

### 1.1 Basic Elements

LD uses relay-logic symbolism. In CODESYS, LD is implemented as an editor within POUs (Programs or Function Blocks).

#### Contact Symbols

```
Normally Open (NO) Contact:     ---| |---
  Passes power when variable is TRUE

Normally Closed (NC) Contact:   ---|/|---
  Passes power when variable is FALSE

Positive Transition Contact:    ---[P]---
  Passes power for one scan on rising edge

Negative Transition Contact:    ---[N]---
  Passes power for one scan on falling edge
```

#### Coil Symbols

```
Standard Coil:    ---( )---
  Sets output to current power flow state

Set Coil:         ---(S)---
  Sets output TRUE and holds (latch)

Reset Coil:       ---(R)---
  Sets output FALSE and holds (unlatch)

Inverted Coil:    ---(/)---
  Sets output to opposite of power flow
```

### 1.2 Common LD Patterns

#### Motor Start-Stop with Latch

```
    bStart     bStop        bMotorRun
----| |----+---|/|----------( )---
           |
    bMotorRun|
----| |----+

Description:
  - bStart (NO) in parallel with bMotorRun (NO) = latch
  - bStop (NC) in series = break latch
  - bMotorRun stays ON after bStart pulse until bStop
```

ST equivalent:
```iecst
IF bStart THEN
    bMotorRun := TRUE;
END_IF
IF bStop THEN
    bMotorRun := FALSE;
END_IF
```

#### Motor Start-Stop with Set/Reset Coils

```
    bStart                       bMotorRun
----| |---+----------------------(S)---
         |
    bOverload|
----|/|---+

    bStop                        bMotorRun
----| |--------------------------(R)---

    bMotorRun                    bMotorRun
----| |---+----------------------( )---
         |
    bMotorFb|
----| |---+
```

#### Three-Wire Control with Interlock

```
    bStart    bSafetyDoor  bEStop    bMotorRun
----| |---+-----| |---+-----| |---------+-----( )---
         |           |               |
    bMotorRun    bOverload        bMaintMode
----| |---+-----|/|---+-----|/|-----+
```

#### Timer in LD

```
    bStart                      tonDelay (TON)
----| |-------+--------------------[ TON, T#5S ]---
           |
    bMotorRun|
----| |----+

    tonDelay.Q                  bMotorRun
----| |--------------------------( )---
```

ST equivalent:
```iecst
tonDelay(IN := bStart OR bMotorRun, PT := T#5S);
IF tonDelay.Q THEN
    bMotorRun := TRUE;
END_IF
```

#### Counter in LD

```
    bPartDetected               ctuParts (CTU)
----| |--------------------------[ CTU, PV=100 ]---

    ctuParts.CV                 nPartCount
----[ ]--------------------------[ MOVE ]---

    bReset                      ctuParts
----| |--------------------------[ RESET ]---
```

#### Comparison Block in LD

```
    rTemperature                rTempOK
----[ >= 50.0 ]-----------------( )---

    rTemperature                rTempAlarm
----[ > 80.0 ]------------------(S)---

    bAckAlarm                   rTempAlarm
----| |--------------------------(R)---
```

#### Jump and Label

```
    bBypass                     lblSkip
----| |--------------------------( JMP )

    // Normal logic here
    bInput1                     bOutput1
----| |--------------------------( )---

lblSkip:
    // Bypassed logic when bBypass is TRUE
    bInput2                     bOutput2
----| |--------------------------( )---
```

### 1.3 LD Best Practices

1. **Power flow left-to-right**: Always draw contacts on left, coils on right
2. **Parallel paths**: Use parallel branches for OR conditions, not separate rungs
3. **Series contacts**: Use series contacts for AND conditions
4. **Avoid nested logic**: Keep rungs simple — complex logic belongs in ST
5. **One output per rung**: Each rung should drive one coil
6. **Order matters**: Place safety interlocks before normal logic
7. **Use Set/Reset sparingly**: Prefer standard coils for deterministic behavior
8. **Document**: Add rung comments explaining the purpose

### 1.4 When to Use LD vs ST

| Scenario | Recommended | Reason |
|----------|------------|--------|
| Motor start/stop | LD | Familiar to electricians, visual |
| Safety interlocks | LD | Clear visual chain of conditions |
| Simple boolean logic | LD | Easy to debug visually |
| Complex algorithms | ST | LD becomes unwieldy for math |
| Array operations | ST | LD has no loop constructs |
| State machines | ST | CASE statement is cleaner |
| String manipulation | ST | LD has no string operations |
| Communication logic | ST | Complex data handling |

---

## 2. Function Block Diagram (FBD) Fundamentals

### 2.1 Basic Elements

FBD uses graphical blocks connected by lines representing signal flow.

#### Logic Blocks

```
AND block:    a ---|& |--- output
              b ---|  |

OR block:     a ---|>=1|--- output
              b ---|   |

NOT block:    a ---|& |--- output (with NOT on output)
              1 ---|  |    (input tied to TRUE)

XOR block:    a ---|=1|--- output
              b ---|  |
```

#### Arithmetic Blocks

```
ADD block:    a ---|ADD|--- result
              b ---|   |

SUB block:    a ---|SUB|--- result
              b ---|   |

MUL block:    a ---|MUL|--- result
              b ---|   |

DIV block:    a ---|DIV|--- result
              b ---|   |
```

#### Comparison Blocks

```
GT block:     a ---|> |--- output (TRUE if a > b)
              b ---|  |

LT block:     a ---|< |--- output (TRUE if a < b)
              b ---|  |

GE block:     a ---|>=|--- output (TRUE if a >= b)
              b ---|  |

EQ block:     a ---|= |--- output (TRUE if a = b)
              b ---|  |
```

#### Selection Blocks

```
MUX block:    sel ---|MUX|--- output (selects input[sel])
              in0 ---|   |
              in1 ---|   |
              in2 ---|   |

SEL block:    sel ---|SEL|--- output (sel=0: out=in0, sel=1: out=in1)
              in0 ---|   |
              in1 ---|   |

LIMIT block:  val ---|LIMIT|--- output (clamped between min and max)
              min ---|     |
              max ---|     |
```

### 2.2 Common FBD Patterns

#### Signal Conditioning Chain

```
                 +-------+        +-------+        +--------+
rRawSensor  -----| SCALE |--->----| FILTER|--->----| LIMIT  |--- rEngValue
                 | 0-1k  |        | T#100MS|       | 0-100  |
                 +-------+        +-------+        +--------+
```

ST equivalent:
```iecst
rScaled := rRawSensor * 100.0 / 1000.0;  // Scale 0-1000 to 0-100
tonFilter(IN := TRUE, PT := T#100MS);
rFiltered := rScaled;  // Simplified — real filter needs FB
rEngValue := LIMIT(0.0, rFiltered, 100.0);
```

#### PID Control Loop

```
rSetPoint ---+--->----|PID |--- rOutput
             |        |    |
rActValue ---+--->----|    |
             |        |    |
             +--->----|    |  (feedback)
                      +----+
```

ST equivalent:
```iecst
fbPID(
    rSetPoint := rSetPoint,
    rActValue := rActValue,
    rKp := 2.0,
    rTi := T#5S,
    rTd := T#500MS,
    rOutput => rOutput
);
```

#### Alarm Detection with Hysteresis

```
rValue ---+--->---[ > 80.0 ]---+---|SR |--- bAlarmHigh
          |                     |   |   |
          +--->---[ < 75.0 ]---+---|R  |
                                |   +---+
                          bAck ---|R  |
```

ST equivalent:
```iecst
IF rValue > 80.0 THEN
    bAlarmHigh := TRUE;
END_IF
IF (rValue < 75.0) OR bAck THEN
    bAlarmHigh := FALSE;
END_IF
```

#### Motor Control with Feedback

```
bStart ---+--->---|SR |---+--- bMotorCmd
          |       |   |   |
bStop  ---|---+---|R  |   |
          |   |   +---+   |
          |   |           |
bRunning---+---+-----------+--->---[ & ]--- bMotorReady
              |
bFault -------|/|---+
```

#### Multi-Input Voting Logic (2oo3)

```
bSensor1 ---+--->---[ >=2 ]--- bVoteResult
bSensor2 ---|---+---|     |
bSensor3 ---+---+---|     |
              |     +-----+
              |
    +--- AND ---+      +--- AND ---+      +--- AND ---+
bSensor1-|& |   | bSensor2-|& |   | bSensor3-|& |   |
bSensor2-|  |---+ bSensor3-|  |---+ bSensor1-|  |---+
        +---+             +---+             +---+
         |                  |                  |
         +-------- OR -------+-------- OR ------+
                            |
                      bVoteResult
```

ST equivalent (simpler for voting):
```iecst
nVoteCount := 0;
IF bSensor1 THEN nVoteCount := nVoteCount + 1; END_IF
IF bSensor2 THEN nVoteCount := nVoteCount + 1; END_IF
IF bSensor3 THEN nVoteCount := nVoteCount + 1; END_IF
bVoteResult := (nVoteCount >= 2);
```

### 2.3 FBD with User Function Blocks

Custom FBs appear as blocks in FBD:

```
                 +-------------------+
rSetPoint  -----| SetPoint          |
rActValue  -----| ActValue    FB_PID|----- rControlOutput
bEnable    -----| Enable            |
T#5S       -----| Ti                |
2.0        -----| Kp                |
                 +-------------------+

                 +-------------------+
bStart     -----| bEnable           |
rPosition  -----| SensorValue  FB_Acq|----- bDone
rAngle     -----| TriggerSource     |
10.0       -----| StepInterval      |
                 +-------------------+
```

### 2.4 FBD Feedback Loops

FBD requires careful handling of feedback (cyclic dependencies):

```
// PROBLEM: Direct feedback creates circular dependency
//   Block A output → Block B input → Block A input

// SOLUTION: Use VAR intermediate variable
VAR
    rFeedback : LREAL;  // Breaks the cycle
END_VAR

// In FBD:
//   Block A output → rFeedback (write)
//   rFeedback (read) → Block B input
//   Block B output → Block A input
```

### 2.5 FBD Best Practices

1. **Signal flow left-to-right**: Inputs on left, outputs on right
2. **Minimize crossings**: Route lines to avoid crossing
3. **Group related blocks**: Place related blocks close together
4. **Label all signals**: Add comments to important signal lines
5. **Use user FBs for reuse**: Encapsulate repeated patterns in custom FBs
6. **Avoid deep nesting**: Maximum 3-4 levels of block nesting per diagram
7. **Use SEL/MUX for mode selection**: Cleaner than multiple AND/OR blocks
8. **Keep diagrams readable**: If a diagram needs more than one screen, split it

### 2.6 When to Use FBD vs ST vs LD

| Scenario | Recommended | Reason |
|----------|------------|--------|
| Analog signal processing | FBD | Visual signal flow |
| PID control loops | FBD | Clear feedback path |
| Multi-input logic | FBD | Easier to see all inputs |
| Safety interlocks | LD | Electrical convention |
| Motor control | LD | Standard relay logic |
| Complex algorithms | ST | Math and array operations |
| State machines | ST | CASE statement |
| Communication | ST | Data structure handling |

---

## 3. Mixed-Language Programming

CODESYS allows calling ST code from LD/FBD and vice versa:

### 3.1 Calling FB from LD

```iecst
// Declare FB instance in LD POU variables
VAR
    fbMotorCtrl : FB_MotorControl;
    bStart : BOOL;
    bStop : BOOL;
    rSpeedSetpoint : LREAL;
    bRunning : BOOL;
    rActualSpeed : LREAL;
END_VAR

// In LD, add "Box" element and select FB_MotorControl
// Wire inputs to left pins, outputs from right pins:
//
//   bStart ---------| bStart            |
//   bStop ----------| bStop   fbMotorCtrl|--- bRunning
//   rSpeedSetpoint--| rSpeedSetpoint    |--- rActualSpeed
//                   +--------------------+
```

### 3.2 Calling FB from FBD

```iecst
// In FBD, add user FB block:
//
//   bStart ---------| bEnable           |
//   rTemp ----------| rInput  FB_Filter |--- rFiltered
//   5 --------------| nWindow           |
//                   +--------------------+
```

### 3.3 Action Blocks in LD/FBD

Use ST code within LD/FBD via Action blocks:

```iecst
// Create Action in POU (right-click → Add → Action)
// Action name: CalculateAverage

// In the Action (ST code):
rAverage := (rValue1 + rValue2 + rValue3) / 3.0;

// In LD: call the action using "Jump" or "Action" element
// In FBD: call using "Execute" block with action name
```

---

## 4. Conversion Between Languages

### 4.1 LD to ST Conversion

| LD Element | ST Equivalent |
|------------|---------------|
| `---| |---` (NO contact) | `IF bVar THEN` |
| `---|/|---` (NC contact) | `IF NOT bVar THEN` |
| Parallel contacts (OR) | `IF bVar1 OR bVar2 THEN` |
| Series contacts (AND) | `IF bVar1 AND bVar2 THEN` |
| `---( )---` (coil) | `bOutput := ...;` |
| `---(S)---` (set coil) | `bOutput := TRUE;` |
| `---(R)---` (reset coil) | `bOutput := FALSE;` |
| TON timer | `tonDelay(IN := ..., PT := ...);` |
| Comparison `---[> 5]---` | `IF rValue > 5.0 THEN` |

### 4.2 FBD to ST Conversion

| FBD Block | ST Equivalent |
|-----------|---------------|
| AND | `bResult := bA AND bB;` |
| OR | `bResult := bA OR bB;` |
| ADD | `rResult := rA + rB;` |
| SUB | `rResult := rA - rB;` |
| MUL | `rResult := rA * rB;` |
| DIV | `rResult := rA / rB;` |
| GT (`>`) | `bResult := rA > rB;` |
| LIMIT | `rResult := LIMIT(rMin, rVal, rMax);` |
| SEL | `rResult := SEL(bSel, rIn0, rIn1);` |
| MUX | `rResult := MUX(nSel, rIn0, rIn1, rIn2);` |
