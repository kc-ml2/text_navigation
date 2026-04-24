![textnavigation_mapping__online-video-cutter com___1___1__360](https://github.com/user-attachments/assets/ee077fd4-7444-4bac-9b7f-03853c1c14f0)
# TextMap Example

TextMap is an open-source project for leveraging textual information across navigation pipeline, including SLAM, path planning, and goal commands, all on CPU.

TextMap comprises four modules:

| Package | Role | Repository |
|---|---|---|
| NavOCR | Navigation-relevant text detection & recognition | [kc-ml2/NavOCR](https://github.com/kc-ml2/NavOCR) |
| TextMap | Text landmark mapping (Adding text landmarks to the map during SLAM) | [kc-ml2/TextMap](https://github.com/kc-ml2/TextMap) |
| text_nav_bridge | Goal setting through text commands & Integration with Nav2 | [kc-ml2/text_nav_bridge](https://github.com/kc-ml2/text_nav_bridge) |
| text_nav_sim | Gazebo simulation with text signs (based on turtlebot3_simulation) | [kc-ml2/text_nav_sim](https://github.com/kc-ml2/text_nav_sim) |

## Examples

This repository provides examples of how to use TextMap.

| Environment | SLAM package | Text detection model format | Dataset | Coverage | Link |
|---|---|---|---|---|---|
| Gazebo simulation | slam_toolbox | OpenVINO | real-time simulation | Text mapping + Navigation | [example](#quickstart) |
| Gazebo simulation | slam_toolbox | ONNX | real-time simulation | Text mapping + Navigation | [example](#quickstart) |
| Real-world | rtabmap | | real-time Realsense D455 camera | Text mapping only | [example](examples/realworld.rtabmap.realsense_d455.guide.md) |
| Real-world | rtabmap | | rosbag dataset | Text mapping only | [example](examples/realworld.rtabmap.rosbag.guide.md) |


## Demo

### Text Landmark Mapping

`NavOCR` detects text in RGB camera images. `TextMap` then lifts each detection into a 3D SLAM map.

<p align="center">
  <img src="[https://github.com/user-attachments/assets/b3156f7b-a9f4-4d24-8045-b2fbb6aa6327](https://github.com/user-attachments/assets/8e0ff73e-1efb-43e4-a0dd-7cfc786dd80a)" width="1080" height="704" alt="textmap landmark SLAM" />
</p>


### Text-command Navigation (with `Nav2`)
Given a text command, `text_nav_bridge` finds the closest-matching landmark from the saved map, and sends it to Nav2 as a `NavigateToPose` action.

<p align="center">
  <img src="https://github.com/user-attachments/assets/55a5d4d0-0b4b-486e-9050-efdd3e11c806" width="1080" height="704" alt="text-command navigation" />
</p>

---

<a id="quickstart"></a>
## Quick Start (Simulation)

End-to-end TextMap pipeline in Gazebo: build a map with text landmarks, then drive to them by name.

### 1. Prerequisites

This tutorial is primarily tested on ROS 2 `humble`, but it is designed to work across ROS 2 distributions.

```bash
# Make sure your ROS 2 environment is sourced.
echo $ROS_DISTRO

# Nav2 and the simulation SLAM backend
sudo apt install \
  ros-${ROS_DISTRO}-navigation2 ros-${ROS_DISTRO}-nav2-bringup \
  ros-${ROS_DISTRO}-slam-toolbox

# Gazebo and TurtleBot3
sudo apt install \
  ros-${ROS_DISTRO}-gazebo-ros-pkgs \
  ros-${ROS_DISTRO}-turtlebot3-gazebo \
  ros-${ROS_DISTRO}-turtlebot3-description \
  ros-${ROS_DISTRO}-turtlebot3-teleop

# (Optional) if you are using python venv
python3 -m venv ~/.venvs/textmap
source ~/.venvs/textmap/bin/activate
pip install --upgrade pip
pip install colcon-common-extensions

# Prerequisite for NavOCR (openvino)
pip install openvino pyyaml opencv-python numpy
```

### 2. Clone and build

```bash
cd ~/ros2_ws/src  # your ros2 workspace

git clone git@github.com:kc-ml2/NavOCR.git
git clone git@github.com:kc-ml2/TextMap.git
git clone git@github.com:kc-ml2/text_nav_bridge.git
git clone git@github.com:kc-ml2/text_nav_sim.git

cd ..

source /opt/ros/${ROS_DISTRO}/setup.bash

rosdep update
rosdep install --from-paths src --ignore-src -r -y

colcon build

# (Optional) if you are using python venv
python -m colcon build --symlink-install

source install/setup.bash
```

### 3. Initialize Simulation

Gazebo simulation environment with text signs:

```bash
ros2 launch text_nav_sim simulation.launch.py
```

On the other terminal, run robot teleoperation:

```bash
ros2 run turtlebot3_teleop teleop_keyboard
```

### 4. SLAM & Text Landmark Mapping
On the other terminal, launch Textmap:

```bash
ros2 launch textmap textmap_slamtoolbox.launch.py \
  landmark_save_path:=~/map/sim_run/landmarks.yaml
```

`NavOCR` detects text in the camera images, `slam_toolbox` builds the SLAM map, and `textmap` anchors the detected text as 3D landmarks.

When you are done exploration and mapping, save the occupancy map **before** pressing `Ctrl+C` on the textmap terminal (`slam_toolbox` must still be running):

```bash
# Occupancy grid for AMCL (.pgm + .yaml)
ros2 run nav2_map_server map_saver_cli -f ~/map/sim_run/map \
  --ros-args -p map_subscribe_transient_local:=true
```

Then `Ctrl+C` the textmap terminal. `textmap` auto-exports `landmarks.yaml` on shutdown to the path given by `landmark_save_path`.

You should now have:

```
~/map/sim_run/
  map.pgm           # occupancy grid image
  map.yaml          # map_server metadata
  landmarks.yaml    # detected text landmarks with 3D positions
```

### 5. Navigation

Turn off text landmark mapping, and launch Nav2, amcl, and text_nav_bridge:

```bash
ros2 launch text_nav_bridge text_nav_sim.launch.py \
landmark_file:=$HOME/map/sim_run/landmarks.yaml \
map_yaml_file:=$HOME/map/sim_run/map.yaml
```

Send a text command — it will be matched against the landmarks and converted into a Nav2 goal:

```bash
ros2 topic pub --once /text_nav/command std_msgs/msg/String "data: 'Kitchen'"
```

Monitor progress:

```bash
ros2 topic echo /text_nav/status
```

Try other landmarks from the sim world: `Bedroom`, `Bathroom`, `Office`, `Exit`, `Living Room`, `Laundry`, `Storage Room`, `Garage`, `Closet`.


### Troubleshooting

#### `spawn_entity.py` hangs or `odom` frame does not exist.
A previous Gazebo process is still alive. Kill it before relaunching:

```bash
killall -9 gzserver gzclient; pkill -9 -f ros2
```

#### `FLAGS_enable_pir_*` / oneDNN errors from NavOCR on Intel CPU.
Set these before launching:

```bash
export FLAGS_enable_pir_api=0
export FLAGS_enable_pir_in_executor=0
```

#### Conda `(base)` breaks `spawn_entity.py`.
Deactivate conda before launching.
We recommend to use `venv` for compatibility wirh ROS 2.


## License

Apache License 2.0. See `LICENSE` for the full text and per-package notices.
