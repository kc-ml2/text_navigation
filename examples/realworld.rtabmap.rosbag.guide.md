## Quick Start (Real-world, RTAB-Map + Rosbag)

Text landmark mapping from a recorded stereo camera rosbag: replay the bag to build an RTAB-Map SLAM database and save detected text landmarks.

This guide uses `rtabmap` as the SLAM backend.

### 1. Prerequisites

This tutorial assumes ROS 2 is already installed. It is primarily tested on ROS 2 `humble`, but it is designed to work across ROS 2 distributions.

```bash
# Make sure your ROS 2 environment is sourced.
echo $ROS_DISTRO

# RTAB-Map SLAM and IMU filter
sudo apt install \
  ros-${ROS_DISTRO}-rtabmap-ros \
  ros-${ROS_DISTRO}-imu-filter-madgwick

# (Optional) if you are using python venv
python3 -m venv ~/.venvs/textmap
source ~/.venvs/textmap/bin/activate
pip install --upgrade pip

# NavOCR runtime dependencies
pip install openvino pyyaml opencv-python numpy
```

### 2. Clone and build

Clone the real-world TextMap stack into your ROS 2 workspace. `text_nav_sim` is intentionally omitted here because this guide uses RTAB-Map and rosbag playback.

```bash
cd ~/ros2_ws  # your ros2 workspace
cd src/

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

### 3. Download rosbag file

Download example rosbag under a local dataset directory.
This rosbag file was recorded with Realsense D455 camera.

```bash
cd ~/ros2_ws/
pip install gdown==5.2.0
gdown https://drive.google.com/uc?id=1Zp2PsnTVyCEnEJABwPb5UDFVyB12FFg2
unzip rosbag_demo.zip
```

### 4. SLAM & Text Landmark Mapping

Launch TextMap with the RTAB-Map SLAM backend:

```bash
mkdir -p ~/map/real_rtabmap_bag

ros2 launch textmap textmap_rtabmap.launch.py \
  use_sim_time:=true \
  landmark_save_path:=~/map/real_rtabmap_bag/landmarks.yaml
```

On the other terminal, play the rosbag:

```bash
cd ~/ros2_ws/
ros2 bag play rosbag_demo --clock
```

`NavOCR` detects text in the replayed camera images, `rtabmap` builds the SLAM map, and `textmap` anchors the detected text as 3D landmarks.

When bag playback is finished, press `Ctrl+C` in the TextMap terminal. `textmap` writes `landmarks.yaml` automatically, and RTAB-Map writes its database to `~/.ros/rtabmap.db` by default.

You should now have:

```text
~/map/real_rtabmap_bag/
  landmarks.yaml      # detected text landmarks with 3D positions

~/.ros/
  rtabmap.db          # RTAB-Map SLAM database
```

### Troubleshooting

Make sure every node uses simulation time and the bag publishes `/clock`:

```bash
ros2 bag play rosbag_demo --clock
ros2 param get /textmap_node use_sim_time
```

**`rtabmap` or `rgbd_odometry` does not receive camera data.** Check the bag topics and adjust the launch arguments if your bag uses different names:

```bash
ros2 bag info rosbag_demo --clock

ros2 launch textmap textmap_rtabmap.launch.py \
  use_sim_time:=true \
  depth_topic:=/camera/depth/image_rect_raw \
  rgb_topic:=/camera/infra1/image_rect_raw \
  rgb_info_topic:=/camera/infra1/camera_info \
  camera_info_topic:=/camera/infra1/camera_info \
  imu_topic:=/camera/imu \
  landmark_save_path:=~/map/real_rtabmap_bag/landmarks.yaml
```

**`FLAGS_enable_pir_*` / oneDNN errors from NavOCR on CPU.** `textmap_rtabmap.launch.py` already injects the common workarounds. If you run NavOCR separately, set:

```bash
export FLAGS_enable_pir_api=0
export FLAGS_enable_pir_in_executor=0
export FLAGS_use_mkldnn=0
```

**Conda `(base)` breaks ROS 2 Python packages.** Deactivate conda before launching. We recommend using `venv` for compatibility with ROS 2.
