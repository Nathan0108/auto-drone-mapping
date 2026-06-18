# Autonomous Drone Mapping

A simulation-first pipeline for autonomously exploring and mapping unknown 3D environments with a quadrotor UAV. It unifies three sub-systems (physics-based simulation, LiDAR-inertial SLAM, and RL path planning) into a single closed loop that produces accurate 3D digital replicas of physical spaces.

> Developed by Nathan Miller, Samuel Yoon, Neerav Poluri, Tanish Rathi, Nathan Chou.

This repo holds the **simulation, SLAM, and CAD** components. The RL policy lives in a separate repo: **[drone-isaaclab](https://github.com/Nathan0108/drone-isaaclab)**.

---

## Motivation

This project distills published SLAM reserach into one system optimized for **coverage completeness, energy efficiency, and flight time**. The simulation-first approach lets us iterate against a high-fidelity simulated drone before committing to hardware, with the same software designed to transfer to a physical airframe later.

## System Architecture

Three layers: sensor data flows from the simulator, through the SLAM stack into a 3D occupancy map, and back out as RL-selected flight commands.

**1. Simulation**: Isaac Sim provides physics and rendering; Pegasus Sim manages the drone and spawns a **PX4 SITL** flight controller. Motor commands feed back into policy to close the loop.

**2. SLAM**: builds a map while localizing the drone, in three stages:

1. **Fast-LIO2** turns raw LiDAR + IMU into accurate odometry → `/Odometry`.
2. An **EKF** (`robot_localization`) fuses Fast-LIO2 odom with PX4's raw odometry (`/fmu/out/vehicle_odometry`) → `/Fused_odom`.
3. **RTAB-Map** ingests camera/LiDAR/IMU plus the fused odom (visual odometry off), runs loop closure, and generates the **3D Occupancy Map** (Octomap).

> EKF wiring: [drone_slam/config/ekf.yaml](drone_slam/config/ekf.yaml) — `odom0` = Fast-LIO2 pose (X/Y/Z + RPY), `odom1` = PX4 velocities.

**3. RL Planning**: [drone-isaaclab](https://github.com/Nathan0108/drone-isaaclab).

## Hardware & Sensors

The simulated drone mirrors the planned physical airframe (X500 frame + Jetson companion; CAD in [CAD/](CAD/)) so software transfers with minimal change.

| Sensor        | Model                | Stream                     |
| ------------- | -------------------- | -------------------------- |
| Stereo camera | Intel RealSense D455 | RGB + depth frames         |
| LiDAR         | Hesai XT32           | 3D point cloud             |
| IMU           | onboard              | linear accel + angular vel |

**Sim-to-real:** we plan to domain-randomize lighting, obstacle placement, and sensor noise in Isaac Sim.

## Getting Started

**Prerequisites:** ROS 2 (Humble) on Ubuntu 22.04, `colcon`, `ros-$ROS_DISTRO-robot-localization`, `rtabmap` + `rtabmap_launch`, the [Livox ROS 2 driver](https://github.com/Livox-SDK/livox_ros_driver2), [FAST_LIO_ROS2](https://github.com/Ericsii/FAST_LIO_ROS2), and Git LFS.

> Bag files in `bag_files/` (`.db3`/`.zstd`) are tracked via **Git LFS** ([.gitattributes](.gitattributes)). Run `git lfs install` before cloning.

The Livox driver and Fast-LIO2 must be built in **separate, isolated workspaces** sourced in a strict order (ROS 2 custom-message constraints). Full build and `.bashrc` overlay steps: **[drone_slam/launch_with_lidar.md](drone_slam/launch_with_lidar.md)** (Phases 1-4).

## Running the SLAM Pipeline

Drive the stack from recorded ROS 2 bags (no simulator needed) or a live Isaac Sim session. Each mode has a guide with the exact per-terminal commands:

- **Full pipeline (Fast-LIO2 + EKF + RTAB-Map):** [drone_slam/launch_with_lidar.md](drone_slam/launch_with_lidar.md) (Phases 5-6)
- **Vision-only / RGB-D from bags** (no Fast-LIO2): [drone_slam/launch_no_lidar.md](drone_slam/launch_no_lidar.md)
- **RTAB-Map replay without Fast-LIO2:** [drone_slam/launch_no_fast.md](drone_slam/launch_no_fast.md)

Config in [drone_slam/config/](drone_slam/config/): `ekf.yaml`, `qos_overrides.yaml` (bag-playback QoS), `rtabmap_params.yaml`.

## ROS 2 Topics

Canonical sensor topics ([interfaces.md](interfaces.md)):

| Topic                       | Type                      |
| --------------------------- | ------------------------- |
| `/camera/color/image_raw`   | `sensor_msgs/Image`       |
| `/camera/depth/image_raw`   | `sensor_msgs/Image`       |
| `/camera/color/camera_info` | `sensor_msgs/CameraInfo`  |
| `/lidar/points`             | `sensor_msgs/PointCloud2` |
| `/imu/data`                 | `sensor_msgs/Imu`         |
| `/clock`                    | `rosgraph_msgs/Clock`     |

Odom chain: `/Odometry` (Fast-LIO2) → `/Fused_odom` (EKF) → RTAB-Map; PX4 odom enters the EKF on `/fmu/out/vehicle_odometry`. Per-node sub/pub map: [drone_slam/topic_mapping.md](drone_slam/topic_mapping.md).

> `/imu/data` must be `sensor_msgs/Imu` for the LiDAR-inertial pipeline; some early bags recorded it as `tf2_msgs/TFMessage`, blocking filter init. Bag-recording spec: [drone_slam/request.md](drone_slam/request.md).

## Results

PPO trained for 200k timesteps in the warehouse (2048 parallel Isaac Lab envs, single GPU):

- **Mean reward** rose from ~0 to ~880 by ~100k steps, then plateaued (stable convergence).
- **Max reward** reached ~1,570 (near-optimal episodes).
- **Policy std** peaked ~50k steps then decayed to ~1.0 (healthy exploration→exploitation).

RTAB-Map produces dense colored 3D reconstructions of the Isaac Sim warehouse from fused LiDAR + RGB-D (sample artifact: [occupancy_map/cloud.ply](occupancy_map/cloud.ply)).

## References

- Achiam, J. _Spinning Up in Deep RL._ OpenAI, 2018.
- Campos, C. et al. _ORB-SLAM3._ IEEE T-RO 37(6), 2021. doi:10.1109/TRO.2021.3075644
- Xu, W. et al. _FAST-LIO2._ IEEE T-RO 38(4), 2022. doi:10.1109/TRO.2022.3141876
- Placed, J. A. et al. _A Survey on Active SLAM._ IEEE T-RO 39(3), 2023. arXiv:2207.00254
- Lajoie, P.-Y. & Beltrame, G. _Swarm-SLAM._ IEEE RA-L 9(1), 2024. doi:10.1109/LRA.2023.3333742
- Panerati, J. et al. _Learning to Fly._ IROS 2021. doi:10.1109/IROS51168.2021.9635857
- Yu, C. et al. _The Surprising Effectiveness of PPO in Cooperative Multi-Agent Games._ NeurIPS 35, 2022.

## License

[MIT](LICENSE) (c) 2026 Nathan0108.
