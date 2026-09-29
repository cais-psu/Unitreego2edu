# Unitreego2edu
Unitreego2edu forDT


# Unitree Go2 EDU — Connection and RGB-D Capture Guide

Quick-start instructions for our Go2 EDU, a MacBook, and the mounted Intel RealSense D435. This guide covers access to the onboard computer, camera operation, and data export. Use the paired Unitree controller and the lab's approved procedure for standing, walking, and stopping the robot.

Last updated: September 29, 2026.

## 1. Our Setup

| Item | Configuration |
|---|---|
| Robot | Unitree Go2 EDU |
| Onboard OS | Ubuntu 20.04.5 LTS, ARM64 (`aarch64`) |
| NVIDIA software | L4T 35.3.1; kernel `5.10.104-tegra` |
| Robot Ethernet IP | `192.168.123.18/24` |
| SSH username | `unitree` |
| Mac Ethernet IP | `192.168.123.222/24` |
| Camera | RealSense D435, connected by USB to the Go2 onboard computer |
| Python dependencies | Python 3.8.10, NumPy 1.17.4, OpenCV 4.2.0 — confirmed installed |

**Current status:** Wired SSH and software downloads through the Mac are working. The D435 is visible in `lsusb`, but was last detected at USB 2 speed (`480M`). SDK installation and successful RGB-D capture still need confirmation on the robot.

## 2. Connect the Mac to Go2

1. Power on Go2 using the normal Unitree procedure and allow the onboard computer to boot. Keep the robot stationary during setup.
2. Connect the rear Ethernet port to the Mac using an Ethernet cable and a USB Ethernet adapter.
3. On the Mac, open **System Settings → Network → the Ethernet adapter → Details → TCP/IP**. Set:

| Setting | Value |
|---|---|
| Configure IPv4 | Manually |
| IP address | `192.168.123.222` |
| Subnet mask | `255.255.255.0` |
| Router | Leave blank for this direct connection |
| DNS | Not required for this direct connection |

Our Mac adapter is named **USB 10/100/1000 LAN**. Keep Mac Wi-Fi connected if the Mac needs Internet access. Preserve the robot's existing Ethernet configuration.

**On the Mac, open Terminal:**

```bash
ssh -o ConnectTimeout=10 unitree@192.168.123.18
```

Enter the robot account password. Password characters are not displayed while typing.

If the login menu asks `ros:foxy(1) noetic(2) ?`, enter **2** for the current camera workflow. The Python capture script does not require ROS. The separate CMU autonomy stack uses ROS 2 Foxy, selected with **1**; that stack has not been installed or validated by this guide.

**Command locations:** A prompt such as `yifeili@... ~ %` is the **Mac**. A prompt such as `unitree@ubuntu:~$` is **Go2**. Commands after SSH login run on Go2.

## 3. Install the Camera SDK — Once

Skip to Section 4 if the SDK is already installed in `~/venvs/d435`.

The project folder contains the official Python 3.8 / ARM64 wheel, its checksum, USB permission rules, and `capture_rgbd.py`. These installation files are for Go2, not the Mac's Python environment.

**Mac — upload the files:**

```bash
cd /Users/yifeili/Documents/ChatGPT/Unitree
scp -r d435_install capture_rgbd.py unitree@192.168.123.18:~/
```

On another team member's Mac, replace the local folder path with the location of this project.

**Go2 — verify the package:**

```bash
cd ~/d435_install
sha256sum -c SHA256SUMS
```

Continue only when the checksum reports `OK`. The required `python3-venv` and USB library were installed during setup.

**Go2 — create the environment and install the SDK:**

```bash
python3 -m venv --system-site-packages ~/venvs/d435
source ~/venvs/d435/bin/activate
python -m pip install --no-index ~/d435_install/pyrealsense2-2.55.1.6486-cp38-cp38-manylinux2014_aarch64.whl
```

The environment reuses the installed NumPy and OpenCV. SDK installation is offline and does not require the Internet proxy.

**Go2 — install camera access rules:**

```bash
sudo install -m 644 ~/d435_install/99-realsense-libusb.rules /etc/udev/rules.d/99-realsense-libusb.rules
sudo udevadm control --reload-rules
```

Unplug and reconnect **only the D435 USB cable**. Keep Ethernet connected. A robot reboot is not required for this step.

## 4. Check the Camera — Each Session

**Go2:**

```bash
source ~/venvs/d435/bin/activate
python -c "import pyrealsense2 as rs; print([(d.get_info(rs.camera_info.name), d.get_info(rs.camera_info.serial_number)) for d in rs.context().query_devices()])"
```

Expected: a list containing the D435 name and serial number. An empty list (`[]`) means the SDK did not find a device. Device detection alone does not confirm that frames can be captured.

Check the USB connection if needed:

```bash
lsusb
lsusb -t
```

The D435 USB ID is `8086:0b07`. Check the **camera's own row**: `480M` means USB 2; `5000M` or higher indicates USB 3. Prefer a suitable USB 3 cable and port for scanning. The USB root hub's speed alone does not establish the camera's connection speed.

## 5. Capture and View the First RGB-D Sequence

Keep Go2 stationary and point the camera toward a textured scene, such as a desk and room corner.

**Go2:**

```bash
python ~/capture_rgbd.py --out ~/dt_capture/test01 --seconds 20 --fps 15
```

This attempts 640 × 480 color and depth at 15 fps. The terminal reports saved frame pairs, valid depth percentage, center depth, and actual save rate. Keep this SSH window open until capture finishes. If an error appears, retain its full text before changing the setup.

**Use a new output directory for every run:** `test02`, `test03`, etc. The script refuses to overwrite existing data.

**Mac — download and open the previews:**

```bash
scp -r unitree@192.168.123.18:~/dt_capture/test01 ~/Downloads/
open ~/Downloads/test01/preview_first.png
open ~/Downloads/test01/preview_last.png
```

The preview shows color on the left and false-color depth on the right. The preview itself is not metric depth data.

| Output | Contents |
|---|---|
| `rgb/` | Color PNG images |
| `depth/` | Depth aligned to color; 16-bit PNG, millimeters, zero = invalid |
| `depth_native/` | Original depth values; unit specified in `calibration.json` |
| `calibration.json` | Camera intrinsics, distortion, internal depth-to-color extrinsics, and depth scale |
| `frames.csv` | Frame pairs, frame numbers, timestamps, and timestamp domains |
| `summary.json` | Capture status, frame count, save rate, and frame-number gaps |

**Camera poses are not included yet.** The D435 does not directly provide its position and orientation in the room. Subsequent RGB-D SLAM is needed to estimate those poses and build a map. Camera calibration and timestamps are not room-relative poses.

For a later moving scan, first pass the stationary test, secure the camera and cables, and use supervised manual robot control. Use slow motion and overlapping views; return to the starting area for a possible loop closure. Arrange recording that survives an SSH disconnect before attempting an untethered run. A foreground SSH command may stop when the connection drops.

## 6. Reconnect After an SSH Disconnect

**Mac:**

```bash
ssh -o ConnectTimeout=10 unitree@192.168.123.18
```

**Go2 — reactivate the Python environment:**

```bash
source ~/venvs/d435/bin/activate
```

Installed files remain after a disconnect. If capture was running, check for a surviving process before starting another one:

```bash
pgrep -af '[c]apture_rgbd.py'
```

If a capture process is still running, let it finish before starting another. Inspect the existing output directory before choosing a new recording name.

If SSH times out, check robot power, Ethernet cables, and the Mac's static IP. On the **Mac**, these checks can help:

```bash
route -n get 192.168.123.18
nc -vz -G 5 192.168.123.18 22
```

The route should use the wired adapter; the port check tests whether SSH is reachable. Rebooting Go2 is not a routine reconnect step.

## 7. Optional: Download Software Through the Mac

Use this only when Go2 needs Internet downloads. Offline SDK installation and local capture do not need it.

If Go2's clock is wrong after boot, run this from the **Mac**, whose clock should be correct:

```bash
ssh -t unitree@192.168.123.18 "sudo date -u -s '$(date -u '+%Y-%m-%d %H:%M:%S')'"
```

**Mac, Terminal A — start the proxy and leave this window open:**

```bash
ssh -N -o ExitOnForwardFailure=yes -o ServerAliveInterval=30 -R 127.0.0.1:1080 unitree@192.168.123.18
```

No prompt after login is normal. This command must start on the Mac, not inside an existing Go2 SSH session.

**Mac, Terminal B — SSH into Go2, then run on Go2:**

```bash
sudo apt-get \
  -o Acquire::http::Proxy=socks5h://127.0.0.1:1080/ \
  -o Acquire::https::Proxy=socks5h://127.0.0.1:1080/ \
  update
```

Use the same proxy options for `apt-get install`. This provides connectivity to proxy-aware programs, not general Internet routing for the whole robot. Press `Ctrl+C` in Terminal A to close the proxy.

Ubuntu package indexes have downloaded successfully through this setup. The ROS/ROS 2 mirrors still report `EXPKEYSIG F42ED6FBAB17C654`; their key configuration needs separate maintenance. Preserve signature verification. The current Python camera workflow does not require ROS packages or a full system upgrade.

## 8. Known Wi-Fi Issue and Optional Hardware

Activating our current USB adapter (`0e8d:7612`, MediaTek MT7612U, `mt76x2u`) was followed by onboard-computer resets. A changed boot ID and the reset reason `BCCPLEXWDT` confirmed a watchdog reset in one test. The exact driver/kernel/hardware cause remains unresolved. Continue using wired SSH for this setup rather than repeating the same hotspot activation test.

The CMU project lists a **Panda Wireless PAU03**, a 2.4 GHz adapter, as an alternative. It is a candidate for a single-unit trial; compatibility with our exact Jetson software has not been established.

Wireless HDMI is optional: the CMU team uses it to display the onboard Ubuntu desktop on a separate monitor. It is not required for our headless camera capture or offline mapping workflow and does not provide camera poses.

## References

- [CMU Go2 autonomy stack and real-robot setup](https://github.com/jizhang-cmu/autonomy_stack_go2#real-robot-setup)
- [Official RealSense Python SDK release used here](https://pypi.org/project/pyrealsense2/2.55.1.6486/)
- [RealSense Jetson installation and backend options](https://github.com/realsenseai/librealsense/blob/v2.55.1/doc/installation_jetson.md)
- [Panda PAU03 manufacturer specifications](https://www.pandawireless.com/pandaUltra.htm)

Companion files: `capture_rgbd.py`, `prepare_rtabmap.py`, and the detailed Chinese workflow `D435_DT_操作指南.md`.
