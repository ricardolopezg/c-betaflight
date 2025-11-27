---
title: Processing MSP Packets
---
This document describes the flow for handling an incoming MSP packet, interpreting its command, and generating a response. This is the main protocol interface for telemetry and configuration, enabling communication between the flight controller and external tools. The flow covers both data retrieval and configuration changes.

```mermaid
flowchart TD
  node1["Starting MSP Packet Processing"]:::HeadingStyle
  click node1 goToHeading "Starting MSP Packet Processing"
  node1 --> node2["Dispatching MSP Command"]:::HeadingStyle
  click node2 goToHeading "Dispatching MSP Command"
  node2 --> node3{"Which type of command?"}
  node3 -->|"Standard"| node4["Handling Standard MSP Commands"]:::HeadingStyle
  click node4 goToHeading "Handling Standard MSP Commands"
  node3 -->|"Fallback"| node5["Fallback MSP Command Handling"]:::HeadingStyle
  click node5 goToHeading "Fallback MSP Command Handling"
  node5 --> node6{"Advanced or Final Fallback?"}
  node6 -->|"Advanced"| node7["Advanced MSP Command Handling"]:::HeadingStyle
  click node7 goToHeading "Advanced MSP Command Handling"
  node6 -->|"Final Fallback"| node8["Final MSP Command Fallbacks"]:::HeadingStyle
  click node8 goToHeading "Final MSP Command Fallbacks"
  node8 --> node9{"Is this a config command?"}
  node9 -->|"Yes"| node10["Processing MSP Input/Config Commands"]:::HeadingStyle
  click node10 goToHeading "Processing MSP Input/Config Commands"
  node10 --> node11["Deep Config and EEPROM Handling"]:::HeadingStyle
  click node11 goToHeading "Deep Config and EEPROM Handling"
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

# Starting MSP Packet Processing

<SwmSnippet path="/src/main/telemetry/msp_shared.c" line="144">

---

In <SwmToken path="src/main/telemetry/msp_shared.c" pos="144:4:4" line-data="static void processMspPacket(void)">`processMspPacket`</SwmToken>, we prep the response buffer and clear out any previous state, then immediately call into <SwmToken path="src/main/telemetry/msp_shared.c" pos="152:4:4" line-data="    if (mspFcProcessCommand(mspSharedDescriptor, &amp;requestPacket, &amp;responsePacket, &amp;mspPostProcessFn) == MSP_RESULT_ERROR) {">`mspFcProcessCommand`</SwmToken> to actually handle the incoming MSP command. This is where the main protocol logic happens, so we need to call into <SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath> next to interpret the request and fill out the response. If that fails, we write an error byte to the response buffer.

```c
static void processMspPacket(void)
{
    responsePacket.cmd = 0;
    responsePacket.result = 0;
    responsePacket.buf.ptr = responseBuffer;
    responsePacket.buf.end = ARRAYEND(responseBuffer);

    mspPostProcessFnPtr mspPostProcessFn = NULL;
    if (mspFcProcessCommand(mspSharedDescriptor, &requestPacket, &responsePacket, &mspPostProcessFn) == MSP_RESULT_ERROR) {
        sbufWriteU8(&responsePacket.buf, TELEMETRY_MSP_ERROR);
    }
```

---

</SwmSnippet>

## Dispatching MSP Command

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
    node1["Receive MSP command"] --> node2{"Is command a common outgoing command?"}
    click node1 openCode "src/main/msp/msp.c:4421:4430"
    node2 -->|"Yes"| node3["Handling Standard MSP Commands"]
    click node2 openCode "src/main/msp/msp.c:4430:4431"
    
    node2 -->|"No"| node4{"Is command a specific outgoing command?"}
    node4 -->|"Yes"| node5["Advanced MSP Command Handling"]
    click node4 openCode "src/main/msp/msp.c:4432:4434"
    
    node4 -->|"No"| node6{"Is command MSP_SET_PASSTHROUGH?"}
    node6 -->|"Yes"| node7["Call mspFcSetPassthroughCommand"]
    click node6 openCode "src/main/msp/msp.c:4436:4438"
    node6 -->|"No"| node8{"Is command MSP_DATAFLASH_READ?"}
    node8 -->|"Yes"| node9["Call mspFcDataFlashReadCommand"]
    click node8 openCode "src/main/msp/msp.c:4440:4442"
    node8 -->|"No"| node10{"Is command a common incoming command?"}
    node10 -->|"Yes"| node11["Processing MSP Input/Config Commands"]
    click node10 openCode "src/main/msp/msp.c:4444:4445"
    
    node10 -->|"No"| node12["Deep Config and EEPROM Handling"]
    
    node3 --> node13["Set reply and return result"]
    node5 --> node13
    node7 --> node13
    node9 --> node13
    node11 --> node13
    node12 --> node13
    click node13 openCode "src/main/msp/msp.c:4447:4449"
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
click node3 goToHeading "Handling Standard MSP Commands"
node3:::HeadingStyle
click node5 goToHeading "Advanced MSP Command Handling"
node5:::HeadingStyle
click node11 goToHeading "Processing MSP Input/Config Commands"
node11:::HeadingStyle
click node12 goToHeading "Deep Config and EEPROM Handling"
node12:::HeadingStyle

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%     node1["Receive MSP command"] --> node2{"Is command a common outgoing command?"}
%%     click node1 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4421:4430"
%%     node2 -->|"Yes"| node3["Handling Standard MSP Commands"]
%%     click node2 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4430:4431"
%%     
%%     node2 -->|"No"| node4{"Is command a specific outgoing command?"}
%%     node4 -->|"Yes"| node5["Advanced MSP Command Handling"]
%%     click node4 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4432:4434"
%%     
%%     node4 -->|"No"| node6{"Is command <SwmToken path="src/main/msp/msp.c" pos="4436:12:12" line-data="    } else if (cmdMSP == MSP_SET_PASSTHROUGH) {">`MSP_SET_PASSTHROUGH`</SwmToken>?"}
%%     node6 -->|"Yes"| node7["Call <SwmToken path="src/main/msp/msp.c" pos="4437:1:1" line-data="        mspFcSetPassthroughCommand(dst, src, mspPostProcessFn);">`mspFcSetPassthroughCommand`</SwmToken>"]
%%     click node6 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4436:4438"
%%     node6 -->|"No"| node8{"Is command <SwmToken path="src/main/msp/msp.c" pos="4440:12:12" line-data="    } else if (cmdMSP == MSP_DATAFLASH_READ) {">`MSP_DATAFLASH_READ`</SwmToken>?"}
%%     node8 -->|"Yes"| node9["Call <SwmToken path="src/main/msp/msp.c" pos="4441:1:1" line-data="        mspFcDataFlashReadCommand(dst, src);">`mspFcDataFlashReadCommand`</SwmToken>"]
%%     click node8 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4440:4442"
%%     node8 -->|"No"| node10{"Is command a common incoming command?"}
%%     node10 -->|"Yes"| node11["Processing MSP Input/Config Commands"]
%%     click node10 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4444:4445"
%%     
%%     node10 -->|"No"| node12["Deep Config and EEPROM Handling"]
%%     
%%     node3 --> node13["Set reply and return result"]
%%     node5 --> node13
%%     node7 --> node13
%%     node9 --> node13
%%     node11 --> node13
%%     node12 --> node13
%%     click node13 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4447:4449"
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
%% click node3 goToHeading "Handling Standard MSP Commands"
%% node3:::HeadingStyle
%% click node5 goToHeading "Advanced MSP Command Handling"
%% node5:::HeadingStyle
%% click node11 goToHeading "Processing MSP Input/Config Commands"
%% node11:::HeadingStyle
%% click node12 goToHeading "Deep Config and EEPROM Handling"
%% node12:::HeadingStyle
```

<SwmSnippet path="/src/main/msp/msp.c" line="4421">

---

In <SwmToken path="src/main/msp/msp.c" pos="4421:2:2" line-data="mspResult_e mspFcProcessCommand(mspDescriptor_t srcDesc, mspPacket_t *cmd, mspPacket_t *reply, mspPostProcessFnPtr *mspPostProcessFn)">`mspFcProcessCommand`</SwmToken>, we prep the reply buffer and copy over the command, then immediately try to handle the command with <SwmToken path="src/main/msp/msp.c" pos="4430:4:4" line-data="    if (mspCommonProcessOutCommand(cmdMSP, dst, mspPostProcessFn)) {">`mspCommonProcessOutCommand`</SwmToken>. This is the fast path for standard MSP commands. If that doesn't handle it, we fall through to other handlers.

```c
mspResult_e mspFcProcessCommand(mspDescriptor_t srcDesc, mspPacket_t *cmd, mspPacket_t *reply, mspPostProcessFnPtr *mspPostProcessFn)
{
    int ret = MSP_RESULT_ACK;
    sbuf_t *dst = &reply->buf;
    sbuf_t *src = &cmd->buf;
    const int16_t cmdMSP = cmd->cmd;
    // initialize reply by default
    reply->cmd = cmd->cmd;

    if (mspCommonProcessOutCommand(cmdMSP, dst, mspPostProcessFn)) {
```

---

</SwmSnippet>

### Handling Standard MSP Commands

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
    node1{"Which command is requested?"}
    click node1 openCode "src/main/msp/msp.c:631:1081"
    node1 -->|"API Version"| node2["Send API version info"]
    click node2 openCode "src/main/msp/msp.c:632:636"
    node1 -->|"Board Info"| node3["Send board info (features: targetCapabilities, issues: configurationProblems)"]
    click node3 openCode "src/main/msp/msp.c:655:754"
    node1 -->|"OSD Config"| node4["Send OSD configuration (hardware: osdFlags, items, alarms, timers)"]
    click node4 openCode "src/main/msp/msp.c:943:1068"
    node1 -->|"Debug Values"| node5["Send debug values"]
    click node5 openCode "src/main/msp/msp.c:773:775"
    subgraph loop1["For each debug value"]
      node5
    end
    node1 -->|"Voltage Meters"| node6["Send voltage meter values"]
    click node6 openCode "src/main/msp/msp.c:813:828"
    subgraph loop2["For each voltage meter"]
      node6
    end
    node1 -->|"Current Meters"| node7["Send current meter values"]
    click node7 openCode "src/main/msp/msp.c:832:847"
    subgraph loop3["For each current meter"]
      node7
    end
    node1 -->|"Transponder Config"| node8{"Which provider is selected?"}
    click node8 openCode "src/main/msp/msp.c:916:934"
    node8 -->|"Provider selected"| node9["Send provider data"]
    click node9 openCode "src/main/msp/msp.c:928:934"
    node8 -->|"No provider"| node10["Send no provider info"]
    click node10 openCode "src/main/msp/msp.c:937:937"
    node1 -->|"Other"| node11["Send specific data for requested command"]
    click node11 openCode "src/main/msp/msp.c:631:1081"
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%     node1{"Which command is requested?"}
%%     click node1 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:631:1081"
%%     node1 -->|"API Version"| node2["Send API version info"]
%%     click node2 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:632:636"
%%     node1 -->|"Board Info"| node3["Send board info (features: <SwmToken path="src/main/msp/msp.c" pos="674:3:3" line-data="        uint8_t targetCapabilities = 0;">`targetCapabilities`</SwmToken>, issues: <SwmToken path="src/main/msp/msp.c" pos="727:3:3" line-data="        uint32_t configurationProblems = 0;">`configurationProblems`</SwmToken>)"]
%%     click node3 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:655:754"
%%     node1 -->|"OSD Config"| node4["Send OSD configuration (hardware: <SwmToken path="src/main/msp/msp.c" pos="953:3:3" line-data="        uint8_t osdFlags = 0;">`osdFlags`</SwmToken>, items, alarms, timers)"]
%%     click node4 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:943:1068"
%%     node1 -->|"Debug Values"| node5["Send debug values"]
%%     click node5 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:773:775"
%%     subgraph loop1["For each debug value"]
%%       node5
%%     end
%%     node1 -->|"Voltage Meters"| node6["Send voltage meter values"]
%%     click node6 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:813:828"
%%     subgraph loop2["For each voltage meter"]
%%       node6
%%     end
%%     node1 -->|"Current Meters"| node7["Send current meter values"]
%%     click node7 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:832:847"
%%     subgraph loop3["For each current meter"]
%%       node7
%%     end
%%     node1 -->|"Transponder Config"| node8{"Which provider is selected?"}
%%     click node8 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:916:934"
%%     node8 -->|"Provider selected"| node9["Send provider data"]
%%     click node9 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:928:934"
%%     node8 -->|"No provider"| node10["Send no provider info"]
%%     click node10 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:937:937"
%%     node1 -->|"Other"| node11["Send specific data for requested command"]
%%     click node11 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:631:1081"
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/src/main/msp/msp.c" line="627">

---

In <SwmToken path="src/main/msp/msp.c" pos="627:4:4" line-data="static bool mspCommonProcessOutCommand(int16_t cmdMSP, sbuf_t *dst, mspPostProcessFnPtr *mspPostProcessFn)">`mspCommonProcessOutCommand`</SwmToken>, we switch on the command and write protocol-specific data to the output buffer for each recognized MSP command. The function ignores the <SwmToken path="src/main/msp/msp.c" pos="627:20:20" line-data="static bool mspCommonProcessOutCommand(int16_t cmdMSP, sbuf_t *dst, mspPostProcessFnPtr *mspPostProcessFn)">`mspPostProcessFn`</SwmToken> argument, and the actual output depends on both the command and the build configuration. If the command isn't handled here, we return false so other handlers can try.

```c
static bool mspCommonProcessOutCommand(int16_t cmdMSP, sbuf_t *dst, mspPostProcessFnPtr *mspPostProcessFn)
{
    UNUSED(mspPostProcessFn);

    switch (cmdMSP) {
    case MSP_API_VERSION:
        sbufWriteU8(dst, MSP_PROTOCOL_VERSION);
        sbufWriteU8(dst, API_VERSION_MAJOR);
        sbufWriteU8(dst, API_VERSION_MINOR);
        break;

    case MSP_FC_VARIANT:
        sbufWriteData(dst, flightControllerIdentifier, FLIGHT_CONTROLLER_IDENTIFIER_LENGTH);
        break;

    case MSP_FC_VERSION:
        sbufWriteU8(dst, (uint8_t)(FC_VERSION_YEAR - FC_CALVER_BASE_YEAR)); // year since 2000
        sbufWriteU8(dst, FC_VERSION_MONTH);
        sbufWriteU8(dst, FC_VERSION_PATCH_LEVEL);
        sbufWritePString(dst, FC_VERSION_STRING);
        break;

    case MSP2_MCU_INFO: {
        sbufWriteU8(dst, getMcuTypeId());
        sbufWritePString(dst, getMcuTypeName());
        break;
    }

    case MSP_BOARD_INFO: {
        sbufWriteData(dst, systemConfig()->boardIdentifier, BOARD_IDENTIFIER_LENGTH);
#ifdef USE_HARDWARE_REVISION_DETECTION
        sbufWriteU16(dst, hardwareRevision);
#else
        sbufWriteU16(dst, 0); // No other build targets currently have hardware revision detection.
#endif
#if defined(USE_MAX7456)
        sbufWriteU8(dst, 2);  // 2 == FC with MAX7456
#else
        sbufWriteU8(dst, 0);  // 0 == FC
#endif

        // Target capabilities (uint8)
#define TARGET_HAS_VCP 0
#define TARGET_HAS_SOFTSERIAL 1
#define TARGET_HAS_FLASH_BOOTLOADER 3
#define TARGET_SUPPORTS_RX_BIND 6

        uint8_t targetCapabilities = 0;
#ifdef USE_VCP
        targetCapabilities |= BIT(TARGET_HAS_VCP);
#endif
#if defined(USE_SOFTSERIAL)
        targetCapabilities |= BIT(TARGET_HAS_SOFTSERIAL);
#endif
#if defined(USE_FLASH_BOOT_LOADER)
        targetCapabilities |= BIT(TARGET_HAS_FLASH_BOOTLOADER);
#endif
#if defined(USE_RX_BIND)
        if (getRxBindSupported()) {
            targetCapabilities |= BIT(TARGET_SUPPORTS_RX_BIND);
        }
#endif

        sbufWriteU8(dst, targetCapabilities);

        // Target name with explicit length
        sbufWritePString(dst, targetName);

#if defined(USE_BOARD_INFO)
        // Board name with explicit length
        sbufWritePString(dst, getBoardName());

        // Manufacturer id with explicit length
        sbufWritePString(dst, getManufacturerId());
#else
        sbufWriteU8(dst, 0);
        sbufWriteU8(dst, 0);
#endif

#if defined(USE_SIGNATURE)
        // Signature
        sbufWriteData(dst, getSignature(), SIGNATURE_LENGTH);
#else
        uint8_t emptySignature[SIGNATURE_LENGTH];
        memset(emptySignature, 0, sizeof(emptySignature));
        sbufWriteData(dst, &emptySignature, sizeof(emptySignature));
#endif

        sbufWriteU8(dst, getMcuTypeId());

        // Added in API version 1.42
        sbufWriteU8(dst, systemConfig()->configurationState);

        // Added in API version 1.43
        sbufWriteU16(dst, gyro.sampleRateHz); // informational so the configurator can display the correct gyro/pid frequencies in the drop-down

        // Configuration warnings / problems (uint32_t)
#define PROBLEM_ACC_NEEDS_CALIBRATION 0
#define PROBLEM_MOTOR_PROTOCOL_DISABLED 1

        uint32_t configurationProblems = 0;

#if defined(USE_ACC)
        if (!accHasBeenCalibrated()) {
            configurationProblems |= BIT(PROBLEM_ACC_NEEDS_CALIBRATION);
        }
#endif

        if (!checkMotorProtocolEnabled(&motorConfig()->dev, NULL)) {
            configurationProblems |= BIT(PROBLEM_MOTOR_PROTOCOL_DISABLED);
        }

        sbufWriteU32(dst, configurationProblems);

        // Added in MSP API 1.44
#if defined(USE_SPI)
        sbufWriteU8(dst, spiGetRegisteredDeviceCount());
#else
        sbufWriteU8(dst, 0);
#endif
#if defined(USE_I2C)
        sbufWriteU8(dst, i2cGetRegisteredDeviceCount());
#else
        sbufWriteU8(dst, 0);
#endif

        break;
    }

    case MSP_BUILD_INFO:
        sbufWriteData(dst, buildDate, BUILD_DATE_LENGTH);
        sbufWriteData(dst, buildTime, BUILD_TIME_LENGTH);
        sbufWriteData(dst, shortGitRevision, GIT_SHORT_REVISION_LENGTH);
        // Added in API version 1.46
        sbufWriteBuildInfoFlags(dst);
        break;

    case MSP_ANALOG:
        sbufWriteU8(dst, (uint8_t)constrain(getLegacyBatteryVoltage(), 0, 255));
        sbufWriteU16(dst, (uint16_t)constrain(getMAhDrawn(), 0, 0xFFFF)); // milliamp hours drawn from battery
        sbufWriteU16(dst, getRssi());
        sbufWriteU16(dst, (int16_t)constrain(getAmperage(), -0x8000, 0x7FFF)); // send current in 0.01 A steps, range is -320A to 320A
        sbufWriteU16(dst, getBatteryVoltage());
        break;

    case MSP_DEBUG:
        for (int i = 0; i < DEBUG16_VALUE_COUNT; i++) {
            sbufWriteU16(dst, debug[i]);      // 4 variables are here for general monitoring purpose
        }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="778">

---

Here we're just continuing the switch statement, handling more MSP commands like UID, feature config, beeper config, and battery state. Each case is self-contained and writes its own data to the output buffer.

```c
    case MSP_UID:
        sbufWriteU32(dst, U_ID_0);
        sbufWriteU32(dst, U_ID_1);
        sbufWriteU32(dst, U_ID_2);
        break;

    case MSP_FEATURE_CONFIG:
        sbufWriteU32(dst, featureConfig()->enabledFeatures);
        break;

#ifdef USE_BEEPER
    case MSP_BEEPER_CONFIG:
        sbufWriteU32(dst, beeperConfig()->beeper_off_flags);
        sbufWriteU8(dst, beeperConfig()->dshotBeaconTone);
        sbufWriteU32(dst, beeperConfig()->dshotBeaconOffFlags);
        break;
#endif

    case MSP_BATTERY_STATE: {
        // battery characteristics
        sbufWriteU8(dst, (uint8_t)constrain(getBatteryCellCount(), 0, 255)); // 0 indicates battery not detected.
        sbufWriteU16(dst, batteryConfig()->batteryCapacity); // in mAh

        // battery state
        sbufWriteU8(dst, (uint8_t)constrain(getLegacyBatteryVoltage(), 0, 255)); // in 0.1V steps
        sbufWriteU16(dst, (uint16_t)constrain(getMAhDrawn(), 0, 0xFFFF)); // milliamp hours drawn from battery
        sbufWriteU16(dst, (int16_t)constrain(getAmperage(), -0x8000, 0x7FFF)); // send current in 0.01 A steps, range is -320A to 320A

        // battery alerts
        sbufWriteU8(dst, (uint8_t)getBatteryState());

        sbufWriteU16(dst, getBatteryVoltage()); // in 0.01V steps
        break;
    }

    case MSP_VOLTAGE_METERS: {
        // write out id and voltage meter values, once for each meter we support
        uint8_t count = supportedVoltageMeterCount;
#ifdef USE_ESC_SENSOR
        count -= VOLTAGE_METER_ID_ESC_COUNT - getMotorCount();
#endif

        for (int i = 0; i < count; i++) {

            voltageMeter_t meter;
            uint8_t id = (uint8_t)voltageMeterIds[i];
            voltageMeterRead(id, &meter);

            sbufWriteU8(dst, id);
            sbufWriteU8(dst, (uint8_t)constrain((meter.displayFiltered + 5) / 10, 0, 255));
        }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="832">

---

This part handles current meter commands, looping through all supported meters and writing their data to the buffer. It's just another set of cases in the big switch.

```c
    case MSP_CURRENT_METERS: {
        // write out id and current meter values, once for each meter we support
        uint8_t count = supportedCurrentMeterCount;
#ifdef USE_ESC_SENSOR
        count -= VOLTAGE_METER_ID_ESC_COUNT - getMotorCount();
#endif
        for (int i = 0; i < count; i++) {

            currentMeter_t meter;
            uint8_t id = (uint8_t)currentMeterIds[i];
            currentMeterRead(id, &meter);

            sbufWriteU8(dst, id);
            sbufWriteU16(dst, (uint16_t)constrain(meter.mAhDrawn, 0, 0xFFFF)); // milliamp hours drawn from battery
            sbufWriteU16(dst, (uint16_t)constrain(meter.amperage * 10, 0, 0xFFFF)); // send amperage in 0.001 A steps (mA). Negative range is truncated to zero
        }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="851">

---

Here we're handling voltage meter config, writing out the config for each ADC voltage sensor. The payload is structured so clients can parse out each sensor's settings.

```c
    case MSP_VOLTAGE_METER_CONFIG: {
        // by using a sensor type and a sub-frame length it's possible to configure any type of voltage meter,
        // e.g. an i2c/spi/can sensor or any sensor not built directly into the FC such as ESC/RX/SPort/SBus that has
        // different configuration requirements.
        STATIC_ASSERT(VOLTAGE_SENSOR_ADC_VBAT == 0, VOLTAGE_SENSOR_ADC_VBAT_incorrect); // VOLTAGE_SENSOR_ADC_VBAT should be the first index
        sbufWriteU8(dst, MAX_VOLTAGE_SENSOR_ADC); // voltage meters in payload
        for (int i = VOLTAGE_SENSOR_ADC_VBAT; i < MAX_VOLTAGE_SENSOR_ADC; i++) {
            const uint8_t adcSensorSubframeLength = 1 + 1 + 1 + 1 + 1; // length of id, type, vbatscale, vbatresdivval, vbatresdivmultipler, in bytes
            sbufWriteU8(dst, adcSensorSubframeLength); // ADC sensor sub-frame length

            sbufWriteU8(dst, voltageMeterADCtoIDMap[i]); // id of the sensor
            sbufWriteU8(dst, VOLTAGE_SENSOR_TYPE_ADC_RESISTOR_DIVIDER); // indicate the type of sensor that the next part of the payload is for

            sbufWriteU8(dst, voltageSensorADCConfig(i)->vbatscale);
            sbufWriteU8(dst, voltageSensorADCConfig(i)->vbatresdivval);
            sbufWriteU8(dst, voltageSensorADCConfig(i)->vbatresdivmultiplier);
        }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="872">

---

This chunk outputs config for each supported current sensor, handling both ADC and virtual sensors if enabled. The structure is ready for more sensor types if needed.

```c
    case MSP_CURRENT_METER_CONFIG: {
        // the ADC and VIRTUAL sensors have the same configuration requirements, however this API reflects
        // that this situation may change and allows us to support configuration of any current sensor with
        // specialist configuration requirements.

        int currentMeterCount = 1;

#ifdef USE_VIRTUAL_CURRENT_METER
        currentMeterCount++;
#endif
        sbufWriteU8(dst, currentMeterCount);

        const uint8_t adcSensorSubframeLength = 1 + 1 + 2 + 2; // length of id, type, scale, offset, in bytes
        sbufWriteU8(dst, adcSensorSubframeLength);
        sbufWriteU8(dst, CURRENT_METER_ID_BATTERY_1); // the id of the meter
        sbufWriteU8(dst, CURRENT_SENSOR_ADC); // indicate the type of sensor that the next part of the payload is for
        sbufWriteU16(dst, currentSensorADCConfig()->scale);
        sbufWriteU16(dst, currentSensorADCConfig()->offset);

#ifdef USE_VIRTUAL_CURRENT_METER
        const int8_t virtualSensorSubframeLength = 1 + 1 + 2 + 2; // length of id, type, scale, offset, in bytes
        sbufWriteU8(dst, virtualSensorSubframeLength);
        sbufWriteU8(dst, CURRENT_METER_ID_VIRTUAL_1); // the id of the meter
        sbufWriteU8(dst, CURRENT_SENSOR_VIRTUAL); // indicate the type of sensor that the next part of the payload is for
        sbufWriteU16(dst, currentSensorVirtualConfig()->scale);
        sbufWriteU16(dst, currentSensorVirtualConfig()->offset);
#endif

        // if we had any other current sensors, this is where we would output any needed configuration
        break;
    }

    case MSP_BATTERY_CONFIG:
        sbufWriteU8(dst, (batteryConfig()->vbatmincellvoltage + 5) / 10);
        sbufWriteU8(dst, (batteryConfig()->vbatmaxcellvoltage + 5) / 10);
        sbufWriteU8(dst, (batteryConfig()->vbatwarningcellvoltage + 5) / 10);
        sbufWriteU16(dst, batteryConfig()->batteryCapacity);
        sbufWriteU8(dst, batteryConfig()->voltageMeterSource);
        sbufWriteU8(dst, batteryConfig()->currentMeterSource);
        sbufWriteU16(dst, batteryConfig()->vbatmincellvoltage);
        sbufWriteU16(dst, batteryConfig()->vbatmaxcellvoltage);
        sbufWriteU16(dst, batteryConfig()->vbatwarningcellvoltage);
        break;

    case MSP_TRANSPONDER_CONFIG: {
#ifdef USE_TRANSPONDER
        // Backward compatibility to BFC 3.1.1 is lost for this message type
        sbufWriteU8(dst, TRANSPONDER_PROVIDER_COUNT);
        for (unsigned int i = 0; i < TRANSPONDER_PROVIDER_COUNT; i++) {
            sbufWriteU8(dst, transponderRequirements[i].provider);
            sbufWriteU8(dst, transponderRequirements[i].dataLength);
        }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="925">

---

Here we're handling transponder config, writing out the provider and its data bytes. The loop adapts to the provider's expected data length.

```c
        uint8_t provider = transponderConfig()->provider;
        sbufWriteU8(dst, provider);

        if (provider) {
            uint8_t requirementIndex = provider - 1;
            uint8_t providerDataLength = transponderRequirements[requirementIndex].dataLength;

            for (unsigned int i = 0; i < providerDataLength; i++) {
                sbufWriteU8(dst, transponderConfig()->data[i]);
            }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="937">

---

If there are no transponder providers, we just write a zero byte to the buffer. This signals to the client that the list is empty.

```c
        sbufWriteU8(dst, 0); // no providers
#endif
        break;
    }

#if defined(USE_OSD)
    case MSP_OSD_CONFIG: {
#define OSD_FLAGS_OSD_FEATURE           (1 << 0)
//#define OSD_FLAGS_OSD_SLAVE             (1 << 1)
#define OSD_FLAGS_RESERVED_1            (1 << 2)
#define OSD_FLAGS_OSD_HARDWARE_FRSKYOSD (1 << 3)
#define OSD_FLAGS_OSD_HARDWARE_MAX_7456 (1 << 4)
#define OSD_FLAGS_OSD_DEVICE_DETECTED   (1 << 5)
#define OSD_FLAGS_OSD_MSP_DEVICE        (1 << 6)
#define OSD_FLAGS_OSD_HARDWARE_AIRBOT_THEIA_OSD (1 << 7)

        uint8_t osdFlags = 0;

        osdFlags |= OSD_FLAGS_OSD_FEATURE;

        osdDisplayPortDevice_e deviceType;
        displayPort_t *osdDisplayPort = osdGetDisplayPort(&deviceType);
        bool displayIsReady = osdDisplayPort && displayCheckReady(osdDisplayPort, true);
        switch (deviceType) {
        case OSD_DISPLAYPORT_DEVICE_MAX7456:
            osdFlags |= OSD_FLAGS_OSD_HARDWARE_MAX_7456;
            if (displayIsReady) {
                osdFlags |= OSD_FLAGS_OSD_DEVICE_DETECTED;
            }

            break;
        case OSD_DISPLAYPORT_DEVICE_FRSKYOSD:
            osdFlags |= OSD_FLAGS_OSD_HARDWARE_FRSKYOSD;
            if (displayIsReady) {
                osdFlags |= OSD_FLAGS_OSD_DEVICE_DETECTED;
            }

            break;
        case OSD_DISPLAYPORT_DEVICE_MSP:
            osdFlags |= OSD_FLAGS_OSD_MSP_DEVICE
#ifdef USE_MSP_DISPLAYPORT_FONT
                        | OSD_FLAGS_OSD_HARDWARE_AIRBOT_THEIA_OSD
#endif
            ;
            if (displayIsReady) {
                osdFlags |= OSD_FLAGS_OSD_DEVICE_DETECTED;
            }

            break;
        default:
            break;
        }

        sbufWriteU8(dst, osdFlags);

#ifdef USE_OSD_SD
        // send video system (AUTO/PAL/NTSC/HD)
        sbufWriteU8(dst, vcdProfile()->video_system);
#else
        sbufWriteU8(dst, VIDEO_SYSTEM_HD);
#endif // USE_OSD_SD

        // OSD specific, not applicable to OSD slaves.

        // Configuration
        sbufWriteU8(dst, osdConfig()->units);

        // Alarms
        sbufWriteU8(dst, osdConfig()->rssi_alarm);
        sbufWriteU16(dst, osdConfig()->cap_alarm);

        // Reuse old timer alarm (U16) as OSD_ITEM_COUNT
        sbufWriteU8(dst, 0);
        sbufWriteU8(dst, OSD_ITEM_COUNT);

        sbufWriteU16(dst, osdConfig()->alt_alarm);

        // Element position and visibility
        for (int i = 0; i < OSD_ITEM_COUNT; i++) {
            sbufWriteU16(dst, osdElementConfig()->item_pos[i]);
        }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="1019">

---

This part handles OSD config, writing out all the flags, config values, alarms, element positions, and statistics. It's a big chunk of structured data for the client.

```c
        // Post flight statistics
        sbufWriteU8(dst, OSD_STAT_COUNT);
        for (int i = 0; i < OSD_STAT_COUNT; i++ ) {
            sbufWriteU8(dst, osdStatGetState(i));
        }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="1025">

---

Here we write out the OSD timer count and each timer's value. This lets the client know how many timers there are and what their current settings are.

```c
        // Timers
        sbufWriteU8(dst, OSD_TIMER_COUNT);
        for (int i = 0; i < OSD_TIMER_COUNT; i++) {
            sbufWriteU16(dst, osdConfig()->timers[i]);
        }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="1031">

---

After all the switch cases, if the command wasn't handled, we return false. Otherwise, we return true. The output and supported commands depend on both the command value and the build config, so the actual response can vary a lot.

```c
        // Enabled warnings
        // Send low word first for backwards compatibility (API < 1.41)
        sbufWriteU16(dst, (uint16_t)(osdConfig()->enabledWarnings & 0xFFFF));
        // API >= 1.41
        // Send the warnings count and 32bit enabled warnings flags.
        // Add currently active OSD profile (0 indicates OSD profiles not available).
        // Add OSD stick overlay mode (0 indicates OSD stick overlay not available).
        sbufWriteU8(dst, OSD_WARNING_COUNT);
        sbufWriteU32(dst, osdConfig()->enabledWarnings);

#ifdef USE_OSD_PROFILES
        sbufWriteU8(dst, OSD_PROFILE_COUNT);            // available profiles
        sbufWriteU8(dst, osdConfig()->osdProfileIndex); // selected profile
#else
        // If the feature is not available there is only 1 profile and it's always selected
        sbufWriteU8(dst, 1);
        sbufWriteU8(dst, 1);
#endif // USE_OSD_PROFILES

#ifdef USE_OSD_STICK_OVERLAY
        sbufWriteU8(dst, osdConfig()->overlay_radio_mode);
#else
        sbufWriteU8(dst, 0);
#endif // USE_OSD_STICK_OVERLAY

        // API >= 1.43
        // Add the camera frame element width/height
        sbufWriteU8(dst, osdConfig()->camera_frame_width);
        sbufWriteU8(dst, osdConfig()->camera_frame_height);

        // API >= 1.46
        sbufWriteU16(dst, osdConfig()->link_quality_alarm);

        // API >= 1.47
        sbufWriteU16(dst, osdConfig()->rssi_dbm_alarm);

        break;
    }
#endif // USE_OSD

    case MSP_OSD_CANVAS: {
#ifdef USE_OSD
        sbufWriteU8(dst, osdConfig()->canvas_cols);
        sbufWriteU8(dst, osdConfig()->canvas_rows);
#endif
        break;
    }

    default:
        return false;
    }
    return true;
}
```

---

</SwmSnippet>

### Fallback MSP Command Handling

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
    node1["Receive flight controller command"] --> node2{"Can primary method handle command?"}
    click node1 openCode "src/main/msp/msp.c:4431:4432"
    node2 -->|"Yes"| node3["Processing result: acknowledged"]
    click node2 openCode "src/main/msp/msp.c:4431:4432"
    click node3 openCode "src/main/msp/msp.c:4431:4432"
    node2 -->|"No"| node4{"Can secondary method handle command?"}
    click node4 openCode "src/main/msp/msp.c:4432:4433"
    node4 -->|"Yes"| node3
    node4 -->|"No"| node5{"Can tertiary method handle command?"}
    click node5 openCode "src/main/msp/msp.c:4434:4434"
    node5 -->|"Yes"| node3
    node5 -->|"No"| node6["Processing result: not acknowledged"]
    click node6 openCode "src/main/msp/msp.c:4434:4434"

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%     node1["Receive flight controller command"] --> node2{"Can primary method handle command?"}
%%     click node1 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4431:4432"
%%     node2 -->|"Yes"| node3["Processing result: acknowledged"]
%%     click node2 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4431:4432"
%%     click node3 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4431:4432"
%%     node2 -->|"No"| node4{"Can secondary method handle command?"}
%%     click node4 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4432:4433"
%%     node4 -->|"Yes"| node3
%%     node4 -->|"No"| node5{"Can tertiary method handle command?"}
%%     click node5 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4434:4434"
%%     node5 -->|"Yes"| node3
%%     node5 -->|"No"| node6["Processing result: not acknowledged"]
%%     click node6 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4434:4434"
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/src/main/msp/msp.c" line="4431">

---

We just got back from <SwmToken path="src/main/msp/msp.c" pos="627:4:4" line-data="static bool mspCommonProcessOutCommand(int16_t cmdMSP, sbuf_t *dst, mspPostProcessFnPtr *mspPostProcessFn)">`mspCommonProcessOutCommand`</SwmToken>. If it didn't handle the command, we try <SwmToken path="src/main/msp/msp.c" pos="4432:8:8" line-data="    } else if (mspProcessOutCommand(srcDesc, cmdMSP, dst)) {">`mspProcessOutCommand`</SwmToken> next, and if that doesn't work, we fall back to <SwmToken path="src/main/msp/msp.c" pos="4434:13:13" line-data="    } else if ((ret = mspFcProcessOutCommandWithArg(srcDesc, cmdMSP, src, dst, mspPostProcessFn)) != MSP_RESULT_CMD_UNKNOWN) {">`mspFcProcessOutCommandWithArg`</SwmToken>. Each handler covers a different set of commands.

```c
        ret = MSP_RESULT_ACK;
    } else if (mspProcessOutCommand(srcDesc, cmdMSP, dst)) {
        ret = MSP_RESULT_ACK;
    } else if ((ret = mspFcProcessOutCommandWithArg(srcDesc, cmdMSP, src, dst, mspPostProcessFn)) != MSP_RESULT_CMD_UNKNOWN) {
```

---

</SwmSnippet>

### Advanced MSP Command Handling

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
    node1["Receive command with argument"] --> node2{"Which command was received?"}
    click node1 openCode "src/main/msp/msp.c:2326:2329"
    node2 -->|"BOXNAMES/BOXIDS"| node3["Return box info (with page)"]
    click node2 openCode "src/main/msp/msp.c:2329:2335"
    node2 -->|"REBOOT"| node4{"Is reboot mode valid?"}
    click node4 openCode "src/main/msp/msp.c:2343:2352"
    node4 -->|"Yes"| node5["Perform reboot and respond"]
    click node5 openCode "src/main/msp/msp.c:2353:2378"
    node4 -->|"No"| node6["Return error"]
    click node6 openCode "src/main/msp/msp.c:2351:2352"
    node2 -->|"MULTIPLE_MSP"| node7["Process multiple sub-commands"]
    click node7 openCode "src/main/msp/msp.c:2383:2410"
    subgraph loop1["For each sub-command in MULTIPLE_MSP"]
        node7 --> node8["Read sub-command and process"]
        click node8 openCode "src/main/msp/msp.c:2391:2395"
        node8 --> node7
    end
    node2 -->|"VTXTABLE_BAND"| node9{"Is band in range?"}
    click node9 openCode "src/main/msp/msp.c:2418:2424"
    node9 -->|"Yes"| node10["Return band info"]
    click node10 openCode "src/main/msp/msp.c:2420:2430"
    subgraph loop2["For each channel in band"]
        node10 --> node11["Write channel frequency"]
        click node11 openCode "src/main/msp/msp.c:2428:2430"
        node11 --> node10
    end
    node9 -->|"No"| node12["Return error"]
    click node12 openCode "src/main/msp/msp.c:2419:2420"
    node2 -->|"VTXTABLE_POWERLEVEL"| node13{"Is power level in range?"}
    click node13 openCode "src/main/msp/msp.c:2443:2449"
    node13 -->|"Yes"| node14["Return power level info"]
    click node14 openCode "src/main/msp/msp.c:2444:2449"
    subgraph loop3["For each label in power level"]
        node14 --> node15["Write power label"]
        click node15 openCode "src/main/msp/msp.c:2447:2449"
        node15 --> node14
    end
    node13 -->|"No"| node16["Return error"]
    click node16 openCode "src/main/msp/msp.c:2443:2444"
    node2 -->|"SIMPLIFIED_TUNING"| node17["Return simplified tuning data"]
    click node17 openCode "src/main/msp/msp.c:2464:2467"
    node2 -->|"CALCULATE_SIMPLIFIED_PID"| node18["Calculate and return PID tuning"]
    click node18 openCode "src/main/msp/msp.c:2472:2475"
    node2 -->|"CALCULATE_SIMPLIFIED_DTERM"| node19["Calculate and return D-term tuning"]
    click node19 openCode "src/main/msp/msp.c:2482:2484"
    node2 -->|"CALCULATE_SIMPLIFIED_GYRO"| node20["Calculate and return gyro tuning"]
    click node20 openCode "src/main/msp/msp.c:2491:2493"
    node2 -->|"VALIDATE_SIMPLIFIED_TUNING"| node21["Validate tuning for all axes"]
    click node21 openCode "src/main/msp/msp.c:2503:2510"
    subgraph loop4["For each axis in tuning validation"]
        node21 --> node22{"Are PID values equal?"}
        click node22 openCode "src/main/msp/msp.c:2504:2510"
        node22 -->|"Yes"| node21
        node22 -->|"No"| node23["Set result to false"]
        click node23 openCode "src/main/msp/msp.c:2501:2502"
    end
    node2 -->|"RESET_CONF"| node24{"Is controller armed?"}
    click node24 openCode "src/main/msp/msp.c:2552:2559"
    node24 -->|"No"| node25["Reset configuration and respond"]
    click node25 openCode "src/main/msp/msp.c:2553:2558"
    node24 -->|"Yes"| node26["Return failure"]
    click node26 openCode "src/main/msp/msp.c:2552:2553"
    node2 -->|"GET_TEXT"| node27{"Is requested text available?"}
    click node27 openCode "src/main/msp/msp.c:2574:2603"
    node27 -->|"Yes"| node28["Return text"]
    click node28 openCode "src/main/msp/msp.c:2606:2607"
    node27 -->|"No"| node29["Return error"]
    click node29 openCode "src/main/msp/msp.c:2603:2604"
    node2 -->|"GET_LED_STRIP_CONFIG"| node30["Return LED strip config"]
    click node30 openCode "src/main/msp/msp.c:2612:2614"
    node2 -->|"Unknown"| node31["Return unknown command error"]
    click node31 openCode "src/main/msp/msp.c:2618:2619"
    node3 --> node32["Return result"]
    node5 --> node32
    node6 --> node32
    node7 --> node32
    node10 --> node32
    node12 --> node32
    node14 --> node32
    node16 --> node32
    node17 --> node32
    node18 --> node32
    node19 --> node32
    node20 --> node32
    node23 --> node32
    node25 --> node32
    node26 --> node32
    node28 --> node32
    node29 --> node32
    node30 --> node32
    node31 --> node32
    click node32 openCode "src/main/msp/msp.c:2621:2622"
classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%     node1["Receive command with argument"] --> node2{"Which command was received?"}
%%     click node1 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2326:2329"
%%     node2 -->|"BOXNAMES/BOXIDS"| node3["Return box info (with page)"]
%%     click node2 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2329:2335"
%%     node2 -->|"REBOOT"| node4{"Is reboot mode valid?"}
%%     click node4 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2343:2352"
%%     node4 -->|"Yes"| node5["Perform reboot and respond"]
%%     click node5 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2353:2378"
%%     node4 -->|"No"| node6["Return error"]
%%     click node6 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2351:2352"
%%     node2 -->|"MULTIPLE_MSP"| node7["Process multiple sub-commands"]
%%     click node7 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2383:2410"
%%     subgraph loop1["For each sub-command in MULTIPLE_MSP"]
%%         node7 --> node8["Read sub-command and process"]
%%         click node8 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2391:2395"
%%         node8 --> node7
%%     end
%%     node2 -->|"VTXTABLE_BAND"| node9{"Is band in range?"}
%%     click node9 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2418:2424"
%%     node9 -->|"Yes"| node10["Return band info"]
%%     click node10 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2420:2430"
%%     subgraph loop2["For each channel in band"]
%%         node10 --> node11["Write channel frequency"]
%%         click node11 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2428:2430"
%%         node11 --> node10
%%     end
%%     node9 -->|"No"| node12["Return error"]
%%     click node12 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2419:2420"
%%     node2 -->|"VTXTABLE_POWERLEVEL"| node13{"Is power level in range?"}
%%     click node13 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2443:2449"
%%     node13 -->|"Yes"| node14["Return power level info"]
%%     click node14 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2444:2449"
%%     subgraph loop3["For each label in power level"]
%%         node14 --> node15["Write power label"]
%%         click node15 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2447:2449"
%%         node15 --> node14
%%     end
%%     node13 -->|"No"| node16["Return error"]
%%     click node16 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2443:2444"
%%     node2 -->|"SIMPLIFIED_TUNING"| node17["Return simplified tuning data"]
%%     click node17 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2464:2467"
%%     node2 -->|"CALCULATE_SIMPLIFIED_PID"| node18["Calculate and return PID tuning"]
%%     click node18 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2472:2475"
%%     node2 -->|"CALCULATE_SIMPLIFIED_DTERM"| node19["Calculate and return D-term tuning"]
%%     click node19 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2482:2484"
%%     node2 -->|"CALCULATE_SIMPLIFIED_GYRO"| node20["Calculate and return gyro tuning"]
%%     click node20 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2491:2493"
%%     node2 -->|"VALIDATE_SIMPLIFIED_TUNING"| node21["Validate tuning for all axes"]
%%     click node21 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2503:2510"
%%     subgraph loop4["For each axis in tuning validation"]
%%         node21 --> node22{"Are PID values equal?"}
%%         click node22 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2504:2510"
%%         node22 -->|"Yes"| node21
%%         node22 -->|"No"| node23["Set result to false"]
%%         click node23 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2501:2502"
%%     end
%%     node2 -->|"RESET_CONF"| node24{"Is controller armed?"}
%%     click node24 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2552:2559"
%%     node24 -->|"No"| node25["Reset configuration and respond"]
%%     click node25 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2553:2558"
%%     node24 -->|"Yes"| node26["Return failure"]
%%     click node26 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2552:2553"
%%     node2 -->|"GET_TEXT"| node27{"Is requested text available?"}
%%     click node27 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2574:2603"
%%     node27 -->|"Yes"| node28["Return text"]
%%     click node28 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2606:2607"
%%     node27 -->|"No"| node29["Return error"]
%%     click node29 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2603:2604"
%%     node2 -->|"GET_LED_STRIP_CONFIG"| node30["Return LED strip config"]
%%     click node30 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2612:2614"
%%     node2 -->|"Unknown"| node31["Return unknown command error"]
%%     click node31 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2618:2619"
%%     node3 --> node32["Return result"]
%%     node5 --> node32
%%     node6 --> node32
%%     node7 --> node32
%%     node10 --> node32
%%     node12 --> node32
%%     node14 --> node32
%%     node16 --> node32
%%     node17 --> node32
%%     node18 --> node32
%%     node19 --> node32
%%     node20 --> node32
%%     node23 --> node32
%%     node25 --> node32
%%     node26 --> node32
%%     node28 --> node32
%%     node29 --> node32
%%     node30 --> node32
%%     node31 --> node32
%%     click node32 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2621:2622"
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/src/main/msp/msp.c" line="2326">

---

In <SwmToken path="src/main/msp/msp.c" pos="2326:4:4" line-data="static mspResult_e mspFcProcessOutCommandWithArg(mspDescriptor_t srcDesc, int16_t cmdMSP, sbuf_t *src, sbuf_t *dst, mspPostProcessFnPtr *mspPostProcessFn)">`mspFcProcessOutCommandWithArg`</SwmToken>, we handle a bunch of advanced and Betaflight-specific MSP commands, each with custom logic. For <SwmToken path="src/main/msp/msp.c" pos="2381:3:3" line-data="    case MSP_MULTIPLE_MSP:">`MSP_MULTIPLE_MSP`</SwmToken>, we do a two-pass scan: first to count how many commands fit, then to actually process and write each one with its reply size.

```c
static mspResult_e mspFcProcessOutCommandWithArg(mspDescriptor_t srcDesc, int16_t cmdMSP, sbuf_t *src, sbuf_t *dst, mspPostProcessFnPtr *mspPostProcessFn)
{

    switch (cmdMSP) {
    case MSP_BOXNAMES:
        {
            const int page = sbufBytesRemaining(src) ? sbufReadU8(src) : 0;
            serializeBoxReply(dst, page, &serializeBoxNameFn);
        }
        break;
    case MSP_BOXIDS:
        {
            const int page = sbufBytesRemaining(src) ? sbufReadU8(src) : 0;
            serializeBoxReply(dst, page, &serializeBoxPermanentIdFn);
        }
        break;
    case MSP_REBOOT:
        if (sbufBytesRemaining(src)) {
            rebootMode = sbufReadU8(src);

            if (rebootMode >= MSP_REBOOT_COUNT
#if !defined(USE_USB_MSC)
                || rebootMode == MSP_REBOOT_MSC || rebootMode == MSP_REBOOT_MSC_UTC
#endif
                ) {
                return MSP_RESULT_ERROR;
            }
        } else {
            rebootMode = MSP_REBOOT_FIRMWARE;
        }

        sbufWriteU8(dst, rebootMode);

#if defined(USE_USB_MSC)
        if (rebootMode == MSP_REBOOT_MSC) {
            if (mscCheckFilesystemReady()) {
                sbufWriteU8(dst, 1);
            } else {
                sbufWriteU8(dst, 0);

                return MSP_RESULT_ACK;
            }
        }
#endif

#if defined(USE_MSP_OVER_TELEMETRY)
        if (featureIsEnabled(FEATURE_RX_SPI) && srcDesc == getMspTelemetryDescriptor()) {
            dispatchAdd(&mspRebootEntry, MSP_DISPATCH_DELAY_US);
        } else
#endif
        if (mspPostProcessFn) {
            *mspPostProcessFn = mspRebootFn;
        }

        break;
    case MSP_MULTIPLE_MSP:
        {
            uint8_t maxMSPs = 0;
            if (sbufBytesRemaining(src) == 0) {
                return MSP_RESULT_ERROR;
            }
            int bytesRemaining = sbufBytesRemaining(dst);
            mspPacket_t packetIn, packetOut;
            sbufInit(&packetIn.buf, src->end, src->end); // there is no paramater for MSP_MULTIPLE_MSP
            uint8_t* initialInputPtr = src->ptr;
            while (sbufBytesRemaining(src) && bytesRemaining > 0) {
                uint8_t newMSP = sbufReadU8(src);
                sbufInit(&packetOut.buf, dst->ptr + 1, dst->end); // reserve 1 byte for length
                packetIn.cmd = newMSP;
                mspFcProcessCommand(srcDesc, &packetIn, &packetOut, NULL);
                uint8_t mspSize = sbufPtr(&packetOut.buf) - dst->ptr; // length included
                bytesRemaining -= mspSize;
                if (bytesRemaining >= 0) {
                    maxMSPs++;
                }
            }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="2402">

---

After counting how many MSP commands fit, we rewind the input pointer and process each command for real, writing the reply and its size. This makes sure we don't overflow the output buffer.

```c
            src->ptr = initialInputPtr;
            sbufInit(&packetOut.buf, dst->ptr, dst->end);
            for (int i = 0; i < maxMSPs; i++) {
                uint8_t* sizePtr = sbufPtr(&packetOut.buf);
                sbufWriteU8(&packetOut.buf, 0); // placeholder for reply size
                packetIn.cmd = sbufReadU8(src);
                mspFcProcessCommand(srcDesc, &packetIn, &packetOut, NULL);
                *sizePtr = sbufPtr(&packetOut.buf) - (sizePtr + 1);
            }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="2411">

---

After the two-pass <SwmToken path="src/main/msp/msp.c" pos="2381:3:3" line-data="    case MSP_MULTIPLE_MSP:">`MSP_MULTIPLE_MSP`</SwmToken> handling in <SwmToken path="src/main/msp/msp.c" pos="2326:4:4" line-data="static mspResult_e mspFcProcessOutCommandWithArg(mspDescriptor_t srcDesc, int16_t cmdMSP, sbuf_t *src, sbuf_t *dst, mspPostProcessFnPtr *mspPostProcessFn)">`mspFcProcessOutCommandWithArg`</SwmToken>, we update the output pointer to reflect the actual end of the written data. This keeps the buffer state consistent for the caller.

```c
            dst->ptr = packetOut.buf.ptr;
        }
        break;

#ifdef USE_VTX_TABLE
    case MSP_VTXTABLE_BAND:
        {
            const uint8_t band = sbufBytesRemaining(src) ? sbufReadU8(src) : 0;
            if (band > 0 && band <= VTX_TABLE_MAX_BANDS) {
                sbufWriteU8(dst, band);  // band number (same as request)
                sbufWriteU8(dst, VTX_TABLE_BAND_NAME_LENGTH); // band name length
                for (int i = 0; i < VTX_TABLE_BAND_NAME_LENGTH; i++) { // band name bytes
                    sbufWriteU8(dst, vtxTableConfig()->bandNames[band - 1][i]);
                }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="2425">

---

This part handles VTX table band commands, writing out all the band info and channel frequencies for the requested band. Uses a bunch of protocol constants for structure.

```c
                sbufWriteU8(dst, vtxTableConfig()->bandLetters[band - 1]); // band letter
                sbufWriteU8(dst, vtxTableConfig()->isFactoryBand[band - 1]); // CUSTOM = 0; FACTORY = 1
                sbufWriteU8(dst, vtxTableConfig()->channels); // number of channel frequencies to follow
                for (int i = 0; i < vtxTableConfig()->channels; i++) { // the frequency for each channel
                    sbufWriteU16(dst, vtxTableConfig()->frequency[band - 1][i]);
                }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="2435">

---

Here we're handling VTX power level commands, using protocol constants to validate the input and structure the output. Only valid power levels are processed.

```c
            setMspVtxDeviceStatusReady(srcDesc);
#endif
        }
        break;

    case MSP_VTXTABLE_POWERLEVEL:
        {
            const uint8_t powerLevel = sbufBytesRemaining(src) ? sbufReadU8(src) : 0;
            if (powerLevel > 0 && powerLevel <= VTX_TABLE_MAX_POWER_LEVELS) {
                sbufWriteU8(dst, powerLevel);  // powerLevel number (same as request)
                sbufWriteU16(dst, vtxTableConfig()->powerValues[powerLevel - 1]);
                sbufWriteU8(dst, VTX_TABLE_POWER_LABEL_LENGTH); // powerLevel label length
                for (int i = 0; i < VTX_TABLE_POWER_LABEL_LENGTH; i++) { // powerlevel label bytes
                    sbufWriteU8(dst, vtxTableConfig()->powerLabels[powerLevel - 1][i]);
                }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="2454">

---

This chunk handles more tuning and config commands, each writing out their own data using helper functions. It's just more cases in the big switch.

```c
            setMspVtxDeviceStatusReady(srcDesc);
#endif
        }
        break;
#endif // USE_VTX_TABLE

#ifdef USE_SIMPLIFIED_TUNING
    // Added in MSP API 1.44
    case MSP_SIMPLIFIED_TUNING:
        {
            writeSimplifiedPids(currentPidProfile, dst);
            writeSimplifiedDtermFilters(currentPidProfile, dst);
            writeSimplifiedGyroFilters(gyroConfig(), dst);
        }
        break;

    case MSP_CALCULATE_SIMPLIFIED_PID:
        {
            pidProfile_t tempPidProfile = *currentPidProfile;
            readSimplifiedPids(&tempPidProfile, src);
            applySimplifiedTuningPids(&tempPidProfile);
            writePidfs(&tempPidProfile, dst);
        }
        break;

    case MSP_CALCULATE_SIMPLIFIED_DTERM:
        {
            pidProfile_t tempPidProfile = *currentPidProfile;
            readSimplifiedDtermFilters(&tempPidProfile, src);
            applySimplifiedTuningDtermFilters(&tempPidProfile);
            writeSimplifiedDtermFilters(&tempPidProfile, dst);
        }
        break;

    case MSP_CALCULATE_SIMPLIFIED_GYRO:
        {
            gyroConfig_t tempGyroConfig = *gyroConfig();
            readSimplifiedGyroFilters(&tempGyroConfig, src);
            applySimplifiedTuningGyroFilters(&tempGyroConfig);
            writeSimplifiedGyroFilters(&tempGyroConfig, dst);
        }
        break;

    case MSP_VALIDATE_SIMPLIFIED_TUNING:
        {
            pidProfile_t tempPidProfile = *currentPidProfile;
            applySimplifiedTuningPids(&tempPidProfile);
            bool result = true;

            for (int i = 0; i < XYZ_AXIS_COUNT; i++) {
                result = result &&
                    tempPidProfile.pid[i].P == currentPidProfile->pid[i].P &&
                    tempPidProfile.pid[i].I == currentPidProfile->pid[i].I &&
                    tempPidProfile.pid[i].D == currentPidProfile->pid[i].D &&
                    tempPidProfile.d_max[i] == currentPidProfile->d_max[i] &&
                    tempPidProfile.pid[i].F == currentPidProfile->pid[i].F;
            }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="2512">

---

After all the cases, if the command isn't recognized or the buffer is wrong, we return <SwmToken path="src/main/msp/msp.c" pos="2619:3:3" line-data="        return MSP_RESULT_CMD_UNKNOWN;">`MSP_RESULT_CMD_UNKNOWN`</SwmToken> or <SwmToken path="src/main/msp/msp.c" pos="2600:3:3" line-data="                    return MSP_RESULT_ERROR;">`MSP_RESULT_ERROR`</SwmToken>. Each command is handled with Betaflight-specific logic, and the function expects the buffers to be set up correctly.

```c
            sbufWriteU8(dst, result);

            gyroConfig_t tempGyroConfig = *gyroConfig();
            applySimplifiedTuningGyroFilters(&tempGyroConfig);
            result =
                tempGyroConfig.gyro_lpf1_static_hz == gyroConfig()->gyro_lpf1_static_hz &&
                tempGyroConfig.gyro_lpf2_static_hz == gyroConfig()->gyro_lpf2_static_hz;

#if defined(USE_DYN_LPF)
            result = result &&
                tempGyroConfig.gyro_lpf1_dyn_min_hz == gyroConfig()->gyro_lpf1_dyn_min_hz &&
                tempGyroConfig.gyro_lpf1_dyn_max_hz == gyroConfig()->gyro_lpf1_dyn_max_hz;
#endif

            sbufWriteU8(dst, result);

            applySimplifiedTuningDtermFilters(&tempPidProfile);
            result =
                tempPidProfile.dterm_lpf1_static_hz == currentPidProfile->dterm_lpf1_static_hz &&
                tempPidProfile.dterm_lpf2_static_hz == currentPidProfile->dterm_lpf2_static_hz;

#if defined(USE_DYN_LPF)
            result = result &&
                tempPidProfile.dterm_lpf1_dyn_min_hz == currentPidProfile->dterm_lpf1_dyn_min_hz &&
                tempPidProfile.dterm_lpf1_dyn_max_hz == currentPidProfile->dterm_lpf1_dyn_max_hz;
#endif

            sbufWriteU8(dst, result);
        }
        break;
#endif

    case MSP_RESET_CONF:
        {
            if (sbufBytesRemaining(src) >= 1) {
                // Added in MSP API 1.42
                sbufReadU8(src);
            }

            bool success = false;
            if (!ARMING_FLAG(ARMED)) {
                success = resetEEPROM();

                if (success && mspPostProcessFn) {
                    rebootMode = MSP_REBOOT_FIRMWARE;
                    *mspPostProcessFn = mspRebootFn;
                }
            }

            // Added in API version 1.42
            sbufWriteU8(dst, success);
        }

        break;

    case MSP2_GET_TEXT:
        {
            // type byte, then length byte followed by the actual characters
            const uint8_t textType = sbufBytesRemaining(src) ? sbufReadU8(src) : 0;

            const char *textVar;

            switch (textType) {
                case MSP2TEXT_PILOT_NAME:
                    textVar = pilotConfigMutable()->pilotName;
                    break;

                case MSP2TEXT_CRAFT_NAME:
                    textVar = pilotConfigMutable()->craftName;
                    break;

                case MSP2TEXT_PID_PROFILE_NAME:
                    textVar = currentPidProfile->profileName;
                    break;

                case MSP2TEXT_RATE_PROFILE_NAME:
                    textVar = currentControlRateProfile->profileName;
                    break;

                case MSP2TEXT_BUILDKEY:
                    textVar = buildKey;
                    break;

                case MSP2TEXT_RELEASENAME:
                    textVar = releaseName;
                    break;

                default:
                    return MSP_RESULT_ERROR;
            }

            if (!textVar) return MSP_RESULT_ERROR;

            //  type byte, then length byte followed by the actual characters
            sbufWriteU8(dst, textType);
            sbufWritePString(dst, textVar);
        }
        break;
#ifdef USE_LED_STRIP
    case MSP2_GET_LED_STRIP_CONFIG_VALUES:
        sbufWriteU8(dst, ledStripConfig()->ledstrip_brightness);
        sbufWriteU16(dst, ledStripConfig()->ledstrip_rainbow_delta);
        sbufWriteU16(dst, ledStripConfig()->ledstrip_rainbow_freq);
        break;
#endif

    default:
        return MSP_RESULT_CMD_UNKNOWN;
    }
    return MSP_RESULT_ACK;
}
```

---

</SwmSnippet>

### Final MSP Command Fallbacks

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
    node1["Receive flight controller command"]
    click node1 openCode "src/main/msp/msp.c:4435:4436"
    node1 --> node2{"Command type?"}
    click node2 openCode "src/main/msp/msp.c:4436:4444"
    node2 -->|"Passthrough"| node3["Process passthrough command and acknowledge"]
    click node3 openCode "src/main/msp/msp.c:4437:4438"
    node2 -->|"Dataflash read"| node4["Process dataflash read command and acknowledge"]
    click node4 openCode "src/main/msp/msp.c:4441:4442"
    node2 -->|"Other"| node5["Process standard command and return result"]
    click node5 openCode "src/main/msp/msp.c:4445:4446"

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%     node1["Receive flight controller command"]
%%     click node1 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4435:4436"
%%     node1 --> node2{"Command type?"}
%%     click node2 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4436:4444"
%%     node2 -->|"Passthrough"| node3["Process passthrough command and acknowledge"]
%%     click node3 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4437:4438"
%%     node2 -->|"Dataflash read"| node4["Process dataflash read command and acknowledge"]
%%     click node4 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4441:4442"
%%     node2 -->|"Other"| node5["Process standard command and return result"]
%%     click node5 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4445:4446"
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/src/main/msp/msp.c" line="4435">

---

After <SwmToken path="src/main/msp/msp.c" pos="2326:4:4" line-data="static mspResult_e mspFcProcessOutCommandWithArg(mspDescriptor_t srcDesc, int16_t cmdMSP, sbuf_t *src, sbuf_t *dst, mspPostProcessFnPtr *mspPostProcessFn)">`mspFcProcessOutCommandWithArg`</SwmToken>, if the command still isn't handled, we check for passthrough and flash read commands, and if those don't match, we finally call <SwmToken path="src/main/msp/msp.c" pos="4445:5:5" line-data="        ret = mspCommonProcessInCommand(srcDesc, cmdMSP, src, mspPostProcessFn);">`mspCommonProcessInCommand`</SwmToken> to handle <SwmPath>[src/main/config/](src/main/config/)</SwmPath> commands.

```c
        /* ret */;
    } else if (cmdMSP == MSP_SET_PASSTHROUGH) {
        mspFcSetPassthroughCommand(dst, src, mspPostProcessFn);
        ret = MSP_RESULT_ACK;
#ifdef USE_FLASHFS
    } else if (cmdMSP == MSP_DATAFLASH_READ) {
        mspFcDataFlashReadCommand(dst, src);
        ret = MSP_RESULT_ACK;
#endif
    } else {
        ret = mspCommonProcessInCommand(srcDesc, cmdMSP, src, mspPostProcessFn);
    }
```

---

</SwmSnippet>

### Processing MSP Input/Config Commands

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
    node1["Receive configuration command"] --> node2{"Which command type?"}
    click node1 openCode "src/main/msp/msp.c:4124:4130"
    node2 -->|"Transponder"| node3{"Is provider valid?"}
    click node2 openCode "src/main/msp/msp.c:4130:4132"
    node3 -->|"Yes"| node4{"Does data size match requirements?"}
    click node3 openCode "src/main/msp/msp.c:4138:4143"
    node3 -->|"No"| node20["Return error"]
    click node20 openCode "src/main/msp/msp.c:4139:4140"
    node4 -->|"Yes"| node5["Set transponder provider"]
    click node4 openCode "src/main/msp/msp.c:4145:4146"
    node4 -->|"No"| node20
    node5 --> node6{"Is provider 'None'?"}
    click node5 openCode "src/main/msp/msp.c:4147:4149"
    node6 -->|"No"| node7["Stop repeating if provider changed"]
    click node7 openCode "src/main/msp/msp.c:4155:4157"
    node6 -->|"Yes"| node11["Return ACK"]
    node7 --> node8["Clear transponder data"]
    click node8 openCode "src/main/msp/msp.c:4159:4160"
    node8 --> node9
    subgraph loop1["For each byte in transponder data"]
        node9["Write byte to transponder config"]
        click node9 openCode "src/main/msp/msp.c:4161:4163"
    end
    node9 --> node10["Update transponder data"]
    click node10 openCode "src/main/msp/msp.c:4164:4165"
    node10 --> node11["Return ACK"]
    click node11 openCode "src/main/msp/msp.c:4415:4416"
    node2 -->|"Voltage Meter"| node12["Find matching voltage sensor"]
    click node12 openCode "src/main/msp/msp.c:4170:4179"
    subgraph loop2["For each voltage sensor"]
        node12 --> node13{"Is sensor ID matched?"}
        click node13 openCode "src/main/msp/msp.c:4177:4178"
    end
    node13 -->|"Yes"| node14["Update voltage sensor config"]
    click node14 openCode "src/main/msp/msp.c:4183:4186"
    node13 -->|"No"| node15["Skip sensor config"]
    click node15 openCode "src/main/msp/msp.c:4187:4191"
    node14 --> node11
    node15 --> node11
    node2 -->|"Current Meter"| node16{"Which meter type?"}
    click node16 openCode "src/main/msp/msp.c:4198:4213"
    node16 -->|"Battery"| node17["Update battery current sensor config"]
    click node17 openCode "src/main/msp/msp.c:4200:4202"
    node16 -->|"Virtual"| node18["Update virtual current sensor config"]
    click node18 openCode "src/main/msp/msp.c:4205:4207"
    node16 -->|"Other"| node19["Skip current sensor config"]
    click node19 openCode "src/main/msp/msp.c:4210:4212"
    node17 --> node11
    node18 --> node11
    node19 --> node11
    node2 -->|"Battery"| node21["Update battery thresholds and sources"]
    click node21 openCode "src/main/msp/msp.c:4217:4230"
    node21 --> node11
    node2 -->|"OSD"| node22{"Is address for general settings?"}
    click node22 openCode "src/main/msp/msp.c:4232:4351"
    node22 -->|"Yes"| node23["Update OSD settings, alarms, warnings"]
    click node23 openCode "src/main/msp/msp.c:4237:4324"
    node22 -->|"No"| node24{"Is address for timers?"}
    click node24 openCode "src/main/msp/msp.c:4325:4332"
    node24 -->|"Yes"| node25["Update OSD timer"]
    click node25 openCode "src/main/msp/msp.c:4327:4331"
    node24 -->|"No"| node26["Update OSD statistics or element positions"]
    click node26 openCode "src/main/msp/msp.c:4333:4345"
    node23 --> node11
    node25 --> node11
    node26 --> node11
    node2 -->|"OSD Char Write"| node27{"Is display port available?"}
    click node27 openCode "src/main/msp/msp.c:4379:4382"
    node27 -->|"Yes"| node28{"Was font character written?"}
    click node28 openCode "src/main/msp/msp.c:4384:4386"
    node27 -->|"No"| node20
    node28 -->|"Yes"| node11
    node28 -->|"No"| node20
    subgraph loop3["For each byte in OSD character data"]
        node29["Write byte to OSD character"]
        click node29 openCode "src/main/msp/msp.c:4376:4378"
    end
    node2 -->|"OSD Canvas"| node30["Update OSD canvas size and video system"]
    click node30 openCode "src/main/msp/msp.c:4391:4408"
    node30 --> node11
    node2 -->|"Other"| node31["Process with default handler"]
    click node31 openCode "src/main/msp/msp.c:4412:4413"
    node31 --> node11

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%     node1["Receive configuration command"] --> node2{"Which command type?"}
%%     click node1 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4124:4130"
%%     node2 -->|"Transponder"| node3{"Is provider valid?"}
%%     click node2 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4130:4132"
%%     node3 -->|"Yes"| node4{"Does data size match requirements?"}
%%     click node3 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4138:4143"
%%     node3 -->|"No"| node20["Return error"]
%%     click node20 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4139:4140"
%%     node4 -->|"Yes"| node5["Set transponder provider"]
%%     click node4 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4145:4146"
%%     node4 -->|"No"| node20
%%     node5 --> node6{"Is provider 'None'?"}
%%     click node5 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4147:4149"
%%     node6 -->|"No"| node7["Stop repeating if provider changed"]
%%     click node7 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4155:4157"
%%     node6 -->|"Yes"| node11["Return ACK"]
%%     node7 --> node8["Clear transponder data"]
%%     click node8 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4159:4160"
%%     node8 --> node9
%%     subgraph loop1["For each byte in transponder data"]
%%         node9["Write byte to transponder config"]
%%         click node9 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4161:4163"
%%     end
%%     node9 --> node10["Update transponder data"]
%%     click node10 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4164:4165"
%%     node10 --> node11["Return ACK"]
%%     click node11 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4415:4416"
%%     node2 -->|"Voltage Meter"| node12["Find matching voltage sensor"]
%%     click node12 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4170:4179"
%%     subgraph loop2["For each voltage sensor"]
%%         node12 --> node13{"Is sensor ID matched?"}
%%         click node13 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4177:4178"
%%     end
%%     node13 -->|"Yes"| node14["Update voltage sensor config"]
%%     click node14 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4183:4186"
%%     node13 -->|"No"| node15["Skip sensor config"]
%%     click node15 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4187:4191"
%%     node14 --> node11
%%     node15 --> node11
%%     node2 -->|"Current Meter"| node16{"Which meter type?"}
%%     click node16 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4198:4213"
%%     node16 -->|"Battery"| node17["Update battery current sensor config"]
%%     click node17 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4200:4202"
%%     node16 -->|"Virtual"| node18["Update virtual current sensor config"]
%%     click node18 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4205:4207"
%%     node16 -->|"Other"| node19["Skip current sensor config"]
%%     click node19 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4210:4212"
%%     node17 --> node11
%%     node18 --> node11
%%     node19 --> node11
%%     node2 -->|"Battery"| node21["Update battery thresholds and sources"]
%%     click node21 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4217:4230"
%%     node21 --> node11
%%     node2 -->|"OSD"| node22{"Is address for general settings?"}
%%     click node22 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4232:4351"
%%     node22 -->|"Yes"| node23["Update OSD settings, alarms, warnings"]
%%     click node23 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4237:4324"
%%     node22 -->|"No"| node24{"Is address for timers?"}
%%     click node24 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4325:4332"
%%     node24 -->|"Yes"| node25["Update OSD timer"]
%%     click node25 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4327:4331"
%%     node24 -->|"No"| node26["Update OSD statistics or element positions"]
%%     click node26 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4333:4345"
%%     node23 --> node11
%%     node25 --> node11
%%     node26 --> node11
%%     node2 -->|"OSD Char Write"| node27{"Is display port available?"}
%%     click node27 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4379:4382"
%%     node27 -->|"Yes"| node28{"Was font character written?"}
%%     click node28 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4384:4386"
%%     node27 -->|"No"| node20
%%     node28 -->|"Yes"| node11
%%     node28 -->|"No"| node20
%%     subgraph loop3["For each byte in OSD character data"]
%%         node29["Write byte to OSD character"]
%%         click node29 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4376:4378"
%%     end
%%     node2 -->|"OSD Canvas"| node30["Update OSD canvas size and video system"]
%%     click node30 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4391:4408"
%%     node30 --> node11
%%     node2 -->|"Other"| node31["Process with default handler"]
%%     click node31 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4412:4413"
%%     node31 --> node11
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/src/main/msp/msp.c" line="4124">

---

In <SwmToken path="src/main/msp/msp.c" pos="4124:4:4" line-data="static mspResult_e mspCommonProcessInCommand(mspDescriptor_t srcDesc, int16_t cmdMSP, sbuf_t *src, mspPostProcessFnPtr *mspPostProcessFn)">`mspCommonProcessInCommand`</SwmToken>, we switch on the command and read config values from the buffer, updating internal state or config as needed. Each case expects the buffer to match the command's format.

```c
static mspResult_e mspCommonProcessInCommand(mspDescriptor_t srcDesc, int16_t cmdMSP, sbuf_t *src, mspPostProcessFnPtr *mspPostProcessFn)
{
    UNUSED(mspPostProcessFn);
    const unsigned int dataSize = sbufBytesRemaining(src);
    UNUSED(dataSize); // maybe unused due to compiler options

    switch (cmdMSP) {
#ifdef USE_TRANSPONDER
    case MSP_SET_TRANSPONDER_CONFIG: {
        // Backward compatibility to BFC 3.1.1 is lost for this message type

        uint8_t provider = sbufReadU8(src);
        uint8_t bytesRemaining = dataSize - 1;

        if (provider > TRANSPONDER_PROVIDER_COUNT) {
            return MSP_RESULT_ERROR;
        }

        const uint8_t requirementIndex = provider - 1;
        const uint8_t transponderDataSize = transponderRequirements[requirementIndex].dataLength;

        transponderConfigMutable()->provider = provider;

        if (provider == TRANSPONDER_NONE) {
            break;
        }

        if (bytesRemaining != transponderDataSize) {
            return MSP_RESULT_ERROR;
        }

        if (provider != transponderConfig()->provider) {
            transponderStopRepeating();
        }

        memset(transponderConfigMutable()->data, 0, sizeof(transponderConfig()->data));

        for (unsigned int i = 0; i < transponderDataSize; i++) {
            transponderConfigMutable()->data[i] = sbufReadU8(src);
        }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="4164">

---

Here we're handling more config commands, reading values from the buffer and updating config. We check buffer size before reading extra fields to support different protocol versions.

```c
        transponderUpdateData();
        break;
    }
#endif

    case MSP_SET_VOLTAGE_METER_CONFIG: {
        int8_t id = sbufReadU8(src);

        //
        // find and configure an ADC voltage sensor
        //
        int8_t voltageSensorADCIndex;
        for (voltageSensorADCIndex = 0; voltageSensorADCIndex < MAX_VOLTAGE_SENSOR_ADC; voltageSensorADCIndex++) {
            if (id == voltageMeterADCtoIDMap[voltageSensorADCIndex]) {
                break;
            }
        }

        if (voltageSensorADCIndex < MAX_VOLTAGE_SENSOR_ADC) {
            voltageSensorADCConfigMutable(voltageSensorADCIndex)->vbatscale = sbufReadU8(src);
            voltageSensorADCConfigMutable(voltageSensorADCIndex)->vbatresdivval = sbufReadU8(src);
            voltageSensorADCConfigMutable(voltageSensorADCIndex)->vbatresdivmultiplier = sbufReadU8(src);
        } else {
            // if we had any other types of voltage sensor to configure, this is where we'd do it.
            sbufReadU8(src);
            sbufReadU8(src);
            sbufReadU8(src);
        }
        break;
    }

    case MSP_SET_CURRENT_METER_CONFIG: {
        int id = sbufReadU8(src);

        switch (id) {
            case CURRENT_METER_ID_BATTERY_1:
                currentSensorADCConfigMutable()->scale = sbufReadU16(src);
                currentSensorADCConfigMutable()->offset = sbufReadU16(src);
                break;
#ifdef USE_VIRTUAL_CURRENT_METER
            case CURRENT_METER_ID_VIRTUAL_1:
                currentSensorVirtualConfigMutable()->scale = sbufReadU16(src);
                currentSensorVirtualConfigMutable()->offset = sbufReadU16(src);
                break;
#endif
            default:
                sbufReadU16(src);
                sbufReadU16(src);
                break;
        }
        break;
    }

    case MSP_SET_BATTERY_CONFIG:
        batteryConfigMutable()->vbatmincellvoltage = sbufReadU8(src) * 10;      // vbatlevel_warn1 in MWC2.3 GUI
        batteryConfigMutable()->vbatmaxcellvoltage = sbufReadU8(src) * 10;      // vbatlevel_warn2 in MWC2.3 GUI
        batteryConfigMutable()->vbatwarningcellvoltage = sbufReadU8(src) * 10;  // vbatlevel when buzzer starts to alert
        batteryConfigMutable()->batteryCapacity = sbufReadU16(src);
        batteryConfigMutable()->voltageMeterSource = sbufReadU8(src);
        batteryConfigMutable()->currentMeterSource = sbufReadU8(src);
        if (sbufBytesRemaining(src) >= 6) {
            batteryConfigMutable()->vbatmincellvoltage = sbufReadU16(src);
            batteryConfigMutable()->vbatmaxcellvoltage = sbufReadU16(src);
            batteryConfigMutable()->vbatwarningcellvoltage = sbufReadU16(src);
        }
        break;

#if defined(USE_OSD)
    case MSP_SET_OSD_CONFIG:
        {
            const uint8_t addr = sbufReadU8(src);

            if ((int8_t)addr == -1) {
                /* Set general OSD settings */
                videoSystem_e video_system = sbufReadU8(src);

                if ((video_system == VIDEO_SYSTEM_HD) && (vcdProfile()->video_system != VIDEO_SYSTEM_HD)) {
                    // If switching to HD, don't wait for the VTX to communicate the correct resolution, just
#ifdef USE_OSD_HD
                    // If an HD build, increase the canvas size to the HD default as that is what the user will expect
                    osdConfigMutable()->canvas_cols = OSD_HD_COLS;
                    osdConfigMutable()->canvas_rows = OSD_HD_ROWS;
                    // Also force use of MSP displayport
                    osdConfigMutable()->displayPortDevice = OSD_DISPLAYPORT_DEVICE_MSP;
#else
                    // must have an SD build option, keep existing SD video_system, do not change canvas size
                    video_system = vcdProfile()->video_system;
#endif
                } else if ((video_system != VIDEO_SYSTEM_HD) && (vcdProfile()->video_system == VIDEO_SYSTEM_HD)) {
                    // Switching away from HD to SD
#ifdef USE_OSD_SD
                    // SD is in the build; set canvas size to SD and displayport device to auto
                    osdConfigMutable()->canvas_cols = OSD_SD_COLS;
                    osdConfigMutable()->canvas_rows = (video_system == VIDEO_SYSTEM_NTSC) ? VIDEO_LINES_NTSC : VIDEO_LINES_PAL;
                    osdConfigMutable()->displayPortDevice = OSD_DISPLAYPORT_DEVICE_AUTO;
#else
                    // must have an HD build option, keep existing HD video_system, do not change canvas size
                    video_system = VIDEO_SYSTEM_HD;
#endif
                }

                vcdProfileMutable()->video_system = video_system;

                osdConfigMutable()->units = sbufReadU8(src);

                // Alarms
                osdConfigMutable()->rssi_alarm = sbufReadU8(src);
                osdConfigMutable()->cap_alarm = sbufReadU16(src);
                sbufReadU16(src); // Skip unused (previously fly timer)
                osdConfigMutable()->alt_alarm = sbufReadU16(src);

                if (sbufBytesRemaining(src) >= 2) {
                    /* Enabled warnings */
                    // API < 1.41 supports only the low 16 bits
                    osdConfigMutable()->enabledWarnings = sbufReadU16(src);
                }

                if (sbufBytesRemaining(src) >= 4) {
                    // 32bit version of enabled warnings (API >= 1.41)
                    osdConfigMutable()->enabledWarnings = sbufReadU32(src);
                }

                if (sbufBytesRemaining(src) >= 1) {
                    // API >= 1.41
                    // selected OSD profile
#ifdef USE_OSD_PROFILES
                    changeOsdProfileIndex(sbufReadU8(src));
#else
                    sbufReadU8(src);
#endif // USE_OSD_PROFILES
                }

                if (sbufBytesRemaining(src) >= 1) {
                    // API >= 1.41
                    // OSD stick overlay mode

#ifdef USE_OSD_STICK_OVERLAY
                    osdConfigMutable()->overlay_radio_mode = sbufReadU8(src);
#else
                    sbufReadU8(src);
#endif // USE_OSD_STICK_OVERLAY

                }

                if (sbufBytesRemaining(src) >= 2) {
                    // API >= 1.43
                    // OSD camera frame element width/height
                    osdConfigMutable()->camera_frame_width = sbufReadU8(src);
                    osdConfigMutable()->camera_frame_height = sbufReadU8(src);
                }

                if (sbufBytesRemaining(src) >= 2) {
                    // API >= 1.46
                    osdConfigMutable()->link_quality_alarm = sbufReadU16(src);
                }

                if (sbufBytesRemaining(src) >= 2) {
                    // API >= 1.47
                    osdConfigMutable()->rssi_dbm_alarm = sbufReadU16(src);
                }

            } else if ((int8_t)addr == -2) {
                // Timers
                uint8_t index = sbufReadU8(src);
                if (index > OSD_TIMER_COUNT) {
                    return MSP_RESULT_ERROR;
                }
                osdConfigMutable()->timers[index] = sbufReadU16(src);
            } else {
                const uint16_t value = sbufReadU16(src);

                /* Get screen index, 0 is post flight statistics, 1 and above are in flight OSD screens */
                const uint8_t screen = (sbufBytesRemaining(src) >= 1) ? sbufReadU8(src) : 1;

                if (screen == 0 && addr < OSD_STAT_COUNT) {
                    /* Set statistic item enable */
                    osdStatSetState(addr, (value != 0));
                } else if (addr < OSD_ITEM_COUNT) {
                    /* Set element positions */
                    osdElementConfigMutable()->item_pos[addr] = value;
                    osdAnalyzeActiveElements();
                } else {
                    return MSP_RESULT_ERROR;
                }
            }
        }
        break;

    case MSP_OSD_CHAR_WRITE:
        {
            osdCharacter_t chr;
            size_t osdCharacterBytes;
            uint16_t addr;
            if (dataSize >= OSD_CHAR_VISIBLE_BYTES + 2) {
                if (dataSize >= OSD_CHAR_BYTES + 2) {
                    // 16 bit address, full char with metadata
                    addr = sbufReadU16(src);
                    osdCharacterBytes = OSD_CHAR_BYTES;
                } else if (dataSize >= OSD_CHAR_BYTES + 1) {
                    // 8 bit address, full char with metadata
                    addr = sbufReadU8(src);
                    osdCharacterBytes = OSD_CHAR_BYTES;
                } else {
                    // 16 bit character address, only visible char bytes
                    addr = sbufReadU16(src);
                    osdCharacterBytes = OSD_CHAR_VISIBLE_BYTES;
                }
            } else {
                // 8 bit character address, only visible char bytes
                addr = sbufReadU8(src);
                osdCharacterBytes = OSD_CHAR_VISIBLE_BYTES;
            }
            for (unsigned ii = 0; ii < MIN(osdCharacterBytes, sizeof(chr.data)); ii++) {
                chr.data[ii] = sbufReadU8(src);
            }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="4379">

---

After all the config command cases, if the command isn't handled, we call <SwmToken path="src/main/msp/msp.c" pos="4413:3:3" line-data="        return mspProcessInCommand(srcDesc, cmdMSP, src);">`mspProcessInCommand`</SwmToken> as a final fallback. This covers any remaining commands or returns an error.

```c
            displayPort_t *osdDisplayPort = osdGetDisplayPort(NULL);
            if (!osdDisplayPort) {
                return MSP_RESULT_ERROR;
            }

            if (!displayWriteFontCharacter(osdDisplayPort, addr, &chr)) {
                return MSP_RESULT_ERROR;
            }
        }
        break;

#ifdef USE_OSD_HD
    case MSP_SET_OSD_CANVAS:
        {
            osdConfigMutable()->canvas_cols = sbufReadU8(src);
            osdConfigMutable()->canvas_rows = sbufReadU8(src);

            if ((vcdProfile()->video_system != VIDEO_SYSTEM_HD) ||
                (osdConfig()->displayPortDevice != OSD_DISPLAYPORT_DEVICE_MSP)) {
                // An HD VTX has communicated it's canvas size, so we must be in HD mode
                vcdProfileMutable()->video_system = VIDEO_SYSTEM_HD;
                // And using MSP displayport
                osdConfigMutable()->displayPortDevice = OSD_DISPLAYPORT_DEVICE_MSP;

                // Save settings and reboot or the user won't see the effect and will have to manually save
                writeEEPROM();
                systemReset();
            }
        }
        break;
#endif //USE_OSD_HD
#endif // OSD

    default:
        return mspProcessInCommand(srcDesc, cmdMSP, src);
    }
    return MSP_RESULT_ACK;
}
```

---

</SwmSnippet>

### Deep Config and EEPROM Handling

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
  node1["Receive incoming command"]
  click node1 openCode "src/main/msp/msp.c:2647:4122"
  node1 --> node2{"What type of command?"}
  click node2 openCode "src/main/msp/msp.c:2652:4117"
  node2 -->|"Profile selection/copy"| node3{"Is craft disarmed?"}
  click node3 openCode "src/main/msp/msp.c:2656:2661"
  node3 -->|"No"| node8["Return error"]
  click node8 openCode "src/main/msp/msp.c:4119:4120"
  node3 -->|"Yes"| node4{"Is profile index valid?"}
  click node4 openCode "src/main/msp/msp.c:2657:2658"
  node4 -->|"No"| node8
  node4 -->|"Yes"| node5["Update profile"]
  click node5 openCode "src/main/msp/msp.c:2660:2661"
  node2 -->|"Sensor calibration"| node6{"Is craft disarmed?"}
  click node6 openCode "src/main/msp/msp.c:3315:3316"
  node6 -->|"No"| node8
  node6 -->|"Yes"| node7["Start calibration"]
  click node7 openCode "src/main/msp/msp.c:3316:3316"
  node2 -->|"src/main/config update"| node9{"Does command update multiple items?"}
  click node9 openCode "src/main/msp/msp.c:2728:2732"
  node9 -->|"Yes"| loop1["For each item, update setting"]
  subgraph loop1["For each item to update"]
    loop1a["Update item setting"]
    click loop1a openCode "src/main/msp/msp.c:2729:2732"
  end
  loop1 --> node11["Return acknowledgment"]
  click node11 openCode "src/main/msp/msp.c:4121:4122"
  node9 -->|"No"| node10["Update src/main/config"]
  click node10 openCode "src/main/msp/msp.c:2733:2734"
  node10 --> node11
  node5 --> node11
  node7 --> node11
  node2 -->|"Other"| node12["Perform action"]
  click node12 openCode "src/main/msp/msp.c:4117:4118"
  node12 --> node11
  node8 --> node13["Return error"]
  click node13 openCode "src/main/msp/msp.c:4119:4120"

classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;

%% Swimm:
%% %%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
%% flowchart TD
%%   node1["Receive incoming command"]
%%   click node1 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2647:4122"
%%   node1 --> node2{"What type of command?"}
%%   click node2 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2652:4117"
%%   node2 -->|"Profile selection/copy"| node3{"Is craft disarmed?"}
%%   click node3 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2656:2661"
%%   node3 -->|"No"| node8["Return error"]
%%   click node8 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4119:4120"
%%   node3 -->|"Yes"| node4{"Is profile index valid?"}
%%   click node4 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2657:2658"
%%   node4 -->|"No"| node8
%%   node4 -->|"Yes"| node5["Update profile"]
%%   click node5 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2660:2661"
%%   node2 -->|"Sensor calibration"| node6{"Is craft disarmed?"}
%%   click node6 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:3315:3316"
%%   node6 -->|"No"| node8
%%   node6 -->|"Yes"| node7["Start calibration"]
%%   click node7 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:3316:3316"
%%   node2 -->|"<SwmPath>[src/main/config/](src/main/config/)</SwmPath> update"| node9{"Does command update multiple items?"}
%%   click node9 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2728:2732"
%%   node9 -->|"Yes"| loop1["For each item, update setting"]
%%   subgraph loop1["For each item to update"]
%%     loop1a["Update item setting"]
%%     click loop1a openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2729:2732"
%%   end
%%   loop1 --> node11["Return acknowledgment"]
%%   click node11 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4121:4122"
%%   node9 -->|"No"| node10["Update <SwmPath>[src/main/config/](src/main/config/)</SwmPath>"]
%%   click node10 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:2733:2734"
%%   node10 --> node11
%%   node5 --> node11
%%   node7 --> node11
%%   node2 -->|"Other"| node12["Perform action"]
%%   click node12 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4117:4118"
%%   node12 --> node11
%%   node8 --> node13["Return error"]
%%   click node13 openCode "<SwmPath>[src/…/msp/msp.c](src/main/msp/msp.c)</SwmPath>:4119:4120"
%% 
%% classDef HeadingStyle fill:#777777,stroke:#333,stroke-width:2px;
```

<SwmSnippet path="/src/main/msp/msp.c" line="2647">

---

In <SwmToken path="src/main/msp/msp.c" pos="2647:4:4" line-data="static mspResult_e mspProcessInCommand(mspDescriptor_t srcDesc, int16_t cmdMSP, sbuf_t *src)">`mspProcessInCommand`</SwmToken>, we switch on the command and parse config data from the buffer, updating internal state or config. Each case is tightly coupled to Betaflight's data structures, and we use feature flags to include or exclude command handlers. For profile selection/copy, we clamp indices to valid ranges.

```c
static mspResult_e mspProcessInCommand(mspDescriptor_t srcDesc, int16_t cmdMSP, sbuf_t *src)
{
    uint32_t i;
    uint8_t value;
    const unsigned int dataSize = sbufBytesRemaining(src);
    switch (cmdMSP) {
    case MSP_SELECT_SETTING:
        value = sbufReadU8(src);
        if ((value & RATEPROFILE_MASK) == 0) {
            if (!ARMING_FLAG(ARMED)) {
                if (value >= PID_PROFILE_COUNT) {
                    value = 0;
                }
                changePidProfile(value);
            }
        } else {
            value = value & ~RATEPROFILE_MASK;

            if (value >= CONTROL_RATE_PROFILE_COUNT) {
                value = 0;
            }
            changeControlRateProfile(value);
        }
        break;

    case MSP_COPY_PROFILE:
        value = sbufReadU8(src);        // 0 = pid profile, 1 = control rate profile
        uint8_t dstProfileIndex = sbufReadU8(src);
        uint8_t srcProfileIndex = sbufReadU8(src);
        if (value == 0) {
            pidCopyProfile(dstProfileIndex, srcProfileIndex);
        }
        else if (value == 1) {
            copyControlRateProfile(dstProfileIndex, srcProfileIndex);
        }
        break;

#if defined(USE_GPS) || defined(USE_MAG)
    case MSP_SET_HEADING:
        magHold = sbufReadU16(src);
        break;
#endif

    case MSP_SET_RAW_RC:
#ifdef USE_RX_MSP
        {
            uint8_t channelCount = dataSize / sizeof(uint16_t);
            if (channelCount > MAX_SUPPORTED_RC_CHANNEL_COUNT) {
                return MSP_RESULT_ERROR;
            } else {
                uint16_t frame[MAX_SUPPORTED_RC_CHANNEL_COUNT];
                for (int i = 0; i < channelCount; i++) {
                    frame[i] = sbufReadU16(src);
                }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="2701">

---

Here we're handling more config commands, and for each one, we check if there are enough bytes left before reading optional fields. This lets us support multiple protocol versions and keep things compatible.

```c
                rxMspFrameReceive(frame, channelCount);
            }
        }
#endif
        break;
#if defined(USE_ACC)
    case MSP_SET_ACC_TRIM:
        accelerometerConfigMutable()->accelerometerTrims.values.pitch = sbufReadU16(src);
        accelerometerConfigMutable()->accelerometerTrims.values.roll  = sbufReadU16(src);

        break;
#endif
    case MSP_SET_ARMING_CONFIG:
        armingConfigMutable()->auto_disarm_delay = sbufReadU8(src);
        sbufReadU8(src); // reserved. disarm_kill_switch was removed in #5073
        if (sbufBytesRemaining(src)) {
          imuConfigMutable()->small_angle = sbufReadU8(src);
        }
        if (sbufBytesRemaining(src)) {
            armingConfigMutable()->gyro_cal_on_first_arm = sbufReadU8(src);
        }
        break;

    case MSP_SET_PID_CONTROLLER:
        break;

    case MSP_SET_PID:
        for (int i = 0; i < PID_ITEM_COUNT; i++) {
            currentPidProfile->pid[i].P = sbufReadU8(src);
            currentPidProfile->pid[i].I = sbufReadU8(src);
            currentPidProfile->pid[i].D = sbufReadU8(src);
        }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="2733">

---

After reading new PID values from the buffer, we call <SwmToken path="src/main/msp/msp.c" pos="2733:1:1" line-data="        pidInitConfig(currentPidProfile);">`pidInitConfig`</SwmToken> to apply the changes to the PID controller. This keeps the system in sync with the new config.

```c
        pidInitConfig(currentPidProfile);
        break;

    case MSP_SET_MODE_RANGE:
        i = sbufReadU8(src);
        if (i < MAX_MODE_ACTIVATION_CONDITION_COUNT) {
            modeActivationCondition_t *mac = modeActivationConditionsMutable(i);
            i = sbufReadU8(src);
            const box_t *box = findBoxByPermanentId(i);
            if (box) {
                mac->modeId = box->boxId;
                mac->auxChannelIndex = sbufReadU8(src);
                mac->range.startStep = sbufReadU8(src);
                mac->range.endStep = sbufReadU8(src);
                if (sbufBytesRemaining(src) != 0) {
                    mac->modeLogic = sbufReadU8(src);

                    i = sbufReadU8(src);
                    mac->linkedTo = findBoxByPermanentId(i)->boxId;
                }
                rcControlsInit();
            } else {
                return MSP_RESULT_ERROR;
            }
        } else {
            return MSP_RESULT_ERROR;
        }
        break;

    case MSP_SET_ADJUSTMENT_RANGE:
        i = sbufReadU8(src);
        if (i < MAX_ADJUSTMENT_RANGE_COUNT) {
            adjustmentRange_t *adjRange = adjustmentRangesMutable(i);
            sbufReadU8(src); // was adjRange->adjustmentIndex
            adjRange->auxChannelIndex = sbufReadU8(src);
            adjRange->range.startStep = sbufReadU8(src);
            adjRange->range.endStep = sbufReadU8(src);
            adjRange->adjustmentConfig = sbufReadU8(src);
            adjRange->auxSwitchChannelIndex = sbufReadU8(src);

            activeAdjustmentRangeReset();
        } else {
            return MSP_RESULT_ERROR;
        }
        break;

    case MSP_SET_RC_TUNING:
        if (sbufBytesRemaining(src) >= 10) {
            value = sbufReadU8(src);
            if (currentControlRateProfile->rcRates[FD_PITCH] == currentControlRateProfile->rcRates[FD_ROLL]) {
                currentControlRateProfile->rcRates[FD_PITCH] = value;
            }
            currentControlRateProfile->rcRates[FD_ROLL] = value;

            value = sbufReadU8(src);
            if (currentControlRateProfile->rcExpo[FD_PITCH] == currentControlRateProfile->rcExpo[FD_ROLL]) {
                currentControlRateProfile->rcExpo[FD_PITCH] = value;
            }
            currentControlRateProfile->rcExpo[FD_ROLL] = value;

            for (int i = 0; i < 3; i++) {
                currentControlRateProfile->rates[i] = sbufReadU8(src);
            }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="2797">

---

Here we're updating RC tuning and motor config, reading as many fields as are present in the buffer. If some fields are missing, we just skip them, so older clients still work.

```c
            sbufReadU8(src);    // tpa_rate is moved to PID profile
            currentControlRateProfile->thrMid8 = sbufReadU8(src);
            currentControlRateProfile->thrExpo8 = sbufReadU8(src);
            sbufReadU16(src);   // tpa_breakpoint is moved to PID profile

            if (sbufBytesRemaining(src) >= 1) {
                currentControlRateProfile->rcExpo[FD_YAW] = sbufReadU8(src);
            }

            if (sbufBytesRemaining(src) >= 1) {
                currentControlRateProfile->rcRates[FD_YAW] = sbufReadU8(src);
            }

            if (sbufBytesRemaining(src) >= 1) {
                currentControlRateProfile->rcRates[FD_PITCH] = sbufReadU8(src);
            }

            if (sbufBytesRemaining(src) >= 1) {
                currentControlRateProfile->rcExpo[FD_PITCH] = sbufReadU8(src);
            }

            // version 1.41
            if (sbufBytesRemaining(src) >= 2) {
                currentControlRateProfile->throttle_limit_type = sbufReadU8(src);
                currentControlRateProfile->throttle_limit_percent = sbufReadU8(src);
            }

            // version 1.42
            if (sbufBytesRemaining(src) >= 6) {
                currentControlRateProfile->rate_limit[FD_ROLL] = sbufReadU16(src);
                currentControlRateProfile->rate_limit[FD_PITCH] = sbufReadU16(src);
                currentControlRateProfile->rate_limit[FD_YAW] = sbufReadU16(src);
            }

            // version 1.43
            if (sbufBytesRemaining(src) >= 1) {
                currentControlRateProfile->rates_type = sbufReadU8(src);
            }

            // version 1.47
            if (sbufBytesRemaining(src) >= 1) {
                currentControlRateProfile->thrHover8 = sbufReadU8(src);
            }

            initRcProcessing();
        } else {
            return MSP_RESULT_ERROR;
        }
        break;

    case MSP_SET_MOTOR_CONFIG:
        sbufReadU16(src);   // minthrottle deprecated in 4.6
        motorConfigMutable()->maxthrottle = sbufReadU16(src);
        motorConfigMutable()->mincommand = sbufReadU16(src);

        // version 1.42
        if (sbufBytesRemaining(src) >= 2) {
            motorConfigMutable()->motorPoleCount = sbufReadU8(src);
#if defined(USE_DSHOT_TELEMETRY)
            motorConfigMutable()->dev.useDshotTelemetry = sbufReadU8(src);
#else
            sbufReadU8(src);
#endif
        }
        break;

#ifdef USE_GPS
    case MSP_SET_GPS_CONFIG:
        gpsConfigMutable()->provider = sbufReadU8(src);
        gpsConfigMutable()->sbasMode = sbufReadU8(src);
        gpsConfigMutable()->autoConfig = sbufReadU8(src);
        gpsConfigMutable()->autoBaud = sbufReadU8(src);
        if (sbufBytesRemaining(src) >= 2) {
            // Added in API version 1.43
            gpsConfigMutable()->gps_set_home_point_once = sbufReadU8(src);
            gpsConfigMutable()->gps_ublox_use_galileo = sbufReadU8(src);
        }
        break;
#endif

#ifdef USE_MAG
    case MSP_SET_COMPASS_CONFIG:
        imuConfigMutable()->mag_declination = sbufReadU16(src);
        break;
#endif

#ifdef USE_GPS
#ifdef USE_GPS_RESCUE
#ifndef USE_WING
    case MSP_SET_GPS_RESCUE:
        gpsRescueConfigMutable()->maxRescueAngle = sbufReadU16(src);
        gpsRescueConfigMutable()->returnAltitudeM = sbufReadU16(src);
        gpsRescueConfigMutable()->descentDistanceM = sbufReadU16(src);
        gpsRescueConfigMutable()->groundSpeedCmS = sbufReadU16(src);
        autopilotConfigMutable()->throttleMin = sbufReadU16(src);
        autopilotConfigMutable()->throttleMax = sbufReadU16(src);
        autopilotConfigMutable()->hoverThrottle = sbufReadU16(src);
        gpsRescueConfigMutable()->sanityChecks = sbufReadU8(src);
        gpsRescueConfigMutable()->minSats = sbufReadU8(src);
        if (sbufBytesRemaining(src) >= 6) {
            // Added in API version 1.43
            gpsRescueConfigMutable()->ascendRate = sbufReadU16(src);
            gpsRescueConfigMutable()->descendRate = sbufReadU16(src);
            gpsRescueConfigMutable()->allowArmingWithoutFix = sbufReadU8(src);
            gpsRescueConfigMutable()->altitudeMode = sbufReadU8(src);
        }
        if (sbufBytesRemaining(src) >= 2) {
            // Added in API version 1.44
            gpsRescueConfigMutable()->minStartDistM = sbufReadU16(src);
        }
        if (sbufBytesRemaining(src) >= 2) {
            // Added in API version 1.46
            gpsRescueConfigMutable()->initialClimbM = sbufReadU16(src);
        }
        break;

    case MSP_SET_GPS_RESCUE_PIDS:
        autopilotConfigMutable()->altitudeP = sbufReadU16(src);
        autopilotConfigMutable()->altitudeI = sbufReadU16(src);
        autopilotConfigMutable()->altitudeD = sbufReadU16(src);
        // altitude_F not included in msp yet
        gpsRescueConfigMutable()->velP = sbufReadU16(src);
        gpsRescueConfigMutable()->velI = sbufReadU16(src);
        gpsRescueConfigMutable()->velD = sbufReadU16(src);
        gpsRescueConfigMutable()->yawP = sbufReadU16(src);
        break;
#endif // !USE_WING
#endif
#endif

    case MSP_SET_MOTOR:
        for (int i = 0; i < getMotorCount(); i++) {
            motor_disarmed[i] = motorConvertFromExternal(sbufReadU16(src));
        }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="2938">

---

This section handles commands that do more than just update config—they can trigger calibration, write to EEPROM, or send hardware commands. After updating config, we call routines to reinit filters or mixers so changes take effect everywhere.

```c
        i = sbufReadU8(src);
        if (i >= MAX_SUPPORTED_SERVOS) {
            return MSP_RESULT_ERROR;
        } else {
            servoParamsMutable(i)->min = sbufReadU16(src);
            servoParamsMutable(i)->max = sbufReadU16(src);
            servoParamsMutable(i)->middle = sbufReadU16(src);
            servoParamsMutable(i)->rate = sbufReadU8(src);
            servoParamsMutable(i)->forwardFromChannel = sbufReadU8(src);
            servoParamsMutable(i)->reversedSources = sbufReadU32(src);
        }
#endif
        break;

    case MSP_SET_SERVO_MIX_RULE:
#ifdef USE_SERVOS
        i = sbufReadU8(src);
        if (i >= MAX_SERVO_RULES) {
            return MSP_RESULT_ERROR;
        } else {
            customServoMixersMutable(i)->targetChannel = sbufReadU8(src);
            customServoMixersMutable(i)->inputSource = sbufReadU8(src);
            customServoMixersMutable(i)->rate = sbufReadU8(src);
            customServoMixersMutable(i)->speed = sbufReadU8(src);
            customServoMixersMutable(i)->min = sbufReadU8(src);
            customServoMixersMutable(i)->max = sbufReadU8(src);
            customServoMixersMutable(i)->box = sbufReadU8(src);
            loadCustomServoMixer();
        }
#endif
        break;

    case MSP_SET_MOTOR_3D_CONFIG:
        flight3DConfigMutable()->deadband3d_low = sbufReadU16(src);
        flight3DConfigMutable()->deadband3d_high = sbufReadU16(src);
        flight3DConfigMutable()->neutral3d = sbufReadU16(src);
        break;

    case MSP_SET_RC_DEADBAND:
        rcControlsConfigMutable()->deadband = sbufReadU8(src);
        rcControlsConfigMutable()->yaw_deadband = sbufReadU8(src);
#if defined(USE_POSITION_HOLD) && !defined(USE_WING)
        posHoldConfigMutable()->deadband = sbufReadU8(src);
#else
        sbufReadU8(src);
#endif
        flight3DConfigMutable()->deadband3d_throttle = sbufReadU16(src);
        break;

    case MSP_SET_RESET_CURR_PID:
        resetPidProfile(currentPidProfile);
        break;

    case MSP_SET_SENSOR_ALIGNMENT: {
        sbufReadU8(src);
        sbufReadU8(src);  // discard deprecated acc_align
#if defined(USE_MAG)
        compassConfigMutable()->mag_alignment = sbufReadU8(src);
#else
        sbufReadU8(src);
#endif

        gyroConfigMutable()->gyro_enabled_bitmask = sbufReadU8(src);

        if (sbufBytesRemaining(src) >= 6) {
#ifdef USE_MAG
            compassConfigMutable()->mag_customAlignment.roll = sbufReadU16(src);
            compassConfigMutable()->mag_customAlignment.pitch = sbufReadU16(src);
            compassConfigMutable()->mag_customAlignment.yaw = sbufReadU16(src);
#else
            sbufReadU16(src);
            sbufReadU16(src);
            sbufReadU16(src);
#endif
        }

        break;
    }

    case MSP_SET_ADVANCED_CONFIG:
        sbufReadU8(src); // was gyroConfigMutable()->gyro_sync_denom - removed in API 1.43
        pidConfigMutable()->pid_process_denom = sbufReadU8(src);
        motorConfigMutable()->dev.useContinuousUpdate = sbufReadU8(src);
        motorConfigMutable()->dev.motorProtocol = sbufReadU8(src);
        motorConfigMutable()->dev.motorPwmRate = sbufReadU16(src);
        if (sbufBytesRemaining(src) >= 2) {
            motorConfigMutable()->motorIdle = sbufReadU16(src);
        }
        if (sbufBytesRemaining(src)) {
            sbufReadU8(src); // DEPRECATED: gyro_use_32khz
        }
        if (sbufBytesRemaining(src)) {
            motorConfigMutable()->dev.motorInversion = sbufReadU8(src);
        }
        if (sbufBytesRemaining(src) >= 8) {
            sbufReadU8(src); // deprecated gyro_to_use
            gyroConfigMutable()->gyro_high_fsr = sbufReadU8(src);
            gyroConfigMutable()->gyroMovementCalibrationThreshold = sbufReadU8(src);
            gyroConfigMutable()->gyroCalibrationDuration = sbufReadU16(src);
            gyroConfigMutable()->gyro_offset_yaw = sbufReadU16(src);
            gyroConfigMutable()->checkOverflow = sbufReadU8(src);
        }
        if (sbufBytesRemaining(src) >= 1) {
            //Added in MSP API 1.42
            systemConfigMutable()->debug_mode = sbufReadU8(src);
        }

        validateAndFixGyroConfig();

        break;
    case MSP_SET_FILTER_CONFIG:
        gyroConfigMutable()->gyro_lpf1_static_hz = sbufReadU8(src);
        currentPidProfile->dterm_lpf1_static_hz = sbufReadU16(src);
        currentPidProfile->yaw_lowpass_hz = sbufReadU16(src);
        if (sbufBytesRemaining(src) >= 8) {
            gyroConfigMutable()->gyro_soft_notch_hz_1 = sbufReadU16(src);
            gyroConfigMutable()->gyro_soft_notch_cutoff_1 = sbufReadU16(src);
            currentPidProfile->dterm_notch_hz = sbufReadU16(src);
            currentPidProfile->dterm_notch_cutoff = sbufReadU16(src);
        }
        if (sbufBytesRemaining(src) >= 4) {
            gyroConfigMutable()->gyro_soft_notch_hz_2 = sbufReadU16(src);
            gyroConfigMutable()->gyro_soft_notch_cutoff_2 = sbufReadU16(src);
        }
        if (sbufBytesRemaining(src) >= 1) {
            currentPidProfile->dterm_lpf1_type = sbufReadU8(src);
        }
        if (sbufBytesRemaining(src) >= 10) {
            gyroConfigMutable()->gyro_hardware_lpf = sbufReadU8(src);
            sbufReadU8(src); // DEPRECATED: gyro_32khz_hardware_lpf
            gyroConfigMutable()->gyro_lpf1_static_hz = sbufReadU16(src);
            gyroConfigMutable()->gyro_lpf2_static_hz = sbufReadU16(src);
            gyroConfigMutable()->gyro_lpf1_type = sbufReadU8(src);
            gyroConfigMutable()->gyro_lpf2_type = sbufReadU8(src);
            currentPidProfile->dterm_lpf2_static_hz = sbufReadU16(src);
        }
        if (sbufBytesRemaining(src) >= 9) {
            // Added in MSP API 1.41
            currentPidProfile->dterm_lpf2_type = sbufReadU8(src);
#if defined(USE_DYN_LPF)
            gyroConfigMutable()->gyro_lpf1_dyn_min_hz = sbufReadU16(src);
            gyroConfigMutable()->gyro_lpf1_dyn_max_hz = sbufReadU16(src);
            currentPidProfile->dterm_lpf1_dyn_min_hz = sbufReadU16(src);
            currentPidProfile->dterm_lpf1_dyn_max_hz = sbufReadU16(src);
#else
            sbufReadU16(src);
            sbufReadU16(src);
            sbufReadU16(src);
            sbufReadU16(src);
#endif
        }
        if (sbufBytesRemaining(src) >= 8) {
            // Added in MSP API 1.42
#if defined(USE_DYN_NOTCH_FILTER)
            sbufReadU8(src); // DEPRECATED 1.43: dyn_notch_range
            sbufReadU8(src); // DEPRECATED 1.44: dyn_notch_width_percent
            dynNotchConfigMutable()->dyn_notch_q = sbufReadU16(src);
            dynNotchConfigMutable()->dyn_notch_min_hz = sbufReadU16(src);
#else
            sbufReadU8(src);
            sbufReadU8(src);
            sbufReadU16(src);
            sbufReadU16(src);
#endif
#if defined(USE_RPM_FILTER)
            rpmFilterConfigMutable()->rpm_filter_harmonics = sbufReadU8(src);
            rpmFilterConfigMutable()->rpm_filter_min_hz = sbufReadU8(src);
#else
            sbufReadU8(src);
            sbufReadU8(src);
#endif
        }
        if (sbufBytesRemaining(src) >= 2) {
#if defined(USE_DYN_NOTCH_FILTER)
            // Added in MSP API 1.43
            dynNotchConfigMutable()->dyn_notch_max_hz = sbufReadU16(src);
#else
            sbufReadU16(src);
#endif
        }
        if (sbufBytesRemaining(src) >= 2) {
            // Added in MSP API 1.44
#if defined(USE_DYN_LPF)
            currentPidProfile->dterm_lpf1_dyn_expo = sbufReadU8(src);
#else
            sbufReadU8(src);
#endif
#if defined(USE_DYN_NOTCH_FILTER)
            dynNotchConfigMutable()->dyn_notch_count = sbufReadU8(src);
#else
            sbufReadU8(src);
#endif
        }

        // reinitialize the gyro filters with the new values
        validateAndFixGyroConfig();
        gyroInitFilters();
        // reinitialize the PID filters with the new values
        pidInitFilters(currentPidProfile);

        break;
    case MSP_SET_PID_ADVANCED:
        sbufReadU16(src);
        sbufReadU16(src);
        sbufReadU16(src); // was pidProfile.yaw_p_limit
        sbufReadU8(src); // reserved
        sbufReadU8(src); // was vbatPidCompensation
#if defined(USE_FEEDFORWARD)
        currentPidProfile->feedforward_transition = sbufReadU8(src);
#else
        sbufReadU8(src);
#endif
        sbufReadU8(src); // was low byte of currentPidProfile->dtermSetpointWeight
        sbufReadU8(src); // reserved
        sbufReadU8(src); // reserved
        sbufReadU8(src); // reserved
        currentPidProfile->rateAccelLimit = sbufReadU16(src);
        currentPidProfile->yawRateAccelLimit = sbufReadU16(src);
        if (sbufBytesRemaining(src) >= 2) {
            currentPidProfile->angle_limit = sbufReadU8(src);
            sbufReadU8(src); // was pidProfile.levelSensitivity
        }
        if (sbufBytesRemaining(src) >= 4) {
            sbufReadU16(src); // was currentPidProfile->itermThrottleThreshold
            currentPidProfile->anti_gravity_gain = sbufReadU16(src);
        }
        if (sbufBytesRemaining(src) >= 2) {
            sbufReadU16(src); // was currentPidProfile->dtermSetpointWeight
        }
        if (sbufBytesRemaining(src) >= 14) {
            // Added in MSP API 1.40
            currentPidProfile->iterm_rotation = sbufReadU8(src);
            sbufReadU8(src); // was currentPidProfile->smart_feedforward
#if defined(USE_ITERM_RELAX)
            currentPidProfile->iterm_relax = sbufReadU8(src);
            currentPidProfile->iterm_relax_type = sbufReadU8(src);
#else
            sbufReadU8(src);
            sbufReadU8(src);
#endif
#if defined(USE_ABSOLUTE_CONTROL)
            currentPidProfile->abs_control_gain = sbufReadU8(src);
#else
            sbufReadU8(src);
#endif
#if defined(USE_THROTTLE_BOOST)
            currentPidProfile->throttle_boost = sbufReadU8(src);
#else
            sbufReadU8(src);
#endif
#if defined(USE_ACRO_TRAINER)
            currentPidProfile->acro_trainer_angle_limit = sbufReadU8(src);
#else
            sbufReadU8(src);
#endif
            // PID controller feedforward terms
            currentPidProfile->pid[PID_ROLL].F = sbufReadU16(src);
            currentPidProfile->pid[PID_PITCH].F = sbufReadU16(src);
            currentPidProfile->pid[PID_YAW].F = sbufReadU16(src);
            sbufReadU8(src); // was currentPidProfile->antiGravityMode
        }
        if (sbufBytesRemaining(src) >= 7) {
            // Added in MSP API 1.41
#ifdef USE_D_MAX
            currentPidProfile->d_max[PID_ROLL] = sbufReadU8(src);
            currentPidProfile->d_max[PID_PITCH] = sbufReadU8(src);
            currentPidProfile->d_max[PID_YAW] = sbufReadU8(src);
            currentPidProfile->d_max_gain = sbufReadU8(src);
            currentPidProfile->d_max_advance = sbufReadU8(src);
#else
            sbufReadU8(src);
            sbufReadU8(src);
            sbufReadU8(src);
            sbufReadU8(src);
            sbufReadU8(src);
#endif
#if defined(USE_INTEGRATED_YAW_CONTROL)
            currentPidProfile->use_integrated_yaw = sbufReadU8(src);
            currentPidProfile->integrated_yaw_relax = sbufReadU8(src);
#else
            sbufReadU8(src);
            sbufReadU8(src);
#endif
        }
        if(sbufBytesRemaining(src) >= 1) {
            // Added in MSP API 1.42
#if defined(USE_ITERM_RELAX)
            currentPidProfile->iterm_relax_cutoff = sbufReadU8(src);
#else
            sbufReadU8(src);
#endif
        }
        if (sbufBytesRemaining(src) >= 3) {
            // Added in MSP API 1.43
            currentPidProfile->motor_output_limit = sbufReadU8(src);
            currentPidProfile->auto_profile_cell_count = sbufReadU8(src);
#if defined(USE_DYN_IDLE)
            currentPidProfile->dyn_idle_min_rpm = sbufReadU8(src);
#else
            sbufReadU8(src);
#endif
        }
        if (sbufBytesRemaining(src) >= 7) {
            // Added in MSP API 1.44
#if defined(USE_FEEDFORWARD)
            currentPidProfile->feedforward_averaging = sbufReadU8(src);
            currentPidProfile->feedforward_smooth_factor = sbufReadU8(src);
            currentPidProfile->feedforward_boost = sbufReadU8(src);
            currentPidProfile->feedforward_max_rate_limit = sbufReadU8(src);
            currentPidProfile->feedforward_jitter_factor = sbufReadU8(src);
#else
            sbufReadU8(src);
            sbufReadU8(src);
            sbufReadU8(src);
            sbufReadU8(src);
            sbufReadU8(src);
#endif

#if defined(USE_BATTERY_VOLTAGE_SAG_COMPENSATION)
            currentPidProfile->vbat_sag_compensation = sbufReadU8(src);
#else
            sbufReadU8(src);
#endif
#if defined(USE_THRUST_LINEARIZATION)
            currentPidProfile->thrustLinearization = sbufReadU8(src);
#else
            sbufReadU8(src);
#endif
        }
        if (sbufBytesRemaining(src) >= 4) {
            // Added in API 1.45
            currentPidProfile->tpa_mode = sbufReadU8(src);
            currentPidProfile->tpa_rate = MIN(sbufReadU8(src), TPA_MAX);
            currentPidProfile->tpa_breakpoint = sbufReadU16(src);
        }

        pidInitConfig(currentPidProfile);
        initEscEndpoints();
        mixerInitProfile();

        break;
    case MSP_SET_SENSOR_CONFIG:
#if defined(USE_ACC)
        accelerometerConfigMutable()->acc_hardware = sbufReadU8(src);
#else
        sbufReadU8(src);
#endif
#if defined(USE_BARO)
        barometerConfigMutable()->baro_hardware = sbufReadU8(src);
#else
        sbufReadU8(src);
#endif
#if defined(USE_MAG)
        compassConfigMutable()->mag_hardware = sbufReadU8(src);
#else
        sbufReadU8(src);
#endif

        if (sbufBytesRemaining(src) >= 1) {
#ifdef USE_RANGEFINDER
            rangefinderConfigMutable()->rangefinder_hardware = sbufReadU8(src);
#else
            sbufReadU8(src);
#endif
        }

        if (sbufBytesRemaining(src) >= 1) {
#ifdef USE_OPTICALFLOW
            opticalflowConfigMutable()->opticalflow_hardware = sbufReadU8(src);
#else
            sbufReadU8(src);
#endif
        }
        break;

#ifdef USE_ACC
    case MSP_ACC_CALIBRATION:
        if (!ARMING_FLAG(ARMED))
            accStartCalibration();
        break;
#endif

#if defined(USE_MAG)
    case MSP_MAG_CALIBRATION:
        if (!ARMING_FLAG(ARMED)) {
            compassStartCalibration();
        }
#endif

        break;
    case MSP_EEPROM_WRITE:
        if (ARMING_FLAG(ARMED)) {
            return MSP_RESULT_ERROR;
        }

        // This is going to take some time and won't be done where real-time performance is needed so
        // ignore how long it takes to avoid confusing the scheduler
        schedulerIgnoreTaskStateTime();

#if defined(USE_MSP_OVER_TELEMETRY)
        if (featureIsEnabled(FEATURE_RX_SPI) && srcDesc == getMspTelemetryDescriptor()) {
            dispatchAdd(&writeReadEepromEntry, MSP_DISPATCH_DELAY_US);
        } else
#endif
        {
            writeReadEeprom(NULL);
        }

        break;

#ifdef USE_BLACKBOX
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="420">

---

<SwmToken path="src/main/msp/msp.c" pos="420:4:4" line-data="static void writeReadEeprom(dispatchEntry_t* self)">`writeReadEeprom`</SwmToken> skips everything if the system is armed, then writes and reads EEPROM to keep config in sync. If the VTX table needs reinit, we do that too. The <SwmToken path="src/main/msp/msp.c" pos="420:6:6" line-data="static void writeReadEeprom(dispatchEntry_t* self)">`dispatchEntry_t`</SwmToken> argument is ignored.

```c
static void writeReadEeprom(dispatchEntry_t* self)
{
    UNUSED(self);

    if (ARMING_FLAG(ARMED)) {
        return;
    }

    writeEEPROM();
    readEEPROM();

#ifdef USE_VTX_TABLE
    if (vtxTableNeedsInit) {
        vtxTableNeedsInit = false;
        vtxTableInit();  // Reinitialize and refresh the in-memory copies
    }
#endif
}
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="3349">

---

After syncing EEPROM, we keep handling more config commands.

```c
    case MSP_SET_BLACKBOX_CONFIG:
        // Don't allow config to be updated while Blackbox is logging
        if (blackboxMayEditConfig()) {
            blackboxConfigMutable()->device = sbufReadU8(src);
            const int rateNum = sbufReadU8(src); // was rate_num
            const int rateDenom = sbufReadU8(src); // was rate_denom
            uint16_t pRatio = 0;
            if (sbufBytesRemaining(src) >= 2) {
                // p_ratio specified, so use it directly
                pRatio = sbufReadU16(src);
            } else {
                // p_ratio not specified in MSP, so calculate it from old rateNum and rateDenom
                pRatio = blackboxCalculatePDenom(rateNum, rateDenom);
            }

            if (sbufBytesRemaining(src) >= 1) {
                // sample_rate specified, so use it directly
                blackboxConfigMutable()->sample_rate = sbufReadU8(src);
            } else {
                // sample_rate not specified in MSP, so calculate it from old p_ratio
                blackboxConfigMutable()->sample_rate = blackboxCalculateSampleRate(pRatio);
            }

            // Added in MSP API 1.45
            if (sbufBytesRemaining(src) >= 4) {
                blackboxConfigMutable()->fields_disabled_mask = sbufReadU32(src);
            }
        }
        break;
#endif

#ifdef USE_VTX_COMMON
    case MSP_SET_VTX_CONFIG:
        {
            vtxDevice_t *vtxDevice = vtxCommonDevice();
            vtxDevType_e vtxType = VTXDEV_UNKNOWN;
            if (vtxDevice) {
                vtxType = vtxCommonGetDeviceType(vtxDevice);
            }
            uint16_t newFrequency = sbufReadU16(src);
            if (newFrequency <= VTXCOMMON_MSP_BANDCHAN_CHKVAL) {  // Value is band and channel
                const uint8_t newBand = (newFrequency / 8) + 1;
                const uint8_t newChannel = (newFrequency % 8) + 1;
                vtxSettingsConfigMutable()->band = newBand;
                vtxSettingsConfigMutable()->channel = newChannel;
                vtxSettingsConfigMutable()->freq = vtxCommonLookupFrequency(vtxDevice, newBand, newChannel);
            } else if (newFrequency <= VTX_SETTINGS_MAX_FREQUENCY_MHZ) { // Value is frequency in MHz
                vtxSettingsConfigMutable()->band = 0;
                vtxSettingsConfigMutable()->freq = newFrequency;
            }

            if (sbufBytesRemaining(src) >= 2) {
                vtxSettingsConfigMutable()->power = sbufReadU8(src);
                const uint8_t newPitmode = sbufReadU8(src);
                if (vtxType != VTXDEV_UNKNOWN) {
                    // Delegate pitmode to vtx directly
                    unsigned vtxCurrentStatus;
                    vtxCommonGetStatus(vtxDevice, &vtxCurrentStatus);
                    if ((bool)(vtxCurrentStatus & VTX_STATUS_PIT_MODE) != (bool)newPitmode) {
                        vtxCommonSetPitMode(vtxDevice, newPitmode);
                    }
                }
            }

            if (sbufBytesRemaining(src)) {
                    vtxSettingsConfigMutable()->lowPowerDisarm = sbufReadU8(src);
            }

            // API version 1.42 - this parameter kept separate since clients may already be supplying
            if (sbufBytesRemaining(src) >= 2) {
                vtxSettingsConfigMutable()->pitModeFreq = sbufReadU16(src);
            }

            // API version 1.42 - extensions for non-encoded versions of the band, channel or frequency
            if (sbufBytesRemaining(src) >= 4) {
                // Added standalone values for band, channel and frequency to move
                // away from the flawed encoded combined method originally implemented.
                uint8_t newBand = sbufReadU8(src);
                const uint8_t newChannel = sbufReadU8(src);
                uint16_t newFreq = sbufReadU16(src);
                if (newBand) {
                    newFreq = vtxCommonLookupFrequency(vtxDevice, newBand, newChannel);
                }
                vtxSettingsConfigMutable()->band = newBand;
                vtxSettingsConfigMutable()->channel = newChannel;
                vtxSettingsConfigMutable()->freq = newFreq;
            }

            // API version 1.42 - extensions for vtxtable support
            if (sbufBytesRemaining(src) >= 4) {
#ifdef USE_VTX_TABLE
                const uint8_t newBandCount = sbufReadU8(src);
                const uint8_t newChannelCount = sbufReadU8(src);
                const uint8_t newPowerCount = sbufReadU8(src);

                if ((newBandCount > VTX_TABLE_MAX_BANDS) ||
                    (newChannelCount > VTX_TABLE_MAX_CHANNELS) ||
                    (newPowerCount > VTX_TABLE_MAX_POWER_LEVELS)) {
                    return MSP_RESULT_ERROR;
                }
                vtxTableConfigMutable()->bands = newBandCount;
                vtxTableConfigMutable()->channels = newChannelCount;
                vtxTableConfigMutable()->powerLevels = newPowerCount;

                // boolean to determine whether the vtxtable should be cleared in
                // expectation that the detailed band/channel and power level messages
                // will follow to repopulate the tables
                if (sbufReadU8(src)) {
                    for (int i = 0; i < VTX_TABLE_MAX_BANDS; i++) {
                        vtxTableConfigClearBand(vtxTableConfigMutable(), i);
                        vtxTableConfigClearChannels(vtxTableConfigMutable(), i, 0);
                    }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="3461">

---

Here we're updating the VTX table config, checking that the new values are within allowed limits. If not, we bail out with an error. Otherwise, we update the config and clear the tables if requested.

```c
                    vtxTableConfigClearPowerLabels(vtxTableConfigMutable(), 0);
                    vtxTableConfigClearPowerValues(vtxTableConfigMutable(), 0);
                }
#else
                sbufReadU8(src);
                sbufReadU8(src);
                sbufReadU8(src);
                sbufReadU8(src);
#endif
            }
#ifdef USE_VTX_MSP
            setMspVtxDeviceStatusReady(srcDesc);
#endif
        }
        break;
#endif

#ifdef USE_VTX_TABLE
    case MSP_SET_VTXTABLE_BAND:
        {
            char bandName[VTX_TABLE_BAND_NAME_LENGTH + 1];
            memset(bandName, 0, VTX_TABLE_BAND_NAME_LENGTH + 1);
            uint16_t frequencies[VTX_TABLE_MAX_CHANNELS];
            const uint8_t band = sbufReadU8(src);
            const uint8_t bandNameLength = sbufReadU8(src);
            for (int i = 0; i < bandNameLength; i++) {
                const char nameChar = sbufReadU8(src);
                if (i < VTX_TABLE_BAND_NAME_LENGTH) {
                    bandName[i] = toupper(nameChar);
                }
            }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="3492">

---

Here we're parsing the VTX band config from the buffer: we grab the band letter, factory flag, and channel count, then loop to read each channel's frequency. Only frequencies up to the supported channel count are stored, so we don't overflow the config array. This sets up the data for the next step, where we actually update the VTX table config.

```c
            const char bandLetter = toupper(sbufReadU8(src));
            const bool isFactoryBand = (bool)sbufReadU8(src);
            const uint8_t channelCount = sbufReadU8(src);
            for (int i = 0; i < channelCount; i++) {
                const uint16_t frequency = sbufReadU16(src);
                if (i < vtxTableConfig()->channels) {
                    frequencies[i] = frequency;
                }
            }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="3502">

---

Next we validate the band index and, if it's valid, update the VTX table config with the new band name, letter, factory flag, and channel frequencies. We also flag the table for reinit. If the band is invalid, we bail out with an error. This leads into the next step, which handles the currently selected band and triggers any needed updates.

```c
            if (band > 0 && band <= vtxTableConfig()->bands) {
                vtxTableStrncpyWithPad(vtxTableConfigMutable()->bandNames[band - 1], bandName, VTX_TABLE_BAND_NAME_LENGTH);
                vtxTableConfigMutable()->bandLetters[band - 1] = bandLetter;
                vtxTableConfigMutable()->isFactoryBand[band - 1] = isFactoryBand;
                for (int i = 0; i < vtxTableConfig()->channels; i++) {
                    vtxTableConfigMutable()->frequency[band - 1][i] = frequencies[i];
                }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="3509">

---

Here we check if the updated band is currently selected. If so, we update the VTX frequency to match the new channel's frequency. We also flag the table for reinit. If the band is invalid, we return an error. After this, we move on to the next MSP command case.

```c
                // If this is the currently selected band then reset the frequency
                if (band == vtxSettingsConfig()->band) {
                    uint16_t newFreq = 0;
                    if (vtxSettingsConfig()->channel > 0 && vtxSettingsConfig()->channel <= vtxTableConfig()->channels) {
                        newFreq = frequencies[vtxSettingsConfig()->channel - 1];
                    }
                    vtxSettingsConfigMutable()->freq = newFreq;
                }
                vtxTableNeedsInit = true;  // reinintialize vtxtable after eeprom write
            } else {
                return MSP_RESULT_ERROR;
            }
#ifdef USE_VTX_MSP
            setMspVtxDeviceStatusReady(srcDesc);
#endif
        }
        break;

    case MSP_SET_VTXTABLE_POWERLEVEL:
        {
            char powerLevelLabel[VTX_TABLE_POWER_LABEL_LENGTH + 1];
            memset(powerLevelLabel, 0, VTX_TABLE_POWER_LABEL_LENGTH + 1);
            const uint8_t powerLevel = sbufReadU8(src);
            const uint16_t powerValue = sbufReadU16(src);
            const uint8_t powerLevelLabelLength = sbufReadU8(src);
            for (int i = 0; i < powerLevelLabelLength; i++) {
                const char labelChar = sbufReadU8(src);
                if (i < VTX_TABLE_POWER_LABEL_LENGTH) {
                    powerLevelLabel[i] = toupper(labelChar);
                }
            }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="3541">

---

Here we're handling VTX power level config: we validate the power level index, update the value and label if it's valid, and flag the table for reinit. If the index is out of range, we return an error. This is just another config update before moving on to the next command.

```c
            if (powerLevel > 0 && powerLevel <= vtxTableConfig()->powerLevels) {
                vtxTableConfigMutable()->powerValues[powerLevel - 1] = powerValue;
                vtxTableStrncpyWithPad(vtxTableConfigMutable()->powerLabels[powerLevel - 1], powerLevelLabel, VTX_TABLE_POWER_LABEL_LENGTH);
                vtxTableNeedsInit = true;  // reinintialize vtxtable after eeprom write
            } else {
                return MSP_RESULT_ERROR;
            }
#ifdef USE_VTX_MSP
            setMspVtxDeviceStatusReady(srcDesc);
#endif
        }
        break;
#endif

    case MSP2_SET_MOTOR_OUTPUT_REORDERING:
        {
            const uint8_t arraySize = sbufReadU8(src);

            for (unsigned i = 0; i < MAX_SUPPORTED_MOTORS; i++) {
                uint8_t value = i;

                if (i < arraySize) {
                    value = sbufReadU8(src);
                }

                motorConfigMutable()->dev.motorOutputReordering[i] = value;
            }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="3572">

---

Here we're handling motor output reordering: we read the new order from the buffer, but if the client doesn't provide enough values, we leave the rest in their default order. This lets clients partially update the mapping without breaking things. Next up is handling DSHOT commands.

```c
    case MSP2_SEND_DSHOT_COMMAND:
        {
            const bool armed = ARMING_FLAG(ARMED);

            if (!armed) {
                const uint8_t commandType = sbufReadU8(src);
                const uint8_t motorIndex = sbufReadU8(src);
                const uint8_t commandCount = sbufReadU8(src);

                if (DSHOT_CMD_TYPE_BLOCKING == commandType) {
                    motorDisable();
                }

                for (uint8_t i = 0; i < commandCount; i++) {
                    const uint8_t commandIndex = sbufReadU8(src);
                    dshotCommandWrite(motorIndex, getMotorCount(), commandIndex, commandType);
                }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="3590">

---

Here we're handling DSHOT command dispatch: if the system isn't armed, we process the command(s) from the buffer. For blocking commands, we disable motors before sending and re-enable them after. This is a hardware action, not just a config update. After this, we move on to more config commands.

```c
                if (DSHOT_CMD_TYPE_BLOCKING == commandType) {
                    motorEnable();
                }
            }
        }
        break;
#endif

#ifdef USE_SIMPLIFIED_TUNING
    // Added in MSP API 1.44
    case MSP_SET_SIMPLIFIED_TUNING:
        {
            readSimplifiedPids(currentPidProfile, src);
            readSimplifiedDtermFilters(currentPidProfile, src);
            readSimplifiedGyroFilters(gyroConfigMutable(), src);
            applySimplifiedTuning(currentPidProfile, gyroConfigMutable());
        }
        break;
#endif

#ifdef USE_CAMERA_CONTROL
    case MSP_CAMERA_CONTROL:
        {
            if (ARMING_FLAG(ARMED)) {
                return MSP_RESULT_ERROR;
            }

            const uint8_t key = sbufReadU8(src);
            cameraControlKeyPress(key, 0);
        }
        break;
#endif

    case MSP_SET_ARMING_DISABLED:
        {
            const uint8_t command = sbufReadU8(src);
            uint8_t disableRunawayTakeoff = 0;
#ifndef USE_RUNAWAY_TAKEOFF
            UNUSED(disableRunawayTakeoff);
#endif
            if (sbufBytesRemaining(src)) {
                disableRunawayTakeoff = sbufReadU8(src);
            }
            if (command) {
#ifndef SIMULATOR_BUILD // In simulator mode we can safely arm with MSP link.
                mspArmingDisableByDescriptor(srcDesc);
                setArmingDisabled(ARMING_DISABLED_MSP);
                if (ARMING_FLAG(ARMED)) {
                    disarm(DISARM_REASON_ARMING_DISABLED);
                }
#endif
#ifdef USE_RUNAWAY_TAKEOFF
                runawayTakeoffTemporaryDisable(false);
#endif
            } else {
                mspArmingEnableByDescriptor(srcDesc);
                if (mspIsMspArmingEnabled()) {
                    unsetArmingDisabled(ARMING_DISABLED_MSP);
#ifdef USE_RUNAWAY_TAKEOFF
                    runawayTakeoffTemporaryDisable(disableRunawayTakeoff);
#endif
                }
            }
        }
        break;

#if defined(USE_FLASHFS) && defined(USE_BLACKBOX)
    case MSP_DATAFLASH_ERASE:
        blackboxEraseAll();

        break;
#endif

#if defined(USE_RANGEFINDER_MT)
    case MSP2_SENSOR_RANGEFINDER_LIDARMT:
        mtRangefinderReceiveNewData(sbufPtr(src));
        break;

    case MSP2_SENSOR_OPTICALFLOW_MT:
        mtOpticalflowReceiveNewData(sbufPtr(src));
        break;
#endif

#ifdef USE_GPS
    case MSP2_SENSOR_GPS:
        (void)sbufReadU8(src);              // instance
        (void)sbufReadU16(src);             // gps_week
        gpsSol.time = sbufReadU32(src);     // ms_tow
        gpsSetFixState(sbufReadU8(src) != 0); // fix_type
        gpsSol.numSat = sbufReadU8(src);    // satellites_in_view
        gpsSol.acc.hAcc = sbufReadU16(src) * 10; // horizontal_pos_accuracy - convert cm to mm
        gpsSol.acc.vAcc = sbufReadU16(src) * 10; // vertical_pos_accuracy - convert cm to mm
        gpsSol.acc.sAcc = sbufReadU16(src) * 10; // horizontal_vel_accuracy - convert cm to mm
        gpsSol.dop.pdop = sbufReadU16(src); // hdop in 4.4 and earlier, pdop in 4.5 and above
        gpsSol.llh.lon = sbufReadU32(src);
        gpsSol.llh.lat = sbufReadU32(src);
        gpsSol.llh.altCm = sbufReadU32(src); // alt
        int32_t ned_vel_north = (int32_t)sbufReadU32(src);  // ned_vel_north
        int32_t ned_vel_east = (int32_t)sbufReadU32(src);   // ned_vel_east
        gpsSol.groundSpeed = (uint16_t)sqrtf((ned_vel_north * ned_vel_north) + (ned_vel_east * ned_vel_east));
        (void)sbufReadU32(src);             // ned_vel_down
        gpsSol.groundCourse = ((uint16_t)sbufReadU16(src) % 36000) / 10; // incoming value expected to be in centidegrees, output value in decidegrees
        (void)sbufReadU16(src);             // true_yaw
        (void)sbufReadU16(src);             // year
        (void)sbufReadU8(src);              // month
        (void)sbufReadU8(src);              // day
        (void)sbufReadU8(src);              // hour
        (void)sbufReadU8(src);              // min
        (void)sbufReadU8(src);              // sec
        GPS_update |= GPS_MSP_UPDATE;        // MSP data signalisation to GPS functions
        break;

    case MSP_SET_RAW_GPS:
        gpsSetFixState(sbufReadU8(src));
        gpsSol.numSat = sbufReadU8(src);
        gpsSol.llh.lat = sbufReadU32(src);
        gpsSol.llh.lon = sbufReadU32(src);
        gpsSol.llh.altCm = sbufReadU16(src) * 100; // alt changed from 1m to 0.01m per lsb since MSP API 1.39 by RTH. Received MSP altitudes in 1m per lsb have to upscaled.
        gpsSol.groundSpeed = sbufReadU16(src);
        GPS_update |= GPS_MSP_UPDATE;        // MSP data signalisation to GPS functions
        break;
#endif // USE_GPS
    case MSP_SET_FEATURE_CONFIG:
        featureConfigReplace(sbufReadU32(src));
        break;

#ifdef USE_BEEPER
    case MSP_SET_BEEPER_CONFIG:
        beeperConfigMutable()->beeper_off_flags = sbufReadU32(src);
        if (sbufBytesRemaining(src) >= 1) {
            beeperConfigMutable()->dshotBeaconTone = sbufReadU8(src);
        }
        if (sbufBytesRemaining(src) >= 4) {
            beeperConfigMutable()->dshotBeaconOffFlags = sbufReadU32(src);
        }
        break;
#endif

    case MSP_SET_BOARD_ALIGNMENT_CONFIG:
        boardAlignmentMutable()->rollDegrees = sbufReadU16(src);
        boardAlignmentMutable()->pitchDegrees = sbufReadU16(src);
        boardAlignmentMutable()->yawDegrees = sbufReadU16(src);
        break;

    case MSP_SET_MIXER_CONFIG:
#ifndef USE_QUAD_MIXER_ONLY
        mixerConfigMutable()->mixerMode = sbufReadU8(src);
#else
        sbufReadU8(src);
#endif
        if (sbufBytesRemaining(src) >= 1) {
            mixerConfigMutable()->yaw_motors_reversed = sbufReadU8(src);
        }
        break;

    case MSP_SET_RX_CONFIG:
        rxConfigMutable()->serialrx_provider = sbufReadU8(src);
        rxConfigMutable()->maxcheck = sbufReadU16(src);
        rxConfigMutable()->midrc = sbufReadU16(src);
        rxConfigMutable()->mincheck = sbufReadU16(src);
        rxConfigMutable()->spektrum_sat_bind = sbufReadU8(src);
        if (sbufBytesRemaining(src) >= 4) {
            rxConfigMutable()->rx_min_usec = sbufReadU16(src);
            rxConfigMutable()->rx_max_usec = sbufReadU16(src);
        }
        if (sbufBytesRemaining(src) >= 4) {
            sbufReadU8(src); // not required in API 1.44, was rxConfigMutable()->rcInterpolation
            sbufReadU8(src); // not required in API 1.44, was rxConfigMutable()->rcInterpolationInterval
            rxConfigMutable()->airModeActivateThreshold = (sbufReadU16(src) - 1000) / 10;
        }
        if (sbufBytesRemaining(src) >= 6) {
#ifdef USE_RX_SPI
            rxSpiConfigMutable()->rx_spi_protocol = sbufReadU8(src);
            rxSpiConfigMutable()->rx_spi_id = sbufReadU32(src);
            rxSpiConfigMutable()->rx_spi_rf_channel_count = sbufReadU8(src);
#else
            sbufReadU8(src);
            sbufReadU32(src);
            sbufReadU8(src);
#endif
        }
        if (sbufBytesRemaining(src) >= 1) {
            rxConfigMutable()->fpvCamAngleDegrees = sbufReadU8(src);
        }
        if (sbufBytesRemaining(src) >= 6) {
            // Added in MSP API 1.40
            sbufReadU8(src); // not required in API 1.44, was rxConfigMutable()->rcSmoothingChannels
#if defined(USE_RC_SMOOTHING_FILTER)
            sbufReadU8(src); // not required in API 1.44, was rc_smoothing_type
            configRebootUpdateCheckU8(&rxConfigMutable()->rc_smoothing_setpoint_cutoff, sbufReadU8(src));
            configRebootUpdateCheckU8(&rxConfigMutable()->rc_smoothing_throttle_cutoff, sbufReadU8(src)); // was rc_smoothing_feedforward_cutoff
            configRebootUpdateCheckU8(&rxConfigMutable()->rc_smoothing_auto_factor_throttle, sbufReadU8(src)); // was rc_smoothing_input_type
            sbufReadU8(src); // not required in API 1.44, was rc_smoothing_derivative_type
#else
            sbufReadU8(src);
            sbufReadU8(src);
            sbufReadU8(src);
            sbufReadU8(src);
            sbufReadU8(src);
#endif
        }
        if (sbufBytesRemaining(src) >= 1) {
            // Added in MSP API 1.40
            // Kept separate from the section above to work around missing Configurator support in version < 10.4.2
#if defined(USE_USB_CDC_HID)
            usbDevConfigMutable()->type = sbufReadU8(src);
#else
            sbufReadU8(src);
#endif
        }
        if (sbufBytesRemaining(src) >= 1) {
            // Added in MSP API 1.42
#if defined(USE_RC_SMOOTHING_FILTER)
            // Added extra validation/range constraint for rc_smoothing_auto_factor as a workaround for a bug in
            // the 10.6 configurator where it was possible to submit an invalid out-of-range value. We might be
            // able to remove the constraint at some point in the future once the affected versions are deprecated
            // enough that the risk is low.
            configRebootUpdateCheckU8(&rxConfigMutable()->rc_smoothing_auto_factor_rpy, constrain(sbufReadU8(src), RC_SMOOTHING_AUTO_FACTOR_MIN, RC_SMOOTHING_AUTO_FACTOR_MAX));
#else
            sbufReadU8(src);
#endif
        }
        if (sbufBytesRemaining(src) >= 1) {
            // Added in MSP API 1.44
#if defined(USE_RC_SMOOTHING_FILTER)
            configRebootUpdateCheckU8(&rxConfigMutable()->rc_smoothing, sbufReadU8(src));
#else
            sbufReadU8(src);
#endif
        }
        if (sbufBytesRemaining(src) >= 6) {
            // Added in MSP API 1.45
#ifdef USE_RX_EXPRESSLRS
            sbufReadData(src, rxExpressLrsSpiConfigMutable()->UID, 6);
#else
            uint8_t emptyUid[6];
            sbufReadData(src, emptyUid, 6);
#endif
        }
        if (sbufBytesRemaining(src) >= 1) {
#ifdef USE_RX_EXPRESSLRS
            // Added in MSP API 1.47
            rxExpressLrsSpiConfigMutable()->modelId = sbufReadU8(src);
#else
            sbufReadU8(src);
#endif
        }
        break;
    case MSP_SET_FAILSAFE_CONFIG:
        failsafeConfigMutable()->failsafe_delay = sbufReadU8(src);
        failsafeConfigMutable()->failsafe_landing_time = sbufReadU8(src);
        failsafeConfigMutable()->failsafe_throttle = sbufReadU16(src);
        failsafeConfigMutable()->failsafe_switch_mode = sbufReadU8(src);
        failsafeConfigMutable()->failsafe_throttle_low_delay = sbufReadU16(src);
        failsafeConfigMutable()->failsafe_procedure = sbufReadU8(src);
        break;

    case MSP_SET_RXFAIL_CONFIG:
        i = sbufReadU8(src);
        if (i < MAX_SUPPORTED_RC_CHANNEL_COUNT) {
            rxFailsafeChannelConfigsMutable(i)->mode = sbufReadU8(src);
            rxFailsafeChannelConfigsMutable(i)->step = CHANNEL_VALUE_TO_RXFAIL_STEP(sbufReadU16(src));
        } else {
            return MSP_RESULT_ERROR;
        }
        break;

    case MSP_SET_RSSI_CONFIG:
        rxConfigMutable()->rssi_channel = sbufReadU8(src);
        break;

    case MSP_SET_RX_MAP:
        for (int i = 0; i < RX_MAPPABLE_CHANNEL_COUNT; i++) {
            rxConfigMutable()->rcmap[i] = sbufReadU8(src);
        }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="3867">

---

Here we're handling serial port config updates: we check the packet size, then loop through each port config in the buffer, updating the settings if the port is recognized. If anything doesn't match up, we return an error. This is just another config update before moving on to the next command.

```c
    case MSP_SET_CF_SERIAL_CONFIG:
        {
            uint8_t portConfigSize = sizeof(uint8_t) + sizeof(uint16_t) + (sizeof(uint8_t) * 4);

            if (dataSize % portConfigSize != 0) {
                return MSP_RESULT_ERROR;
            }

            uint8_t remainingPortsInPacket = dataSize / portConfigSize;

            while (remainingPortsInPacket--) {
                uint8_t identifier = sbufReadU8(src);

                serialPortConfig_t *portConfig = serialFindPortConfigurationMutable(identifier);

                if (!portConfig) {
                    return MSP_RESULT_ERROR;
                }

                portConfig->functionMask = sbufReadU16(src);
                portConfig->msp_baudrateIndex = sbufReadU8(src);
                portConfig->gps_baudrateIndex = sbufReadU8(src);
                portConfig->telemetry_baudrateIndex = sbufReadU8(src);
                portConfig->blackbox_baudrateIndex = sbufReadU8(src);
            }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="3894">

---

Here we're handling another serial config command, this time with a slightly different structure. We read the count, check the size, and update each port config. Any extra bytes are skipped to keep things compatible with future protocol changes. This is just another config update before moving on to the next command.

```c
    case MSP2_COMMON_SET_SERIAL_CONFIG: {
        if (dataSize < 1) {
            return MSP_RESULT_ERROR;
        }
        unsigned count = sbufReadU8(src);
        unsigned portConfigSize = (dataSize - 1) / count;
        unsigned expectedPortSize = sizeof(uint8_t) + sizeof(uint32_t) + (sizeof(uint8_t) * 4);
        if (portConfigSize < expectedPortSize) {
            return MSP_RESULT_ERROR;
        }
        for (unsigned ii = 0; ii < count; ii++) {
            unsigned start = sbufBytesRemaining(src);
            uint8_t identifier = sbufReadU8(src);
            serialPortConfig_t *portConfig = serialFindPortConfigurationMutable(identifier);

            if (!portConfig) {
                return MSP_RESULT_ERROR;
            }

            portConfig->functionMask = sbufReadU32(src);
            portConfig->msp_baudrateIndex = sbufReadU8(src);
            portConfig->gps_baudrateIndex = sbufReadU8(src);
            portConfig->telemetry_baudrateIndex = sbufReadU8(src);
            portConfig->blackbox_baudrateIndex = sbufReadU8(src);
            // Skip unknown bytes
            while (start - sbufBytesRemaining(src) < portConfigSize && sbufBytesRemaining(src)) {
                sbufReadU8(src);
            }
        }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="3927">

---

Here we're updating the LED color palette: we loop through all configurable colors and update each one from the buffer. This lets the client set all colors in a single command. After this, we move on to the next LED config command.

```c
    case MSP_SET_LED_COLORS:
        for (int i = 0; i < LED_CONFIGURABLE_COLOR_COUNT; i++) {
            hsvColor_t *color = &ledStripStatusModeConfigMutable()->colors[i];
            color->h = sbufReadU16(src);
            color->s = sbufReadU8(src);
            color->v = sbufReadU8(src);
        }
```

---

</SwmSnippet>

<SwmSnippet path="/src/main/msp/msp.c" line="3938">

---

We return ACK or ERROR depending on whether the MSP command was handled, with the actual logic depending on build flags and protocol version.

```c
    case MSP_SET_LED_STRIP_CONFIG:
        {
            i = sbufReadU8(src);
            if (i >= LED_STRIP_MAX_LENGTH || dataSize != (1 + 4)) {
                return MSP_RESULT_ERROR;
            }
#ifdef USE_LED_STRIP_STATUS_MODE
            ledConfig_t *ledConfig = &ledStripStatusModeConfigMutable()->ledConfigs[i];
            *ledConfig = sbufReadU32(src);
            reevaluateLedConfig();
#else
            sbufReadU32(src);
#endif
            // API 1.41 - selected ledstrip_profile
            if (sbufBytesRemaining(src) >= 1) {
                ledStripConfigMutable()->ledstrip_profile = sbufReadU8(src);
            }
        }
        break;
#endif

#ifdef USE_LED_STRIP_STATUS_MODE
    case MSP_SET_LED_STRIP_MODECOLOR:
        {
            ledModeIndex_e modeIdx = sbufReadU8(src);
            int funIdx = sbufReadU8(src);
            int color = sbufReadU8(src);

            if (!setModeColor(modeIdx, funIdx, color)) {
                return MSP_RESULT_ERROR;
            }
        }
        break;
#endif

    case MSP_SET_NAME:
        memset(pilotConfigMutable()->craftName, 0, sizeof(pilotConfigMutable()->craftName));
        sbufReadData(src, pilotConfigMutable()->craftName, MIN(ARRAYLEN(pilotConfigMutable()->craftName) - 1, dataSize));
#ifdef USE_OSD
        osdAnalyzeActiveElements();
#endif
        break;

#ifdef USE_RTC_TIME
    case MSP_SET_RTC:
        {
            // Use seconds and milliseconds to make senders
            // easier to implement. Generating a 64 bit value
            // might not be trivial in some platforms.
            int32_t secs = (int32_t)sbufReadU32(src);
            uint16_t millis = sbufReadU16(src);
            rtcTime_t t = rtcTimeMake(secs, millis);
            rtcSet(&t);
        }

        break;
#endif

    case MSP_SET_TX_INFO:
        setRssiMsp(sbufReadU8(src));

        break;

#if defined(USE_BOARD_INFO)
    case MSP_SET_BOARD_INFO:
        if (!boardInformationIsSet()) {
            uint8_t length = sbufReadU8(src);
            char boardName[MAX_BOARD_NAME_LENGTH + 1];
            sbufReadData(src, boardName, MIN(length, MAX_BOARD_NAME_LENGTH));
            if (length > MAX_BOARD_NAME_LENGTH) {
                sbufAdvance(src, length - MAX_BOARD_NAME_LENGTH);
                length = MAX_BOARD_NAME_LENGTH;
            }
            boardName[length] = '\0';
            length = sbufReadU8(src);
            char manufacturerId[MAX_MANUFACTURER_ID_LENGTH + 1];
            sbufReadData(src, manufacturerId, MIN(length, MAX_MANUFACTURER_ID_LENGTH));
            if (length > MAX_MANUFACTURER_ID_LENGTH) {
                sbufAdvance(src, length - MAX_MANUFACTURER_ID_LENGTH);
                length = MAX_MANUFACTURER_ID_LENGTH;
            }
            manufacturerId[length] = '\0';

            setBoardName(boardName);
            setManufacturerId(manufacturerId);
            persistBoardInformation();
        } else {
            return MSP_RESULT_ERROR;
        }

        break;
#if defined(USE_SIGNATURE)
    case MSP_SET_SIGNATURE:
        if (!signatureIsSet()) {
            uint8_t signature[SIGNATURE_LENGTH];
            sbufReadData(src, signature, SIGNATURE_LENGTH);
            setSignature(signature);
            persistSignature();
        } else {
            return MSP_RESULT_ERROR;
        }

        break;
#endif
#endif // USE_BOARD_INFO
#if defined(USE_RX_BIND)
    case MSP2_BETAFLIGHT_BIND:
        if (!startRxBind()) {
            return MSP_RESULT_ERROR;
        }

        break;
#endif

    case MSP2_SET_TEXT:
        {
            // type byte, then length byte followed by the actual characters
            const unsigned textType = sbufReadU8(src);

            char* textVar;
            unsigned textSpace;
            switch (textType) {
                case MSP2TEXT_PILOT_NAME:
                    textVar = pilotConfigMutable()->pilotName;
                    textSpace = sizeof(pilotConfigMutable()->pilotName) - 1;
                    break;

                case MSP2TEXT_CRAFT_NAME:
                    textVar = pilotConfigMutable()->craftName;
                    textSpace = sizeof(pilotConfigMutable()->craftName) - 1;
                    break;

                case MSP2TEXT_PID_PROFILE_NAME:
                    textVar = currentPidProfile->profileName;
                    textSpace = sizeof(currentPidProfile->profileName) - 1;
                    break;

                case MSP2TEXT_RATE_PROFILE_NAME:
                    textVar = currentControlRateProfile->profileName;
                    textSpace = sizeof(currentControlRateProfile->profileName) - 1;
                    break;

                case MSP2TEXT_CUSTOM_MSG_0:
                case MSP2TEXT_CUSTOM_MSG_0 + 1:
                case MSP2TEXT_CUSTOM_MSG_0 + 2:
                case MSP2TEXT_CUSTOM_MSG_0 + 3: {
                    unsigned msgIdx = textType - MSP2TEXT_CUSTOM_MSG_0;
                    if (msgIdx < OSD_CUSTOM_MSG_COUNT) {
                        textVar = pilotConfigMutable()->message[msgIdx];
                        textSpace = sizeof(pilotConfigMutable()->message[msgIdx]) - 1;
                    } else {
                        return MSP_RESULT_ERROR;
                    }
                    break;
                }
                default:
                    return MSP_RESULT_ERROR;
            }

            const unsigned textLength = MIN(textSpace, sbufReadU8(src));
            textVar[textLength] = '\0';
            sbufReadData(src, textVar, textLength);

#ifdef USE_OSD
            if (textType == MSP2TEXT_PILOT_NAME || textType == MSP2TEXT_CRAFT_NAME) {
                osdAnalyzeActiveElements();
            }
#endif
        }
        break;

#ifdef USE_LED_STRIP
    case MSP2_SET_LED_STRIP_CONFIG_VALUES:
        ledStripConfigMutable()->ledstrip_brightness = sbufReadU8(src);
        ledStripConfigMutable()->ledstrip_rainbow_delta = sbufReadU16(src);
        ledStripConfigMutable()->ledstrip_rainbow_freq = sbufReadU16(src);
        break;
#endif

    default:
        // we do not know how to handle the (valid) message, indicate error MSP $M!
        return MSP_RESULT_ERROR;
    }
    return MSP_RESULT_ACK;
}
```

---

</SwmSnippet>

### Returning MSP Command Results to the Caller

<SwmSnippet path="/src/main/msp/msp.c" line="4447">

---

We store and return the command result from <SwmToken path="src/main/telemetry/msp_shared.c" pos="152:4:4" line-data="    if (mspFcProcessCommand(mspSharedDescriptor, &amp;requestPacket, &amp;responsePacket, &amp;mspPostProcessFn) == MSP_RESULT_ERROR) {">`mspFcProcessCommand`</SwmToken> so the caller knows if it worked.

```c
    reply->result = ret;
    return ret;
}
```

---

</SwmSnippet>

## Finalizing the MSP Packet and Running Post-Processing

<SwmSnippet path="/src/main/telemetry/msp_shared.c" line="155">

---

We run any post-processing and prep the response buffer for sending after handling the MSP command.

```c
    if (mspPostProcessFn) {
        mspPostProcessFn(NULL);
    }

    sbufSwitchToReader(&responsePacket.buf, responseBuffer);
}
```

---

</SwmSnippet>

&nbsp;

*This is an auto-generated document by Swimm 🌊 and has not yet been verified by a human*

<SwmMeta version="3.0.0" repo-id="Z2l0aHViJTNBJTNBYy1iZXRhZmxpZ2h0JTNBJTNBcmljYXJkb2xvcGV6Zw==" repo-name="c-betaflight"><sup>Powered by [Swimm](https://app.swimm.io/)</sup></SwmMeta>
