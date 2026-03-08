# WIP UNFINISHED ML@P Drone Mapping — SLAM Subteam Setup & Integration Guide

> Documented from hands-on setup sessions. Covers clean installation, custom source build, simulation launch, RTAB-Map configuration, known issues, and fixes.
> **Team members:** Peter & Samuel

---

## Table of Contents

1. [System Requirements](#1-system-requirements)
2. [Environment Setup](#2-environment-setup)
3. [Custom Workspace Build](#3-custom-workspace-build)
4. [Launching the Simulation](#4-launching-the-simulation)
5. [Visualization in RViz](#5-visualization-in-rviz)
6. [Critical Fixes & Known Issues](#6-critical-fixes--known-issues)
7. [Parameter Tuning (Drone Transition Prep)](#7-parameter-tuning-drone-transition-prep)
8. [Verification Pipeline](#8-verification-pipeline)
9. [Advanced Debugging: Manual Override](#9-advanced-debugging-manual-override)
10. [Common Issues Quick Reference](#10-common-issues-quick-reference)
11. [Reference Links](#11-reference-links)

---

## 1. System Requirements

- Ubuntu 22.04 LTS (dual-boot)
- ROS 2 Humble
- 8GB+ RAM (11GB recommended)
- 8GB swap space recommended

---

## 2. Environment Setup

### 2.1 Clean Up Previous Attempts

If you've tried installing before and things are broken, start fresh.

Remove any apt-installed rtabmap binaries — **this is critical**, as building from source won't work if these exist:

```bash
sudo apt remove ros-humble-rtabmap*
sudo apt autoremove -y
```

Remove any partial workspace:

```bash
rm -rf ~/ros2_ws
mkdir -p ~/ros2_ws/src
```

### 2.2 Install CycloneDDS

```bash
sudo apt install ros-humble-rmw-cyclonedds-cpp
```

Add to `.bashrc` so it's always set:

```bash
echo "export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp" >> ~/.bashrc
source ~/.bashrc
```

### 2.3 Install Dependencies

General ROS 2 dependencies:

```bash
sudo apt install python3-colcon-common-extensions python3-rosdep git-lfs
```

SLAM-specific dependencies:

```bash
sudo apt install ros-humble-octomap-server \
  ros-humble-robot-localization \
  ros-humble-pointcloud-to-laserscan \
  ros-humble-tf2-tools
```

Also ensure the following ROS packages are available for the full stack: `turtlebot4_simulator`, `rtabmap_ros`, `rtabmap_viz`, `nav2_bringup`, `navigation2`.

---

## 3. Custom Workspace Build

To enable Multi-RGBD synchronization and user data features, RTAB-Map **must be built from source** using specific CMake flags.

### 3.1 Install Livox SDK2 (System Library)

> ⚠️ The SDK2 is a **system-level C++ library**, NOT a ROS package. It must be built and installed separately — do **NOT** put it inside your `ros2_ws`.

```bash
cd ~
git clone https://github.com/Livox-SDK/Livox-SDK2.git
cd Livox-SDK2
mkdir build && cd build
cmake ..
make -j4
sudo make install
sudo ldconfig
```

Verify installation:

```bash
ls /usr/local/lib | grep livox
ls /usr/local/include | grep livox
```

You should see:

```
liblivox_lidar_sdk_shared.so
liblivox_lidar_sdk_static.a
livox_lidar_api.h
livox_lidar_cfg.h
livox_lidar_def.h
```

### 3.2 Clone All ROS Packages

```bash
cd ~/ros2_ws/src

# RTAB-Map core and ROS 2 wrapper
git clone https://github.com/introlab/rtabmap.git
git clone --branch ros2 https://github.com/introlab/rtabmap_ros.git

# Livox ROS 2 driver (needed by FAST-LIO2)
git clone https://github.com/Livox-SDK/livox_ros_driver2.git

# FAST-LIO2 for ROS 2
git clone https://github.com/Ericsii/FAST_LIO_ROS2.git
```

Your `src` folder should look like:

```
~/ros2_ws/
└── src/
    ├── rtabmap/
    ├── rtabmap_ros/
    ├── livox_ros_driver2/
    └── FAST_LIO_ROS2/
```

### 3.3 Build the Livox ROS 2 Driver

This driver has its own build script. Run it **before** the main colcon build:

```bash
source /opt/ros/humble/setup.sh
cd ~/ros2_ws/src/livox_ros_driver2
./build.sh humble
```

> If `livox_ros_driver2` fails with a `NOTFOUND` error regarding `LIVOX_INTERFACES_INCLUDE_DIRECTORIES`, colcon is failing to link the SDK. Ensure the SDK was installed natively first (Step 3.1), then use the `./build.sh humble` script rather than colcon directly.

### 3.4 Build the Full Workspace

Once Livox is built (or ignored via `COLCON_IGNORE`), build the rest of the workspace:

```bash
cd ~/ros2_ws
rosdep update
rosdep install --from-paths src --ignore-src -r -y

export MAKEFLAGS="-j2"  # Use -j1 if you have <11GB RAM

colcon build --symlink-install --executor sequential --cmake-args \
  -DRTABMAP_SYNC_MULTI_RGBD=ON \
  -DRTABMAP_SYNC_USER_DATA=ON \
  -DCMAKE_BUILD_TYPE=Release
```

> ⚠️ This build will take approximately **15–40 minutes**. Do not interrupt it. CMake warnings about `HUMBLE_ROS` or `ROS_EDITION` are harmless and can be ignored.

**Why `--executor sequential` and `MAKEFLAGS="-j2"`?** Compiling RTAB-Map with heavy synchronization flags can consume >2GB of RAM per file, which triggers the Linux OOM killer and crashes the build. These flags force sequential, controlled compilation to stay within memory limits.

### 3.5 Source the Workspace

```bash
source ~/ros2_ws/install/setup.bash
```

Add to `.bashrc` permanently — these three lines must appear **in this order**:

```bash
source /opt/ros/humble/setup.bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
source ~/ros2_ws/install/setup.bash
```

---

## 4. Launching the Simulation

### 4.1 Baseline Simulation & Teleop

**Terminal 1 — Launch TurtleBot 4 Sim:**

```bash
ros2 launch turtlebot4_ignition_bringup turtlebot4_ignition.launch.py
```

**Terminal 2 — Launch Teleop (Manual Control):**

```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

Teleop keys:

```
i = forward        , = backward
j = rotate left    l = rotate right
k = stop
q = increase speed
```

> ⚠️ **Critical startup order:**
> 1. Wait for Gazebo and RViz2 to fully load.
> 2. Click the **Play (▶)** button at the bottom-left of the Gazebo window — the simulation is **paused by default**. If you skip this, `sim_time` is frozen and all ROS 2 nodes will reject incoming data due to timestamp mismatches.
> 3. Click **".."** in the top-right of the Gazebo HMI panel → select **Undock**.
> 4. Wait a few seconds for the robot to fully undock before driving.

### 4.2 Launching RTAB-Map (3D Mapping)

**Terminal 2 — RTAB-Map 3D Mapping node:**

```bash
ros2 launch rtabmap_launch rtabmap.launch.py \
    rtabmap_args:="--delete_db_on_start" \
    rgb_topic:=/oakd/rgb/preview/image_raw \
    depth_topic:=/oakd/rgb/preview/depth/image_raw \
    camera_info_topic:=/oakd/rgb/preview/camera_info \
    frame_id:=base_link \
    approx_sync:=true \
    use_sim_time:=true \
    qos:=1 \
    rtabmap_viz:=true
```

**Critical flags explained:**

- `use_sim_time:=true` — Prevents the **"56-year delay" TF bug**, where RTAB-Map rejects simulation timestamps (year ~1970) by comparing them to the real system clock (year 2026), silently dropping every frame.
- `qos:=1` — Forces the "Sensor Data" QoS profile, required to receive data from Ignition simulated sensors.
- `approx_sync:=true` — Enables approximate time synchronization between RGB and depth streams when they are not perfectly co-timestamped.
- `--delete_db_on_start` — Clears corrupted databases from previous sessions on launch.

Alternatively, use the demo launch file:

```bash
ros2 launch rtabmap_demos turtlebot4_sim_demo.launch.py rtabmap_args:="--delete_db_on_start"
```

### 4.3 Autonomous Navigation (Nav2)

To move the robot autonomously within the mapped environment, enable the full Nav2 stack in one command:

```bash
ros2 launch turtlebot4_ignition_bringup turtlebot4_ignition.launch.py nav2:=true slam:=true rviz:=true
```

- `nav2:=true` — Enables the planner, controller, and recovery behaviors.
- `slam:=true` — Keeps mapping active so the robot can navigate and map simultaneously.

**Commanding the robot in RViz:**

1. **2D Pose Estimate:** If the robot's "ghost" doesn't match its real position, click this tool and click/drag on the map at the robot's current location to help it localize.
2. **Nav2 Goal:** Click this button, then click and drag on the map where you want the robot to go. The arrow indicates the final heading the robot should face.
3. **Observation:** You will see a **Global Path** (thin green line) and a **Local Plan** (blue/yellow line) showing how the robot intends to avoid obstacles.

---

## 5. Visualization in RViz

To verify sensor data and the OAK-D Field of View (FOV) alignment, open RViz (`rviz2`) and configure the following displays:

### Essential Displays

| Display Type | Topic | Reliability |
|---|---|---|
| Image | `/oakd/rgb/preview/image_raw` | Best Effort |
| DepthCloud | Depth Map: `/oakd/rgb/preview/depth` / Color Image: `/oakd/rgb/preview/image_raw` | Best Effort |
| LaserScan | `/scan` | Best Effort |
| Map | `/map` | Reliable |

> **Fixed Frame:** Set to `map` for navigation/mapping. Use `base_link` for FOV verification.

### FOV Verification (OAK-D Alignment Check)

1. Set the Fixed Frame to `base_link`.
2. Add the `DepthCloud` display.
3. Compare the yellow DepthCloud arrows to the red LiDAR scan lines. If they do not align, the URDF parameter `<horizontal_fov>` still needs adjustment (e.g., changing from `1.25` to `1.047`).
4. After verification, change the Fixed Frame back to `map` for normal operation.

---

## 6. Critical Fixes & Known Issues

### 6.1 Fix: ICP Odometry Failure (Most Common Issue)

**Symptoms:** RTAB-Map terminal spams `no odometry is provided. Image 0 is ignored!`, `lost: true, matches: 0` in `/odom_info`, and the map never generates.

**Cause:** The default `turtlebot4_slam.launch.py` configures RTAB-Map to use LiDAR-based ICP Odometry (`icp_odom`). In simulation, ICP fails to find enough features, halting the entire SLAM pipeline.

**Fix:** Re-route RTAB-Map to use reliable wheel odometry (`/odom`) instead.

Edit the SLAM launch file:

```bash
nano ~/ros2_ws/src/rtabmap_ros/rtabmap_demos/launch/turtlebot4/turtlebot4_slam.launch.py
```

**Change 1** — Fix `icp_parameters` (around line 40):

```python
# Change FROM:
icp_parameters={
    'odom_frame_id': 'icp_odom',
    'guess_frame_id': 'odom'
}

# Change TO:
icp_parameters={
    'odom_frame_id': 'odom',
    'guess_frame_id': 'odom'
}
```

**Change 2** — Fix the topic remappings (around line 66):

```python
# Change FROM:
('odom', 'icp_odom'),

# Change TO:
('odom', '/odom'),

# Remaining remappings should look like:
('rgb/image', '/oakd/rgb/preview/image_raw'),
('rgb/camera_info', '/oakd/rgb/preview/camera_info'),
('depth/image', '/oakd/rgb/preview/depth'),
```

> ⚠️ Do **NOT** add a leading slash to frame IDs (`odom_frame_id`, `guess_frame_id`) — ROS 2 frame IDs don't use slashes. Only the topic remapping gets the slash. Also watch for missing trailing commas in Python arrays — a syntax error will cause the `rtabmap` node to crash silently without appearing in `ros2 node list`.

After editing, rebuild to apply changes:

```bash
cd ~/ros2_ws
colcon build --symlink-install --packages-select rtabmap_demos
source install/setup.bash
```

### 6.2 Fix: Blank RTAB-Map GUI / No 3D Points

If the RTAB-Map GUI opens but the Odometry panel is empty and no 3D points appear:

**A. The robot must move.** RTAB-Map is designed to save memory and will often ignore perfectly stationary frames. Drive the robot forward for at least 1–2 meters. Watch the "Loop Closure Detection" window in the GUI for a green flash indicating the first map node has been created.

**B. Verify TF tree delay:**

```bash
ros2 run tf2_ros tf2_monitor
```

If the `Average Delay` is a massive number (e.g., `1.7718e+09`), `use_sim_time:=true` is not being applied properly and RTAB-Map is dropping all frames. Also confirm a continuous TF chain from `map -> odom -> base_link -> oakd_rgb_camera_optical_frame`.

**C. Check topic bandwidth:**

```bash
ros2 topic hz /oakd/rgb/preview/image_raw
ros2 topic hz /odom
```

If `/odom` is dead, RTAB-Map cannot place 3D points in space. If the image topic is dead, verify the simulation is unpaused and the bridge is running.

**D. `Frame [map] does not exist` in RViz2** means RTAB-Map hasn't initialized yet. Drive the robot to trigger the first keyframe.

### 6.3 Fix: RAM Crashes During Build

**Cause:** Compiling RTAB-Map with heavy synchronization flags can consume >2GB of RAM per file, triggering the Linux OOM killer.

**Fix 1:** Force single-threaded compilation:

```bash
export MAKEFLAGS="-j1"
colcon build --symlink-install --executor sequential ...
```

**Fix 2:** Add swap space if you have <11GB RAM:

```bash
sudo fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

### 6.4 Fix: Livox SDK / FAST_LIO Build Errors

If `livox_ros_driver2` fails with a `NOTFOUND` error regarding `LIVOX_INTERFACES_INCLUDE_DIRECTORIES`:

1. Install the SDK natively first:

    ```bash
    cd ~/Livox-SDK2/build
    sudo make install
    sudo ldconfig
    ```

2. Build the driver using its internal script instead of colcon:

    ```bash
    cd ~/ros2_ws/src/livox_ros_driver2
    ./build.sh humble
    ```

---

## 7. Parameter Tuning (Drone Transition Prep)

We identified critical parameters in `rtabmap_params.yaml` that control map quality and loop closure behavior. These were tuned specifically to prepare for the transition from TurtleBot simulation to drone hardware.

| Parameter | Recommended Value | Purpose |
|---|---|---|
| `Vis/MinInliers` | `10 – 12` | Lower this if loop closures are being rejected in sparse environments. |
| `Grid/RangeMax` | `5.0 – 7.0` | Controls how far the camera/LiDAR "sees" to build the occupancy grid. |
| `Grid/CellSize` | `0.05` | Resolution of the map in meters. |
| `RGBD/OptimizeMaxError` | `3.0` | Increases tolerance for odometry drift during loop closure. |

### Live Tuning (Dynamic Reconfigure)

To change a parameter while the simulation is running without restarting:

```bash
ros2 param set /rtabmap/rtabmap Vis/MinInliers 12
```

---

## 8. Verification Pipeline

Run through these checks after every launch to confirm the pipeline is healthy.

**1. TF Tree integrity:**

```bash
ros2 run tf2_tools view_frames
```

Ensure the path `map -> odom -> base_link -> sensors` is unbroken. Any gap will break localization.

**2. Topic health:**

```bash
ros2 topic list
ros2 topic hz /odom                               # Should be ~20Hz
ros2 topic hz /oakd/rgb/preview/depth/points      # Should be ~10Hz
ros2 topic hz /rgbd_image
```

**3. Odometry status:**

```bash
ros2 topic echo /odom_info --once
```

Healthy output shows `lost: false` with non-zero `matches` and `inliers`. If you see `lost: true, matches: 0`, the ICP odometry fix in Section 6.1 may not have been applied correctly.

**4. Active nodes:**

```bash
ros2 node list
```

You should see `/rtabmap`, `/icp_odometry`, `/rviz2`, and `/rtabmap_viz` among others.

**5. Loop closures:** Monitor `rtabmap_viz`. A successful loop closure is indicated by **green lines** connecting a current frame to a past frame.

**6. Data recording** — use ROS Bags to record tests for offline playback:

```bash
ros2 bag record -a -o drone_test_bag
```

---

## 9. Advanced Debugging: Manual Override

If the launch file continues to fail silently (the `/rtabmap` node disappears from `ros2 node list`), bypass the `.launch.py` file entirely to isolate the issue.

**Kill all lingering processes first:**

```bash
pkill -9 -f rtabmap
pkill -9 -f gazebo
pkill -9 -f robot_state_publisher
```

**Terminal 1 — Start simulator only:**

```bash
ros2 launch turtlebot4_ignition_bringup turtlebot4_ignition.launch.py world:=warehouse
```

*(Press Play and Undock as described in Section 4.1)*

**Terminal 2 — Run RTAB-Map manually:**

This command forces synchronization, uses simulation time, and hard-codes the correct topic routing:

```bash
ros2 run rtabmap_slam rtabmap \
    --ros-args \
    -p use_sim_time:=true \
    -p subscribe_depth:=true \
    -p approx_sync:=true \
    -r rgb/image:=/oakd/rgb/preview/image_raw \
    -r rgb/camera_info:=/oakd/rgb/preview/camera_info \
    -r depth/image:=/oakd/rgb/preview/depth \
    -r odom:=/odom \
    -p frame_id:=base_link
```

If this works, the issue is strictly a **syntax or namespace error in the launch file**, not a hardware or driver problem.

---

## 10. Common Issues Quick Reference

| Problem | Cause | Fix |
|---|---|---|
| `liblivox_lidar_sdk_shared.so` not found | SDK2 not installed to system | Run `sudo make install && sudo ldconfig` in `Livox-SDK2/build` |
| Build crashes with no output | OOM killer triggered | Add 8GB swap (Section 6.3) and use `MAKEFLAGS="-j1"` |
| `Frame [map] does not exist` in RViz2 | RTAB-Map not initialized yet | Drive the robot to generate the first keyframe |
| `lost: true, matches: 0` in odom_info | ICP odometry failing | Apply the remapping fix in Section 6.1 |
| Blank RTAB-Map GUI, no 3D points | Robot stationary or `use_sim_time` wrong | Move the robot; verify `use_sim_time:=true` is applied |
| Robot not moving in Gazebo | Simulation is paused | Click the Play (▶) button in Gazebo bottom-left |
| `/rtabmap` node missing from `ros2 node list` | Launch file syntax error | Run RTAB-Map manually (Section 9) to isolate |
| Massive TF delay (`1.7718e+09`) | `use_sim_time` not applied | Ensure `use_sim_time:=true` is set on the RTAB-Map node |
| Teleop not responding | Terminal lost focus | Click on the teleop terminal window first |
| CMake warnings about `HUMBLE_ROS` | Harmless build artifact | Ignore |
| DepthCloud and LiDAR scan misaligned | Incorrect URDF `<horizontal_fov>` | Adjust from `1.25` to `1.047` in URDF |

---

## 11. Reference Links

- [RTAB-Map GitHub](https://github.com/introlab/rtabmap_ros)
- [FAST-LIO2 ROS2](https://github.com/Ericsii/FAST_LIO_ROS2)
- [Livox SDK2](https://github.com/Livox-SDK/Livox-SDK2)
- [Livox ROS2 Driver](https://github.com/Livox-SDK/livox_ros_driver2)
- [TurtleBot4 Demo Launch](https://github.com/introlab/rtabmap_ros/tree/ros2/rtabmap_demos#turtlebot4-nav2-2d-lidar-and-rgb-d-slam)
- [OctoMap](https://octomap.github.io/)
