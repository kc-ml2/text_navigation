## Quick Start (Real-world, RTAB-Map + RealSense D455)

Text landmark mapping with a live Intel RealSense D455: build an RTAB-Map SLAM database and save detected text landmarks.

### 1. Prerequisites

This tutorial assumes ROS 2 is already installed. It is primarily tested on ROS 2 `humble`, but it is designed to work across ROS 2 distributions.

```bash
# Make sure your ROS 2 environment is sourced.
echo $ROS_DISTRO

# RTAB-Map SLAM, IMU filter, and RealSense driver
sudo apt install \
  ros-${ROS_DISTRO}-rtabmap-ros \
  ros-${ROS_DISTRO}-imu-filter-madgwick \
  ros-${ROS_DISTRO}-realsense2-camera

# (Optional) if you are using python venv
python3 -m venv ~/.venvs/textmap
source ~/.venvs/textmap/bin/activate
pip install --upgrade pip

# NavOCR runtime dependencies
pip install openvino pyyaml opencv-python numpy
```

### 2. Clone and build

Clone the real-world TextMap stack into your ROS 2 workspace. `text_nav_sim` is intentionally omitted here because this guide uses RTAB-Map and a real camera.

```bash
cd ~/ros2_ws/src  # your ros2 workspace

git clone git@github.com:kc-ml2/NavOCR.git
git clone git@github.com:kc-ml2/TextMap.git

cd ~/ros2_ws
source /opt/ros/${ROS_DISTRO}/setup.bash

rosdep update
rosdep install --from-paths src --ignore-src -r -y
colcon build

# (Optional) if you are using python venv
python -m colcon build --symlink-install

source install/setup.bash
```

### 3. Launch RealSense Camera

Terminal 1, launch RealSense camera with infra/depth/IMU streams.

```bash
ros2 launch realsense2_camera rs_launch.py \
  camera_namespace:='' \
  enable_infra1:=true \
  enable_infra2:=true \
  enable_depth:=true \
  enable_gyro:=true \
  enable_accel:=true \
  unite_imu_method:=2 \
  enable_sync:=true \
  depth_module.emitter_enabled:=0
```

### 4. SLAM & Text Landmark Mapping

On the other terminal, launch TextMap with the RTAB-Map SLAM backend:

```bash
mkdir -p ~/map/real_rtabmap_run

ros2 launch textmap textmap_rtabmap.launch.py \
  use_sim_time:=false \
  landmark_save_path:=~/map/real_rtabmap_run/landmarks.yaml
```

Explore with camera or drive the robot around the environment. `NavOCR` detects text in the camera images, `rtabmap` builds the SLAM map, and `textmap` anchors the detected text as 3D landmarks.

When mapping is finished, press `Ctrl+C` in the TextMap terminal. `textmap` writes `landmarks.yaml` automatically, and RTAB-Map writes its database to `~/.ros/rtabmap.db` by default.

You should now have:

```text
~/map/real_rtabmap_run/
  landmarks.yaml      # detected text landmarks with 3D positions

~/.ros/
  rtabmap.db          # RTAB-Map SLAM database
```

### Troubleshooting

**`rtabmap` or `rgbd_odometry` does not receive camera data.** Check the D455 topics and adjust the launch arguments if your camera namespace differs:

```bash
ros2 topic list | grep camera

ros2 launch textmap textmap_rtabmap.launch.py \
  use_sim_time:=false \
  depth_topic:=/camera/depth/image_rect_raw \
  rgb_topic:=/camera/infra1/image_rect_raw \
  rgb_info_topic:=/camera/infra1/camera_info \
  camera_info_topic:=/camera/infra1/camera_info \
  imu_topic:=/camera/imu \
  landmark_save_path:=~/map/real_rtabmap_run/landmarks.yaml
```

**`FLAGS_enable_pir_*` / oneDNN errors from NavOCR on CPU.** `textmap_rtabmap.launch.py` already injects the common workarounds. If you run NavOCR separately, set:

```bash
export FLAGS_enable_pir_api=0
export FLAGS_enable_pir_in_executor=0
export FLAGS_use_mkldnn=0
```

**Conda `(base)` breaks ROS 2 Python packages.** Deactivate conda before launching. We recommend using `venv` for compatibility with ROS 2.
