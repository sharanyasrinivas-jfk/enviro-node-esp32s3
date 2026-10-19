# Design Notes — IoT Environmental Sensor Board Rev 2.1

## Revision History

| Rev | Date | Changes |
|-----|------|---------|
| 1.0 | 2025-11-20 | Initial design, 2-layer, AMS1117-3.3 regulator |
| 2.0 | 2026-02-08 | Migrated to 4-layer, replaced AMS1117 with AP2112K, added CCS811 |
| 2.1 | 2026-04-15 | Fixed SHT40 footprint pad spacing, improved ground plane stitching |

## Component Selection Rationale

### MCU — ESP32-S3-WROOM-1-N8R2
Selected for native USB support (eliminating CP2102 bridge), dual-core 240MHz, and integrated antenna. The N8R2 variant provides 8MB flash + 2MB PSRAM, sufficient for OTA firmware + MQTT buffering. BLE 5.0 enables low-power mesh networking for multi-node deployments.

**Rejected alternatives:**
- STM32WB55: Lower power, but no Wi-Fi — requires external module
- nRF52840: Excellent BLE but no native Wi-Fi
- ESP32-C3: Single-core RISC-V insufficient for concurrent sensor polling + Wi-Fi

### LDO — AP2112K-3.3
The AP2112K was chosen over the ubiquitous AMS1117-3.3 for several reasons:
- **Lower dropout:** 250mV vs 1.0V → operates from 3.55V input (critical for USB under-voltage)
- **Lower quiescent current:** 55µA vs 5mA → meaningful for battery-backed applications
- **Smaller footprint:** SOT-23-5 vs SOT-223
- **Stable with ceramic output caps:** No ESR requirements (AMS1117 needs tantalum)

Thermal calculation at worst case (5V → 3.3V at 85mA):
```
P_diss = (5.0 - 3.3) × 0.085 = 144.5mW
T_junction = 25°C + (144.5mW × 260°C/W) = 62.6°C  ← well under 125°C max
```

### Sensor Bus Architecture

All three sensors share a single I2C bus at 400kHz. Address map:

| Sensor | I2C Address | Pull-up |
|--------|-------------|---------|
| BMP280 | 0x76 (SDO→GND) | Shared 4.7kΩ |
| SHT40 | 0x44 (fixed) | Shared 4.7kΩ |
| CCS811 | 0x5A (ADDR→GND) | Shared 4.7kΩ |

Pull-up value derived from I2C spec for 400kHz fast mode:
```
R_pullup = t_rise / (0.8473 × C_bus)
C_bus ≈ 25pF (3 devices × ~5pF + 10pF trace)
t_rise ≤ 300ns (fast mode spec)
R_pullup = 300ns / (0.8473 × 25pF) = 14.2kΩ (max)
R_pullup_min = (VDD - VOL) / IOL = (3.3 - 0.4) / 3mA = 967Ω (min)
Selected: 4.7kΩ — safe middle range, 0.7mA sink current
```

## PCB Layout Decisions

### Layer Stackup
Moved from 2-layer (Rev 1.0) to 4-layer to achieve:
1. Unbroken ground plane under all signal traces (EMI reduction)
2. Dedicated power plane for clean 3.3V distribution
3. Reduced loop area for I2C traces (signal on L1/L4, return on L2 GND)
4. Better thermal dissipation for LDO through via array to inner GND

### CCS811 Placement
The CCS811 VOC sensor requires careful placement:
- Positioned at board edge with gas inlet holes in soldermask
- Analog supply decoupled with dedicated 100nF capacitor within 2mm
- Via-stitched ground guard ring around sensor footprint
- Keepout zone: no copper pour within 1mm of gas inlet

### Thermal Relief
ESP32-S3 GND pad connected to inner ground plane with 4× 0.3mm thermal relief vias arranged in a 2×2 grid directly under the module's ground paddle.

## Known Issues

1. **SHT40 cross-sensitivity:** Self-heating from nearby ESP32 can bias humidity readings +2-3%RH. Mitigated by routing SHT40 at board edge, opposite side from MCU.
2. **CCS811 burn-in:** Requires 48-hour continuous operation for baseline calibration. Firmware handles automatic baseline save/restore via NVS.
3. **USB-C orientation:** Only CC1 resistor populated — does not detect cable flip. Acceptable for power-only USB (no USB data).

## Rev 2.1 Change Log
- Fixed SHT40 footprint: pad spacing corrected from 0.5mm to 0.65mm per datasheet errata
