# esp-dac.pretty — Custom Footprint Library

This directory is the placeholder for project-specific footprints that are not
available in the standard KiCad libraries.

## When to add footprints here

Add a custom footprint to this library when:
- The exact component is not found in `KiCad_Footprints` (the standard library)
- A vendor-supplied footprint needs local modification (land pattern, courtyard)
- A new connector or mechanical part has a unique pinout or body

## Standard footprints to use from KiCad libraries

Many components in this project have direct matches in the standard libraries:

| Component | Suggested KiCad Library:Footprint |
|-----------|----------------------------------|
| TAS5825M HTSSOP-32 | `Package_SO:HTSSOP-32-1EP_6.1x11mm_P0.65mm_EP3.4x5mm_ThermalVias` |
| TAS5825M VQFN-32 | `Package_QFN:VQFN-32-1EP_5x5mm_P0.5mm_EP3.4x3.4mm_ThermalVias` |
| ESP32-S3-WROOM-1 | `RF_Module:ESP32-S3-WROOM-1` |
| M3 Mounting hole | `MountingHole:MountingHole_3.2mm_M3_Pad` |
| USB-C receptacle | `Connector_USB:USB_C_Receptacle_GCT_USB4135-GF-A` *(verify)* |
| microSD push-push | `Connector_Card:microSD_HC_Hirose_DM3D-SF` *(verify)* |
| Screw terminal 5mm | `TerminalBlock_Phoenix:TerminalBlock_Phoenix_PT-1,5-2-5.0-H_1x02_P5.00mm_Horizontal` |

> Always verify footprint land patterns against the actual component datasheet
> before ordering PCBs.

## Footprint naming convention

```
esp-dac:<RefDes>_<ManufacturerPN>_<Package>
```

Example: `esp-dac:U2_TAS5825M_VQFN32`
