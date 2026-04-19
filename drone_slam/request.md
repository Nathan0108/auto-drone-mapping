To get the new Fast-LIO2 + EKF SLAM pipeline working, I need a new Gazebo bag file with some specific parameters. The last one was missing the physical IMU data, so the math filters couldn't initialize.

- Could you record a longer bag file for at least 60 seconds? Please have the drone fly around so I can test IMU and closed loop algorithm.
- What specific LiDAR are we simulating? Is it a Livox Mid-360 or something else? I need to know this so I can tell my software which scanning pattern to expect.
- My code expects the following topics. If you could run the bag record with at least these topics, that would be great.
```bash
ros2 bag record -o drone_warehouse_full_flight \
  /lidar/points \
  /imu/data \
  /fmu/out/vehicle_odometry \
  /camera/color/image_raw \
  /camera/depth/image_raw \
  /camera/color/camera_info \
  /tf \
  /tf_static
```
- Currently /imu/data is of type tf2_msgs/msg/TFMessage. It should be sensor_msgs/msg/Imu.
