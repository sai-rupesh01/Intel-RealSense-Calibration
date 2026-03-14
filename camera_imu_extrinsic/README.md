# 🔧 Kalibr Docker — Visual-Inertial Calibration for ROS 2 Users

<div align="center">

![ROS 2](https://img.shields.io/badge/ROS_2-Humble%20%7C%20Jazzy-blue?logo=ros&logoColor=white)
![ROS 1](https://img.shields.io/badge/ROS_1-Noetic-orange?logo=ros&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Containerized-2496ED?logo=docker&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-20.04_Focal-E95420?logo=ubuntu&logoColor=white)

**A complete, containerized workflow for calibrating stereo cameras and IMUs using [Kalibr](https://github.com/ethz-asl/kalibr) — without touching your ROS 2 environment.**

[Overview](#overview) · [Prerequisites](#prerequisites) · [Setup](#1-dockerfile-setup) · [Recording](#2-recording-data-in-ros-2) · [Conversion](#3-converting-ros-2-bag-to-ros-1) · [Calibration](#5-running-calibration) · [Config Files](#configuration-files)

</div>

---

## Overview

[Kalibr](https://github.com/ethz-asl/kalibr) is the industry standard for visual-inertial calibration. However, it runs most reliably under **ROS 1 Noetic**, while most modern robotics stacks (ArduPilot, Nav2, etc.) run on **ROS 2**.

This repository bridges that gap with a clean, self-contained workflow:

```
ROS 2 (record) ──► rosbags-convert ──► ROS 1 .bag ──► Kalibr Docker (calibrate)
```

**What's included:**
- 🐳 A ready-to-build `Dockerfile` with all Kalibr dependencies
- 📦 ROS 2 bag recording instructions for Intel RealSense cameras
- 🔄 ROS 2 → ROS 1 bag conversion using `rosbags-convert`
- ⚙️ Pre-configured YAML files for stereo + IMU calibration
- 📋 Step-by-step calibration commands with expected output

> **Tested Hardware:** Intel RealSense D435i / D455 (infrared stereo + IMU)

---

## Prerequisites

Before you begin, make sure the following are installed on your host:

| Requirement | Version | Notes |
|---|---|---|
| Docker | ≥ 20.x | [Install Docker](https://docs.docker.com/get-docker/) |
| ROS 2 | Humble / Jazzy | Running on host |
| Python 3 | ≥ 3.8 | For bag conversion |
| `realsense2_camera` | Latest | ROS 2 RealSense driver |
| X11 / Display | — | For Kalibr GUI output |

---

## Repository Structure

```
kalibr-docker/
├── Dockerfile              # Builds the Kalibr ROS 1 Noetic container
├── config/
│   ├── camchain.yaml       # Stereo camera intrinsics & extrinsics
│   ├── imu.yaml            # IMU noise parameters
│   └── aprilgrid.yaml      # Calibration target definition
└── README.md
```

---

## 1. Dockerfile Setup

Create a `Dockerfile` in your project root. This image is built on **Ubuntu 20.04** and includes ROS Noetic, all Kalibr Python/C++ dependencies, and a pre-built Kalibr catkin workspace.

<details>
<summary><b>📄 Click to view full Dockerfile</b></summary>

```dockerfile
# Use a stable Ubuntu Focal base image
FROM ubuntu:20.04

# Set non-interactive mode and timezone
ENV DEBIAN_FRONTEND=noninteractive
ENV TZ=Etc/UTC

# -----------------------
# Base utilities
# -----------------------
RUN apt-get update && apt-get install -y --no-install-recommends \
    sudo curl wget gnupg2 lsb-release \
    python3 python3-pip python3-dev python-is-python3 \
    build-essential cmake pkg-config git \
    nano

# -----------------------
# Kalibr dependencies
# -----------------------
RUN apt-get update \
    && apt-get install -y --no-install-recommends -o Acquire::Retries=3 \
    python3-numpy python3-scipy python3-matplotlib python3-yaml \
    libeigen3-dev libboost-all-dev libopencv-dev \
    libatlas-base-dev libsuitesparse-dev \
    libgoogle-glog-dev libglew-dev \
    libv4l-dev libyaml-cpp-dev libjpeg-dev \
    python3-wxgtk4.0 python3-wxgtk-media4.0 \
    libwxgtk3.0-gtk3-dev libwxgtk-webview3.0-gtk3-dev \
    python3-tk \
    && rm -rf /var/lib/apt/lists/*

# -----------------------
# Add ROS Noetic repo and GPG key
# -----------------------
RUN curl -s https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
    -o /usr/share/keyrings/ros-archive-keyring.gpg

RUN echo "deb [signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] \
http://packages.ros.org/ros/ubuntu focal main" \
    > /etc/apt/sources.list.d/ros-noetic.list

# -----------------------
# Install ROS Noetic
# -----------------------
RUN apt-get update && apt-get install -y --no-install-recommends \
    ros-noetic-ros-base \
    python3-rosdep python3-rosinstall-generator python3-catkin-tools \
    ros-noetic-cv-bridge

RUN rosdep init || true
RUN rosdep update

ENV ROS_DISTRO=noetic
ENV ROS_VERSION=1
ENV ROS_PYTHON_VERSION=3

RUN echo "source /opt/ros/noetic/setup.bash" >> /root/.bashrc

# -----------------------
# Kalibr workspace
# -----------------------
RUN mkdir -p /kalibr_ws/src
WORKDIR /kalibr_ws/src
RUN git clone https://github.com/ethz-asl/kalibr.git

WORKDIR /kalibr_ws
RUN bash -c "source /opt/ros/noetic/setup.bash && \
             rosdep install --from-paths src --ignore-src -r -y"

RUN bash -c "source /opt/ros/noetic/setup.bash && \
             catkin config --extend /opt/ros/noetic && \
             catkin build -DCMAKE_BUILD_TYPE=Release"

RUN echo "source /kalibr_ws/devel/setup.bash" >> /root/.bashrc

CMD ["bash"]
```

</details>

### Build the Docker Image

```bash
docker build -t kalibr_container .
```

> ⏳ First build takes ~15–25 minutes. Subsequent builds are cached.

---

## 2. Recording Data in ROS 2

Place your **Kalibr AprilGrid target** (6×8 tags) on a flat wall or board.

### Step 1 — Launch the RealSense Camera

Disable color/depth streams and enable infrared + IMU (optimal for VIO calibration):

```bash
ros2 launch realsense2_camera rs_launch.py \
    enable_color:=false enable_depth:=false \
    enable_infra1:=true enable_infra2:=true \
    enable_accel:=true enable_gyro:=true \
    enable_emitter:=false infra_fps:=15
```

> 💡 **Tip:** Set `infra_fps:=15` to keep image rate low enough for Kalibr's feature extraction — 20+ fps can cause issues.

### Step 2 — Record the Bag

```bash
mkdir -p ~/kalibr_data

ros2 bag record -o ~/kalibr_data/calibration_bag_ros2 \
    /camera/camera/infra1/image_rect_raw \
    /camera/camera/infra1/camera_info \
    /camera/camera/infra2/image_rect_raw \
    /camera/camera/infra2/camera_info \
    /camera/camera/gyro/sample \
    /camera/camera/accel/sample
```

### Recording Tips

> 🎯 **Motion pattern matters.** Move the camera in all 6 degrees of freedom — translate in X, Y, Z and rotate around all three axes. Make each motion slow, smooth, and deliberate.
>
> ⏱️ **Duration:** 60–90 seconds of motion is typically sufficient.
>
> 📐 **Distance:** Keep the full AprilGrid visible in both infrared frames throughout.

---

## 3. Converting ROS 2 Bag to ROS 1

Kalibr requires a ROS 1 `.bag` file. We use `rosbags-convert` for this.

### Install `rosbags`

```bash
pip install rosbags
```

If you get a `command not found` error after install:

```bash
# Locate the binary
find ~/.local -name "rosbags-convert"

# Add to PATH
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Confirm it's available
which rosbags-convert
```

### Run the Conversion

```bash
rosbags-convert \
  --src ~/kalibr_data/calibration_bag_ros2 \
  --dst ~/kalibr_data/calibration_bag_ros1.bag \
  --dst-storage sqlite3 \
  --dst-typestore ros1_noetic
```

### Verify the Converted Bag

```bash
rosbag info ~/kalibr_data/calibration_bag_ros1.bag
```

You should see all recorded topics listed with message counts and duration.

---

## 4. Running the Kalibr Docker Container

Ensure your `~/kalibr_data/` folder contains:

```
~/kalibr_data/
├── calibration_bag_ros1.bag
├── camchain.yaml
├── imu.yaml
└── aprilgrid.yaml
```

### Allow GUI Access

```bash
xhost +local:docker
```

### Launch the Container

```bash
sudo docker run -it \
   --net=host \
   -e DISPLAY=$DISPLAY \
   -e QT_X11_NO_MITSHM=1 \
   -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
   -v ~/kalibr_data/:/data \
   kalibr_container
```

Your data directory is now available at `/data` inside the container.

---

## 5. Running Calibration

All commands below are run **inside the Docker container**.

```bash
cd /data
```

### Option A — Stereo Camera Calibration

```bash
rosrun kalibr kalibr_calibrate_cameras \
    --bag calibration_bag_ros1.bag \
    --topics /camera/infra1/image_rect /camera/infra2/image_rect \
    --models pinhole-equi pinhole-equi \
    --target aprilgrid.yaml
```

### Option B — Camera–IMU Calibration

```bash
rosrun kalibr kalibr_calibrate_imu_camera \
   --bag calibration_bag_ros1.bag \
   --cam camchain.yaml \
   --imu imu.yaml \
   --target aprilgrid.yaml \
   --timeoffset-padding 0.1 \
   --show-extraction
```

### Expected Console Output

```
importing libraries
Initializing IMUs:
  Update rate: 200.0
  Accelerometer:
    Noise density: 0.02
    Noise density (discrete): 0.282842712474619
    Random walk: 0.0002
  ...
Reading IMU data (/camera/camera/imu)
  Read 13956 imu readings over 69.9 seconds
Initializing calibration target:
  Type: aprilgrid
  Tags:
    Rows: 6
    Cols: 8
  ...
```

### Output Files

Results are saved to your **host machine's** `~/kalibr_data/` folder:

| File | Description |
|---|---|
| `*-camchain.yaml` | Calibrated camera chain (intrinsics + extrinsics) |
| `*-imu.yaml` | Calibrated IMU–camera transform |
| `*-results-imucam.txt` | Human-readable calibration report |
| `*-report-imucam.pdf` | Visual PDF report with residual plots |

---

## Configuration Files

### `config/camchain.yaml`

```yaml
cam0:
  cam_overlaps: [1]
  camera_model: pinhole
  distortion_coeffs: [0.0, 0.0, 0.0, 0.0]  # Images assumed pre-rectified
  distortion_model: equi
  intrinsics: [424.0, 424.0, 424.0, 240.0]  # [fu, fv, pu, pv]
  resolution: [848, 480]
  rostopic: /camera/infra1/image_rect

cam1:
  cam_overlaps: [0]
  camera_model: pinhole
  distortion_coeffs: [0.0, 0.0, 0.0, 0.0]
  distortion_model: equi
  intrinsics: [424.0, 424.0, 424.0, 240.0]
  resolution: [848, 480]
  rostopic: /camera/infra2/image_rect
  T_cn_cnm1:                          # cam1 pose relative to cam0
    - [1.0, 0.0, 0.0, -0.05]          # ~50mm baseline on X axis
    - [0.0, 1.0, 0.0,  0.0]
    - [0.0, 0.0, 1.0,  0.0]
    - [0.0, 0.0, 0.0,  1.0]
```

### `config/imu.yaml`

```yaml
rostopic: /camera/camera/imu
update_rate: 200.0                    # Hz

accelerometer_noise_density: 0.02    # [m/s²/√Hz]
accelerometer_random_walk:   0.0002  # [m/s³/√Hz]
gyroscope_noise_density:     0.02    # [rad/s/√Hz]
gyroscope_random_walk:       0.0002  # [rad/s²/√Hz]
```

> ⚠️ **Important:** These are generic RealSense values. For best accuracy, replace with values from your camera's datasheet or the [Allan Deviation](https://github.com/gaowenliang/imu_utils) of your specific unit.

### `config/aprilgrid.yaml`

```yaml
target_type: 'aprilgrid'
tagCols:    8
tagRows:    6
tagSize:    0.08      # Tag edge length in meters
tagSpacing: 0.3125   # Ratio of gap to tag size (0.025m / 0.08m)
```

> 📏 **Physical target:** Print the AprilGrid at the correct scale so each tag is exactly `tagSize` meters. Verify with a ruler before calibrating.

---

## Troubleshooting

<details>
<summary><b>❌ <code>rosbags-convert: command not found</code></b></summary>

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

</details>

<details>
<summary><b>❌ Docker GUI / display errors</b></summary>

Run `xhost +local:docker` on your host before launching the container. Ensure `$DISPLAY` is set (usually `:0` or `:1`).

</details>

<details>
<summary><b>❌ Kalibr fails with "No corners detected"</b></summary>

- Ensure the full AprilGrid is visible in **both** camera frames.
- Reduce infrared fps to `15` or lower.
- Check that `tagSize` and `tagSpacing` in `aprilgrid.yaml` match your printed target exactly.
- Improve lighting — infrared works best without strong direct sunlight.

</details>

<details>
<summary><b>❌ IMU topic mismatch inside container</b></summary>

The IMU topic in your `.bag` file must match `rostopic` in `imu.yaml`. You can inspect topics with:

```bash
rosbag info calibration_bag_ros1.bag
```

</details>

---

## Workflow Summary

```
┌─────────────────────────────────────────────────────────┐
│                     HOST MACHINE                         │
│                                                          │
│  1. ros2 launch realsense2_camera  ──► camera streams   │
│  2. ros2 bag record                ──► ROS 2 bag        │
│  3. rosbags-convert                ──► ROS 1 .bag       │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │              DOCKER CONTAINER                     │   │
│  │                                                   │   │
│  │  4. kalibr_calibrate_imu_camera                   │   │
│  │        ──► camchain-results.yaml                  │   │
│  │        ──► report.pdf                             │   │
│  │                                                   │   │
│  └──────────────────────────────────────────────────┘   │
│                          │                               │
│        Results saved to ~/kalibr_data/ on host          │
└─────────────────────────────────────────────────────────┘
```

---

## Credits & References

- [Kalibr — ETH ASL](https://github.com/ethz-asl/kalibr)
- [rosbags — ternaris](https://github.com/rpng/rosbags)
- [Intel RealSense ROS 2 Wrapper](https://github.com/IntelRealSense/realsense-ros)
- [Kalibr Wiki — Camera-IMU Calibration](https://github.com/ethz-asl/kalibr/wiki/camera-imu-calibration)

---

<div align="center">
  <sub>Built for roboticists who don't want ROS 1 on their machine.</sub>
</div>
