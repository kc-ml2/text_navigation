# text navigation 런치 재편 계획

## 배경
- rtabmap 계열(실제 로봇): `textmap/textmap_rtabmap.launch.py`(mapping) + `text_nav_bridge/text_nav.launch.py`(navigation) — rosbag/카메라와 분리
- 시뮬 계열(현재): `text_nav_sim/sim_mapping.launch.py`, `sim_navigation.launch.py`에 Gazebo + mapping/navigation이 한 묶음 — 대칭 깨짐
- 게다가 4월 20일 `c86fb7c`(textmap) + `17a2892`(text_nav_sim) 콤보 커밋으로 시뮬 mapping이 깨진 상태
  - `textmap_slam_toolbox.yaml` 삭제됐지만 `text_nav_sim`으로 이관 안 됨
  - `sim_mapping.launch.py`가 실제 로봇용 기본값인 `textmap_tf_only.yaml`을 쓰면서 `camera_info_topic`/`camera_frame` 오버라이드 누락
  - 결과: camera_info 수신 실패 → detection 전량 폐기 → landmark 0건

## 목표 구조
```
text_nav_sim/launch/
  simulation.launch.py            (신규: Gazebo + RSP + spawn + optical static_tf)

textmap/launch/
  textmap_rtabmap.launch.py       (유지)
  textmap_sim.launch.py           (신규: NavOCR + textmap_node + slam_toolbox + RViz)

text_nav_bridge/launch/
  text_nav.launch.py              (유지, rtabmap localization)
  text_nav_sim.launch.py          (신규: map_server + AMCL + Nav2 + bridge + RViz)
```

## 이관 목록 (순환 의존 회피 목적)
| From | To |
|---|---|
| `text_nav_sim/config/slam_toolbox_mapping.yaml` | `textmap/config/slam_toolbox_mapping.yaml` |
| `text_nav_sim/config/slam_toolbox_localization.yaml` | `text_nav_bridge/config/slam_toolbox_localization.yaml` |
| `text_nav_sim/config/nav2_sim_params.yaml` | `text_nav_bridge/config/nav2_sim_params.yaml` |
| `text_nav_sim/rviz/sim_mapping.rviz` | `textmap/rviz/textmap_sim.rviz` |
| `text_nav_sim/rviz/sim_navigation.rviz` | `text_nav_bridge/rviz/text_nav_sim.rviz` |
| (부활) | `textmap/config/textmap_sim.yaml` — 삭제된 textmap_slam_toolbox.yaml 값 기반 |

## 부활시킬 textmap_sim.yaml 핵심 값
```yaml
camera_info_topic: "/camera/rgbd_camera/camera_info"  # (런타임 확인 필요)
camera_frame: "camera_rgb_optical_frame"
depth_topic: "/camera/rgbd_camera/depth/image_raw"
# 시뮬 튜닝값
confidence_threshold: 0.35
sensor_noise_std: 0.2
min_observations: 3
chi2_threshold: 11.34
min_confidence_for_output: 0.45
min_depth_mm: 300
merge_distance_threshold: 2.0
max_angular_velocity: 0.3
```

## 삭제 대상
- `text_nav_sim/launch/sim_mapping.launch.py`
- `text_nav_sim/launch/sim_navigation.launch.py`
- 위의 이관 완료 후 원본 config/rviz 파일

## package.xml 조정
- `textmap/package.xml`: `<exec_depend>slam_toolbox</exec_depend>` 추가
- `text_nav_bridge/package.xml`: `nav2_bringup`, `nav2_map_server`, `nav2_amcl`, `nav2_lifecycle_manager` 추가
- `text_nav_sim/package.xml`: `navocr`, `textmap`, `text_nav_bridge`, `slam_toolbox`, `nav2_*`, `teleop_twist_keyboard` 제거 후 실제 사용만 유지

## CMakeLists.txt 조정
- `textmap`: `install(DIRECTORY launch config rviz ...)` — rviz 디렉토리 추가 (현재 없으면)
- `text_nav_bridge`: 동일
- `text_nav_sim`: rviz 디렉토리 제거 (이관 후)

## LaunchArgument (최소 노출 방침)
- `simulation.launch.py`: `use_sim_time`, `world_file`, `spawn_x`, `spawn_y`, `spawn_z`, `spawn_yaw`
- `textmap_sim.launch.py`: `use_sim_time`, `landmark_save_path`, `navocr_params_file`, `launch_navocr`, `launch_rviz`
- `text_nav_bridge/text_nav_sim.launch.py`: `use_sim_time`, `map_yaml_file`, `landmark_file`, `match_threshold`

## 실행 순서 (재편 후 사용 흐름)
```
터미널 1: ros2 launch text_nav_sim simulation.launch.py
터미널 2: ros2 launch textmap textmap_sim.launch.py
(또는)   ros2 launch text_nav_bridge text_nav_sim.launch.py map_yaml_file:=... landmark_file:=...
```

## 동반 작업 보류 (별도 PR)
- `approach_distance` unused 파라미터 제거
- textmap README의 `slam_backend: "slam_toolbox"` 표기 정정
- `text_nav_sim/package.xml`의 완전한 unused exec_depend 정리

## 구현 단계
1. yaml/rviz 이관 (복사 + 원본 삭제)
2. `textmap_sim.yaml` 부활
3. 신규 런치 3개 작성
4. 기존 sim_*.launch.py 삭제
5. package.xml / CMakeLists.txt 조정
6. colcon build 빌드 검증
7. 실제 런치 실행 + camera_info 수신 + landmark 생성 검증
8. 세 패키지 README 업데이트
