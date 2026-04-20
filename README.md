# M10 — u-blox M10 GNSS Driver

A generic, portable **C driver** for the **u-blox M10** GNSS receiver family. Communication is handled over **UART** using the proprietary **UBX binary protocol**, built on top of the `ubx` component.

---

## Features

- Full UBX protocol configuration via `CFG-VALSET` key-value pairs
- Configurable GNSS constellations: **GPS, Galileo, BeiDou, QZSS**
- Configurable navigation update rate: **1 – 18 Hz**
- Selectable navigation model (portable, pedestrian, automotive, airborne, etc.)
- Three power modes: **Full power**, **PSM ON/OFF (PSMOO)**, **PSM Cyclic Tracking (PSMCT)**
- Configurable UBX and NMEA output messages per-message
- Configuration layer targeting: **RAM**, **BBR**, **Flash**
- Device status polling (fix type, fixOk, TTFF, MSSS)
- Hardware / software / GNSS-only **reset** control
- Baud rate negotiation
- **MGA (AssistNow)** support:
  - Set UTC reference time
  - Export navigation database (`UBX-MGA-DBD`)
  - Import navigation database (from export or AssistNow servers)

---

## Dependencies

| Component | Purpose |
|-----------|---------|
| [`ubx`](https://github.com/<your-repo>/ubx) | UBX framing, UART transport, message pool |

The driver has no platform-specific dependencies. It relies on the `ubx` component for all hardware abstraction (UART read/write, tick timer). Adapt `ubx` to your target platform/RTOS.

---

## Public API

### Initialization

```c
// 1. Configure hm10.hubx (UART port, baud rate, callbacks, …)
M10_ErrorTypeDef M10_InitUART(M10_HandleTypeDef *hm10);

// 2. Apply DeviceConfig and start GNSS
M10_ErrorTypeDef M10_Init(M10_HandleTypeDef *hm10);
```

`M10_InitUART()` must be called **before** `M10_Init()`.

### Configuration

```c
// Send an arbitrary array of CFG-VALSET key-value pairs
M10_ErrorTypeDef M10_SendConfig(
    M10_HandleTypeDef      *hm10,
    M10_ConfigDataTypeDef  *CfgData,
    uint32_t                CfgDataLen,
    uint8_t                 Layers,     // M10_CONFIG_LAYER_RAM | BBR | FLASH
    uint8_t                 SkipAck);
```

### Status / Info

```c
M10_ErrorTypeDef M10_GetStatus (M10_HandleTypeDef *hm10, M10_DeviceStatusTypeDef  *Status);
M10_ErrorTypeDef M10_GetVersion(M10_HandleTypeDef *hm10, M10_DeviceVersionTypeDef *Version);
uint8_t          M10_HasValidFix(M10_HandleTypeDef *hm10);   // 1 = 3D fix + fixOk
```

### Controls

```c
M10_ErrorTypeDef M10_Reset    (M10_HandleTypeDef *hm10, M10_NavBbrMaskTypeDef BbrMask, M10_ResetModeTypeDef ResetMode);
M10_ErrorTypeDef M10_GnssStop (M10_HandleTypeDef *hm10);
M10_ErrorTypeDef M10_GnssStart(M10_HandleTypeDef *hm10);
M10_ErrorTypeDef M10_SetBaudRate(M10_HandleTypeDef *hm10, UBX_BaudRateTypeDef BaudRate, uint8_t Layers);
```

### MGA (AssistNow)

```c
// Provide current UTC time to the receiver
M10_ErrorTypeDef M10_SetUTC(M10_HandleTypeDef *hm10,
    uint64_t TimestampMs, uint16_t SAccuracy, uint32_t NSAccuracy, uint32_t TimeoutMs);

// Export the receiver's navigation database (e.g. to save in flash)
M10_ErrorTypeDef M10_ExportNavData(M10_HandleTypeDef *hm10,
    uint8_t (*HandleDataMessage)(uint8_t *ChunkContent, uint32_t Len),
    uint32_t *DataLen, uint32_t TimeoutMs);

// Import a previously exported (or AssistNow) navigation database
M10_ErrorTypeDef M10_ImportNavData(M10_HandleTypeDef *hm10,
    uint8_t *Data, uint32_t DataLen, uint32_t TimeoutMs);
```

### ISR / UART RX callback

```c
// Call this from the UART receive task/ISR whenever a complete message arrives
void M10_SignalMessageReceived(M10_HandleTypeDef *hm10,
    M10_MessageTypeTypeDef MessageType, void *Payload);
```

---

## Data Structures

### `M10_HandleTypeDef`
| Field          | Type                  | Description                          |
|----------------|-----------------------|--------------------------------------|
| `hubx`         | `UBX_HandleTypeDef`   | Underlying UBX transport handle      |
| `DeviceConfig` | `M10_ConfigTypeDef`   | Device configuration (see below)     |

### `M10_ConfigTypeDef`
| Field                       | Type                            | Description |
|-----------------------------|---------------------------------|-------------|
| `UpdateRate`                | `M10_UpdateRateTypeDef`         | 1 – 18 Hz |
| `Constellations`            | `uint8_t`                       | Bitmask: `M10_CONSTELLATION_GPS \| GALILEO \| BEIDOU \| QZSS` |
| `UBXOutputMessages`         | `uint64_t`                      | Bitmask of UBX messages to enable on UART |
| `NMEAOutputMessages`        | `uint32_t`                      | Bitmask of NMEA sentences to enable on UART |
| `PowerConfiguration`        | `M10_PowerConfigurationTypeDef` | `FULL`, `PSMOO`, or `PSMCT` |
| `PositionUpdatePeriodSeconds` | `uint32_t`                    | Sleep period for PSMOO mode (5 s … 1 week) |
| `NavModel`                  | `M10_NavModelTypeDef`           | Dynamic platform model |
| `PDOP`                      | `uint16_t`                      | Position DOP limit ×10 (default 250 = 25.0) |
| `TDOP`                      | `uint16_t`                      | Time DOP limit ×10 (default 250 = 25.0) |
| `ConfigLayers`              | `uint8_t`                       | Target layers for `M10_Init()` config writes |

### `M10_DeviceStatusTypeDef`
| Field      | Description |
|------------|-------------|
| `Fix`      | `M10_DeviceFixTypeDef`: NO_FIX / 2D / 3D (READY) / TIME_ONLY |
| `FixOk`    | 1 if navigation fix is valid |
| `WknValid` | GPS week number valid |
| `ToWValid` | GPS time-of-week valid |
| `Ttff`     | Time to first fix (ms) |
| `Msss`     | Milliseconds since last receiver start / GNSS restart |

---

## Error Codes (`M10_ErrorTypeDef`)

| Value | Meaning |
|-------|---------|
| `M10_ERROR_OK`                  | Success |
| `M10_ERROR_UBX`                 | Underlying UBX transport error |
| `M10_ERROR_UBX_PAYLOAD`         | No payload pool item available |
| `M10_ERROR_TIMEOUT`             | Operation timed out |
| `M10_ERROR_BAUD_RATE_NOT_FOUND` | Baud rate detection failed |
| `M10_ERROR_BAUD_RATE_UPDATE`    | Baud rate update failed |
| `M10_ERROR_CFG_INVALID_KEY`     | Unknown CFG-VALSET key |
| `M10_ERROR_INVALID_TIMESTAMP`   | Invalid Unix timestamp passed to `M10_SetUTC` |
| `M10_ERROR_MGA_NOACK`           | No MGA-ACK received within timeout |
| `M10_ERROR_MGA_NOT_ACCEPTED`    | MGA message rejected by receiver |
| `M10_ERROR_MGA_DATA_NOT_HANDLED`| Export callback returned non-zero |
| `M10_ERROR_INVALID_IMPORT_DATA` | Malformed import data (bad UBX header) |

---

## Minimal Usage Example

```c
#include "m10.h"

static M10_HandleTypeDef hm10 = {
    .hubx = {
        // fill in UBX handle fields (UART callbacks, tick callback, …)
    },
    .DeviceConfig = {
        .UpdateRate          = M10_URATE_1HZ,
        .Constellations      = M10_CONSTELLATION_GPS | M10_CONSTELLATION_GALILEO,
        .UBXOutputMessages   = M10_UBX_MSG_NAV_PVT,
        .NMEAOutputMessages  = 0,
        .PowerConfiguration  = M10_PWR_CFG_FULL,
        .NavModel            = M10_NAVMODEL_PORT,
        .PDOP                = 250,
        .TDOP                = 250,
        .ConfigLayers        = M10_CONFIG_LAYER_RAM,
    }
};

int main(void) {
    M10_InitUART(&hm10);
    M10_Init(&hm10);

    while (!M10_HasValidFix(&hm10)) {
        // wait / sleep
    }

    M10_DeviceStatusTypeDef status;
    M10_GetStatus(&hm10, &status);
    // status.Ttff contains time-to-first-fix in ms
}
```

---

## Reset Recipes

| Scenario | BbrMask | ResetMode |
|----------|---------|-----------|
| Normal soft reset | `M10_BBR_MSK_HOT_START` | `M10_RST_MODE_SW_RESET` |
| GNSS-only soft reset (UART stays active) | `M10_BBR_MSK_HOT_START` | `M10_RST_MODE_SW_GNSS_ONLY_RESET` |
| Force reset (unresponsive module) | `M10_BBR_MSK_HOT_START` | `M10_RST_MODE_HW_RESET` |
| Warm start (erase ephemeris) | `M10_BBR_MSK_WARM_START` | `M10_RST_MODE_SW_RESET` |
| Cold start (erase all BBR) | `M10_BBR_MSK_COLD_START` | `M10_RST_MODE_SW_RESET` |
| Inject AssistNow data | Stop: `HOT_START` / `GNSS_STOP` → import → `GNSS_START` | |

---
