# ros2-ardupilot-ethernet-guide

This repo shows how to run **native ROS 2 (DDS) directly with ArduPilot** over a **wired Ethernet link**—no MAVLink bridge required. The flight controller publishes XRCE-DDS topics straight to your companion computer (Jetson/RPi/PC), giving you **low-latency, high-bandwidth, many-to-many** ROS 2 communications for IMU, EKF pose, GPS, params, and more. Compared to UART, Ethernet enables **true DDS discovery/QoS**, easier debugging (tcpdump, iptables), robust wiring, and headroom for **mapping/SLAM/image processing** workloads. You’ll set static IPs (e.g., FC `192.168.144.14`, companion `192.168.144.2`), enable `NET_*` + `DDS_*` params on ArduPilot (4.7-dev with Micro XRCE-DDS), and run the micro-ROS agent on the companion to start consuming `/ap/*` topics in ROS 2.



## Hardware Requirements

- Flight Controller having **ETH PORT** (PIXHWAK 6X IS USED FOR TESTING)
- Companion Computer (RPI4 IS USED FOR TESTING)
- ETHERNET CABLE
- ETHERNET BREAKOUT BOARD
- 4 PIN 2.5MM JST CABLE




## Pin Mapping

Map the Pixhawk’s 4-pin Ethernet (JST-GH) to your RJ45 breakout as follows:

| Pixhawk ETH (4-pin) | RJ45 Breakout Pin |
| ------------------- | ----------------- |
| **Pin 1** →         | **Pin 1**         |
| **Pin 2** →         | **Pin 2**         |
| **Pin 3** →         | **Pin 3**         |
| **Pin 4** →         | **Pin 6**         |

Use ethenet cable to connect RJ45 breakout board to companion computer's ethernet port.




## Flashing Custom Ardupilot 4.7.0 dev

Open the ArduPilot Custom Build [Server](https://custom.ardupilot.org/add_build) and select the options exactly as shown in [Image 1](https://github.com/Shreyashkinhikar/ros2-ardupilot-ethernet-guide/blob/main/Image%201.png), then submit the build.For Flashing custom Tutotial follow this [link](https://ardupilot.org/planner/docs/common-loading-firmware-onto-pixhawk.html)



## Parameters (ArduPilot → ROS 2 over Ethernet)

| Param | Value |
|---|---|
| NET_ENABLE | 1 |
| NET_DHCP | 0 |
| NET_IPADDR0..3 | 192.168.144.14 |
| NET_NETMASK | 24 |
| NET_GWADDR0..3 | 192.168.144.1 |
| DDS_ENABLE | 1 |
| DDS_IP0..3 | 192.168.144.2 |  companion’s eth0 (your RPi/Jetson)
| DDS_UDP_PORT | 2019 |  agent port (use same below)
| DDS_DOMAIN_ID | 0 |  ROS2 DOMAIN ID
| DDS_TIMEOUT_MS | 0 | for infinite try
| DDS_MAX_RETRY | 60 |


## Install ArduPilot ROS 2 Workspace & micro-ROS Agent (RPi 4)

Install ArduPilot ros2 workspace so it can publish XRCE-DDS data, and install the micro-ROS agent on the RPi 4 to expose those as native ROS 2 topics you can subscribe to.


```bash
mkdir -p ~/ardu_ws/src
cd ~/ardu_ws
vcs import --recursive --input  https://raw.githubusercontent.com/ArduPilot/ardupilot/master/Tools/ros2/ros2.repos src
cd ~/ardu_ws
sudo apt update
rosdep update
source /opt/ros/humble/setup.bash
rosdep install --from-paths src --ignore-src -r -y
sudo apt install default-jre
cd ~/ardu_ws
git clone --recurse-submodules https://github.com/ardupilot/Micro-XRCE-DDS-Gen.git
cd Micro-XRCE-DDS-Gen
./gradlew assemble
echo "export PATH=\$PATH:$PWD/scripts" >> ~/.bashrc
source ~/.bashrc
microxrceddsgen -help
# microxrceddsgen usage:
#     microxrceddsgen [options] <file> [<file> ...]
#     where the options are:
#             -help: shows this help
#             -version: shows the current version of eProsima Micro XRCE-DDS Gen.
#             -example: Generates an example.
#             -replace: replaces existing generated files.
#             -ppDisable: disables the preprocessor.
#             -ppPath: specifies the preprocessor path.
#             -I <path>: add directory to preprocessor include paths.
#             -d <path>: sets an output directory for generated files.
#             -t <temp dir>: sets a specific directory as a temporary directory.
#             -cs: IDL grammar apply case sensitive matching.
#     and the supported input files are:
#     * IDL files.
cd ~/ardu_ws
colcon build --packages-up-to ardupilot_dds_tests

  
```
    
## Demo

Follow the steps given

- Connect the pixhwak to laptop(Window's laptop is best as mission planner is supported)


- Power the Rpi4 and create 3 ssh terminals

  Terminal 1:
  ```bash
  export ROS_DOMAIN_ID=0
  source /opt/ros/humble/setup.bash
  ros2 run micro_ros_agent micro_ros_agent udp4 -p 2019 -v6

  ```
  Terminal 2:
  ```bash
  ros2 topic list
  ```
  Terminal 3:
  ```bash
  ros2 topic echo /ap/clock # or any topic of your choice
  ```


## Screenshots

![Terminal](https://github.com/Shreyashkinhikar/ros2-ardupilot-ethernet-guide/blob/main/DEMO.png)
![Hardware Setup](https://github.com/Shreyashkinhikar/ros2-ardupilot-ethernet-guide/blob/main/hardware.jpeg)



## Debugging: Verify eth0 IP & Match DDS Params (with expected output)
Use these quick checks if /ap/* topics don’t appear.

1 Check / set companion IP (RPi/Jetson)
```bash
 ip -brief addr show dev eth0
 output
 eth0 UP 192.168.144.2/24 ... # do not run this as a command

  ```

2 Link & traffic sanity
```bash
 sudo ethtool eth0 | grep -i 'Link detected'
 output
 Link detected: yes # do not run this as a command

 ping -c2 192.168.144.14
 output
 2 packets transmitted, 2 received, 0% packet loss # do not run this as a command
 rtt min/avg/max/... ≈ 0.2–1.0 ms# do not run this as a command

sudo tcpdump -n -i eth0 'src 192.168.144.14 and udp port 2019' -vv -c 3
output
IP 192.168.144.14.62510 > 192.168.144.2.2019: UDP, length 340# do not run this as a command
IP 192.168.144.14.62512 > 192.168.144.2.2019: UDP, length 128# do not run this as a command
...


  ```

## References
- [ROS2](https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debs.html)
- [ARDUPILOT](https://ardupilot.org/dev/docs/ros2-over-ethernet.html)
- [CUSTOM BUILD](https://custom.ardupilot.org/add_build)
