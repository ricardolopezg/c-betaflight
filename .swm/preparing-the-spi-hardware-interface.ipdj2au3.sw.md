---
title: Preparing the SPI Hardware Interface
---
This document describes how the SPI hardware interface is prepared for communication with external devices. The process ensures the SPI peripheral is powered, reset, and configured for reliable operation with connected hardware.

# Enabling and Resetting SPI Peripheral and Configuring GPIO

<SwmSnippet path="/src/platform/STM32/bus_spi_stdperiph.c" line="79">

---

In <SwmToken path="src/platform/STM32/bus_spi_stdperiph.c" pos="79:2:2" line-data="void spiInitDevice(spiDevice_e device)">`spiInitDevice`</SwmToken>, we start by checking if the SPI device is valid, then enable its clock using <SwmToken path="src/platform/STM32/bus_spi_stdperiph.c" pos="88:1:1" line-data="    RCC_ClockCmd(spi-&gt;rcc, ENABLE);">`RCC_ClockCmd`</SwmToken>. This is needed so the SPI hardware is powered up and ready for further configuration. Next, we need to call into <SwmToken path="src/platform/STM32/bus_spi_stdperiph.c" pos="88:1:1" line-data="    RCC_ClockCmd(spi-&gt;rcc, ENABLE);">`RCC_ClockCmd`</SwmToken> in <SwmPath>[src/…/STM32/rcc_stm32.c](src/platform/STM32/rcc_stm32.c)</SwmPath> to actually manipulate the hardware registers and turn on the clock for the SPI peripheral.

```c
void spiInitDevice(spiDevice_e device)
{
    spiDevice_t *spi = &(spiDevice[device]);

    if (!spi->dev) {
        return;
    }

    // Enable SPI clock
    RCC_ClockCmd(spi->rcc, ENABLE);
```

---

</SwmSnippet>

<SwmSnippet path="/src/platform/STM32/rcc_stm32.c" line="24">

---

<SwmToken path="src/platform/STM32/rcc_stm32.c" pos="24:2:2" line-data="void RCC_ClockCmd(rccPeriphTag_t periphTag, FunctionalState NewState)">`RCC_ClockCmd`</SwmToken> decodes the peripheral tag to figure out which bus and bit to manipulate, then uses macros to set the enable bit in the right RCC register. It handles different STM32 families and drivers using conditional compilation, and the suffix macro lets it pick the right register variant for each MCU type.

```c
void RCC_ClockCmd(rccPeriphTag_t periphTag, FunctionalState NewState)
{
    int tag = periphTag >> 5;
    uint32_t mask = 1 << (periphTag & 0x1f);

#if defined(USE_HAL_DRIVER)

// Note on "suffix" macro parameter:
// ENR and RSTR naming conventions for buses with multiple registers per bus differs among MCU types.
// ST decided to use AxBn{L,H}ENR convention for H7 which can be handled with simple "ENR" (or "RSTR") contatenation,
// while use AxBnENR{1,2} convention for G4 which requires extra "suffix" to be concatenated.
// Here, we use "suffix" for all MCU types and leave it as empty where not applicable.

#define NOSUFFIX // Empty

#define __HAL_RCC_CLK_ENABLE(bus, suffix, enbit)   do {      \
        __IO uint32_t tmpreg;                                \
        SET_BIT(RCC->bus ## ENR ## suffix, enbit);           \
        /* Delay after an RCC peripheral clock enabling */   \
        tmpreg = READ_BIT(RCC->bus ## ENR ## suffix, enbit); \
        UNUSED(tmpreg);                                      \
    } while(0)

#define __HAL_RCC_CLK_DISABLE(bus, suffix, enbit) (RCC->bus ## ENR ## suffix &= ~(enbit))

#define __HAL_RCC_CLK(bus, suffix, enbit, newState) \
    if (newState == ENABLE) {                       \
        __HAL_RCC_CLK_ENABLE(bus, suffix, enbit);   \
    } else {                                        \
        __HAL_RCC_CLK_DISABLE(bus, suffix, enbit);  \
    }

    switch (tag) {
    case RCC_AHB1:
        __HAL_RCC_CLK(AHB1, NOSUFFIX, mask, NewState);
        break;

    case RCC_AHB2:
        __HAL_RCC_CLK(AHB2, NOSUFFIX, mask, NewState);
        break;

#if !(defined(STM32H7) || defined(STM32G4))
    case RCC_APB1:
        __HAL_RCC_CLK(APB1, NOSUFFIX, mask, NewState);
        break;
#endif

    case RCC_APB2:
        __HAL_RCC_CLK(APB2, NOSUFFIX, mask, NewState);
        break;

#ifdef STM32H7

    case RCC_AHB3:
        __HAL_RCC_CLK(AHB3, NOSUFFIX, mask, NewState);
        break;

    case RCC_AHB4:
        __HAL_RCC_CLK(AHB4, NOSUFFIX, mask, NewState);
        break;

    case RCC_APB1L:
        __HAL_RCC_CLK(APB1L, NOSUFFIX, mask, NewState);
        break;

    case RCC_APB1H:
        __HAL_RCC_CLK(APB1H, NOSUFFIX, mask, NewState);
        break;

    case RCC_APB3:
        __HAL_RCC_CLK(APB3, NOSUFFIX, mask, NewState);
        break;

    case RCC_APB4:
        __HAL_RCC_CLK(APB4, NOSUFFIX, mask, NewState);
        break;
#endif

#ifdef STM32G4

    case RCC_APB11:
        __HAL_RCC_CLK(APB1, 1, mask, NewState);
        break;

    case RCC_APB12:
        __HAL_RCC_CLK(APB1, 2, mask, NewState);
        break;

    case RCC_AHB3:
        __HAL_RCC_CLK(AHB3, NOSUFFIX, mask, NewState);
        break;

#endif
    }
#elif defined(USE_ATBSP_DRIVER)

#define NOSUFFIX // Empty

#define __AT_RCC_CLK_ENABLE(bus, suffix, enbit)   do {      \
        __IO uint32_t tmpreg;                               \
        SET_BIT(CRM->bus ## en ## suffix, enbit);           \
        /* Delay after an RCC peripheral clock enabling */  \
        tmpreg = READ_BIT(CRM->bus ## en ## suffix, enbit); \
        UNUSED(tmpreg);                                     \
    } while(0)

#define __AT_RCC_CLK_DISABLE(bus, suffix, enbit) (CRM->bus ## en ## suffix &= ~(enbit))

#define __AT_RCC_CLK(bus, suffix, enbit, newState) \
    if (newState == ENABLE) {                      \
        __AT_RCC_CLK_ENABLE(bus, suffix, enbit);   \
    } else {                                       \
        __AT_RCC_CLK_DISABLE(bus, suffix, enbit);  \
    }

    switch (tag) {
    case RCC_AHB1:
        __AT_RCC_CLK(ahb, 1, mask, NewState);
        break;
    case RCC_AHB2:
        __AT_RCC_CLK(ahb, 2, mask, NewState);
        break;
    case RCC_AHB3:
        __AT_RCC_CLK(ahb, 3, mask, NewState);
        break;
    case RCC_APB1:
        __AT_RCC_CLK(apb1, NOSUFFIX, mask, NewState);
        break;
    case RCC_APB2:
        __AT_RCC_CLK(apb2, NOSUFFIX, mask, NewState);
        break;
    }
#else
    switch (tag) {
    case RCC_APB2:
        RCC_APB2PeriphClockCmd(mask, NewState);
        break;
    case RCC_APB1:
        RCC_APB1PeriphClockCmd(mask, NewState);
        break;
#if defined(STM32F4)
    case RCC_AHB1:
        RCC_AHB1PeriphClockCmd(mask, NewState);
        break;
#endif
    }
#endif
}
```

---

</SwmSnippet>

<SwmSnippet path="/src/platform/STM32/bus_spi_stdperiph.c" line="89">

---

Back in <SwmToken path="src/platform/STM32/bus_spi_stdperiph.c" pos="79:2:2" line-data="void spiInitDevice(spiDevice_e device)">`spiInitDevice`</SwmToken>, after enabling the SPI clock, we immediately reset the peripheral with <SwmToken path="src/platform/STM32/bus_spi_stdperiph.c" pos="89:1:1" line-data="    RCC_ResetCmd(spi-&gt;rcc, ENABLE);">`RCC_ResetCmd`</SwmToken>. This clears any previous state and makes sure the hardware starts clean. Next, we call into <SwmToken path="src/platform/STM32/bus_spi_stdperiph.c" pos="89:1:1" line-data="    RCC_ResetCmd(spi-&gt;rcc, ENABLE);">`RCC_ResetCmd`</SwmToken> in <SwmPath>[src/…/STM32/rcc_stm32.c](src/platform/STM32/rcc_stm32.c)</SwmPath> to actually trigger the hardware reset.

```c
    RCC_ResetCmd(spi->rcc, ENABLE);

```

---

</SwmSnippet>

<SwmSnippet path="/src/platform/STM32/rcc_stm32.c" line="173">

---

<SwmToken path="src/platform/STM32/rcc_stm32.c" pos="173:2:2" line-data="void RCC_ResetCmd(rccPeriphTag_t periphTag, FunctionalState NewState)">`RCC_ResetCmd`</SwmToken> decodes the peripheral tag to select the right bus and bit, then uses macros to force or release the reset on the peripheral. It handles different STM32 families and drivers with conditional compilation, so the reset logic works across hardware variants.

```c
void RCC_ResetCmd(rccPeriphTag_t periphTag, FunctionalState NewState)
{
    int tag = periphTag >> 5;
    uint32_t mask = 1 << (periphTag & 0x1f);

// Peripheral reset control relies on RSTR bits are identical to ENR bits where applicable

#define __HAL_RCC_FORCE_RESET(bus, suffix, enbit) (RCC->bus ## RSTR ## suffix |= (enbit))
#define __HAL_RCC_RELEASE_RESET(bus, suffix, enbit) (RCC->bus ## RSTR ## suffix &= ~(enbit))
#define __HAL_RCC_RESET(bus, suffix, enbit, NewState) \
    if (NewState == ENABLE) {                         \
        __HAL_RCC_RELEASE_RESET(bus, suffix, enbit);  \
    } else {                                          \
        __HAL_RCC_FORCE_RESET(bus, suffix, enbit);    \
    }

#if defined(USE_HAL_DRIVER)

    switch (tag) {
    case RCC_AHB1:
        __HAL_RCC_RESET(AHB1, NOSUFFIX, mask, NewState);
        break;

    case RCC_AHB2:
        __HAL_RCC_RESET(AHB2, NOSUFFIX, mask, NewState);
        break;

#if !(defined(STM32H7) || defined(STM32G4))
    case RCC_APB1:
        __HAL_RCC_RESET(APB1, NOSUFFIX, mask, NewState);
        break;
#endif

    case RCC_APB2:
        __HAL_RCC_RESET(APB2, NOSUFFIX, mask, NewState);
        break;

#ifdef STM32H7

    case RCC_AHB3:
        __HAL_RCC_RESET(AHB3, NOSUFFIX, mask, NewState);
        break;

    case RCC_AHB4:
        __HAL_RCC_RESET(AHB4, NOSUFFIX, mask, NewState);
        break;

    case RCC_APB1L:
        __HAL_RCC_RESET(APB1L, NOSUFFIX, mask, NewState);
        break;

    case RCC_APB1H:
        __HAL_RCC_RESET(APB1H, NOSUFFIX, mask, NewState);
        break;

    case RCC_APB3:
        __HAL_RCC_RESET(APB3, NOSUFFIX, mask, NewState);
        break;

    case RCC_APB4:
        __HAL_RCC_RESET(APB4, NOSUFFIX, mask, NewState);
        break;
#endif

#ifdef STM32G4

    case RCC_APB11:
        __HAL_RCC_RESET(APB1, 1, mask, NewState);
        break;

    case RCC_APB12:
        __HAL_RCC_RESET(APB1, 2, mask, NewState);
        break;

    case RCC_AHB3:
        __HAL_RCC_RESET(AHB3, NOSUFFIX, mask, NewState);
        break;

#endif
    }
#elif defined(USE_ATBSP_DRIVER)
#define __AT_RCC_FORCE_RESET(bus, suffix, enbit) (CRM->bus ## rst ## suffix |= (enbit))
#define __AT_RCC_RELEASE_RESET(bus, suffix, enbit) (CRM->bus ## rst ## suffix &= ~(enbit))
#define __AT_RCC_RESET(bus, suffix, enbit, NewState) \
    if (NewState == ENABLE) {                        \
        __AT_RCC_RELEASE_RESET(bus, suffix, enbit);  \
    } else {                                         \
        __AT_RCC_FORCE_RESET(bus, suffix, enbit);    \
    }

    switch (tag) {
    case RCC_AHB1:
        __AT_RCC_RESET(ahb, 1, mask, NewState);
        break;
    case RCC_AHB2:
        __AT_RCC_RESET(ahb, 2, mask, NewState);
        break;
    case RCC_AHB3:
        __AT_RCC_RESET(ahb, 3, mask, NewState);
        break;
    case RCC_APB1:
        __AT_RCC_RESET(apb1, NOSUFFIX, mask, NewState);
        break;
    case RCC_APB2:
        __AT_RCC_RESET(apb2, NOSUFFIX, mask, NewState);
        break;
    }
#else

    switch (tag) {
    case RCC_APB2:
        RCC_APB2PeriphResetCmd(mask, NewState);
        break;
    case RCC_APB1:
        RCC_APB1PeriphResetCmd(mask, NewState);
        break;
#if defined(STM32F4)
    case RCC_AHB1:
        RCC_AHB1PeriphResetCmd(mask, NewState);
        break;
#endif
    }
#endif
}
```

---

</SwmSnippet>

<SwmSnippet path="/src/platform/STM32/bus_spi_stdperiph.c" line="91">

---

Back in <SwmToken path="src/platform/STM32/bus_spi_stdperiph.c" pos="79:2:2" line-data="void spiInitDevice(spiDevice_e device)">`spiInitDevice`</SwmToken>, after resetting the SPI peripheral, we set up the SCK, MISO, and MOSI pins using <SwmToken path="src/platform/STM32/bus_spi_stdperiph.c" pos="91:1:1" line-data="    IOInit(IOGetByTag(spi-&gt;sck),  OWNER_SPI_SCK,  RESOURCE_INDEX(device));">`IOInit`</SwmToken> and <SwmToken path="src/platform/STM32/bus_spi_stdperiph.c" pos="95:1:1" line-data="    IOConfigGPIOAF(IOGetByTag(spi-&gt;sck),  SPI_IO_AF_SCK_CFG, spi-&gt;af);">`IOConfigGPIOAF`</SwmToken>. This maps the pins to the SPI hardware and configures them for the right alternate function, so the peripheral can actually use them for communication. Next, we call into <SwmToken path="src/platform/STM32/bus_spi_stdperiph.c" pos="95:1:1" line-data="    IOConfigGPIOAF(IOGetByTag(spi-&gt;sck),  SPI_IO_AF_SCK_CFG, spi-&gt;af);">`IOConfigGPIOAF`</SwmToken> in <SwmPath>[src/…/STM32/io_stm32.c](src/platform/STM32/io_stm32.c)</SwmPath> to handle the pin setup.

```c
    IOInit(IOGetByTag(spi->sck),  OWNER_SPI_SCK,  RESOURCE_INDEX(device));
    IOInit(IOGetByTag(spi->miso), OWNER_SPI_SDI, RESOURCE_INDEX(device));
    IOInit(IOGetByTag(spi->mosi), OWNER_SPI_SDO, RESOURCE_INDEX(device));

    IOConfigGPIOAF(IOGetByTag(spi->sck),  SPI_IO_AF_SCK_CFG, spi->af);
    IOConfigGPIOAF(IOGetByTag(spi->miso), SPI_IO_AF_SDI_CFG, spi->af);
    IOConfigGPIOAF(IOGetByTag(spi->mosi), SPI_IO_AF_CFG, spi->af);

```

---

</SwmSnippet>

<SwmSnippet path="/src/platform/STM32/io_stm32.c" line="199">

---

<SwmToken path="src/platform/STM32/io_stm32.c" pos="199:2:2" line-data="void IOConfigGPIOAF(IO_t io, ioConfig_t cfg, uint8_t af)">`IOConfigGPIOAF`</SwmToken> uses <SwmToken path="src/platform/STM32/io_stm32.c" pos="205:7:7" line-data="    rccPeriphTag_t rcc = ioPortDefs[IO_GPIOPortIdx(io)].rcc;">`ioPortDefs`</SwmToken> and <SwmToken path="src/platform/STM32/io_stm32.c" pos="205:9:9" line-data="    rccPeriphTag_t rcc = ioPortDefs[IO_GPIOPortIdx(io)].rcc;">`IO_GPIOPortIdx`</SwmToken> to figure out which GPIO port and clock to enable, then extracts Mode, Speed, and Pull from the cfg bitfield to set up the pin. It assumes the IO and cfg parameters are valid and encodes all the settings needed for the alternate function.

```c
void IOConfigGPIOAF(IO_t io, ioConfig_t cfg, uint8_t af)
{
    if (!io) {
        return;
    }

    rccPeriphTag_t rcc = ioPortDefs[IO_GPIOPortIdx(io)].rcc;
    RCC_ClockCmd(rcc, ENABLE);

    GPIO_InitTypeDef init = {
        .Pin = IO_Pin(io),
        .Mode = (cfg >> 0) & 0x13,
        .Speed = (cfg >> 2) & 0x03,
        .Pull = (cfg >> 5) & 0x03,
        .Alternate = af
    };

    HAL_GPIO_Init(IO_GPIO(io), &init);
}
```

---

</SwmSnippet>

<SwmSnippet path="/src/platform/STM32/bus_spi_stdperiph.c" line="99">

---

After setting up the GPIOs in <SwmToken path="src/platform/STM32/bus_spi_stdperiph.c" pos="79:2:2" line-data="void spiInitDevice(spiDevice_e device)">`spiInitDevice`</SwmToken>, we deinit the SPI hardware, disable DMA requests, reinitialize with default settings, and finally enable the SPI. This makes sure the peripheral is clean and ready for use with the new pin configuration.

```c
    // Init SPI hardware
    SPI_I2S_DeInit(spi->dev);

    SPI_I2S_DMACmd(spi->dev, SPI_I2S_DMAReq_Tx | SPI_I2S_DMAReq_Rx, DISABLE);
    SPI_Init(spi->dev, &defaultInit);
    SPI_Cmd(spi->dev, ENABLE);
}
```

---

</SwmSnippet>

&nbsp;

*This is an auto-generated document by Swimm 🌊 and has not yet been verified by a human*

<SwmMeta version="3.0.0" repo-id="Z2l0aHViJTNBJTNBYy1iZXRhZmxpZ2h0JTNBJTNBcmljYXJkb2xvcGV6Zw==" repo-name="c-betaflight"><sup>Powered by [Swimm](https://app.swimm.io/)</sup></SwmMeta>
