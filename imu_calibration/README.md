# 📐 Intel RealSense IMU Intrinsic Calibration

<div align="center">

![RealSense](https://img.shields.io/badge/Intel-RealSense-0071C5?logo=intel&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.x-3776AB?logo=python&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-20.04%20%7C%2022.04-E95420?logo=ubuntu&logoColor=white)
![IMU](https://img.shields.io/badge/IMU-Intrinsic_Calibration-brightgreen)

**Calibrate the IMU on your Intel RealSense camera using Intel's official Python script — stores calibration directly into the device's onboard memory.**

 [Requirements](#requirements) · [Build SDK](#2-build-librealsense) · [Run Calibration](#4-run-the-calibration-script) · [Orientations](#5-calibration-procedure) · [Verify](#7-verify-calibration)

</div>

---

## Requirements

### System Dependencies

```bash
sudo apt install git cmake build-essential
sudo apt install libusb-1.0-0-dev pkg-config
sudo apt install libgtk-3-dev
sudo apt install python3 python3-pip
```

### Python Dependencies

```bash
pip install numpy scipy matplotlib
```

### Hardware

- Intel RealSense camera with IMU (D435i, D455, T265)
- USB 3.0 cable and port
- A **flat, stable surface** (table, not hand-held)

---

## 1. Clone the librealsense SDK

```bash
git clone https://github.com/IntelRealSense/librealsense.git
cd librealsense
```

---

## 2. Build librealsense

```bash
mkdir build && cd build
cmake ..
make -j4
sudo make install
```

> ⏳ Build takes 5–10 minutes depending on your machine. The `-j4` flag parallelizes across 4 cores — increase to `-j$(nproc)` to use all available cores.

---

## 3. Locate the Calibration Script

```bash
cd librealsense/tools/imu-calibration
ls
```

You should see:

```
rs-imu-calibration.py
```

> ℹ️ The script name may vary slightly across SDK versions. If `rs-imu-calibration.py` is not present, look for `imu_calibration.py` in the same directory.

---

## 4. Run the Calibration Script

```bash
python3 rs-imu-calibration.py
```

If you have an older SDK version, try:

```bash
python3 imu_calibration.py
```

The script will automatically detect your connected RealSense device and begin the guided calibration sequence.

---

## 5. Calibration Procedure

> ⚠️ **This is the most important step.** Poor placement = poor calibration. Read carefully before starting.

The script guides you through placing the camera in **6 static orientations**, one at a time. It uses the accelerometer to detect when the camera is stable and correctly oriented before recording each sample.

### The 6 Required Orientations

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   1. FLAT           2. LEFT SIDE       3. RIGHT SIDE            │
│                                                                 │
│   ┌──────┐          ┌──┐               ┌──┐                     │
│   │ CAM  │          │  │               │  │                     │
│   └──────┘          │  │               │  │                     │
│   lens up           └──┘ lens left     └──┘ lens right          │
│                                                                 │
│   4. FRONT          5. BACK            6. UPSIDE DOWN           │
│                                                                 │
│   ┌──────┐          ┌──────┐           └──────┘                 │
│   │ ████ │          │      │           │ CAM  │                 │
│   └──────┘          └──────┘           ┌──────┐                 │
│   lens facing you   lens away          lens down                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### What the Script Does at Each Step

1. Displays the **target orientation** on screen
2. Waits until your camera matches it (detected via accelerometer gravity vector)
3. Records IMU samples for several seconds once stable
4. Moves to the next orientation automatically

> 🔴 **If you move the camera during recording, the script stops collecting data for that orientation.** Hold completely still until it advances.

### Placement Tips

| ✅ Do | ❌ Don't |
|---|---|
| Place on a flat table | Hold in your hand |
| Use a book or box to prop up side orientations | Use a tripod with any tilt |
| Keep the USB cable slack and not pulling | Let the cable tension tilt the camera |
| Wait for the script to confirm each orientation | Rush through orientations |
| Work in a vibration-free environment | Calibrate near HVAC vents or motors |

---

## 6. After Calibration

On success, the script prints:

```
Calibration successful
Writing to device
Done
```

**Summary:**

- The script computed accelerometer and gyroscope intrinsic parameters (bias, scale factors, cross-axis terms)
- These parameters were written to the camera's **onboard NVRAM** (non-volatile memory)
- The RealSense SDK will **automatically apply** this calibration every time the camera is used — no extra configuration or YAML files needed
- The calibration **persists across reboots, reconnections, and host machines**

---

## 7. Verify Calibration

### Using realsense-viewer

```bash
realsense-viewer
```

Navigate to **Motion Module** in the left panel. You should see IMU calibration status marked as enabled/applied.

### Using Python (optional)

You can also confirm the calibration was stored programmatically:

```python
import pyrealsense2 as rs

pipeline = rs.pipeline()
config = rs.config()
config.enable_stream(rs.stream.accel)
config.enable_stream(rs.stream.gyro)
pipeline.start(config)

profile = pipeline.get_active_profile()
accel = profile.get_stream(rs.stream.accel).as_motion_stream_profile()
imu_profile = accel.get_motion_intrinsics()

print("Accelerometer bias:", imu_profile.bias_variances)
print("Accelerometer noise:", imu_profile.noise_variances)
pipeline.stop()
```

Non-zero bias and noise variance values confirm the calibration is active.

---

## Where This Fits in the Full Calibration Pipeline

If you are setting up a VIO / SLAM stack, IMU intrinsic calibration is **Step 1** of a two-step process:

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 |
│  STEP 1 — IMU Intrinsic Calibration          ◄── You are here   │
│                                                                 │
│  Tool:    Intel rs-imu-calibration.py                           │
│  What:    Removes accelerometer & gyroscope bias/scale errors   │
│  Output:  Written to device NVRAM (automatic)                   │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 2 — Camera–IMU Extrinsic Calibration                      │
│                                                                 │
│  Tool:    Kalibr (via Docker — see main README)                 │
│  What:    Finds the 3D transform between camera and IMU frames  │
│  Output:  camchain.yaml + imu.yaml for use in your ROS stack    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```


---

## Troubleshooting

<details>
<summary><b>❌ Script cannot find the RealSense device</b></summary>

Make sure the camera is connected via **USB 3.0** (not a hub). Verify it's detected:

```bash
rs-enumerate-devices
```

If not found, check udev rules:

```bash
cd librealsense
sudo cp config/99-realsense-libusb.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules && sudo udevadm trigger
```

Then reconnect the camera.

</details>

<details>
<summary><b>❌ Script fails mid-calibration with "motion detected"</b></summary>

The camera moved during a recording step. Ensure:
- Camera is resting on a solid surface, not hand-held
- USB cable is not pulling or flexing the camera body
- No vibration sources nearby (fans, motors, footsteps)

Restart the script from the beginning — partial calibrations are not saved.

</details>

<details>
<summary><b>❌ "Writing to device" fails</b></summary>

The script needs write access to the device firmware. Try running with elevated permissions:

```bash
sudo python3 rs-imu-calibration.py
```

</details>

<details>
<summary><b>❌ realsense-viewer shows no Motion Module</b></summary>

Your camera may not have an IMU (e.g., D435 without the "i" suffix). Confirm your model supports IMU:

```bash
rs-enumerate-devices | grep -i imu
```

</details>

---

## Quick Reference

```bash
# Install dependencies
sudo apt install git cmake build-essential libusb-1.0-0-dev pkg-config libgtk-3-dev
pip install numpy scipy matplotlib

# Clone & build SDK
git clone https://github.com/IntelRealSense/librealsense.git
cd librealsense && mkdir build && cd build
cmake .. && make -j$(nproc) && sudo make install

# Run calibration
cd ../tools/imu-calibration
python3 rs-imu-calibration.py

# Verify
realsense-viewer  →  Motion Module  →  calibration enabled ✅
```

---

## References

- [Intel RealSense librealsense — GitHub](https://github.com/IntelRealSense/librealsense)
- [Intel IMU Calibration Tool — SDK Docs](https://dev.intelrealsense.com/docs/imu-calibration-tool-for-intel-realsense-depth-camera)
- [RealSense IMU White Paper — Intel](https://www.intelrealsense.com/wp-content/uploads/2019/07/Intel_RealSense_Depth_D435i_IMU_Calibration.pdf)

---

<div align="center">
  <sub>Calibrate once. Use everywhere — ROS, SLAM, VIO, EKF, ArduPilot.</sub>
</div>
