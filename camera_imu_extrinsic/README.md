# 🔧 Kalibr Docker — Visual-Inertial Calibration for ROS 2 Users

<div align="center">

![ROS 2](https://img.shields.io/badge/ROS_2-Humble%20%7C%20Jazzy-blue?logo=ros&logoColor=white)
![ROS 1](https://img.shields.io/badge/ROS_1-Noetic-orange?logo=ros&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-20.04_Focal-E95420?logo=ubuntu&logoColor=white)


**A complete, containerized workflow for calibrating stereo cameras and IMUs using [Kalibr](https://github.com/ethz-asl/kalibr) — without touching your ROS 2 environment.**

[Overview](#overview) · [Prerequisites](#prerequisites) · [Structure](#repository-structure) · [Docker Compose](#2-docker-compose-setup) · [Recording](#3-recording-data-in-ros-2) · [Conversion](#4-converting-ros-2-bag-to-ros-1) · [Calibration](#6-running-calibration) · [Config Files](#configuration-files)

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
- 🧩 A `docker-compose.yml` that handles volumes, display, and networking automatically
- 📂 A shared `data/` volume — bags, configs, and results live here, visible on both host and container
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
| Docker Compose | ≥ 2.x | Bundled with Docker Desktop; `apt install docker-compose-plugin` on Linux |
| ROS 2 | Humble / Jazzy | Running on host |
| Python 3 | ≥ 3.8 | For bag conversion |
| `realsense2_camera` | Latest | ROS 2 RealSense driver |
| X11 / Display | — | For Kalibr GUI output |

---

## Repository Structure

```
kalibr-docker/
├── Dockerfile                   # Builds the Kalibr ROS 1 Noetic image
├── docker-compose.yml           # Manages volumes, display, and container lifecycle
├── config/
│   ├── camchain.yaml            # Stereo camera intrinsics & extrinsics
│   ├── imu.yaml                 # IMU noise parameters
│   └── aprilgrid.yaml           # Calibration target definition
├── data/                        # ← Shared volume (host ↔ container at /data)
│   ├── calibration_bag_ros2/    #   Raw ROS 2 bag (recorded on host)
│   ├── calibration_bag_ros1.bag #   Converted ROS 1 bag
│   └── results/                 #   Kalibr output: YAMLs, PDFs, reports
└── README.md
```

> 📂 **The `data/` folder is the single source of truth.** Everything you record, convert, or calibrate lives here — accessible from both your host machine and the container at `/data` with no manual copying required.

---

## 1. Dockerfile

The image is built on **Ubuntu 20.04** and includes ROS Noetic, all Kalibr Python/C++ dependencies, and a fully compiled Kalibr catkin workspace.

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

---

## 2. Docker Compose Setup

Instead of long `docker run` commands, Docker Compose handles volumes, display forwarding, and networking in a single declarative file.

### `docker-compose.yml`

```yaml
services:
  kalibr:
    build:
      context: .
      dockerfile: Dockerfile
    image: kalibr_container
    container_name: kalibr

    # Keep the container interactive
    stdin_open: true
    tty: true

    # Use host networking so ROS topics resolve correctly
    network_mode: host

    environment:
      - DISPLAY=${DISPLAY}
      - QT_X11_NO_MITSHM=1

    volumes:
      # --- Shared data volume (bags, configs, results) ---
      # Host: ./data/   <-->   Container: /data
      - ./data:/data

      # --- Config files (read-only inside container) ---
      # Host: ./config/   <-->   Container: /config
      - ./config:/config:ro

      # --- X11 socket for GUI display ---
      - /tmp/.X11-unix:/tmp/.X11-unix:rw

    # Always land in /data so you can run Kalibr commands immediately
    working_dir: /data
```

### Volume Mapping at a Glance

| Host path | Container path | Mode | Purpose |
|---|---|---|---|
| `./data/` | `/data` | read-write | Bags, converted bags, calibration results |
| `./config/` | `/config` | read-only | YAML config files |
| `/tmp/.X11-unix` | `/tmp/.X11-unix` | read-write | GUI display passthrough |

> 🔑 **Key point:** Any file written by Kalibr inside `/data` (YAML results, PDF reports) appears **immediately** in `./data/` on your host. No `docker cp` needed.

### Initial Setup

Create the shared directories before first use:

```bash
mkdir -p data/results
```

Your project should now look like:

```
kalibr-docker/
├── Dockerfile
├── docker-compose.yml
├── config/
│   ├── camchain.yaml
│   ├── imu.yaml
│   └── aprilgrid.yaml
└── data/
    └── results/        # Kalibr will write output here
```

### Build and Start

```bash
# Allow Docker to access your X display (run once per session)
xhost +local:docker

# Build the image and start the container
docker compose up --build -d

# Open a shell inside the running container
docker compose exec kalibr bash
```

> ⏳ First build takes ~15–25 minutes. Subsequent builds use the Docker layer cache.

**Other useful Compose commands:**

```bash
# Stop the container without removing it
docker compose stop

# Stop and remove the container (image is preserved)
docker compose down

# Rebuild from scratch (e.g. after Dockerfile changes)
docker compose build --no-cache
```

---

## 3. Recording Data in ROS 2

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

> 💡 **Tip:** Set `infra_fps:=15` to keep the image rate low enough for Kalibr's feature extraction — 20+ fps can cause issues.

### Step 2 — Record the Bag

Record directly into the shared `data/` folder so it's immediately available in the container:

```bash
ros2 bag record -o ./data/calibration_bag_ros2 \
    /camera/camera/infra1/image_rect_raw \
    /camera/camera/infra1/camera_info \
    /camera/camera/infra2/image_rect_raw \
    /camera/camera/infra2/camera_info \
    /camera/camera/gyro/sample \
    /camera/camera/accel/sample
```

### Recording Tips

> 🎯 **Motion pattern matters.** Move the camera in all 6 degrees of freedom — translate in X, Y, Z and rotate around all three axes. Keep each motion slow, smooth, and deliberate.
>
> ⏱️ **Duration:** 60–90 seconds is typically sufficient.
>
> 📐 **Field of view:** Keep the full AprilGrid visible in **both** infrared frames throughout the entire recording.

---

## 4. Converting ROS 2 Bag to ROS 1

Kalibr requires a ROS 1 `.bag` file. Run `rosbags-convert` on the **host** (not inside the container).

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
  --src ./data/calibration_bag_ros2 \
  --dst ./data/calibration_bag_ros1.bag \
  --dst-storage sqlite3 \
  --dst-typestore ros1_noetic
```

### Verify the Converted Bag

```bash
rosbag info ./data/calibration_bag_ros1.bag
```

You should see all recorded topics listed with message counts and duration. The converted `.bag` is now visible inside the container at `/data/calibration_bag_ros1.bag`.

---

## 5. Preparing Config Files

Copy your config YAMLs into the `config/` folder. They are mounted read-only inside the container at `/config/`:

```
config/                     →   /config/  (inside container)
├── camchain.yaml           →   /config/camchain.yaml
├── imu.yaml                →   /config/imu.yaml
└── aprilgrid.yaml          →   /config/aprilgrid.yaml
```

Your full `data/` folder before running calibration:

```
data/
├── calibration_bag_ros2/        # Raw ROS 2 bag (source)
├── calibration_bag_ros1.bag     # Converted ROS 1 bag (input to Kalibr)
└── results/                     # Empty for now — Kalibr writes here
```

---

## 6. Running Calibration

Open a shell inside the running container:

```bash
docker compose exec kalibr bash
```

All commands below are run **inside the container**. Your data is at `/data` and configs at `/config`.

### Option A — Stereo Camera Calibration

```bash
rosrun kalibr kalibr_calibrate_cameras \
    --bag /data/calibration_bag_ros1.bag \
    --topics /camera/infra1/image_rect /camera/infra2/image_rect \
    --models pinhole-equi pinhole-equi \
    --target /config/aprilgrid.yaml \
    --output-dir /data/results
```

### Option B — Camera–IMU Calibration

```bash
rosrun kalibr kalibr_calibrate_imu_camera \
   --bag /data/calibration_bag_ros1.bag \
   --cam /config/camchain.yaml \
   --imu /config/imu.yaml \
   --target /config/aprilgrid.yaml \
   --timeoffset-padding 0.1 \
   --show-extraction \
   --output-dir /data/results
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

Results are written to `/data/results/` inside the container, which maps directly to `./data/results/` on your host:

| File | Description |
|---|---|
| `*-camchain.yaml` | Calibrated camera chain (intrinsics + extrinsics) |
| `*-imu.yaml` | Calibrated IMU–camera transform |
| `*-results-imucam.txt` | Human-readable calibration report |
| `*-report-imucam.pdf` | Visual PDF report with residual plots |

> ✅ All output is instantly available on your host in `./data/results/` — no file transfer needed.

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

> ⚠️ **Important:** These are generic RealSense values. For best accuracy, replace with values from your camera's datasheet or measure them with [imu_utils](https://github.com/gaowenliang/imu_utils) (Allan Deviation analysis).

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

Run `xhost +local:docker` on your host **before** starting the container. Ensure `$DISPLAY` is set:

```bash
echo $DISPLAY   # should return :0 or :1, not empty
```

</details>

<details>
<summary><b>❌ Permission denied writing to <code>./data/</code></b></summary>

The container runs as root. If files created inside the container are unreadable on the host, fix ownership with:

```bash
sudo chown -R $USER:$USER ./data/
```

</details>

<details>
<summary><b>❌ Kalibr fails with "No corners detected"</b></summary>

- Ensure the full AprilGrid is visible in **both** camera frames.
- Reduce infrared fps to `15` or lower.
- Check that `tagSize` and `tagSpacing` in `aprilgrid.yaml` match your **printed** target exactly (measure with a ruler).
- Improve lighting — infrared works best without strong direct sunlight or IR reflections.

</details>

<details>
<summary><b>❌ IMU topic mismatch inside container</b></summary>

The IMU topic recorded in your `.bag` must match `rostopic` in `imu.yaml`. Inspect topics with:

```bash
rosbag info /data/calibration_bag_ros1.bag
```

</details>

<details>
<summary><b>❌ <code>docker compose</code> command not found</b></summary>

You may have the older standalone `docker-compose` (V1). Install the V2 plugin:

```bash
sudo apt install docker-compose-plugin
```

Or replace `docker compose` with `docker-compose` throughout.

</details>

---

## Workflow Summary

```
kalibr-docker/
│
│  HOST MACHINE
│
├─ 1. ros2 launch realsense2_camera      ──► camera streams
│
├─ 2. ros2 bag record -o ./data/...      ──► data/calibration_bag_ros2/
│
├─ 3. rosbags-convert                    ──► data/calibration_bag_ros1.bag
│
├─ 4. docker compose up --build -d       ──► container starts, volumes mounted
│
│   ┌────────────────────────────────────────────────────────────┐
│   │  DOCKER CONTAINER                                          │
│   │                                                            │
│   │  /data    <──── bind mount ────>  ./data/    (host)       │
│   │  /config  <──── bind mount ────>  ./config/  (host, r/o)  │
│   │                                                            │
│   │  5. kalibr_calibrate_imu_camera                           │
│   │       reads:   /data/calibration_bag_ros1.bag             │
│   │                /config/camchain.yaml                       │
│   │                /config/imu.yaml                            │
│   │                /config/aprilgrid.yaml                      │
│   │       writes:  /data/results/*-camchain.yaml               │
│   │                /data/results/*-report-imucam.pdf           │
│   │                /data/results/*-results-imucam.txt          │
│   │                                                            │
│   └────────────────────────────────────────────────────────────┘
│
└─ Results instantly visible in ./data/results/ on host ✅
```

---

## Credits & References

- [Kalibr — ETH ASL](https://github.com/ethz-asl/kalibr)
- [rosbags — ternaris](https://github.com/rpng/rosbags)
- [Intel RealSense ROS 2 Wrapper](https://github.com/IntelRealSense/realsense-ros)
- [Kalibr Wiki — Camera-IMU Calibration](https://github.com/ethz-asl/kalibr/wiki/camera-imu-calibration)
- [imu_utils — Allan Deviation IMU Calibration](https://github.com/gaowenliang/imu_utils)

---

<div align="center">
  <sub>Built for roboticists who don't want ROS 1 on their machine.</sub>
</div>
