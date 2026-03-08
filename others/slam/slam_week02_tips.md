## 1. Running the Baseline Simulation

Start the simulation environment and the teleop controls to move the robot:

**Step 1 — Launch TurtleBot 4 Sim:**

```bash
ros2 launch rtabmap_demos turtlebot4_sim_demo.launch.py
```

**Step 2 — Launch Teleop (Manual Control):**

```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

---

## 2. Visualization in RViz

To see the sensors in action, open RViz (`rviz2`) and configure the following displays:

### Essential Displays

| Display Type | Topic | Reliability |
|---|---|---|
| Image | `/oakd/rgb/preview/image_raw` | Best Effort |
| DepthCloud | Depth Map: `/oakd/rgb/preview/depth` / Color Image: `/oakd/rgb/preview/image_raw` | Best Effort |
| Map | `/map` | Reliable |

> **Fixed Frame:** Set to `map` or `base_cloud`

## 3. Verification Pipeline

1. **TF Tree:** Run the command below to ensure the path `map → odom → base_link → sensors` is unbroken.

    ```bash
    ros2 run tf2_tools view_frames
    ```

2. **Loop Closures:** Monitor `rtabmap_viz`. Success is indicated by green lines connecting a current frame to a past frame.

3. **Data Recording:** Use ROS Bags to record tests for offline playback:

    ```bash
    ros2 bag record -a -o drone_test_bag
    ```
