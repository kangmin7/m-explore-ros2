# m-explore-ros2 (PX4 fork)

This is a fork of [m-explore-ros2](https://github.com/robo-friends/m-explore-ros2) adapted for PX4-based UAV platforms. It adds two parameters — `goal_hysteresis` and `distance_scale` — that improve exploration stability on drones.

**Demo video**: https://www.youtube.com/watch?v=bG_B9l2mHzY

---

## Changes from upstream

### `goal_hysteresis`

Prevents the robot from oscillating between similarly-scored frontiers. When a new frontier is selected, the planner checks if the current goal is still available. If the new frontier's cost is not better than the current one by at least `goal_hysteresis`, the robot stays on its current goal.

- **Default**: `0.0` (disabled, matches upstream behavior)
- **Recommended**: `3.0`
- **Increasing it**: The robot becomes more committed to its current goal. It takes a larger cost advantage for a competing frontier to cause a goal switch. This reduces back-and-forth replanning and is especially useful on drones where replanning mid-flight is costly.

### `distance_scale`

Adds a continuous distance penalty to each frontier's score: `cost += distance_scale * distance_to_frontier`. This makes nearby frontiers continuously more attractive without a hard radius cutoff.

- **Default**: `0.0` (disabled, matches upstream behavior)
- **Recommended**: `3.0`
- **Increasing it**: The robot strongly prefers closer frontiers. Exploration becomes more conservative and local — the robot exhausts nearby unknowns before traveling far. On drones, this also helps conserve battery by minimizing long traversals.

Both parameters can be set in `explore/config/params.yaml`.

---

## Running in simulation (PX4 + Gazebo)

This fork is tested with a simulated X500 quadrotor with a 2D downward-facing lidar in a Gazebo world. Each component runs in a separate terminal.

### 1. PX4 SITL

```bash
# https://github.com/kangmin7/PX4-Autopilot
cd ~/PX4-Autopilot
make px4_sitl gz_x500_lidar_2d_down PX4_GZ_WORLD=husarion_office_empty
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

```bash
# https://github.com/kangmin7/mavros
ros2 launch mavros px4.launch fcu_url:="udp://:14540@127.0.0.1:14557" use_sim_time:=true
```

### 4. SLAM

```bash
# https://github.com/kangmin7/slam_toolbox-px4
ros2 launch slam_toolbox online_async_launch.py use_sim_time:=true
```

### 5. Nav2

```bash
# https://github.com/kangmin7/navigation2-px4
ros2 launch nav2_bringup px4_mavros_navigation_launch.py use_sim_time:=true
```

### 6. Exploration

```bash
ros2 launch explore_lite explore.launch.py
```

### 7. Offboard mode

To let MAVROS/Nav2 command the drone, switch PX4 to offboard mode:

```bash
ros2 service call /mavros/set_mode mavros_msgs/srv/SetMode "{custom_mode: 'OFFBOARD'}"
```

---

COPYRIGHT
---------

Packages are licensed under BSD license. See respective files for details.
