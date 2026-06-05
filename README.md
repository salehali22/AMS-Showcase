# Autonomous Aerial Mapping System

![STM32H743](https://img.shields.io/badge/MCU-STM32H743-blue)
![ESP32-P4](https://img.shields.io/badge/Companion-ESP32--P4-green)
![KiCad](https://img.shields.io/badge/EDA-KiCad-orange)
![ArduPilot](https://img.shields.io/badge/Firmware-ArduPilot-red)
![FreeRTOS](https://img.shields.io/badge/RTOS-FreeRTOS-brightgreen)
![IPC-2221](https://img.shields.io/badge/Standard-IPC--2221-lightgrey)

Full-stack UAV avionics platform for autonomous aerial survey and photogrammetric 3D reconstruction. Two custom multilayer PCBs designed in KiCad, hand-soldered, and validated on a physical airframe: an STM32H743 flight controller with dual redundant IMUs and a custom ArduPilot port, paired with an ESP32-P4 companion computer for synchronized 12MP image capture and 2D LiDAR acquisition with real-time LTE downlink.

![System Overview](Renders/AU6.png)

## Overview

This project implements a complete UAV avionics stack built from scratch for aerial mapping and search-and-rescue applications. The system captures high-resolution geotagged imagery and LiDAR terrain profiles simultaneously during flight, fuses them with flight controller attitude and GPS data via MAVLink v2, and logs everything to SD card for post-flight photogrammetric 3D reconstruction.

The flight controller board handles all flight-critical functions: stabilization, navigation, RC control, and MAVLink telemetry output. The companion computer board operates independently on its own power rail, receiving only UART telemetry from the FC, and handles all payload data acquisition, sensor fusion, encoding, and logging.

Both boards conform to the 30.5mm drone stack mounting standard and draw power from a common 4S LiPo through a power distribution board, with fully independent voltage regulation on each board.

## Key Specifications

### Flight Controller (SALEHFC)

| Parameter | Value |
|---|---|
| MCU | STM32H743, ARM Cortex-M7 @ 480 MHz |
| IMU 1 | ICM-42688-P (primary) |
| IMU 2 | BMI270 (redundant) |
| Barometer | BMP280 |
| OSD | AT7456E |
| Flash Logging | W25Q128 (16 MB) |
| Firmware | ArduPilot / INAV (custom target) |
| Telemetry | MAVLink v2 over UART |
| Mounting | 30.5 x 30.5 mm |
| Design Rules | IPC-2221 |

### Companion Computer

| Parameter | Value |
|---|---|
| MCU | ESP32-P4, Dual-core RISC-V @ 400 MHz |
| Camera | IMX477 (12 MP) via MIPI CSI-2 |
| LiDAR | RPLidar A1 (8000 samples/sec, up to 10 Hz scan rate) |
| Encoding | Hardware H.264/JPEG (1080p @ 30fps, zero CPU load) |
| Cellular | A7670 LTE module |
| Storage | MicroSD (geotagged frames + LiDAR logs) |
| FC Link | MAVLink v2 over UART |
| Input Voltage | 5V from PDB |
| Mounting | 30.5 x 30.5 mm |
| Design Rules | IPC-2221 |

## System Architecture

```mermaid
graph TB
    subgraph Power["Power Distribution"]
        LIPO["4S LiPo Battery"]
        PDB["Power Distribution Board"]
        LIPO --> PDB
    end

    subgraph FC["Flight Controller (STM32H743)"]
        direction TB
        IMU1["ICM-42688-P<br/>Primary IMU"]
        IMU2["BMI270<br/>Redundant IMU"]
        BARO["BMP280<br/>Barometer"]
        GPS["GPS Module"]
        OSD["AT7456E OSD"]
        FLASH["W25Q128<br/>Flash Logger"]
        RC["RC Receiver"]
        MOTORS["ESC / Motors"]
    end

    subgraph COMP["Companion Computer (ESP32-P4)"]
        direction TB
        CAM["IMX477 12MP<br/>MIPI CSI-2"]
        LIDAR["RPLidar A1<br/>UART"]
        LTE["A7670<br/>LTE Module"]
        SD["MicroSD<br/>Fused Data Log"]
        ENCODER["Hardware H.264/JPEG<br/>Encoder"]
    end

    PDB -- "8V/12V + 5V + 3.3V" --> FC
    PDB -- "5V" --> COMP
    FC -- "MAVLink v2<br/>UART" --> COMP

    CAM --> ENCODER
    ENCODER --> SD
    LIDAR --> SD
```

## Data Fusion Pipeline

```mermaid
flowchart LR
    subgraph Inputs
        A["IMX477<br/>12MP MIPI CSI-2"]
        B["RPLidar A1<br/>UART @ 5.5 Hz"]
        C["MAVLink v2<br/>GPS + Attitude @ 10 Hz"]
    end

    subgraph ESP32-P4["ESP32-P4 Processing"]
        D["Hardware JPEG<br/>Encoder"]
        E["Core 0<br/>LiDAR Parse +<br/>MAVLink Parse"]
        F["Core 1<br/>Geotag + Fuse +<br/>SD Write"]
    end

    subgraph Output
        G["MicroSD Card"]
        H["LTE Downlink<br/>to Ground Station"]
    end

    A --> D --> F
    B --> E --> F
    C --> E
    F --> G
    D --> H
```

## Firmware Architecture

### Flight Controller

The STM32H743 runs a custom-built ArduPilot (or INAV) target with a board definition, bootloader, and HAL pin mapping written from scratch. Key functions:

- Sensor fusion across dual redundant IMUs with automatic failover
- GPS-guided autonomous waypoint navigation
- MAVLink v2 telemetry output to companion computer at 10 Hz
- RC override and failsafe handling
- OSD overlay and onboard flash logging

### Companion Computer

The ESP32-P4 firmware runs on both RISC-V cores with a clear task separation:

```mermaid
graph LR
    subgraph Core0["Core 0: Acquisition"]
        T1["LiDAR UART RX<br/>Parse scan frames"]
        T2["MAVLink UART RX<br/>Parse GPS + attitude"]
        T3["Timestamp sync"]
    end

    subgraph HW["Hardware Accelerator"]
        T4["JPEG/H.264 Encode<br/>12MP frames<br/>Zero CPU load"]
    end

    subgraph Core1["Core 1: Fusion + Storage"]
        T5["Geotag frames with<br/>position + attitude"]
        T6["Fuse LiDAR scan<br/>with GPS coordinates"]
        T7["Write to SD card"]
        T8["LTE stream to<br/>ground station"]
    end

    T1 --> T5
    T2 --> T5
    T2 --> T6
    T3 --> T5
    T4 --> T5
    T1 --> T6
    T6 --> T7
    T5 --> T7
    T4 --> T8
```

## Power Architecture

```mermaid
graph TD
    BAT["4S LiPo<br/>14.8V Nominal"] --> PDB["Power Distribution Board"]

    PDB --> FC_REG["FC Voltage Regulators"]
    PDB --> COMP_REG["Companion Board Regulators"]

    FC_REG --> V_SERVO["8V or 12V<br/>Selectable<br/>Servo / Peripherals"]
    FC_REG --> V5["5V<br/>Digital Logic"]
    FC_REG --> V3_MCU["3.3V<br/>MCU"]
    FC_REG --> V3_SENS["3.3V Low Noise<br/>Sensors (IMU, Baro)"]

    COMP_REG --> COMP_3V3["3.3V<br/>ESP32-P4 + Peripherals"]
    COMP_REG --> LTE_BUCK["Dedicated Buck<br/>A7670 LTE Module"]

    style V3_SENS fill:#2d5a27,color:#fff
    style BAT fill:#8b0000,color:#fff
    style LTE_BUCK fill:#1a3a5c,color:#fff
```

The flight controller and companion computer have fully independent power regulation with power protection on every rail across both boards. The companion board expects 5V input from the PDB and regulates down to 3.3V for the ESP32-P4 and peripherals, with a dedicated buck converter supplying the A7670 LTE module on its own isolated rail. The FC provides a selectable 8V/12V rail for servos and peripherals, plus dedicated 5V, 3.3V, and a low-noise 3.3V rail isolated for the IMUs and barometer to minimize sensor noise.

No power dependency exists between the two boards. The companion board receives only UART data from the FC, ensuring a failure on the companion side cannot affect flight-critical systems.

## Hardware

### Flight Controller PCB

![FC Top Render](Renders/AU2.png)
![FC Bottom Render](Renders/AU3.png)

### Companion Computer PCB

![Companion Top Render](Renders/AU4.png)
![Companion Bottom Render](Renders/AU5.png)

### Assembled Stack

![Assembled Photos](Images/assembled_stack.jpg)

## EMI/EMC Validation

Both boards underwent pre-fabrication electromagnetic simulation in OpenEMS with FreeCAD integration. Simulation targets included:

- Signal integrity on high-speed MIPI CSI-2 differential pairs
- Emission compliance around the LTE antenna feed
- Ground plane continuity and return path analysis
- Decoupling capacitor placement optimization for the sensor power rail

Antenna matching on the companion board was informed by prior LTE RF layout experience from a production Modbus RTU pump controller deployment.

## Post-Flight Workflow

1. Remove SD card from the companion board after landing
2. Import geotagged JPEG frames and LiDAR scan logs into photogrammetry software (Agisoft Metashape, Pix4D, or OpenDroneMap)
3. Camera imagery produces dense 3D point clouds and textured mesh through multi-view photogrammetry
4. LiDAR terrain profiles supplement the model with accurate elevation data
5. Output: georeferenced 3D reconstruction for survey, inspection, or search-and-rescue applications

## Project Status

| Component | Status |
|---|---|
| Flight Controller PCB | Fabricated, assembled, and flight-tested |
| FC Firmware (ArduPilot/INAV) | Custom target validated with live sensor fusion and RC control |
| Companion Computer PCB | Design complete, renders finalized, awaiting fabrication |
| Companion Firmware | In development |
| System Integration | Pending companion board fabrication |

## Repository Contents

```
├── Renders/
│   ├── AU2.png
│   ├── AU3.png
│   ├── AU4.png
│   └── AU5.png
├── Images/
│   ├── assembled_stack.jpg
│   ├── scope_captures/
│   └── test_results/
└── README.md
```