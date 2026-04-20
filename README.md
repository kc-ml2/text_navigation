<img width="1606" height="720" alt="navigation" src="https://github.com/user-attachments/assets/343c7657-07c0-48ed-8cad-3dfa2e2a810d" /><img width="720" height="323" alt="demo" src="https://github.com/user-attachments/assets/03e97221-d4fb-4d46-8980-7f327d2e4d07" /># TextMap

Text-driven indoor navigation with OCR-based landmark SLAM for ROS 2.

The robot detects text signs (room names, directions, store fronts) with NavOCR, builds a 3D landmark map on top of SLAM, and then navigates on text commands such as `Kitchen` or `Exit` by converting them into Nav2 goals.

---

## Overview

### Landmark mapping (textmap)

<p align="center">
  <img src="https://github.com/user-attachments/assets/4b2efade-f421-4705-906c-e9283a93f19f" width="1440" height="646" alt="textmap landmark SLAM" />
</p>

NavOCR detects text on the RGB frame. `textmap` lifts each detection into 3D using the depth image, associates it across frames on a spatial grid, and publishes the result as a persistent landmark map anchored to the SLAM pose graph.

### Text-command navigation (text_nav_bridge)

<p align="center">
  <img src="https://github.com/user-attachments/assets/7a0b3d72-e3d5-430b-99e8-16d84c3f8eb3" width="1606" height="720" alt="text-command navigation" />
</p>

Given a text command, `text_nav_bridge` finds the closest-matching landmark from the saved map, ray-marches the robot-to-landmark line on the Nav2 costmap to pick a free-space goal, and sends it to Nav2 as a `NavigateToPose` action.

Both GIFs above are recorded in the Gazebo simulation shipped with this repo — the same pipeline reproduces the pipeline's behavior on real hardware (see [Real Environment](#real-environment)).

---

## Quick Start (Simulation)

End-to-end pipeline in Gazebo: build a map with text landmarks, then drive to them by name. No real hardware required.

### 1. Prerequisites

ROS 2 Humble on Ubuntu 22.04.

```bash
# Nav2 and the simulation SLAM backend
sudo apt install \
  ros-humble-navigation2 ros-humble-nav2-bringup \
  ros-humble-slam-toolbox

# Gazebo and TurtleBot3
sudo apt install \
  ros-humble-gazebo-ros-pkgs \
  ros-humble-turtlebot3-gazebo \
  ros-humble-turtlebot3-description \
  ros-humble-turtlebot3-teleop

# textmap build-time dependency (msg definitions only)
sudo apt install ros-humble-rtabmap-msgs

# NavOCR Python dependencies
pip install paddlepaddle==3.0.0 paddleocr==3.4.0
```

Set the TurtleBot3 model once per shell:

```bash
export TURTLEBOT3_MODEL=waffle
```

### 2. Clone and build

This repository is a colcon workspace umbrella — it pulls in the four packages (NavOCR, textmap, text_nav_bridge, text_nav_sim) as Git submodules. Clone recursively so `src/` is populated in one step.

```bash
git clone --recurse-submodules https://github.com/kc-ml2/text_navigation.git ~/ros2_ws
cd ~/ros2_ws

source /opt/ros/humble/setup.bash
colcon build
source install/setup.bash
```

Already cloned without `--recurse-submodules`? Initialize the submodules after the fact:

```bash
cd ~/ros2_ws
git submodule update --init --recursive
```

### 3. Phase 1 — Mapping

Launch the simulation with SLAM, NavOCR, and `textmap`:

```bash
ros2 launch text_nav_sim sim_mapping.launch.py
```

This brings up Gazebo (house world with text signs), `slam_toolbox` in async mapping mode, `navocr_node`, `textmap_node`, and RViz.

Drive the robot manually in another terminal:

```bash
ros2 run turtlebot3_teleop teleop_keyboard
```

When you are done exploring, **before** pressing `Ctrl+C`, save the occupancy grid (the launch file prints the exact command with a timestamped directory):

```bash
mkdir -p ~/map/sim_run

# Occupancy grid for AMCL (.pgm + .yaml)
ros2 run nav2_map_server map_saver_cli -f ~/map/sim_run/map \
  --ros-args -p map_subscribe_transient_local:=true
```

Press `Ctrl+C` to stop. `textmap` writes `landmarks.yaml` to `~/map/sim_run/` automatically on shutdown.

You should now have:

```
~/map/sim_run/
  map.pgm           # occupancy grid image
  map.yaml          # map_server metadata
  landmarks.yaml    # detected text landmarks with 3D positions
```

### 4. Phase 2 — Navigation

Launch Nav2 with AMCL localization on the saved map and landmarks:

```bash
ros2 launch text_nav_sim sim_navigation.launch.py \
  landmark_file:=~/map/sim_run/landmarks.yaml \
  map_yaml_file:=~/map/sim_run/map.yaml
```

This brings up Gazebo, `map_server + AMCL + lifecycle_manager`, the Nav2 stack, `text_nav_bridge_node`, and RViz.

Send a text command — it will be matched against the landmarks captured in Phase 1 and converted into a Nav2 goal:

```bash
ros2 topic pub --once /text_nav/command std_msgs/msg/String "data: 'Kitchen'"
```

Monitor progress:

```bash
ros2 topic echo /text_nav/status
```

Try other landmarks from the sim world: `Bedroom`, `Bathroom`, `Office`, `Exit`, `Living Room`, `Laundry`, `Storage Room`, `Garage`, `Closet`.

Pose graph localization (using the slam_toolbox `.posegraph`) is planned for a later release; AMCL on the occupancy grid is currently the only supported backend in simulation.

### Troubleshooting

**`spawn_entity.py` hangs or `odom` frame does not exist.** A previous Gazebo process is still alive. Kill it before relaunching:

```bash
killall -9 gzserver gzclient; pkill -9 -f ros2
```

**`FLAGS_enable_pir_*` / oneDNN errors from NavOCR on CPU.** Set these before launching:

```bash
export FLAGS_enable_pir_api=0
export FLAGS_enable_pir_in_executor=0
```

**Conda `(base)` breaks `spawn_entity.py`.** Deactivate conda before launching — Gazebo expects system Python 3.10:

```bash
conda deactivate
```

---

## Real Environment

The same pipeline runs on a physical robot. The simulation and real-hardware flows differ in three ways: (1) the SLAM backend is RTAB-Map instead of slam_toolbox, (2) you need a real RGB-D camera and IMU driver, and (3) you run the navigation stack against an RTAB-Map database (`.db`) rather than an occupancy grid + pose graph.

### Hardware used

This project was tested on an Intel RealSense D455 (RGB-D + built-in IMU) mounted on a mobile base. Any RGB-D + IMU sensor with a ROS 2 driver should work with parameter tweaks to `textmap` (camera frame names, depth topic, camera_info topic).

### Additional prerequisites

On top of the simulation prerequisites, install:

```bash
sudo apt install \
  ros-humble-rtabmap-ros \
  ros-humble-imu-filter-madgwick \
  ros-humble-realsense2-camera
```

### Mapping (real-time)

The default flow is to run the camera driver and `textmap` live while you drive the robot.

```bash
# Terminal 1 — RealSense driver
ros2 launch realsense2_camera rs_launch.py \
  enable_infra1:=true enable_infra2:=true \
  enable_depth:=true enable_gyro:=true enable_accel:=true \
  unite_imu_method:=2

# Terminal 2 — textmap + RTAB-Map SLAM + NavOCR
ros2 launch textmap textmap.launch.py \
  landmark_save_path:=~/map/real_run/landmarks.yaml
```

Drive the robot around the environment. On shutdown (`Ctrl+C` on terminal 2), `textmap` writes `landmarks.yaml` automatically; RTAB-Map writes its database to `~/.ros/rtabmap.db` by default (move it next to the landmark file for the navigation step).

To save landmarks manually while the node is still running:

```bash
ros2 service call /textmap/save_landmarks std_srvs/srv/Trigger
```

<details>
<summary><strong>Alternative: record a rosbag and map from it offline</strong></summary>

If you prefer to separate data collection from mapping, record the RGB-D + IMU + TF topics and play them back later:

```bash
# Record
ros2 bag record -o my_run \
  /camera/color/image_raw /camera/color/camera_info \
  /camera/depth/image_rect_raw /camera/infra1/camera_info \
  /camera/imu /tf /tf_static

# Later: map from the bag
ros2 launch textmap textmap.launch.py \
  landmark_save_path:=~/map/real_run/landmarks.yaml &
ros2 bag play my_run --clock -r 1.0
```

This is useful when you want to re-run mapping with different `textmap` parameters without re-collecting data.

</details>

### Navigation (real-time)

`text_nav_bridge`'s launch resolves both the landmark file and the RTAB-Map database from a single `bag_name` argument, which looks up:

- `src/text_nav_bridge/landmarks/<bag_name>.yaml`
- `src/text_nav_bridge/rtabmap_db/<bag_name>.db`

Copy or symlink the outputs from the mapping step into those directories, then launch the navigation stack and the camera driver:

```bash
# Terminal 1 — Nav2 + RTAB-Map localization + text_nav_bridge
ros2 launch text_nav_bridge text_nav.launch.py bag_name:=real_run

# Terminal 2 — RealSense driver (same as mapping)
ros2 launch realsense2_camera rs_launch.py \
  enable_infra1:=true enable_infra2:=true \
  enable_depth:=true enable_gyro:=true enable_accel:=true \
  unite_imu_method:=2
```

Send a text command:

```bash
ros2 topic pub --once /text_nav/command std_msgs/msg/String "data: 'restroom'"
ros2 topic echo /text_nav/status
```

<details>
<summary><strong>Alternative: replay a rosbag instead of running the camera</strong></summary>

If you have a recorded session, swap the camera driver for rosbag playback and keep the navigation launch the same:

```bash
# Terminal 1 — same as above
ros2 launch text_nav_bridge text_nav.launch.py bag_name:=real_run

# Terminal 2 — rosbag replay
ros2 bag play my_run --clock
```

</details>

### Sample rosbag

A reference rosbag recorded on our setup is available for download:

- **Link:** *TBD — to be added*

The bag corresponds to `bag_name:=TBD` in the navigation launch and can be used to reproduce the `text_nav_bridge` GIF above without physical hardware.

---

## Packages

### This repository

| Package | Role | Repository |
|---|---|---|
| NavOCR | Text detection + OCR (PaddleDetection + PaddleOCR) | [kc-ml2/NavOCR](https://github.com/kc-ml2/NavOCR/tree/add-openvino) |
| textmap | 3D text landmark SLAM (NavOCR + depth + SLAM pose graph) | [kc-ml2/TextMap](https://github.com/kc-ml2/TextMap/tree/slamtoolbox) |
| text_nav_bridge | Text command to Nav2 `NavigateToPose` goal | [kc-ml2/text_nav_bridge](https://github.com/kc-ml2/text_nav_bridge) |
| text_nav_sim | Gazebo simulation (TurtleBot3 + text signs) | [kc-ml2/text_nav_sim](https://github.com/kc-ml2/text_nav_sim) |

### Third-party runtime dependencies

| Package | Role | Upstream |
|---|---|---|
| Nav2 | Navigation stack (planner, controller, recovery) | https://github.com/ros-navigation/navigation2 |
| slam_toolbox | 2D SLAM backend used by the simulation | https://github.com/SteveMacenski/slam_toolbox |
| rtabmap_ros | 3D SLAM backend used on real hardware | https://github.com/introlab/rtabmap_ros |

---

## License

Apache License 2.0. See `LICENSE` for the full text and per-package notices.
