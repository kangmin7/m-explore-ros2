# m-explore-ros2 for PX4

Custom [m-explore-ros2](https://github.com/robo-friends/m-explore-ros2) for PX4 integration, targeting ROS 2 Humble

**Demo**:
[![Demo](https://img.youtube.com/vi/bG_B9l2mHzY/maxresdefault.jpg)](https://www.youtube.com/watch?v=bG_B9l2mHzY)

---

## Changes (Added Parameters)

### `goal_hysteresis`

- **Increasing it**: The robot becomes more committed to its current goal. It takes a larger cost advantage for a competing frontier to cause a goal switch. This reduces back-and-forth replanning and is especially useful on drones where replanning mid-flight is costly. Setting to `0.0` disables it.

### `distance_scale`

- **Increasing it**: The robot strongly prefers closer frontiers. Exploration becomes more conservative and local — the robot exhausts nearby unknowns before traveling far. On drones, this also helps conserve battery by minimizing long traversals. Setting to `0.0` disables it.

---

## Running in simulation (PX4 + Gazebo)

### 1. PX4 SITL

Repo: https://github.com/kangmin7/PX4-Autopilot

```bash
cd ~/PX4-Autopilot
make px4_sitl gz_x500_lidar_2d PX4_GZ_WORLD=husarion_office_empty
```

After PX4 boots, take off:
```
commander takeoff
```

### 2. ROS-Gazebo bridge

```bash
ros2 run ros_gz_bridge parameter_bridge /clock@rosgraph_msgs/msg/Clock[gz.msgs.Clock
ros2 run ros_gz_bridge parameter_bridge /scan@sensor_msgs/msg/LaserScan[gz.msgs.LaserScan
ros2 run tf2_ros static_transform_publisher \
  --x -0.1 --y 0 --z 0.26 \
  --qx 0 --qy 0 --qz 0 --qw 1 \
  --frame-id base_link \
  --child-frame-id link
ros2 param set /ros_gz_bridge use_sim_time true
```

### 3. MAVROS

Repo: https://github.com/kangmin7/mavros

```bash
ros2 launch mavros px4.launch fcu_url:="udp://:14540@127.0.0.1:14557" use_sim_time:=true
```

### 4. SLAM

Repo: https://github.com/kangmin7/slam_toolbox-px4

```bash
ros2 launch slam_toolbox online_async_launch.py use_sim_time:=true
```

### 5. Navigation

Repo: https://github.com/kangmin7/navigation2-px4

```bash
ros2 launch nav2_bringup px4_mavros_navigation_launch.py use_sim_time:=true
```

### 6. Exploration

```bash
ros2 launch explore_lite explore.launch.py
```

### 7. Offboard mode

```bash
ros2 service call /mavros/set_mode mavros_msgs/srv/SetMode "{custom_mode: 'OFFBOARD'}"
```

---

## Docker

A pre-built image with ROS 2 Humble, MAVROS, SLAM Toolbox, Nav2, and PX4 SITL is available on Docker Hub.

```bash
docker pull kangmin7/px4-ros2-humble:latest
```

Run the container:

```bash
docker run -it --rm \
  --network host \
  --privileged \
  -e DISPLAY=$DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  kangmin7/px4-ros2-humble:latest
```

Or open this repo in VS Code and use **Reopen in Container** via the `.devcontainer` setup.
