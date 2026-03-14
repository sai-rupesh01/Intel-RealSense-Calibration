Kalibr-Docker: Visual-Inertial Calibration for ROS 2 Users🚀 OverviewMost modern robotics stacks run on ROS 2, but the industry-standard calibration tool, Kalibr, operates best within a ROS 1 Noetic environment.This repository provides a "zero-install" solution using Docker to run Kalibr. It includes a streamlined workflow to record data in ROS 2, convert it to ROS 1 format, and perform high-precision Stereo Camera and IMU calibration.🛠️ The Workflow at a GlanceRecord: Collect raw sensor data using ROS 2.Convert: Use rosbags-convert to transform .mcap/.db3 files into ROS 1 .bag files.Configure: Set up your yaml files for the target, camera, and IMU.Calibrate: Run the Kalibr container and generate your calibration report.1. Prerequisites & InstallationHost Machine SetupYou will need Docker and the rosbags conversion tool installed on your host.Bash# Install the conversion tool
pip install rosbags

# Ensure the local bin is in your PATH
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
Build the Docker ImageClone this repo and build the container which contains ROS Noetic and Kalibr pre-installed.Bashdocker build -t kalibr_container .
2. Data Collection (ROS 2)Launch your camera (example for Intel RealSense) with specific parameters to ensure raw, unrectified streams and IMU data are active.Bash# Launch Camera
ros2 launch realsense2_camera rs_launch.py \
    enable_color:=false \
    enable_depth:=false \
    enable_infra1:=true \
    enable_infra2:=true \
    enable_accel:=true \
    enable_gyro:=true \
    enable_emitter:=false \
    infra_fps:=15

# Record the Bag
ros2 bag record -o ~/kalibr_data/raw_data \
    /camera/camera/infra1/image_rect_raw \
    /camera/camera/infra2/image_rect_raw \
    /camera/camera/gyro/sample \
    /camera/camera/accel/sample
3. Format ConversionConvert the ROS 2 data into a ROS 1 .bag file that Kalibr can read.Bashrosbags-convert \
  --src ~/kalibr_data/raw_data \
  --dst ~/kalibr_data/calibration_ros1.bag \
  --dst-storage sqlite3 \
  --dst-typestore ros1_noetic
4. Configuration TemplatesPlace these .yaml files in your ~/kalibr_data/ folder.<details><summary><b>Click to expand: aprilgrid.yaml</b></summary>YAMLtarget_type: 'aprilgrid'
tagCols: 8
tagRows: 6
tagSize: 0.08
tagSpacing: 0.3125
</details><details><summary><b>Click to expand: imu.yaml</b></summary>YAMLrostopic: /camera/camera/imu
update_rate: 200.0
accelerometer_noise_density: 0.02
accelerometer_random_walk: 0.0002
gyroscope_noise_density: 0.02
gyroscope_random_walk: 0.0002
</details><details><summary><b>Click to expand: camchain.yaml</b></summary>YAMLcam0:
  cam_overlaps: [1]
  camera_model: pinhole
  distortion_coeffs: [0.0, 0.0, 0.0, 0.0]
  distortion_model: equi
  intrinsics: [424.0, 424.0, 424.0, 240.0]
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
  T_cn_cnm1:
    - [1.0, 0.0, 0.0, -0.05]
    - [0.0, 1.0, 0.0, 0.0]
    - [0.0, 0.0, 1.0, 0.0]
    - [0.0, 0.0, 0.0, 1.0]
</details>5. Running CalibrationStep A: Grant GUI PermissionsTo see the feature extraction in real-time, allow Docker to access your X11 server:Bashxhost +local:docker
Step B: Launch the ContainerMount your data directory to the /data volume inside the container.Bashsudo docker run -it \
   --net=host \
   -e DISPLAY=$DISPLAY \
   -e QT_X11_NO_MITSHM=1 \
   -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
   -v ~/kalibr_data/:/data \
   kalibr_container
Step C: Execute KalibrRun the visual-inertial calibration command:Bashrosrun kalibr kalibr_calibrate_imu_camera \
   --bag /data/calibration_ros1.bag \
   --cam /data/camchain.yaml \
   --imu /data/imu.yaml \
   --target /data/aprilgrid.yaml \
   --timeoffset-padding 0.1 \
   --show-extraction
📊 Interpreting ResultsAfter the process completes, Kalibr generates three files in your /data folder:Report (PDF): Contains reprojection error plots. Aim for a mean error of < 0.15 pixels.Results (YAML): Contains the final transformation matrix ($T_{bc}$) and refined intrinsics.Camchain (YAML): The updated camera model for use in SLAM/VIO.
