# CODESYS WebVisu Visualization Guide

## 1. Visualization Manager Configuration

### 1.1 Setup Steps

1. In device tree, right-click "Device" → "Add Device" → "Visualization Manager"
2. Configure web server settings:
   - Port: default 8080 (change if conflict)
   - Target: HTML5 (for browser access) or Native (for CODESYS app)
3. Add visualization objects: right-click project → "Add Object" → "Visualization"

### 1.2 Access Methods

| Method | URL / Access | Use Case |
|--------|-------------|----------|
| Web browser | `http://<PLC-IP>:8080` | Remote monitoring from any device |
| CODESYS app | Native app on mobile/desktop | Field operator interface |
| VNC client | Via VNC server | Direct screen access |

### 1.3 Security Configuration

```xml
<!-- In Visualization Manager settings -->
- Enable HTTPS for encrypted communication
- Set username/password for access control
- Configure user groups (Operator, Engineer, Admin)
- Restrict write access per user group
```

---

## 2. Visualization Elements

### 2.1 Basic Shapes and Properties

| Element | Purpose | Key Properties |
|---------|---------|---------------|
| Rectangle | Container, background | FillColor, FrameColor, Transparency |
| RoundRect | Panel, button background | CornerRadius, FillColor |
| Ellipse | Lamp, indicator | FillColor (dynamic), Border |
| Polygon | Arrows, custom shapes | Points array, FillType |
| Line | Connectors, dividers | LineWidth, LineType |
| Bitmap | Logos, icons | BitmapID, Stretch |

### 2.2 Interactive Controls

| Element | Purpose | Key Properties |
|---------|---------|---------------|
| Button | Trigger actions | "Variable" (toggle), "Events" (on tap) |
| Text Input | Enter values | "Variable" (write target), InputType |
| Combo Box | Select from list | "Variable" (selected index), Items |
| Table | Display arrays | "Variable" (array ref), Columns |
| Slider | Analog adjust | "Variable" (min/max), Orientation |
| Gauge | Analog display | "Variable" (value), Min/Max, Style |

### 2.3 Display Elements

| Element | Purpose | Key Properties |
|---------|---------|---------------|
| Text Field | Show formatted data | "Variable", "Text" with `%s`/`%d`/`%f` |
| Bar | Level indicator | "Variable", Min/Max, Orientation |
| Lamp (LED) | Status indicator | "Variable" (BOOL), Color on/off |
| Alarm Table | Show active alarms | "AlarmConfiguration" |
| Trend Control | Real-time plot | "Variable" (array), Time window |

---

## 3. Variable Binding

### 3.1 Basic Binding

Bind element properties directly to PLC variables:

```
// Button toggle: binds to PLC BOOL variable
Element Property "Variable" := "PLC_PRG.bMotorStart"

// Text display: shows formatted variable
Element Property "Text" := "Temperature: %.1f C"
Element Property "Variable" := "PLC_PRG.rTemperature"

// Bar gauge: shows analog value
Element Property "Variable" := "PLC_PRG.rPressure"
Element Property "MinValue" := 0
Element Property "MaxValue" := 10
```

### 3.2 Dynamic Properties

Use expressions for dynamic appearance:

```
// Dynamic fill color based on value
Element Property "FillColor" :=
    IF PLC_PRG.bAlarm THEN 16#0000FF  // Red
    ELSIF PLC_PRG.bWarning THEN 16#00FFFF  // Yellow
    ELSE 16#00FF00  // Green
    END_IF

// Dynamic visibility
Element Property "Invisible" := NOT PLC_PRG.bExpertMode

// Dynamic position (animation)
Element Property "X" := INT_TO_REAL(PLC_PRG.nPosX)
Element Property "Y" := INT_TO_REAL(PLC_PRG.nPosY)
```

### 3.3 Color Constants

CODESYS uses RGB color values as DWORD (16#BBGGRR format):

| Color | Value | Description |
|-------|-------|-------------|
| Red | 16#0000FF | Alarm, error, stop |
| Green | 16#00FF00 | Running, OK, active |
| Yellow | 16#00FFFF | Warning, caution |
| Blue | 16#FF0000 | Information, selected |
| Gray | 16#808080 | Disabled, inactive |
| White | 16#FFFFFF | Background |
| Black | 16#000000 | Text, border |
| Orange | 16#00A5FF | Warning level 2 |

### 3.4 Input Modes

| Mode | Behavior | Use Case |
|------|----------|----------|
| Tap (momentary) | TRUE while pressed, FALSE on release | Push button, jog |
| Tap on / Tap off | Toggle on each tap | Start/stop toggle |
| Tap variable | Write specific value | Mode selector |
| Switch on / Switch off | Separate on and off buttons | Motor start/stop |
| Text input | Opens keyboard on tap | Setpoint entry |

---

## 4. Common HMI Design Patterns

### 4.1 Motor Control with Feedback

**Layout**: Button (green) + Lamp (round indicator)

```
// Button configuration:
//   - Variable: "PLC_PRG.bMotorCmd"
//   - Input mode: "Tap on / Tap off"
//   - Text: "MOTOR START" when off, "MOTOR STOP" when on
//   - FillColor: Green when off, Red when on

// Lamp configuration (placed next to button):
//   - Variable: "PLC_PRG.bMotorRunning" (actual feedback)
//   - FillColor: Green when TRUE, Gray when FALSE
//   - Ellipse shape, 20x20 pixels

// Dynamic text:
//   Text property := IF PLC_PRG.bMotorRunning THEN 'RUNNING' ELSE 'STOPPED' END_IF
```

### 4.2 Analog Gauge with Limits

**Layout**: Gauge element + Text field + Alarm indicator

```
// Gauge configuration:
//   - Variable: "PLC_PRG.rPressure"
//   - MinValue: 0
//   - MaxValue: 10
//   - FillColor: Green (normal), Red (over limit)

// Dynamic fill color:
//   FillColor := IF PLC_PRG.rPressure > 8.0 THEN 16#0000FF  // Red
//                ELSIF PLC_PRG.rPressure > 6.0 THEN 16#00FFFF  // Yellow
//                ELSE 16#00FF00  // Green
//                END_IF

// Text field:
//   Text := "Pressure: %.2f bar"
//   Variable := "PLC_PRG.rPressure"
```

### 4.3 Alarm Table Configuration

1. Add "Alarm Configuration" object to project
2. Define alarm classes (Info, Warning, Error, Critical)
3. Define alarms with trigger conditions

```
// Alarm definition example:
//   Name: "TempHigh"
//   Message: "Temperature too high: %.1f C"
//   Variable: "PLC_PRG.rTemperature > 80.0"
//   Class: "Error"
//   Acknowledge: Required

// Alarm table element:
//   - Variable: "" (uses global alarm config)
//   - Columns: Time, Message, Class, Acknowledge button
//   - Sort: Active alarms first, then by time
```

```iecst
// PLC code to support alarm table
VAR
    // Alarm trigger variables (mapped to alarm configuration)
    bAlarmTempHigh   : BOOL;
    bAlarmPressLow   : BOOL;
    bAlarmMotorTrip  : BOOL;

    // Acknowledge feedback
    bAckTempHigh     : BOOL;
    bAckPressLow     : BOOL;
    bAckMotorTrip    : BOOL;
END_VAR

// Trigger alarms
bAlarmTempHigh  := rTemperature > 80.0;
bAlarmPressLow  := rPressure < 2.0;
bAlarmMotorTrip := bMotorFault;

// Reset after acknowledge
IF bAckTempHigh THEN
    // Alarm auto-resets when condition clears
    bAckTempHigh := FALSE;
END_IF
```

### 4.4 Trend Control (Real-Time Plot)

```
// Trend element configuration:
//   - Variable: "PLC_PRG.arrTrendData" (ARRAY[0..999] OF LREAL)
//   - Time window: 60 seconds
//   - Sample interval: 100ms
//   - Y-axis: Auto-scale or fixed (0-100)
//   - Line color: Blue for primary, Red for secondary

// PLC code for trend data buffer
```

```iecst
VAR
    arrTrendData  : ARRAY[0..999] OF LREAL;
    nTrendIdx     : UINT := 0;
    tonTrendSample: TON;
    bTrendEnable  : BOOL := TRUE;
END_VAR

// Sample at 100ms interval
tonTrendSample(IN := bTrendEnable, PT := T#100MS);
IF tonTrendSample.Q THEN
    tonTrendSample(IN := FALSE);

    // Store value in circular buffer
    arrTrendData[nTrendIdx] := rTemperature;
    nTrendIdx := (nTrendIdx + 1) MOD 1000;
END_IF
```

### 4.5 Page Navigation

```iecst
VAR
    // Navigation state
    eActivePage : E_VisuPage := PAGE_MAIN;
END_VAR

TYPE E_VisuPage :
(
    PAGE_MAIN     := 0,
    PAGE_MOTOR    := 10,
    PAGE_TEMP     := 20,
    PAGE_ALARM    := 30,
    PAGE_SETTINGS := 40,
    PAGE_DIAG     := 50
);
END_TYPE

// Navigation buttons set the page variable:
//   Button "Main Menu"  → Variable := "Visu_PRG.eActivePage", Tap value := 0
//   Button "Motor"      → Variable := "Visu_PRG.eActivePage", Tap value := 10
//   Button "Temperature" → Variable := "Visu_PRG.eActivePage", Tap value := 20
//
// Each page frame uses "Invisible" property:
//   Page Main frame:    Invisible := (Visu_PRG.eActivePage <> 0)
//   Page Motor frame:   Invisible := (Visu_PRG.eActivePage <> 10)
//   Page Temp frame:    Invisible := (Visu_PRG.eActivePage <> 20)
```

### 4.6 Recipe Management

```iecst
VAR
    // Recipe structure
    stRecipe : ST_ProcessRecipe;
    arrRecipes : ARRAY[0..19] OF ST_ProcessRecipe;
    nSelectedRecipe : UINT := 0;
    bLoadRecipe : BOOL;
    bSaveRecipe : BOOL;
END_VAR

TYPE ST_ProcessRecipe :
STRUCT
    sName        : STRING(30);
    rTargetTemp  : LREAL;
    rTargetPress : LREAL;
    nMotorSpeed  : UINT;
    tMixTime     : TIME;
    bAutoMode    : BOOL;
END_STRUCT
END_TYPE

// Load recipe
IF bLoadRecipe THEN
    bLoadRecipe := FALSE;
    stRecipe := arrRecipes[nSelectedRecipe];
    rTargetTemp := stRecipe.rTargetTemp;
    rTargetPress := stRecipe.rTargetPress;
    nMotorSpeed := stRecipe.nMotorSpeed;
END_IF

// Save recipe
IF bSaveRecipe THEN
    bSaveRecipe := FALSE;
    stRecipe.rTargetTemp := rTargetTemp;
    stRecipe.rTargetPress := rTargetPress;
    stRecipe.nMotorSpeed := nMotorSpeed;
    arrRecipes[nSelectedRecipe] := stRecipe;
END_IF
```

---

## 5. User Management

### 5.1 User Groups

```iecst
// Configure in Visualization Manager → User Management
// Define groups:
//   - Operator: Can view pages, acknowledge alarms, start/stop
//   - Engineer: Can modify setpoints, view diagnostics
//   - Admin: Can modify recipes, access all settings, manage users

// Use "Access Rights" on each visualization element:
//   - Operator: Read/Write for control elements
//   - Engineer: Read/Write for setpoint elements
//   - Admin: Read/Write for configuration elements
```

### 5.2 Login/Logout via Code

```iecst
VAR
    fbVisuUserMgmt : VisuUserMgmt;
    sUsername      : STRING(50);
    sPassword      : STRING(50);
    bLogin         : BOOL;
    bLogout        : BOOL;
    bIsLoggedIn    : BOOL;
    sCurrentUser   : STRING(50);
END_VAR

// Login
IF bLogin THEN
    bLogin := FALSE;
    fbVisuUserMgmt.Login(
        sUserName := sUsername,
        sPassword := sPassword
    );
END_IF

// Logout
IF bLogout THEN
    bLogout := FALSE;
    fbVisuUserMgmt.Logout();
END_IF

// Check current user
bIsLoggedIn := fbVisuUserMgmt.IsUserLoggedIn();
sCurrentUser := fbVisuUserMgmt.GetCurrentUser();
```

---

## 6. Best Practices for Visualization

### 6.1 Layout Guidelines

- **Resolution**: Design for 1024x768 minimum (standard HMI panel)
- **Touch targets**: Minimum 40x40 pixels for buttons
- **Font size**: Minimum 12pt for text, 16pt for values
- **Color scheme**: Dark background (gray/blue) with bright text for reduced eye strain
- **Grouping**: Use frames to group related elements
- **Navigation**: Always show current page name and provide "Home" button

### 6.2 Performance Tips

- Limit elements per page to ~50 for smooth rendering
- Use "Invisible" property for page switching (not separate visu objects) to avoid reload
- Avoid polling fast-updating variables on every element — use a shared update timer
- Use integer types for display where possible (faster than LREAL rendering)
- Cache alarm history in arrays rather than querying every scan

### 6.3 Responsive Design

```
// Use relative positioning for responsive layout
// In element properties:
//   X := VisuElemWidth * 0.1   (10% from left)
//   Y := VisuElemHeight * 0.2  (20% from top)
//   Width := VisuElemWidth * 0.3  (30% of screen width)
```

### 6.4 Mobile Considerations

- Design for touch (larger buttons, no hover effects)
- Test on actual mobile devices (rendering may differ)
- Use CODESYS app for native mobile experience
- Consider screen orientation (portrait vs landscape)
- Reduce data update rate for mobile (bandwidth concerns)
