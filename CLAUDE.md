# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **Odin ROS2 Driver** package for Odin sensor modules from Manifold Tech Ltd. It provides ROS2 interfaces for point cloud SLAM applications using LiDAR sensors with integrated RGB cameras and IMU. The driver is designed for technical professionals conducting secondary development and requires scenario-specific optimization.

**Version:** v0.6.1

**Supported ROS2 Distributions:** Foxy, Humble (recommended), Jazzy

## Build Commands

### Build the Package
```bash
cd /home/alex-lin/ros2_ws
source /opt/ros/humble/setup.bash  # or jazzy/foxy
colcon build --packages-select odin_ros_driver
```

Or use the build script:
```bash
cd /home/alex-lin/ros2_ws
source /opt/ros/humble/setup.bash
./src/odin_ros_driver/script/build.sh
```

Build script options:
- `./script/build.sh` - Build the package
- `./script/build.sh -c` - Clean build artifacts (removes build/, install/, log/)
- `./script/build.sh -h` - Show help

The script automatically:
- Detects workspace structure
- Runs `colcon build` with multi-core compilation
- Sources the install/setup.bash after successful build

## Launch and Run

```bash
source install/setup.bash
ros2 launch odin_ros_driver odin1.launch.py
```

The launch file starts:
- `host_sdk_sample` - Main driver node
- `pcd2depth_node` - Point cloud to depth image converter
- `rviz2` - Visualization with auto-loaded config

### Setting ROS2 to localhost only (recommended for complex networks)
```bash
export ROS_LOCALHOST_ONLY=1
```

## Configuration System

Primary config file: [config/control_command.yaml](config/control_command.yaml)

Key runtime modes controlled via `custom_map_mode`:
- **Mode 0** (Odometry): Map and odom frames share the same pose
- **Mode 1** (SLAM): Full SLAM with loop closure and map saving capability
- **Mode 2** (Relocalization): Requires pre-built map at `relocalization_map_abs_path`

### Map Saving (SLAM Mode)
When running in SLAM mode (`custom_map_mode = 1`), use the ROS2 service to save the map:
```bash
ros2 service call /odin1/save_map std_srvs/srv/Trigger
```
The service will:
- Send the save command to the device
- Poll device status every 1 second (up to 30 seconds)
- Automatically transfer the map file when ready
- Return success/failure with detailed message

Map saves to path specified by `mapping_result_dest_dir` and `mapping_result_file_name` in config. If not specified, defaults to timestamped filename in the log directory.

**Note**: Requires updated firmware. Old firmware will fail with "Failed to check save_map status" error.

### Topic Control
Enable/disable topics via config flags:
- `sendrgb` - RGB camera (`/odin1/image`)
- `sendimu` - IMU data (`/odin1/imu`)
- `sendodom` - Odometry (`/odin1/odometry`)
- `senddtof` - Raw point cloud (`/odin1/cloud_raw`)
- `sendcloudslam` - SLAM point cloud (`/odin1/cloud_slam`)
- `sendcloudrender` - Rendered point cloud with RGB (`/odin1/cloud_render`)
- `senddepth` - Dense depth image (high CPU usage, demo only)

## Code Structure

### Core Nodes
- **host_sdk_sample.cpp** (main driver): Interfaces with Odin hardware via `lydHostApi` library, publishes ROS2 topics
- **pcd2depth.cpp**: Converts point clouds to depth images
- **rawCloudRender.cpp**: Renders point clouds with RGB color from camera calibration

### Hardware Interface Layer
- **lidar_api.h** / **lidar_api_type.h**: C API for LiDAR device control (provided as static libraries)
- Device communication via USB 3.0 (strict mode configurable via `strict_usb3.0_check`)
- Callback-based architecture for sensor data streams

### Calibration
- **calib.yaml**: Device-specific calibration retrieved from hardware on connection (differs per device)
- **yaml_parser.cpp**: Handles config and calibration file parsing

### Data Pipeline
Point clouds have custom field structure:
```cpp
struct Point {
    float x, y, z;
    uint8_t intensity;      // 0-255
    uint16_t confidence;    // 0-65535
    float offset_time;      // seconds
};
```

SLAM/rendered clouds use standard PointXYZRGB format.

## Platform Support

The CMakeLists.txt automatically detects platform architecture and links the appropriate static library:
- **x86_64**: Uses `lib/liblydHostApi_amd.a`
- **ARM/aarch64**: Uses `lib/liblydHostApi_arm.a`

## Dependencies

Critical versions:
- **OpenCV ≥ 4.5.0** (4.5.5 or 4.8.0 recommended) - **Only one version must be installed**
- yaml-cpp, Eigen3, PCL, OpenSSL, libusb-1.0

Install dependencies:
```bash
sudo apt update
sudo apt install -y build-essential cmake git libgtk2.0-dev pkg-config \
    libavcodec-dev libavformat-dev libswscale-dev \
    libyaml-cpp-dev libusb-1.0-0-dev libopencv-dev
```

## Hardware Setup

Before running, create udev rules for Odin device:
```bash
sudo vim /etc/udev/rules.d/99-odin-usb.rules
# Add: SUBSYSTEM=="usb", ATTR{idVendor}=="2207", ATTR{idProduct}=="0019", MODE="0666", GROUP="plugdev"
sudo udevadm control --reload
sudo udevadm trigger
```

## Common Issues

### Multiple OpenCV Versions
If driver dies immediately after "Device ready and streams activated":
1. Disable RGB topic: Set `sendrgb = 0` in control_command.yaml
2. If this fixes it, purge unused OpenCV versions, keep only one
3. Rebuild driver

### ROS2 Network Issues
If device disconnects immediately after stream start:
- Use `export ROS_LOCALHOST_ONLY=1` for local-only communication
- Or simplify network environment (disable complex WiFi/Ethernet)

### Segfault on Re-launch
Power cycle the Odin device (disconnect/reconnect power)

### Library Binding Failures
```bash
cd /home/alex-lin/ros2_ws
rm -rf build/ install/ log/
./src/odin_ros_driver/script/build.sh
```

## Data Logging

- **devstatuslog**: Logs device status (temperature, CPU/RAM usage) to `log/Driver_{timestamp}/Conn_{timestamp}/dev_status.csv`
- **recorddata**: Records data importable to MindCloud (uses ~9.5GB per 10 minutes)

## Testing Notes

- Relocalization works best within 1m ± 10° of original SLAM trajectory start
- Environment-dependent performance (distinctive scenes = better matching)
- Device can be moved during fallback SLAM mode to improve relocalization
