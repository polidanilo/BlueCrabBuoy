# Blue Crab Monitoring Buoy

An edge AI IoT buoy for autonomous, low-power monitoring of 
*Callinectes sapidus* (blue crab) in lagoonal environments — 
designed to run computer vision inference locally and transmit 
only lightweight ecological payloads over LoRa/MQTT.

> Status: architecture complete, implementation in progress

## Why this project

The blue crab is one of the most damaging invasive species in 
the Mediterranean. Current monitoring relies on manual trap 
inspection campaigns — periodic, labor-intensive, and spatially 
limited. This buoy is designed as a continuous, autonomous 
alternative: a fixed-point camera trap that detects crabs, 
estimates carapace size, and transmits structured ecological 
data without human presence.

The system is inspired by published underwater camera trap 
methodology (L'Hoste et al. 2025, Bilodeau et al. 2021) and 
population biology data for *C. sapidus* in Italian saltmarshes 
(Marchessaux et al. 2023).

## Hardware architecture
```
ESP32 (manager)          STM32H743 (AI coprocessor)
├── JSN-SR04T sonar      ├── OV2640 camera (acrylic window)
├── DS18B20 temperature  ├── YOLOv8n INT8 inference
├── turbidity sensor     └── pixels_to_centimeters estimate
├── LoRa / Wi-Fi / MQTT
└── MicroSD (DTN queue)
```

**Energy-triggered pipeline:** the ESP32 ULP coprocessor pings 
the sonar at 10–150 µA continuously. Only when an object 
approaches the bait does it wake the STM32 for a ~500ms 
inference window, then returns to deep sleep. Target autonomy: 
6+ months on two 18650 cells.

## AI stack

- **Model:** YOLOv8n / YOLO11n fine-tuned on the 
  [Brackish Dataset](https://public.roboflow.com/object-detection/brackish-underwater)
- **Training:** vanilla PyTorch loop with Focal Loss, 
  underwater data augmentation (CLAHE, gaussian noise, 
  HSV shifts), hard negative mining
- **Deployment:** PyTorch → ONNX → TFLite INT8 (PTQ) 
  via CMSIS-NN on STM32 Cortex-M7
- **Biological validation:** carapace size classification 
  (juvenile < 5 cm / subadult / mature > 11.75 cm) based on 
  Marchessaux et al. 2023 L₅₀ data

## Ecological output

Each detection event produces a 23-byte payload:
```
node_id | timestamp | lat | lon | num_crabs | size_class 
| temp_c | turbidity | sonar_cm | flags
```

`size_class` encodes juvenile / subadult / mature — turning 
each detection into a data point on population structure, not 
just presence/absence.

True negatives are logged hourly even when no crab is detected, 
enabling detection frequency analysis (L'Hoste et al. 2025 
methodology).

## Repository structure
```
BlueCrabMonitoring/
├── config/          # YAML configs (hardware, training, deployment)
├── data/            # Dataset loaders, augmentation, splits
├── core/            # YOLO wrapper, CoordConv, NMS, embeddings
├── losses/          # Focal Loss, CIoU/DFL, distillation loss
├── training/        # Vanilla PyTorch training loop + utilities
├── evaluation/      # mAP, confusion matrix, bias analysis
├── distillation/    # Teacher→student knowledge distillation
├── quantization/    # ONNX + TFLite INT8 export pipeline
├── telemetry/       # Payload format, DTN queue, logging
├── deploy/          # C/C++ firmware (STM32 + ESP32)
└── docs/            # Architecture decisions, power budget, 
                     # Biology considerations
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