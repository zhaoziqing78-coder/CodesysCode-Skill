# Cam Table Design Patterns and Algorithms

## 1. Cam Table Fundamentals

A cam table maps a master axis position to a slave axis position:
- X-axis = Master (encoder position, or winding axis angle)
- Y-axis = Slave (desired output position)

In CODESYS, cam tables are used with:
- `MC_CamIn` — activates electronic cam coupling
- `MC_CamTableSelect` — selects a cam table
- CAM Editor — visual curve design tool

## 2. Data Acquisition Patterns

### Pattern A: Angle-Triggered Capture
- **Trigger source**: Winding axis angle (Y-axis feedback)
- **Captured value**: Encoder position (X-axis)
- **Interval**: Every N degrees of winding axis rotation
- **Use case**: Map winding angle → material feed position

### Pattern B: Position-Triggered Capture
- **Trigger source**: Encoder position (X-axis)
- **Captured value**: Winding axis angle (Y-axis)
- **Interval**: Every N units of encoder travel
- **Use case**: Map material feed → required winding angle

### Pattern C: Time-Triggered Capture
- **Trigger source**: Timer/PLC scan count
- **Captured value**: Both encoder and angle
- **Use case**: Diagnostic logging, not for cam table generation

## 3. Bidirectional Capture (ABS Difference Trigger)

> **WARNING**: Do NOT use direction-based triggering (`MoveDir` / `NextTarget`).
> Direction jitter (e.g., 120.0 → 119.99 → 120.01) causes the direction flag to flip,
> which re-satisfies the trigger condition and produces massive duplicate captures.
>
> **Always use absolute difference instead**:

```iecst
// Calculate absolute difference from last captured point
AbsDelta := TriggerSource - LastCaptured;
IF AbsDelta < 0.0 THEN
    AbsDelta := -AbsDelta;
END_IF;

// Trigger when ABS difference reaches step interval
IF AbsDelta >= StepInterval THEN
    // Capture data point
    DataArray[ActualLength] := SensorValue;
    LastCaptured := TriggerSource;
    ActualLength := ActualLength + 1;
END_IF;
```

**Why ABS is better:**
- No direction state → no direction jitter → no duplicate captures
- Works for both forward and reverse rotation automatically
- Self-correcting: if axis jitters back, AbsDelta shrinks, no trigger fires
- Simpler code: 3 variables vs 5 (MoveDir, LastTrigger, NextTarget, etc.)

## 4. Curve Fitting Algorithms

### 4.1 Average Slope (Linear Fit)

```
AvgSlope = (X[end] - X[0]) / (Y[end] - Y[0])
```

Interpretation: For every 1 degree of winding axis rotation, the encoder moves by AvgSlope units.

Linear reference curve:
```
FitX[i] = X[0] + AvgSlope * (Y[i] - Y[0])
FitY[i] = Y[i]
```

### 4.2 Moving Average Smoothing

Window size N (must be odd: 3, 5, 7):

```
Half = (N - 1) / 2
FitX[i] = average(X[i-Half .. i+Half])  // clipped at array boundaries
FitY[i] = average(Y[i-Half .. i+Half])
```

- N=3: light smoothing, preserves detail
- N=5: moderate smoothing, good for noisy signals
- N=7: heavy smoothing, may introduce lag

### 4.3 Exponential Smoothing (Alternative)

```
alpha = 0.3  // smoothing factor (0..1), higher = less smoothing
FitX[0] = X[0]
FitX[i] = alpha * X[i] + (1 - alpha) * FitX[i-1]
```

## 5. Array Design Considerations

- **Compile-time max**: 10000 elements (practical limit for memory)
- **Runtime effective**: controlled by `MaxArrayLength` input
- **Memory**: each LREAL = 8 bytes; 10000 * 2 arrays * 8 = 160KB per FB instance
- **Scan time**: processing 10000 points in one cycle may take 1-5ms depending on CPU

## 6. Integration with CODESYS SoftMotion

```iecst
// Connect to SoftMotion axis
CamGen.EncoderPosition  := Axis_Encoder.Position;
CamGen.WindingAxisAngle := Axis_Winding.Position;

// After fitting, use in MC_CamIn
// Method A: Export to CSV, import to CAM Editor
// Method B: Directly use arrays with MC_CamTableSelect
```

## 7. Quality Metrics

After fitting, evaluate curve quality:
- **R² (coefficient of determination)**: how well linear fit matches data
- **Max deviation**: largest difference between original and fitted
- **Standard deviation**: spread of residuals
- **Smoothness**: second derivative magnitude (lower = smoother)

These can be added as additional FB features if needed.
