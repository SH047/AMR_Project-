# Wheeled AMR | ROS 2 Jazzy | Raspberry Pi 5

## Overview

This repository contains a ROS 2 Jazzy workspace for a wheeled differential-drive AMR running on a Raspberry Pi 5 with Ubuntu Server 24.04 ARM64. The robot uses an Arduino Mega 2560 for low-level motor and encoder handling, an RPLiDAR A1 for 2D scanning, a BNO055 IMU, Nav2 for autonomous navigation, SLAM Toolbox for mapping, and robot_localization EKF for wheel odometry plus IMU fusion.

The project is structured for learning and development, but the files are written to be usable on real hardware with persistent USB names, systemd startup, and standard ROS 2 launch/config conventions.

This stack is also prepared for future Acceleration Robotics deployment work, Modbus TCP/IP readiness, and higher-level dashboard/autonomy integrations.

## Hardware BOM

- Raspberry Pi 5, Ubuntu Server 24.04 ARM64, hostname `amrbot`
- Arduino Mega 2560 over USB, CH340 adapter, persistent symlink `/dev/mega`
- L298N dual H-bridge motor driver
- Two DC drive motors with quadrature encoders
- RPLiDAR A1 over USB, CP2102 adapter, persistent symlink `/dev/lidar`
- BNO055 IMU over I2C
- ST7789 SPI TFT, 320x240 landscape, with rotary encoder UI
- SHANWAN gamepad through ROS 2 `joy` and `teleop_twist_joy`

## System Architecture

```text
                         Mac / Operator
                              |
                              | SSH / ROS tools / future web dashboard
                              v
+---------------------------------------------------------------+
| Raspberry Pi 5 / Ubuntu 24.04 / ROS 2 Jazzy                   |
|                                                               |
|  +-----------------+     +-----------------+                  |
|  | rplidar_ros     | --> | /scan           |                  |
|  +-----------------+     +-----------------+                  |
|                                                               |
|  +-----------------+     +-----------------+                  |
|  | bno055          | --> | /bno055/imu     |                  |
|  +-----------------+     +-----------------+                  |
|                                                               |
|  +-----------------+     +-----------------+                  |
|  | amr_bridge      | --> | /odom + TF      |                  |
|  | /cmd_vel -> USB | <-- | Nav2 / teleop   |                  |
|  +-----------------+     +-----------------+                  |
|                                                               |
|  +-----------------+     +-----------------+                  |
|  | robot_localiz.  | --> | /odometry/filtered                 |
|  +-----------------+     +-----------------+                  |
|                                                               |
|  +-----------------+     +-----------------+                  |
|  | slam_toolbox    | --> | /map            |                  |
|  +-----------------+     +-----------------+                  |
|                                                               |
|  +-----------------+     +-----------------+                  |
|  | Nav2            | --> | /cmd_vel        |                  |
|  +-----------------+     +-----------------+                  |
+------------------------------|--------------------------------+
                               |
                               | USB serial /dev/mega
                               v
+---------------------------------------------------------------+
| Arduino Mega 2560                                              |
|  Serial command: L<pwm>R<pwm>                                  |
|  Telemetry:       L<ticks>R<ticks>                             |
|  L298N motor PWM + direction                                   |
|  Encoder tick counting                                         |
+---------------------------------------------------------------+
```

## Installation

Install ROS 2 Jazzy and the required packages on the Pi:

```bash
sudo apt update
sudo apt install -y \
  ros-jazzy-rplidar-ros \
  ros-jazzy-slam-toolbox \
  ros-jazzy-nav2-bringup \
  ros-jazzy-navigation2 \
  ros-jazzy-robot-localization \
  ros-jazzy-robot-state-publisher \
  ros-jazzy-joint-state-publisher \
  ros-jazzy-xacro \
  ros-jazzy-teleop-twist-joy \
  python3-serial
```

Create and build the workspace:

```bash
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src
git clone https://github.com/SH047/AMR_Project-.git
cd ~/ros2_ws
source /opt/ros/jazzy/setup.bash
colcon build
source install/setup.bash
```

Install persistent USB rules:

```bash
cd ~/ros2_ws/src/AMR_Project-
chmod +x scripts/setup_udev.sh
sudo ./scripts/setup_udev.sh
```

Reconnect the Arduino Mega and RPLiDAR, then verify:

```bash
ls -la /dev/mega /dev/lidar
```

## Bringup Instructions

Run the full stack:

```bash
source /opt/ros/jazzy/setup.bash
source ~/ros2_ws/install/setup.bash
ros2 launch amr_bringup bringup.launch.py
```

Useful checks:

```bash
ros2 topic list
ros2 topic echo /odom --once
ros2 topic echo /scan --once
ros2 topic echo /odometry/filtered --once
ros2 run tf2_tools view_frames
```

## Nav2 Usage

For mapping, drive the robot manually while SLAM Toolbox publishes `/map`. Save the map when complete:

```bash
ros2 run nav2_map_server map_saver_cli -f ~/ros2_ws/maps/amr_map
```

For navigation, use RViz2 or a dashboard to send a `NavigateToPose` goal in the `map` frame. Nav2 uses:

- NavFn global planner
- DWB local controller
- obstacle and inflation costmap layers
- LiDAR obstacle source from `/scan`
- differential-drive footprint constraints

## Teleop Usage

The platform is prepared for SHANWAN gamepad teleoperation through `teleop_twist_joy`. Use L1, button index `6`, as the enable button.

Example:

```bash
ros2 run joy joy_node
ros2 run teleop_twist_joy teleop_node --ros-args \
  -p enable_button:=6 \
  -p axis_linear.x:=1 \
  -p axis_angular.yaw:=0
```

## Known Issues

- The Arduino firmware in this repository follows the requested pin map. If your physical Mega wiring still uses an older pin map, update the firmware constants before uploading.
- `bno055` package launch details vary by driver. The bringup launch assumes an installed `bno055` package can publish `/bno055/imu`.
- RPLiDAR A1 scan quality depends heavily on stable 5V power.
- L298N voltage drop is significant. Tune max velocities and acceleration conservatively.
- The ST7789 TFT UI is hardware-specific and should be kept separate from safety-critical motion control.

## License

MIT License. See package manifests for package-level license declarations.

