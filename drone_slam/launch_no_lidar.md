# **Resimulating Drone Mapping with RTAB-Map and ROS 2 Bags**

This guide explains how to take recorded sensor data (a ROS 2 bag) from the Gazebo drone simulation and use it to generate a 3D environment map using RTAB-Map.  
By playing back bag files, we can test and tune our SLAM (Simultaneous Localization and Mapping) pipeline without needing to run the physics simulator or fly the actual drone.

## **Step 1: Inspect the Bag File**

Before playing a bag, it is good practice to inspect its contents to see what topics were recorded, the duration of the flight, and the message counts.  
Run the following command to view the bag's metadata:  

```bash
ros2 bag info ~/dev/auto-drone-mapping/bag_files/drone_warehouse_bag_0.db3 -s sqlite3
```

*(Note: If the bag folder contains a metadata.yaml file, you can omit the -s sqlite3 flag and just point to the folder directory).*

## **Step 2: Play the Bag (The "Virtual Drone")**

We need to play the bag to simulate the data stream. We will use the --clock flag to publish simulation time, remap any incorrectly named topics, and optionally use -r 0.2 or -- loop to play it at 20% speed so RTAB-Map has time to boot up or automatically rerun the bag simulation.

**Open Terminal 1:**  
```bash
ros2 bag play ~/dev/auto-drone-mapping/bag_files/drone_warehouse_bag_0.db3 -s sqlite3 --clock --remap /imu/data:=/tf --loop
```

## **Step 3: Launch RViz2 (The Visualization)**

We need RViz2 to visualize the point clouds, camera feeds, and the generated 3D map. It must be told to use the simulation time provided by the bag.  

**Open Terminal 2:**  
```bash
ros2 run rviz2 rviz2 --ros-args -p use_sim_time:=true
```

## **Step 4: Publish Static Transforms (Virtual Duct Tape)**

If the simulation bag is missing the /tf_static blueprints that physically connect the sensors to the drone, RTAB-Map will reject the data. We must manually publish these hardware mounts.

**Open Terminal 3 (Camera Mount):**  

```bash
ros2 run tf2_ros static_transform_publisher 0 0 0 -1.57 0 -1.57 body sim_camera
```

## **Step 5: Launch RTAB-Map (The SLAM Brain)**

Depending on what data is available and correctly formatted in the bag, you can launch RTAB-Map in two different modes.

### **Option A: Vision-Only Mode (RGB-D)**

Use this command if LiDAR data is missing or incorrectly formatted (e.g., published in the global map frame instead of a local sensor frame). This relies purely on visual odometry.

**Open Terminal 4:**  
```bash
ros2 launch rtabmap_launch rtabmap.launch.py     rtabmap_args:="--delete_db_on_start"     visual_odometry:=true     frame_id:="body"     rgb_topic:=/camera/color/image_raw     depth_topic:=/camera/depth/image_raw     camera_info_topic:=/camera/color/camera_info     approx_sync:=true     qos:=2     use_sim_time:=true
```

### **Option B: Vision + LiDAR Fusion Mode**

Use this command to fuse the RGB-D cameras with the 3D LiDAR point cloud for a highly accurate map. *(Note: This requires the LiDAR points to be published in a local frame like lidar_link, and requires an additional static transform for the LiDAR).*  

**Open Terminal 4:**  
```bash
ros2 launch rtabmap_launch rtabmap.launch.py \
    rtabmap_args:="--delete_db_on_start" \
    visual_odometry:=true \
    frame_id:="body" \
    rgb_topic:=/camera/color/image_raw \
    depth_topic:=/camera/depth/image_raw \
    camera_info_topic:=/camera/color/camera_info \
    subscribe_scan_cloud:=true \
    scan_cloud_topic:=/lidar/points \
    approx_sync:=true \
    qos:=2 \
    use_sim_time:=true
```

## **Troubleshooting**

* **Black Screen in RTAB-Map if using Option B of Step 5:** Check your ros2 topic echo for your sensor data. If the frame_id is set to map instead of a local frame, RTAB-Map will reject it.  
* **"Did not receive data since 5 seconds":** This usually means QoS mismatch (use qos:=2 for simulator data) or a missing TF static transform.  
* **Time Jump Warnings:** If you loop the bag, RTAB-Map will detect time moving backward and clear its database to protect the map from corruption. To view a stable map, play the bag once without the --loop flag.
