# Intel RealSense Calibration

## 1. On-Chip Calibration

This step uses the Intel RealSense D400 series' internal processor to improve depth accuracy and reduce noise. It updates the camera's internal depth table without relying on host-side processing.

### Prerequisites

* **Software:** Intel RealSense Viewer (`realsense-viewer`) installed via SDK 2.0.
* **Hardware:** Connect the camera via a **USB 3.0** port (USB 2.0 may restrict bandwidth/features).
* **Environment:** A flat, **plain white wall** with uniform lighting.

### Step-by-Step Guide

#### 1. Physical Setup
* **Position:** Face the camera towards a plain white wall.
* **Alignment:** Ensure the camera is parallel to the wall (not tilted).
* **Distance:** The camera must be at least **1 meter (approx. 3.3 feet)** away from the wall.
* **Lighting:** Ensure the wall is well-lit but avoid direct glare or dark shadows on the surface.

#### 2. Launch the Software
Open a terminal (Linux) or command prompt (Windows) and launch the viewer:

realsense-viewer 

#### 3. Run Calibration
Expand the More menu (sometimes listed as "Functions" or a gear icon) under the Stereo Module settings.

Select On-Chip Calibration.

A dialogue box will appear with calibration options. Ensure the settings are optimized for a "White Wall" target if available, or leave defaults as Medium speed.

Click Calibrate.

Note: The camera feed may freeze for a few seconds while the internal processor calculates the new depth table.

#### 4. Analyze Results (The 0.25 Metric)
Once the calculation is complete, the viewer will display a Health Check score and an improvement comparison.

Metric: The Health Check number represents the geometric error.

Pass Criteria: The calibrated value (Health Check) should be less than 0.25.