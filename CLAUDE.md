# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

`m-explore-ros2` is a ROS2 port of the multi-robot exploration framework. It provides two packages:

- **explore_lite**: Frontier-based autonomous exploration (single or multi-robot)
- **multirobot_map_merge**: Merges per-robot occupancy grids into a single map, supporting both known and unknown initial poses

Tested on ROS2 Humble and Jazzy.

## Build Commands

```bash
# Build both packages
colcon build --symlink-install --packages-select explore_lite multirobot_map_merge

# Build only one package
colcon build --symlink-install --packages-select explore_lite
colcon build --symlink-install --packages-select multirobot_map_merge

# Source the workspace after building
source install/setup.bash
```

## Testing

```bash
# Run explore_lite tests
colcon test --packages-select explore_lite
# Or run the binary directly after build:
./build/explore_lite/test_explore

# Run map_merge tests
colcon test --packages-select multirobot_map_merge
./build/multirobot_map_merge/test_merging_pipeline

# View test results
colcon test-result --verbose
```

## Code Style

Formatting uses `.clang-format` (Google style, 80-char column limit, 2-space indent, C++11 base). Run before committing:

```bash
find . -name "*.cpp" -o -name "*.h" -o -name "*.hpp" | xargs clang-format -i
```

## Architecture

### explore_lite

The node subscribes to a nav2 costmap, runs BFS-based frontier detection, scores frontiers, and sends `NavigateToPose` action goals to nav2.

Key classes and their files:
- `Explore` (`explore/src/explore.cpp`) — main node; owns the planning timer, action client, and frontier visualization
- `FrontierSearch` (`explore/src/frontier_search.cpp`) — BFS frontier detection and scoring (distance potential + info gain)
- `CostmapClient` (`explore/src/costmap_client.cpp`) — subscribes to `map` / `map_updates` and exposes the costmap + robot pose via TF

Data flow: `CostmapClient` → `FrontierSearch::searchFrom()` → sorted frontier list → `Explore::makePlan()` → `NavigateToPose` action client.

### multirobot_map_merge

The node dynamically discovers robots by scanning topics, subscribes to per-robot maps, and composites them into a merged occupancy grid.

Key classes and their files:
- `MapMerge` (`map_merge/src/map_merge.cpp`) — main node; runs three independent timers for discovery, estimation, and merging
- `MergingPipeline` (`map_merge/src/combine_grids/merging_pipeline.cpp`) — owns the OpenCV feature matching (AKAZE/ORB) and calls compositor/warper
- `GridCompositor` / `GridWarper` (`map_merge/src/combine_grids/`) — homography-based warping and composition of occupancy grids

Two operating modes controlled by `known_init_poses`:
- `true` — uses TF transforms between robot frames and world frame
- `false` — estimates relative poses via OpenCV feature matching on each robot's map image

### Relationships between packages

The packages are independent. `explore_lite` consumes nav2's costmap and action server. `multirobot_map_merge` consumes per-robot map topics and optionally TF. They can run together or separately.

## Running

```bash
# Single-robot exploration (requires nav2 with SLAM already running)
ros2 launch explore_lite explore.launch.py

# Multi-robot map merging
ros2 launch multirobot_map_merge map_merge.launch.py

# Full multi-robot simulation demo (TurtleBot3 in Gazebo)
export TURTLEBOT3_MODEL=waffle
ros2 launch multirobot_map_merge multi_tb3_simulation_launch.py slam_gmapping:=True
# With unknown initial poses:
ros2 launch multirobot_map_merge multi_tb3_simulation_launch.py slam_gmapping:=True known_init_poses:=False
```

## Key Parameters

**explore_lite** (`explore/config/params.yaml`):
- `robot_base_frame` — robot TF frame (default: `base_link`)
- `planner_frequency` — planning Hz (default: `0.15`)
- `progress_timeout` — seconds before aborting a goal (default: `30.0`)
- `min_frontier_size` — minimum frontier size in meters (default: `0.75`)
- `return_to_init` — return home when exploration finishes (default: `true`)

**multirobot_map_merge** (`map_merge/config/params.yaml`):
- `known_init_poses` — use TF vs. feature matching (default: `true`)
- `merging_rate` / `discovery_rate` / `estimation_rate` — timer frequencies
- `estimation_confidence` — feature matching confidence threshold

## Dependencies

- `nav2_costmap_2d`, `nav2_msgs` — explore_lite costmap and action interface
- `OpenCV >= 3.0` (4.0+ preferred) — map_merge feature matching and warping
- `Boost` (threads) — map_merge
- `slam_gmapping` fork with namespace support for multi-robot demos: `https://github.com/charlielito/slam_gmapping` (branch `feature/namespace_launch`)

## C++ Standards

- `explore_lite`: C++14
- `multirobot_map_merge`: C++17
