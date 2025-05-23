# Contents
1. [Kimera Semantics on custom dataset](#kimera-semantics-on-custom-dataset)
   1. [Setup](#setup)
      1. [Pull From Docker Hub](#a-pull-from-docker-hub)
      2. [Clone repo and build locally](#b-clone-repo-and-build-locally)
   2. [Running Kimera VIO Semantics with ROS in docker container](#running-kimera-vio-semantics-with-ros-in-docker-container)
      1. [Run Kimera VIO Semantics](#run-kimera-vio-semantics)
      2. [Run Kimera VIO on EuRoC dataset](#run-kimera-vio-on-euroc-dataset)
      3. [Kimera LCD module](#kimera-lcd-module)
         1. [RPGO](#rpgo)
         2. [VIO](#vio)
2. [Notes](#notes)
   1. [Docker Peek](#docker-peek)
   2. [Converting ROS2 bags to ROS1 bags](#converting-ros2-bags-to-ros1-bags)
   3. [Install semantics if not in container](#install-semantics-if-not-in-container)
3. [Troubleshooting](#troubleshooting)
4. [Acknowledgements](#acknowledgements)

# Kimera Semantics on custom dataset
You may either use the docker image or clone this github repo and build the docker image locally. \
Please download our datasets from our Google Drive:
- [ground_truth_4.bag](https://drive.google.com/file/d/16pu0glhqrX1xnswjnMnqstqEIOkwyp0_/view?usp=sharing)
- [ground_truth_6.bag](https://drive.google.com/file/d/1cKDJs9B9hSjD9ZxbOhzEE1h0aWyplCqp/view?usp=sharing)
- [loop_closures.bag](https://drive.google.com/file/d/12F7ZLgMf913-mRY1b-_LIXHa7uIA2YsY/view?usp=sharing)
- [strider_to_ARTarena.bag](https://drive.google.com/file/d/1Lr-9MYCM-SkG1wggfkD__S4I2y198DUI/view?usp=sharing)
- [vending_mach_to_strider.bag](https://drive.google.com/file/d/1EER6T_0gMrjT36P-xyhxtqXh-vl-ygh8/view?usp=sharing)

## Setup
### A) Pull From Docker Hub
```bash
[sudo] docker pull aurunima/kimera_vio_sem:latest
```
Change the path to point to the folder where your datasets are stored on your local system and run the docker container
```bash
#!/bin/bash

xhost +local:root
docker run --memory=8g --memory-swap=8g -it \
    --privileged \
    --env="DISPLAY" \
    --env="QT_X11_NO_MITSHM=1" \
    --device=/dev/bus/usb \
    -v /dev/bus/usb:/dev/bus/usb \
    -v /etc/udev/rules.d:/etc/udev/rules.d \
    -v /lib/udev:/lib/udev \
    -v /run/udev:/run/udev \
    --volume="/tmp/.X11-unix:/tmp/.X11-unix:rw" \
    --volume="/home/<user>/<datasets_folder>/:/datasets/" \
    --device /dev/dri:/dev/dri \
    --name kimera_vio_sem aurunima/kimera_vio_sem:latest
xhost -local:root
```
You may save this script (`<script-name>.bash`) and execute it.
```bash
chmod +x /path/to/<script-name>.bash
sudo ./path/to/<script-name>.bash
```

### B) Clone repo and build locally
```bash
git clone https://github.com/Semantic-Digital-Twin/Kimera-VIO-with-Realsense-D435i-Datasets.git
```
Build the docker container
```bash
cd Kimera-VIO-with-Realsense-D435i-Datasets
sudo docker build [--no-cache] -t kimera_vio_sem_ros [-f <path-to-dockerfile>] .
```
Change the path  ```vi scripts/kimera_vio_sem_docker.bash``` to point to the folder where your datasets are stored on your local system and run the docker container. \
If you are using the docker image, you may also need to change the name of the image in the file from ```kimera_vio_sem_ros``` to ```aurunima/kimera_vio_sem:v<X>.<y>```
```bash
[cd Kimera-VIO-with-Realsense-D435i-Datasets]
chmod +x scripts/kimera_vio_sem_docker.bash
sudo ./scripts/kimera_vio_sem_docker.bash
```
Once you are in the container, download the kimera_semantics demo bag
```bash
mkdir /datasets/kimera && cd /datasets/kimera
gdown 1SG8cfJ6JEfY2PGXcxDPAMYzCcGBEh4Qq
```


## Running Kimera VIO Semantics with ROS in docker container
You might need to ensure that auto-initialise is changed from ```0``` to ```1``` in _/catkin_ws/src/Kimera-VIO/params/Euroc/BackendParams.yaml_

Sourcing added to bashrc file in the docker container **(You don't need to do this)**
```bash
echo 'source /opt/ros/noetic/setup.bash' >> ~/.bashrc
echo 'source ~/catkin_ws/devel/setup.bash' >> ~/.bashrc
```
If you need to source, just use
```bash
source ~/.bashrc
```

To list all the ROS packages that are installed
```bash
rospack list-names
```

### Run Kimera VIO Semantics Demo
Copy the demo rosbag to kimera_semantics package
```bash
cp /datasets/kimera/kimera_semantics_demo.bag /catkin_ws/src/kimera_semantics_ros/rosbag/
```
You will need 3 terminals with ```cd /catkin_ws``` in all of them \
**Terminal 1:**
```bash
roscore
```

**Terminal 2:**
```bash
rviz -d $(rospack find kimera_semantics_ros)/rviz/kimera_semantics_gt.rviz
```

**Terminal 3:**
```bash
roslaunch kimera_semantics_ros kimera_semantics.launch play_bag:=true
```

### Run Kimera VIO Semantics on Custom Data

To run Kimera Semantics on the data record at ARTGarage, follow the instructions below:

You will need 5 terminals with ```cd /catkin_ws``` in all of them \
**Terminal 1:**
```bash
roscore
```
**Terminal 2:**
```bash
rviz -d $(rospack find kimera_semantics_ros)/rviz/kimera_semantics_gt_custom.rviz
```
**Terminal 3:**
```bash
roslaunch kimera_semantics_ros kimera_semantics_custom.launch
```
**Terminal 4:**
```bash
roslaunch kimera_vio_ros kimera_vio_ros_realsense_IR.launch use_lcd:=true should_use_sim_time:=true
```

**Terminal 5:**
```bash
rosbag play --pause --clock /path/to/bag/file
```
or if you want to perform the reconstruction live from your camera feed (make sure to change the topic names as per the ones coming from the realsense node).
```bash
roslaunch realsense2_camera rs_camera.launch \
   align_depth:=true \
   unite_imu_method:="linear_interpolation" \
   enable_gyro:=true \
   enable_accel:=true \
   enable_infra1:=true \
   enable_infra2:=true\
   infra_fps:=30 \
   infra_height:=480 \
   infra_width:=640 \
   color_fps:=30 \
   color_height:=480 \
   color_width:=640 \
   depth_fps:=30 \
   depth_height:=480 \
   depth_width:=640 \
```

**Remember:**
- **Change the frame_ids in the kimera_semantics_custom.launch file depending on the left_cam_topic (color or infra1).**
- **Change the topics in the kimera_vio_ros_realsense_IR.launch to match the topics coming from either the bag file or your RealSense camera.**
- **Modify the parameter ```<arg name="metric_semantic_reconstruction" default="true"/>``` to false (either in the launch file or during runtime) if you are NOT performing semantic reconstruction i.e, if you do NOT have semantic segmented images**

**NOTE: The data recorded at ARTGarage is raw sensor data and does NOT have its own Odometry topic or TF Transforms. Kimera-VIO estimates its own left_cam & right_cam to base_link transforms hence there will be no need to record transforms coming from the RealSense. The only missing transform is the World**

### Run Kimera VIO on EuRoC dataset
Ensure you have the corresponding .bag file downloaded in your ```/datasets``` folder. \
You may download the datasets from [MIT Spark's EuRoC Google Drive](https://drive.google.com/drive/folders/1_kwqHojvBusHxilcclqXh6haxelhJW0O) using ```gdown```
```bash
gdown <download ID>
```
Example
```bash
mkdir /datasets/EuRoC && cd /datasets/EuRoC
gdown http://robotics.ethz.ch/~asl-datasets/ijrr_euroc_mav_dataset/vicon_room1/V1_01_easy/V1_01_easy.bag
```

You will need 4 terminals with ```cd /catkin_ws``` in all of them \
**Terminal 1:**
```bash
roscore
```
**Terminal 2:**
```bash
rosbag play /datasets/EuRoC/V1_01_easy.bag
```
Hit spacebar to pause until roslaunch OR use this
```bash
rosbag play --clock --pause /datasets/EuRoC/V1_01_easy.bag --rate 3
```
**Terminal 3:**
```bash
rviz -d $(rospack find kimera_vio_ros)/rviz/kimera_vio_euroc.rviz
```
**Terminal 4:**
```bash
roslaunch kimera_vio_ros kimera_vio_ros_euroc.launch online:=true
```

If not playing rosbag and just using one from storage, use
```bash
roslaunch kimera_vio_ros kimera_vio_ros_euroc.launch online:=false rosbag_path:="/datasets/EuRoC/V1_01_easy.bag"
```

### Kimera LCD module
To use the loop closure detection module and the RPGO, run the vio launch file with the ```use_lcd:=true``` parameter:
```bash
roslaunch kimera_vio_ros kimera_vio_ros_realsense_IR.launch use_lcd:=true
```
You need to tweak the parameters for your use case

#### RPGO
```bash
vi /catkin_ws/src/Kimera-RPGO/include/KimeraRPGO/SolverParams.h
```
RPGO debug (you could add in your debug statements here)
```bash
vi /catkin_ws/src/Kimera-RPGO/Kimera-RPGO/src/RobustSolver.cpp
```
#### VIO
```bash
vi /catkin_ws/src/Kimera-VIO/src/loopclosure/LoopClosureDetector.cpp
```
Similarly modify 
```
/catkin_ws
└── src
    └── Kimera-VIO
        └── params
            └── RealSenseIR
                ├── LeftCameraParams.yaml
                ├── RightCameraParams.yaml
                ├── ImuParams.yaml
                ├── LcdParams.yaml
```
for your use case


# Notes

### Docker Peek
To start a stopped container,
```bash
sudo docker start -i <container-name>
OR
sudo docker start -i <container-id>
```
To open a terminal in the container
```bash
sudo docker exec -it <container-name> /bin/bash
```
If you wish for the container to be removed once you exit it, use the `--rm` flag
```bash
sudo docker run [...] --rm [...] --name <container_name> <image_name>:<tag>
```
Do not forget to commit the changes before exiting if you wish to save the changes you make.

`--memory=8g`: Limits the container’s memory usage to 8 GB. \
`--memory-swap=8g`: Limits the total memory and swap space usage to 8 GB (prevents the container from swapping too much). \
`-it`: Runs the container interactively (-i for interactive) and attaches a terminal (-t).

### Converting ROS2 bags to ROS1 bags
Supposing that you have the ROS2 bags in zipped format, to convert ROS2bag files (.db3) to ROS1bag files (.bag), \
just unzip them in your datasets directory (in your docker container).
```bash
unzip -d </path/to/datasets/destination> </path/to/zipped_rosbag>/vending_mach_to_strider.zip
unzip -d /datasets/realsense/ /datasets/realsense/strider_to_ARTarena.zip
```
Then convert the ROS2 bags to ROS1 bags
```bash
rosbags-convert /datasets/realsense/rosbag2_2025_02_07-14_34_37/ --dst /datasets/realsense/vending_mach_to_strider.bag
rosbags-convert /datasets/realsense/rosbag2_2025_02_07-14_53_07/ --dst /datasets/realsense/strider_to_ARTarena.bag
```
You will find the ROS1 bag file in `/datasets/realsense`

There might be an issue with the camera_info topics. \
To resolve that, use the convert_camerainfo.py script
```bash
python3 /Scripts/convert_cameraInfo.py --source_bag </path/to/source/ROS1bag> --destination_bag </path/to/final/ROS1bag>
python3 /Scripts/convert_cameraInfo.py --source_bag /datasets/realsense/strider_to_ARTarena.bag --destination_bag /datasets/realsense/strider_to_ARTarena_fixed.bag
```

### Install semantics if not in container
In the container, run the following commands
```bash
catkin init
catkin config --extend /opt/ros/noetic
catkin config --cmake-args -DCMAKE_BUILD_TYPE=Release
```
Clone repo 
```bash
cd ~/catkin_ws/src
git clone git@github.com:MIT-SPARK/Kimera-Semantics.git
wstool merge Kimera-Semantics/install/kimera_semantics_ssh.rosinstall
```
Download and update all dependencies
```bash
wstool update
```
Compile code
```bash
catkin build kimera_semantics_ros
```
Refresh workspace
```bash
source ~/catkin_ws/devel/setup.bash
```
OR
```bash
source ~/.bashrc
```

# Troubleshooting

#### Not able to execute script
If you get this error upon trying to execute a script
```bash
sudo: <script-filename>.bash: command not found
```
Ensure that you have made it executable
```bash
chmod +x <script-filename>.bash
```

#### ROSbag conversion not working
Ensure that rosbags-converter is installed in your docker container
```bash
pip3 install rosbags>=0.9.12
apt update && apt upgrade
```
<!-- Why do we not use the names of the rosbags? -->
<!-- Because, then we need to change the filename, folder name and also in the metadata file -->
<!-- rosbags-convert /datasets/realsense/rosbag2_2025_02_07-14_34_37/ --dst /datasets/realsense/vending_mach_to_strider.bag -->
<!-- rosbags-convert /datasets/realsense/rosbag2_2025_02_07-14_53_07/ -->

# Acknowledgements
We have used [MIT SPARK's](https://github.com/MIT-SPARK) [Kimera](https://github.com/MIT-SPARK/Kimera) for our custom dataset. \
The focus of this repo is [Kimera Semantics](https://github.com/MIT-SPARK/Kimera-Semantics)

