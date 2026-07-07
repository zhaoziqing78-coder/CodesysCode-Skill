# Communication Protocols in CODESYS

## 1. Modbus TCP

### 1.1 Modbus TCP Client (Master)

Use the `SysSocket` library or the Modbus TCP FB library (`IoDrvModbusTcp`).

#### FB: Modbus TCP Client via SysSocket

```iecst
FUNCTION_BLOCK FB_ModbusTcpClient

VAR_INPUT
    bEnable     : BOOL;        // Enable communication
    sIpAddr     : STRING(15);  // Target IP, e.g., '192.168.1.100'
    nPort       : UINT := 502; // Modbus TCP port
    nUnitId     : BYTE := 1;   // Slave unit ID
    tTimeout    : TIME := T#2S;
END_VAR

VAR_OUTPUT
    bConnected  : BOOL;
    bError      : BOOL;
    nErrorCode  : UINT;
    nState      : UINT;
END_VAR

VAR
    fbSocket    : SYSSOCKET;
    hSocket     : SYSHANDLE;
    tLastAction : TIME;
    nRetryCount : UINT;
    arrTxBuffer : ARRAY[0..259] OF BYTE; // MBAP header (7) + PDU max 253
    arrRxBuffer : ARRAY[0..259] OF BYTE;
    nTxLength   : UINT;
    nRxLength   : UINT;
    nTransactionId : UINT := 1;
END_VAR

VAR CONSTANT
    STATE_IDLE      : UINT := 0;
    STATE_CONNECT   : UINT := 10;
    STATE_CONNECTED : UINT := 20;
    STATE_SEND      : UINT := 30;
    STATE_RECV      : UINT := 40;
    STATE_ERROR     : UINT := 99;
END_VAR
```

#### Read Holding Registers (Function Code 03)

```iecst
// Build MBAP header + PDU for "Read Holding Registers"
// Transaction ID: 2 bytes (arrTxBuffer[0..1])
// Protocol ID: 2 bytes = 0 (arrTxBuffer[2..3])
// Length: 2 bytes = 6 (arrTxBuffer[4..5])
// Unit ID: 1 byte (arrTxBuffer[6])
// Function code: 03 (arrTxBuffer[7])
// Starting address: 2 bytes (arrTxBuffer[8..9])
// Quantity: 2 bytes (arrTxBuffer[10..11])

arrTxBuffer[0] := BYTE#(nTransactionId / 256);
arrTxBuffer[1] := BYTE#(nTransactionId MOD 256);
arrTxBuffer[2] := 0; // Protocol ID = 0
arrTxBuffer[3] := 0;
arrTxBuffer[4] := 0; // Length = 6
arrTxBuffer[5] := 6;
arrTxBuffer[6] := nUnitId;
arrTxBuffer[7] := 3;  // Function code 03
arrTxBuffer[8] := BYTE#(nStartAddr / 256);
arrTxBuffer[9] := BYTE#(nStartAddr MOD 256);
arrTxBuffer[10] := BYTE#(nQuantity / 256);
arrTxBuffer[11] := BYTE#(nQuantity MOD 256);

nTxLength := 12;
nTransactionId := nTransactionId + 1;
```

#### Write Multiple Registers (Function Code 16)

```iecst
// Function code 16: Write Multiple Registers
// PDU: FC(1) + StartAddr(2) + Quantity(2) + ByteCount(1) + Data(n*2)

arrTxBuffer[7] := 16;  // Function code 0x10
arrTxBuffer[8] := BYTE#(nStartAddr / 256);
arrTxBuffer[9] := BYTE#(nStartAddr MOD 256);
arrTxBuffer[10] := BYTE#(nQuantity / 256);
arrTxBuffer[11] := BYTE#(nQuantity MOD 256);
arrTxBuffer[12] := BYTE#(nQuantity * 2); // Byte count

// Fill register data (little-endian in array, big-endian on wire)
FOR i := 0 TO nQuantity - 1 DO
    arrTxBuffer[13 + i*2]   := BYTE#(arrData[i] / 256);     // High byte
    arrTxBuffer[14 + i*2]   := BYTE#(arrData[i] MOD 256);   // Low byte
END_FOR

nTxLength := 13 + nQuantity * 2;
```

### 1.2 Modbus Register Mapping

| Reference | Address Range | Access | Function Code |
|-----------|--------------|--------|---------------|
| Coils | 00001-09999 | R/W | 01 (read), 05 (write single), 15 (write multiple) |
| Discrete Inputs | 10001-19999 | R | 02 |
| Input Registers | 30001-39999 | R | 04 |
| Holding Registers | 40001-49999 | R/W | 03 (read), 06 (write single), 16 (write multiple) |

### 1.3 Using CODESYS Modbus TCP Slave

Configure via device tree:
1. Add "Modbus TCP Slave Device" under Ethernet adapter
2. Configure IP and port in device settings
3. Add "Modbus Slave Channel" with register mapping
4. Map PLC variables to register addresses

```iecst
// Variables automatically mapped to Modbus registers
// Example: Holding register 40001 mapped to rTemperature
VAR
    rTemperature : LREAL;  // Maps to 2 registers (40001-40002)
    rPressure    : LREAL;  // Maps to 2 registers (40003-40004)
    bMotorRun    : BOOL;   // Maps to coil 00001
END_VAR
```

### 1.4 Modbus RTU

For serial communication (RS-232/RS-485):

```iecst
// Use COM port with SysCom library
VAR
    fbComPort : SYSCOM;
    nPort     : UINT := 1; // COM1
    arrRtuTx  : ARRAY[0..259] OF BYTE;
    arrRtuRx  : ARRAY[0..259] OF BYTE;
    nCrc      : UINT;
END_VAR

// RTU frame: SlaveAddr(1) + Function(1) + Data(n) + CRC(2)
// CRC-16 (Modbus): polynomial 0xA001

// CRC calculation (lookup table version)
FUNCTION FC_CalcCRC16 : UINT
VAR_INPUT
    pData : POINTER TO BYTE;
    nLen  : UINT;
END_VAR
VAR
    nCRC : UINT := 16#FFFF;
    nIdx : UINT;
    nByte: BYTE;
END_VAR

FOR nIdx := 0 TO nLen - 1 DO
    nByte := pData[nIdx];
    nCRC := nCRC XOR UINT#nByte;
    nCRC := (nCRC SHR 1) XOR (16#A001 * (nCRC AND 1));
    nCRC := (nCRC SHR 1) XOR (16#A001 * (nCRC AND 1));
    nCRC := (nCRC SHR 1) XOR (16#A001 * (nCRC AND 1));
    nCRC := (nCRC SHR 1) XOR (16#A001 * (nCRC AND 1));
    nCRC := (nCRC SHR 1) XOR (16#A001 * (nCRC AND 1));
    nCRC := (nCRC SHR 1) XOR (16#A001 * (nCRC AND 1));
    nCRC := (nCRC SHR 1) XOR (16#A001 * (nCRC AND 1));
    nCRC := (nCRC SHR 1) XOR (16#A001 * (nCRC AND 1));
END_FOR

FC_CalcCRC16 := nCRC;
END_FUNCTION
```

---

## 2. EtherCAT

### 2.1 Configuration (Device Tree Based)

EtherCAT is configured in the device tree, not in ST code:

1. Add "EtherCAT Master" under Ethernet adapter
2. Scan for slaves or add from ESI files
3. Configure PDO mapping for cyclic data
4. Map process data to PLC variables

### 2.2 Accessing EtherCAT Process Data

```iecst
// Variables are auto-generated from PDO mapping
// Example: Digital I/O slave
VAR
    // Inputs (from slave to master)
    bDi0 AT %IX0.0 : BOOL;
    bDi1 AT %IX0.1 : BOOL;
    bDi2 AT %IX0.2 : BOOL;
    bDi3 AT %IX0.3 : BOOL;

    // Outputs (from master to slave)
    bDo0 AT %QX0.0 : BOOL;
    bDo1 AT %QX0.1 : BOOL;
    bDo2 AT %QX0.2 : BOOL;
    bDo3 AT %QX0.3 : BOOL;
END_VAR

// Analog I/O example
VAR
    rAnalogIn0 AT %IW0 : INT;   // Raw value 0-32767
    rAnalogOut0 AT %QW0 : INT;
END_VAR

// Convert raw to engineering units
rTemperature := LREAL#rAnalogIn0 * 100.0 / 32767.0; // 0-100 degC
```

### 2.3 SDO Access (Acyclic Parameters)

```iecst
// Use EtherCATInfo FB for SDO read/write
VAR
    fbSdoRead  : FB_EcSdoRead;
    fbSdoWrite : FB_EcSdoWrite;
    bReadTrigger : BOOL;
    bWriteTrigger : BOOL;
    nSdoValue : DINT;
    nSdoResult : UINT;
END_VAR

// SDO Read: Index 0x6040, Subindex 0x00
fbSdoRead(
    sNetName := 'EtherCAT_Master',
    nSlavePos := 0,
    nIndex := 16#6040,
    nSubIndex := 0,
    pData := ADR(nSdoValue),
    nDataSize := SIZEOF(nSdoValue),
    bExecute := bReadTrigger,
    nResult => nSdoResult
);

// SDO Write: Index 0x6060, Subindex 0x00 (mode of operation)
fbSdoWrite(
    sNetName := 'EtherCAT_Master',
    nSlavePos := 0,
    nIndex := 16#6060,
    nSubIndex := 0,
    pData := ADR(nSdoValue),
    nDataSize := SIZEOF(nSdoValue),
    bExecute := bWriteTrigger,
    nResult => nSdoResult
);
```

### 2.4 Drive Control (CiA 402 Profile)

Standard servo drive control via EtherCAT using CiA 402 state machine:

```iecst
VAR
    // Control word (0x6040) and Status word (0x6041)
    wControlWord : WORD AT %QW2;
    wStatusWord  : WORD AT %IW2;

    // Mode of operation (0x6060)
    nOpMode      : SINT AT %QB4;

    // Target position (0x607A) for CSP mode
    nTargetPos   : DINT AT %QD6;

    // Actual position (0x6064)
    nActPos      : DINT AT %ID6;
END_VAR

VAR CONSTANT
    // Control word bits
    CW_ENABLE      : WORD := 16#0001;
    CW_RESET_FAULT : WORD := 16#0080;
    CW_ENABLE_OP   : WORD := 16#000F;
    CW_NEW_SET     : WORD := 16#0010;
    CW_IMMEDIATE   : WORD := 16#0020;
END_VAR

// State machine: Not Ready -> Switch On Disabled -> Ready To Switch On
// -> Switched On -> Operation Enabled

// Fault reset
IF (wStatusWord AND 16#0008) <> 0 THEN  // Fault bit
    wControlWord := CW_RESET_FAULT;
ELSIF (wStatusWord AND 16#0040) <> 0 THEN // Switch on disabled
    wControlWord := CW_ENABLE;
ELSIF (wStatusWord AND 16#0021) = 16#0021 THEN // Ready to switch on
    wControlWord := CW_ENABLE_OP;
END_IF
```

---

## 3. PROFINET

### 3.1 Configuration (Device Tree Based)

1. Add "PROFINET IO Device" under Ethernet adapter
2. Import GSDML files for connected devices
3. Configure slots and subslots with modules
4. Map I/O addresses to PLC variables

### 3.2 Process Data Access

```iecst
// Auto-generated from PROFINET configuration
// Digital input module (4 bytes = 32 bits)
VAR
    bPnDi0   AT %IB10 : BYTE;  // Slot 1, submodule 1
    bPnDi1   AT %IB11 : BYTE;
    bPnDo0   AT %QB10 : BYTE;
    bPnDo1   AT %QB11 : BYTE;

    // Analog input module
    nPnAi0   AT %IW12 : INT;
    nPnAi1   AT %IW13 : INT;
    nPnAo0   AT %QW12 : INT;
END_VAR

// Read PROFINET digital inputs
bSensor1 := bPnDi0.0;
bSensor2 := bPnDi0.1;

// Write PROFINET digital outputs
bPnDo0.0 := bMotorEnable;
bPnDo0.1 := bValveOpen;
```

### 3.3 PROFINET Alarm Handling

```iecst
VAR
    fbPnAlarm : FB_PnAlarm;
    bPnDiag   : BOOL;
    sPnDiagMsg : STRING(80);
END_VAR

fbPnAlarm(
    sDeviceName := 'PN_Device_1',
    bEnable := TRUE,
    bDiagnostic => bPnDiag,
    sMessage => sPnDiagMsg
);
```

---

## 4. OPC UA

### 4.1 Built-in OPC UA Server

CODESYS includes a built-in OPC UA server:

1. Add "OPC UA Server" device in device tree
2. Configure port (default 4840)
3. Enable "Symbol Configuration" to expose PLC variables
4. Select which variables to publish

```iecst
// Variables exposed via OPC UA are automatically accessible
// No additional code needed — just declare variables
VAR
    // These will appear under the OPC UA namespace
    rOpcUa_Temperature : LREAL;
    rOpcUa_Pressure    : LREAL;
    bOpcUa_MotorRun    : BOOL;
    nOpcUa_SensorCount : UINT;
END_VAR
```

### 4.2 OPC UA Client

```iecst
// Requires OPC UA Client library
VAR
    fbOpcUaClient : FB_OpcUaClient;
    fbUaRead      : FB_UaRead;
    fbUaWrite     : FB_UaWrite;

    bConnect      : BOOL;
    bConnected    : BOOL;
    bReadTrigger  : BOOL;
    bWriteTrigger : BOOL;

    nEndpointUrl  : STRING(100) := 'opc.tcp://192.168.1.100:4840';
    nNodeId       : STRING(100) := 'ns=4;s=MyVariable';

    rReadValue    : LREAL;
    rWriteValue   : LREAL;
    nReadStatus   : UINT;
    nWriteStatus  : UINT;
END_VAR

// Connect to OPC UA server
fbOpcUaClient(
    sEndpointUrl := nEndpointUrl,
    eSecurityMode := eUaSecurityMode_None,
    bExecute := bConnect,
    bConnected => bConnected
);

// Read a node
fbUaRead(
    fbOpcUaClient := fbOpcUaClient,
    sNodeId := nNodeId,
    eDataType := eUaType_Double,
    pData := ADR(rReadValue),
    nDataSize := SIZEOF(rReadValue),
    bExecute := bReadTrigger,
    nStatus => nReadStatus
);

// Write a node
fbUaWrite(
    fbOpcUaClient := fbOpcUaClient,
    sNodeId := nNodeId,
    eDataType := eUaType_Double,
    pData := ADR(rWriteValue),
    nDataSize := SIZEOF(rWriteValue),
    bExecute := bWriteTrigger,
    nStatus => nWriteStatus
);
```

### 4.3 OPC UA Subscription (Monitored Items)

```iecst
VAR
    fbUaSubscribe : FB_UaSubscribeItem;
    bSubTrigger   : BOOL;
    rSubValue     : LREAL;
    nSubStatus    : UINT;
    bValueChanged : BOOL;
END_VAR

fbUaSubscribe(
    fbOpcUaClient := fbOpcUaClient,
    sNodeId := nNodeId,
    eDataType := eUaType_Double,
    pData := ADR(rSubValue),
    nDataSize := SIZEOF(rSubValue),
    tPublishingInterval := T#500MS,
    bExecute := bSubTrigger,
    bValueChanged => bValueChanged,
    nStatus => nSubStatus
);
```

---

## 5. Generic Communication FB Template

### FB: Connection Manager with Auto-Reconnect

```iecst
FUNCTION_BLOCK FB_CommManager

VAR_INPUT
    bEnable     : BOOL;       // Enable communication
    tTimeout    : TIME := T#3S;
    nMaxRetries : UINT := 3;  // Max retry attempts
END_VAR

VAR_OUTPUT
    bConnected  : BOOL;
    bError      : BOOL;
    nErrorCode  : UINT;
    nRetryCount : UINT;
    nState      : UINT;
END_VAR

VAR
    LastEnable  : BOOL;
    tTimer      : TON;
    tRetryTimer : TON;
    nRetryIdx   : UINT;
END_VAR

VAR CONSTANT
    ST_DISCONNECTED : UINT := 0;
    ST_CONNECTING   : UINT := 10;
    ST_CONNECTED    : UINT := 20;
    ST_WAIT_RETRY   : UINT := 30;
    ST_ERROR        : UINT := 99;
END_VAR

// Edge-triggered enable
IF bEnable AND NOT LastEnable THEN
    nState := ST_CONNECTING;
    nRetryIdx := 0;
    bError := FALSE;
    nErrorCode := 0;
ELSIF NOT bEnable AND LastEnable THEN
    nState := ST_DISCONNECTED;
    bConnected := FALSE;
    bError := FALSE;
END_IF
LastEnable := bEnable;

// State machine
CASE nState OF
    ST_DISCONNECTED:
        bConnected := FALSE;

    ST_CONNECTING:
        // Subclass implements actual connection logic
        // Call virtual M_DoConnect() or use specific FB
        IF bConnected THEN
            nState := ST_CONNECTED;
            nRetryIdx := 0;
        ELSIF bError THEN
            IF nRetryIdx < nMaxRetries THEN
                nState := ST_WAIT_RETRY;
                tRetryTimer(IN := TRUE, PT := T#5S);
            ELSE
                nState := ST_ERROR;
            END_IF
        END_IF

    ST_CONNECTED:
        // Monitor connection
        IF NOT bConnected THEN
            nState := ST_CONNECTING;
            nRetryIdx := nRetryIdx + 1;
        END_IF

    ST_WAIT_RETRY:
        IF tRetryTimer.Q THEN
            tRetryTimer(IN := FALSE);
            nRetryIdx := nRetryIdx + 1;
            nState := ST_CONNECTING;
            bError := FALSE;
        END_IF

    ST_ERROR:
        bConnected := FALSE;
        bError := TRUE;
END_CASE

nRetryCount := nRetryIdx;

END_FUNCTION_BLOCK
```
