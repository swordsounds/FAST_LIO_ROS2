# Unilidar 2 / Unitree L2 support

This integration consumes the standard ROS 2 messages published by Unitree's `unitree_lidar_ros2` driver. The driver runs separately; FAST-LIO subscribes to its point cloud and IMU topics.

## Data flow

| Step | What happens | Where |
| --- | --- | --- |
| 1. Publish | Unitree's driver publishes a `sensor_msgs/msg/PointCloud2` on `/unilidar/cloud` and a `sensor_msgs/msg/Imu` on `/unilidar/imu`. The cloud header stamp marks the start of the cloud. | [Unitree ROS 2 driver](https://github.com/unitreerobotics/unilidar_sdk2/blob/main/unitree_lidar_ros2/src/unitree_lidar_ros2/include/unitree_lidar_ros2.h#L139-L143), [cloud publication](https://github.com/unitreerobotics/unilidar_sdk2/blob/main/unitree_lidar_ros2/src/unitree_lidar_ros2/include/unitree_lidar_ros2.h#L204-L220) |
| 2. Select handler | `lidar_type: 5` selects `UNILIDAR2` and the `unilidar2_handler` for incoming PointCloud2 messages. | [config](../config/unilidar2.yaml), [dispatch](../src/preprocess.cpp) |
| 3. Read fields | The handler reads `x`, `y`, `z`, `intensity`, `ring`, and `time`. It requires `time` to be `float32`, matching the driver's PCL point definition. | [Unitree PCL point definition](https://github.com/unitreerobotics/unilidar_sdk2/blob/main/unitree_lidar_sdk/include/unitree_lidar_sdk_pcl.h#L41-L53), [handler](../src/preprocess.cpp) |
| 4. Filter and time | Nonfinite coordinates or time, negative time, and points within `preprocess.blind` are discarded. With feature extraction off, `point_filter_num` selects every Nth input point. Each point's relative time is converted from seconds to milliseconds and stored in FAST-LIO's `curvature` field for motion compensation. | [Unitree point and cloud timestamps](https://github.com/unitreerobotics/unilidar_sdk2/blob/main/unitree_lidar_sdk/include/unitree_lidar_utilities.h#L40-L62), [handler](../src/preprocess.cpp) |
| 5. Map | FAST-LIO pairs the cloud with IMU measurements and uses the per point offsets during motion compensation. Clouds with fewer than two accepted points are skipped. | [cloud callback and synchronization](../src/laserMapping.cpp), [IMU processing](../src/IMU_Processing.hpp) |

The supplied config sets `feature_extract_enable: false`, so `ring` is not needed for its normal path. If feature extraction is enabled, the handler groups points by Unitree's one-based ring IDs and uses `preprocess.scan_line` as the ring count.

## Configuration choices

| Setting | Supplied value | Reason |
| --- | --- | --- |
| `common.lid_topic`, `common.imu_topic` | `/unilidar/cloud`, `/unilidar/imu` | These are the [driver's default topic names](https://github.com/unitreerobotics/unilidar_sdk2/blob/main/unitree_lidar_ros2/src/unitree_lidar_ros2/include/unitree_lidar_ros2.h#L93-L96). |
| `preprocess.scan_line` | `18` | Matches the [driver's default `cloud_scan_num`](https://github.com/unitreerobotics/unilidar_sdk2/blob/main/unitree_lidar_ros2/src/unitree_lidar_ros2/include/unitree_lidar_ros2.h#L79-L85). |
| `preprocess.timestamp_unit` | `0` (seconds) | Unitree's point `time` is relative to the cloud start stamp; the SDK's sample timestamps and parser use fractional seconds. FAST-LIO converts seconds to milliseconds. See [SDK point definition](https://github.com/unitreerobotics/unilidar_sdk2/blob/main/unitree_lidar_sdk/include/unitree_lidar_utilities.h#L40-L62) and [packet parser](https://github.com/unitreerobotics/unilidar_sdk2/blob/main/unitree_lidar_sdk/include/unitree_lidar_utilities.h#L118-L168). |
| `mapping.extrinsic_T`, `mapping.extrinsic_R` | `[0.007698, 0.014655, -0.00667]`, identity | LiDAR origin and orientation expressed in the IMU frame, matching [Unitree's published frame relationship](https://github.com/unitreerobotics/unilidar_sdk2#2-coordinate-system-definition) and [driver transform](https://github.com/unitreerobotics/unilidar_sdk2/blob/main/unitree_lidar_ros2/src/unitree_lidar_ros2/include/unitree_lidar_ros2.h#L191-L201). |
| `mapping.extrinsic_est_en` | `false` | Uses the supplied physical offset instead of estimating it at runtime. |

Cloud and IMU header stamps must use the same clock. Adjust the topic names, ring count, and extrinsic values if your driver configuration or sensor setup differs. `livox_ros_driver2` is only needed at build time when Livox `CustomMsg` support is wanted; the Unilidar 2 build uses standard `PointCloud2` messages.

## Run

If this repository was cloned without `--recursive`, fetch its required `ikd-Tree` submodule before building:

```bash
git submodule update --init --recursive
colcon build --symlink-install
```

After building and sourcing both ROS 2 workspaces, start the driver and FAST-LIO in separate terminals:

```bash
ros2 launch unitree_lidar_ros2 launch.py
```

```bash
ros2 launch fast_lio mapping.launch.py config_file:=unilidar2.yaml
```

For installation and driver setup, see the [official Unitree SDK2 README](https://github.com/unitreerobotics/unilidar_sdk2#5-how-to-use-the-ros2-package).

## Sources used

- [Unitree SDK2 README](https://github.com/unitreerobotics/unilidar_sdk2): device coordinate systems, default scan count, and ROS 2 setup.
- [Unitree ROS 2 driver](https://github.com/unitreerobotics/unilidar_sdk2/blob/main/unitree_lidar_ros2/src/unitree_lidar_ros2/include/unitree_lidar_ros2.h): published message types, topics, stamps, and IMU to LiDAR transform.
- [Unitree PCL point definition](https://github.com/unitreerobotics/unilidar_sdk2/blob/main/unitree_lidar_sdk/include/unitree_lidar_sdk_pcl.h): PointCloud2 field names and types.
- [Unitree SDK point and cloud definitions](https://github.com/unitreerobotics/unilidar_sdk2/blob/main/unitree_lidar_sdk/include/unitree_lidar_utilities.h): relative point time and cloud start stamp.
