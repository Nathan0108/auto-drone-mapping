## Phase 6: Launching the Full Pipeline

To test the entire SLAM stack, open **separate terminals** (ensure your `.bashrc` has sourced all workspaces) and run the following commands:

### Terminal 1 — Virtual Drone (Bag Playback)

*Important: To play split bag files, point ROS 2 to the folder containing all .db3 chunks and the metadata.yaml, rather than a specific .db3 file.*

```bash
ros2 bag play /home/spes_ignota/dev/auto-drone-mapping/bag_files/rosbag2_2026_04_26-15_42_48_0-001.db3 \
    --clock \ 
    --read-ahead-queue-size 2000 \
    --qos-profile-overrides-path /home/spes_ignota/dev/auto-drone-mapping/drone_slam/config/qos_overrides.yaml
```

### Terminal 2 — RTAB-Map (The SLAM Brain)

```bash
# Run the modified launch command from Phase 5 here
ros2 launch rtabmap_launch rtabmap.launch.py \
    rtabmap_args:="--delete_db_on_start" \
    visual_odometry:=false \
    odom_topic:=/odom \
    frame_id:="body" \
    rgb_topic:=/camera/color/image_raw \
    depth_topic:=/camera/depth/image_raw \
    camera_info_topic:=/camera/color/camera_info \
    subscribe_scan_cloud:=true \
    scan_cloud_topic:=/point_cloud \
    approx_sync:=true \
    qos:=2 \
    use_sim_time:=true
```

### Terminal 3 — RViz2 (Live Visualizer)

```bash
ros2 run rviz2 rviz2 --ros-args -p use_sim_time:=true
```
