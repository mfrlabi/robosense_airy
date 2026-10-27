# robosense_airy




## 0. Quick Reference

bash
# One-time build
    
    mkdir -p ~/robosense_airy_ws/src && cd ~/robosense_airy_ws/src
    git clone https://github.com/RoboSense-LiDAR/rslidar_sdk.git
    cd rslidar_sdk && git submodule init && git submodule update
    cd ~/robosense_airy_ws && catkin_make && source devel/setup.bash

# One-time config: edit config.yaml -> lidar_type: RSAIRY, imu_port: 6688

# Every session

    source ~/robosense_airy_ws/devel/setup.bash
    ~/robosense_airy_ws/devel/lib/rslidar_sdk/rslidar_sdk_node __name:=my_airy_node

# In another terminal

    rostopic hz /rslidar_points /rslidar_imu_data
    rosbag record -b 32 /rslidar_points /rslidar_imu_data -O ~/airy_data.bag

# Verify

    rosbag info ~/airy_data.bag

# to visualize with rviz   ~/robosense_airy_ws$ roslaunch rslidar_sdk start.launch

    source ~/robosense_airy_ws/devel/setup.bash
    roslaunch rslidar_sdk start.launch


# RoboSense Airy LiDAR on Unitree Jetson (ROS Noetic)

Working setup notes for the RoboSense **Airy** LiDAR + IMU on a Unitree robot's Jetson
(Ubuntu 20.04 aarch64, ROS Noetic). Covers build, config, run, record, and the pitfalls
that cost us the most time.

---

## 1. Hardware / Network Facts (this robot)

| Item | Value |
| :--- | :--- |
| LiDAR model | RoboSense **Airy** |
| LiDAR IP | `192.168.144.200` (Unitree re-IPs it from factory `192.168.1.200`) |
| Jetson `eth0` IP | `192.168.144.3` |
| MSOP port (data) | `6699` |
| DIFOP port (info) | `7788` |
| IMU port | `6688` |
| IMU rate | ~200 Hz |
| Point cloud rate | ~10 Hz |

### Verify connectivity

```bash
ping -c 5 192.168.144.200
```

If it fails and the LiDAR is at factory IP `192.168.1.200`:

```bash
sudo ip addr add 192.168.1.102/24 dev eth0
ping -c 5 192.168.1.200
```

Confirm packets are arriving (needs `sudo apt install tcpdump`):

```bash
sudo tcpdump -i eth0 -n udp port 6699 -c 5    # point cloud
sudo tcpdump -i eth0 -n udp port 6688 -c 5    # IMU
```

---

## 2. System Environment

| Item | Value |
| :--- | :--- |
| OS | Ubuntu 20.04.6 LTS (aarch64) |
| ROS | Noetic (`ROS_VERSION=1`) |
| Cores | 4 |
| RAM | 7.5 GB total — **tight** under robot autonomy load |

### APT fix (Jetson cannot reach overseas repos)

```bash
# Force IPv4 (robot Wi-Fi has no IPv6 route)
echo 'Acquire::ForceIPv4 "true";' | sudo tee /etc/apt/apt.conf.d/99force-ipv4

# Comment out unreachable ports.ubuntu.com
sudo sed -i 's|^deb http://ports.ubuntu.com|# deb http://ports.ubuntu.com|' /etc/apt/sources.list

# Disable NVIDIA + Microsoft repos (not needed for LiDAR)
sudo mv /etc/apt/sources.list.d/nvidia-l4t-apt-source.list{,.disabled} 2>/dev/null
sudo mv /etc/apt/sources.list.d/vscode.sources{,.disabled} 2>/dev/null

sudo apt update     # should end clean: "All packages are up to date."
```

---

## 3. Build `rslidar_sdk` from Source

> The current repo (v1.5.20) ships a **unified `package.xml`** (format 3) that works with
> both ROS1 and ROS2 via `$ROS_VERSION` conditions. **Do NOT** copy `package_ros1.xml`
> (it no longer exists) and **do NOT** edit `COMPILE_METHOD` (no longer in `CMakeLists.txt`).
> Those instructions are for older SDK versions.

```bash
sudo apt install -y libyaml-cpp-dev libpcap-dev

mkdir -p ~/robosense_airy_ws/src
cd ~/robosense_airy_ws/src
git clone https://github.com/RoboSense-LiDAR/rslidar_sdk.git
cd rslidar_sdk
git submodule init
git submodule update

cd ~/robosense_airy_ws
catkin_make
source devel/setup.bash
```

Verify:

```bash
rospack find rslidar_sdk
# -> /home/jetson/robosense_airy_ws/src/rslidar_sdk
ls -la ~/robosense_airy_ws/devel/lib/rslidar_sdk/rslidar_sdk_node
```

---

## 4. Configure the Airy

Edit `~/robosense_airy_ws/src/rslidar_sdk/config/config.yaml`. Only **two** values
need changing from defaults:

```yaml
lidar_type: RSAIRY       # was RSM1 (or whatever default)
imu_port: 6688           # was 0 ("no IMU")
```

Everything else default is correct:

```yaml
msg_source: 1
send_point_cloud_ros: true
msop_port: 6699
difop_port: 7788
min_distance: 0.2
max_distance: 200
use_lidar_clock: 1
dense_points: 0
```

IMU parsing is already enabled by default in `CMakeLists.txt`:

```cmake
option(ENABLE_IMU_DATA_PARSE "Enable imu data parse" ON)
```

No rebuild needed after editing `config.yaml` — it is read at runtime.

---

## 5. Run the Driver

> **CRITICAL:** `rosrun rslidar_sdk rslidar_sdk_node` and `roslaunch rslidar_sdk start.launch`
> both execute the **system-installed** binary at `/opt/ros/noetic/lib/rslidar_sdk/rslidar_sdk_node`.
> That binary loads the **system config** (`/opt/ros/noetic/share/rslidar_sdk/config/config.yaml`)
> and **crashes** with `std::system_error`. Run the workspace binary **directly** instead.

### Launch (workspace binary, isolated name)

```bash
~/robosense_airy_ws/devel/lib/rslidar_sdk/rslidar_sdk_node __name:=my_airy_node
```

**`__name:=my_airy_node` is required** — the robot already runs its own
`/rslidar_sdk_node`. Without it, your node kills the robot's driver and gets killed
back by `rosmon`.

Verify the config path in the log is:

```
Config loaded from PATH:
/home/jetson/robosense_airy_ws/src/rslidar_sdk/config/config.yaml
```

Also confirm `imu_port: 6688` and `min_distance: 0.2` in the printed parameters.

### Verify topics

```bash
rostopic hz /rslidar_points       # ~10 Hz
rostopic hz /rslidar_imu_data     # ~200 Hz
```

> **Topic names are `/rslidar_points` and `/rslidar_imu_data` — NOT `/airy/...`.**
> The topic names in `config.yaml` are absolute paths (`/rslidar_points`), so the node
> namespace has no effect on them.

---

## 6. Record a ROS Bag

The Jetson has only ~100 MB free RAM under robot load. Use a **small** buffer:

```bash
rosbag record -b 32 \
  /rslidar_points /rslidar_imu_data \
  -O ~/airy_data.bag
```

- `-b 32` = 32 MB per-topic buffer (default 256 MB will OOM this system)
- Let it run, then `Ctrl+C` to stop

Verify:

```bash
rosbag info ~/airy_data.bag
```

Expected for a 30 s recording:

```
duration:    ~30s
size:        ~40-60 MB
topics:
    /rslidar_imu_data    ~6000 msgs   : sensor_msgs/Imu
    /rslidar_points      ~300 msgs    : sensor_msgs/PointCloud2
```

If `size` is a few KB and `topics:` is empty, the topic names are wrong — recheck with
`rostopic list | grep rslidar`.

---

## 7. Visualize on Laptop (NOT on Jetson)

The Jetson is headless — RViz on the Jetson fails with
`qt.qpa.xcb: could not connect to display`. Run RViz on your laptop instead.

**On your laptop** (same Wi-Fi as the robot):

```bash
sudo apt install ros-noetic-desktop-full
source /opt/ros/noetic/setup.bash

export ROS_MASTER_URI=http://192.168.12.1:11311   # Jetson IP
export ROS_IP=$(hostname -I | awk '{print $1}')   # your laptop's IP

ping 192.168.12.1        # must respond
rostopic list            # must list Jetson topics
rviz
```

In RViz:
- **Fixed Frame**: `rslidar`
- **Add → PointCloud2** → topic `/rslidar_points`
- **Add → ImU** → topic `/rslidar_imu_data`

---

## 8. Known Pitfalls (what cost us time)

| Symptom | Cause | Fix |
| :--- | :--- | :--- |
| `std::system_error: Invalid argument` on launch | Running `/opt/ros/...` system binary | Run workspace binary directly (step 5) |
| `std::system_error` on `Ctrl+C` | Harmless shutdown quirk in v1.5.20 | Ignore — data is already recorded |
| `imu_port: 0` in runtime log despite config | Stale CMake cache / wrong config path | Rebuild with `rm -rf build devel && catkin_make` |
| `new node registered with same name` | Name clash with robot's driver | `__name:=my_airy_node` |
| Empty 4 KB bag | Recording `/airy/...` (doesn't exist) | Record `/rslidar_points /rslidar_imu_data` |
| RViz crashes: `Could not load Qt platform plugin "xcb"` | Headless SSH, no display | Run RViz on laptop |
| `apt update` times out on NVIDIA / Microsoft / ports.ubuntu | No route to overseas repos | See step 2 |
| `cloud_preproce_node` at 155% CPU | Robot autonomy stack | Optional: stop CV nodes if robot stationary |
| `rosbag record` drops frames | CPU/memory contention (4 cores, ~100 MB free) | Use `-b 32`, `nice -n -5`, stop CV nodes |

---

## 9. Airy-Specific Data Notes

### IMU

- **`linear_acceleration` is in units of `g`**, not `m/s²`. Multiply by **9.80665**
  before feeding into FAST-LIO / LIO-SAM / etc.
- **`orientation` is a fixed default** — the Airy does **not** perform orientation
  estimation.

### Coordinate system

- The Airy uses a **non-standard left-handed** coordinate system (Z is not up).
- For SLAM pipelines, apply a rotation matrix to align to REP-103 (`X forward, Y left,
  Z up`) — or check what `/rslidar_tf_node` publishes.

### Clock

- `use_lidar_clock: true` means point clouds are timestamped by the LiDAR's internal
  clock, not the Jetson's. If your SLAM stack expects system time, set it to `false`.

---

## 10. Quick Reference

```bash
# One-time build
mkdir -p ~/robosense_airy_ws/src && cd ~/robosense_airy_ws/src
git clone https://github.com/RoboSense-LiDAR/rslidar_sdk.git
cd rslidar_sdk && git submodule init && git submodule update
cd ~/robosense_airy_ws && catkin_make && source devel/setup.bash

# One-time config: edit config.yaml -> lidar_type: RSAIRY, imu_port: 6688

# Every session
source ~/robosense_airy_ws/devel/setup.bash
~/robosense_airy_ws/devel/lib/rslidar_sdk/rslidar_sdk_node __name:=my_airy_node

# In another terminal
rostopic hz /rslidar_points /rslidar_imu_data
rosbag record -b 32 /rslidar_points /rslidar_imu_data -O ~/airy_data.bag

# Verify
rosbag info ~/airy_data.bag
```
