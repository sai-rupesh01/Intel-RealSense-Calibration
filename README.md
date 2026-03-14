# Intel-RealSense-Calibration

Workflows and scripts to calibrate **Intel RealSense D400 series** cameras.  
This repository covers:

- On-chip depth calibration  
- Python-based IMU calibration  
- Camera–IMU extrinsic calibration using **Kalibr**

---

## 📌 Features

- Improve depth accuracy and reduce noise
- No host-side processing for on-chip calibration
- Persistent calibration stored in camera flash
- Suitable for robotics, perception, and ROS/ROS2 pipelines

---

## 📂 Repository Structure

```text
Intel-RealSense-Calibration/
├── on_chip_calibration/
│   └── README.md
├── imu_calibration/
│   └── README.md
├── camera_imu_extrinsic/
│   ├── calib.io_kalibr_841x594_4x6_100.pdf  # kalibr image 
|   |── kalibr_calibration_process.png
│   └── README.md
└── README.md
