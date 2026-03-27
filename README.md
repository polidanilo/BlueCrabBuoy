# Blue Crab Monitoring Buoy
An edge AI IoT buoy for autonomous, low-power monitoring of *Callinectes sapidus* (blue crab) in lagoonal environments — designed to transmit lightweight ecological payloads over LoRa/MQTT,
with a planned computer vision layer for on-device species detection.

> Status: hardware design complete · sensor assembly in progress · future AI pipeline planned

<table width="100%" border="0">
  <tr>
    <td width="100%" align="center" valign="top">
      <img src="assets/Screenshot 2026-03-27 010018.png" width="100%"><br>
      <em>Provisional operational diagram: text in Italian, will be replaced with an updated diagram before first field deployment.
      </em>
    </td>
  </tr>
</table>

## Why this project
The blue crab is one of the most ecologically damaging invasive species in the Mediterranean. Current monitoring relies on manual trap inspection campaigns — periodic, labor-intensive, and spatially limited. This buoy is designed as a continuous, autonomous alternative: a fixed-point node that detects environmental triggers, logs structured sensor data, and transmits compact telemetry payloads without human presence.

The system is inspired by published underwater camera trap methodology (L'Hoste et al. 2025, Bilodeau et al. 2021) and population biology data for *C. sapidus* in Italian saltmarshes (Marchessaux et al. 2023).

## Hardware architecture
The buoy is built around two microcontrollers with distinct roles, enclosed in a waterproof chassis with passive anti-fouling measures and different planned options for power supply, including solar charging.

### Enclosure and deployment
| Component | Role |
|---|---|
| IP65/IP68 waterproof enclosure | Protects electronics from water ingress |
| PG7 cable glands | Sealed sensor wire pass-throughs |
| Acrylic optical window | Camera port, flush-mounted and sealed |
| Foam collar | Passive buoyancy |
| Copper tape | Passive anti-biofouling |
| Anchor line + weight | Fixed-point mooring in shallow water |
| Bait compartment | Attracts *C. sapidus* into camera FOV |
| 2× 18650 Li-ion cells | Primary power source |
| Solar panel | Trickle charge for extended deployment |

### Electronics
```
ESP32 (manager / ULP trigger)        STM32H743 (future coprocessor)
├── JSN-SR04T sonar                  ├── OV2640 camera module
├── DS18B20 temperature sensor       └── AI pipeline (planned)
├── Turbidity sensor
├── LoRa / Wi-Fi / MQTT
└── MicroSD (DTN offline queue)
```

**Energy-triggered pipeline:** the ESP32 ULP coprocessor pings the sonar continuously at minimal current draw. Only when an object enters the detection range does it wake the STM32 and the main processing pipeline for a sensor acquisition window, then returns to deep sleep.

## Telemetry and ecological output
Each trigger event produces a compact payload transmitted over LoRa/MQTT:
```
node_id | timestamp | temp_c | turbidity | sonar_cm | flags
```
True negatives are logged hourly even when no trigger occurs, enabling detection frequency and activity pattern analysis.

## AI pipeline (planned)
> The following describes the intended computer vision layer,
> to be implemented after the sensor/telemetry stack is validated in the field.

- **Model:** YOLOv8n / YOLO11n fine-tuned on the [Brackish Dataset](https://public.roboflow.com/object-detection/brackish-underwater).
- **Training:** PyTorch with Focal Loss, underwater augmentation (CLAHE, gaussian noise, HSV shifts), hard negative mining.
- **Deployment:** PyTorch → ONNX → TFLite INT8 (PTQ) via CMSIS-NN on STM32 Cortex-M7.
- **Biological validation:** carapace size classification (juvenile < 5 cm / subadult / mature > 11.75 cm) based on Marchessaux et al. 2023 L₅₀ data.

When active, each detection event will extend the payload to include `num_crabs` and `size_class` (juvenile / subadult / mature), turning each detection into a data point on population structure rather than simple presence/absence.

## Current status & roadmap
| Area | Status |
|---|---|
| Hardware architecture & BOM | ✅ Complete |
| Repository structure & module design | ✅ Complete |
| Enclosure mechanical design | 🔄 In progress |
| Component procurement | 🔄 In progress |
| ESP32 firmware — sonar + ULP trigger | 🔄 In progress |
| ESP32 firmware — DS18B20 + turbidity | ⏳ Planned |
| LoRa telemetry pipeline + DTN queue | ⏳ Planned |
| Field deployment and testing | ⏳ Planned |
| STM32 integration + camera module | ⏳ Planned |
| YOLO fine-tuning on Brackish Dataset | ⏳ Planned |
| TFLite INT8 quantization + CMSIS-NN | ⏳ Planned |
| Carapace size classification | ⏳ Planned |

## Repository structure
```
BlueCrabMonitoring/
├── config/          # YAML configs (hardware, training, deployment)
├── data/            # Dataset loaders, augmentation, splits
├── core/            # YOLO wrapper, CoordConv, NMS, embeddings
├── losses/          # Focal Loss, CIoU/DFL, distillation loss
├── training/        # PyTorch training loop + utilities
├── evaluation/      # mAP, confusion matrix, bias analysis
├── distillation/    # Teacher→student knowledge distillation
├── quantization/    # ONNX + TFLite INT8 export pipeline
├── telemetry/       # Payload format, DTN queue, logging
├── deploy/          # C/C++ firmware (STM32 + ESP32)
└── docs/            # Architecture decisions, power budget, biology considerations
```

## References
- L'Hoste et al. (2025). *A new underwater camera trap for 
  freshwater wildlife monitoring.* Methods in Ecology and 
  Evolution.
- Bilodeau et al. (2021). *A low-cost, long-term underwater 
  camera trap network coupled with deep residual learning.* 
  PLOS ONE.
- Marchessaux et al. (2023). *Environmental drivers of 
  size-based population structure of Callinectes sapidus 
  in the Mediterranean.* PLOS ONE.
- Ciapponi et al. (2025). *WrenNet: enabling multi-species 
  bird classification on low-power bioacoustic loggers.* 
  arXiv.
