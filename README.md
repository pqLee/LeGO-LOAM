# LeGO-LOAM

This repository contains code for a lightweight and ground optimized lidar odometry and mapping (LeGO-LOAM) system for ROS compatible UGVs. The system takes in point cloud  from a Velodyne VLP-16 Lidar (palced horizontally) and optional IMU data as inputs. It outputs 6D pose estimation in real-time. A demonstration of the system can be found here -> https://www.youtube.com/watch?v=O3tz_ftHV48
<!--
[![Watch the video](/LeGO-LOAM/launch/demo.gif)](https://www.youtube.com/watch?v=O3tz_ftHV48)
-->
<p align='center'>
    <img src="/LeGO-LOAM/launch/demo.gif" alt="drawing" width="800"/>
</p>

## Lidar-inertial Odometry

An updated lidar-initial odometry package, [LIO-SAM](https://github.com/TixiaoShan/LIO-SAM), has been open-sourced and available for testing.

## Dependency

- [ROS](http://wiki.ros.org/ROS/Installation) (tested with indigo, kinetic, and melodic)
- [gtsam](https://github.com/borglab/gtsam/releases) (Georgia Tech Smoothing and Mapping library, 4.0.0-alpha2)
  ```
  wget -O ~/Downloads/gtsam.zip https://github.com/borglab/gtsam/archive/4.0.0-alpha2.zip
  cd ~/Downloads/ && unzip gtsam.zip -d ~/Downloads/
  cd ~/Downloads/gtsam-4.0.0-alpha2/
  mkdir build && cd build
  cmake ..
  sudo make install
  ```

## Compile

You can use the following commands to download and compile the package.

```
cd ~/catkin_ws/src
git clone https://github.com/RobustFieldAutonomyLab/LeGO-LOAM.git
cd ..
catkin_make -j1
```
When you compile the code for the first time, you need to add "-j1" behind "catkin_make" for generating some message types. "-j1" is not needed for future compiling.

## The system

LeGO-LOAM is speficifally optimized for a horizontally placed VLP-16 on a ground vehicle. It assumes there is always a ground plane in the scan. The UGV we are using is Clearpath Jackal. It has a built-in IMU. 

<p align='center'>
    <img src="/LeGO-LOAM/launch/jackal-label.jpg" alt="drawing" width="400"/>
</p>

The package performs segmentation before feature extraction.

<p align='center'>
    <img src="/LeGO-LOAM/launch/seg-total.jpg" alt="drawing" width="400"/>
</p>

Lidar odometry performs two-step Levenberg Marquardt optimization to get 6D transformation.

<p align='center'>
    <img src="/LeGO-LOAM/launch/odometry.jpg" alt="drawing" width="400"/>
</p>

## New Lidar

The key thing to adapt the code to a new sensor is making sure the point cloud can be properly projected to an range image and ground can be correctly detected. For example, VLP-16 has a angular resolution of 0.2&deg; and 2&deg; along two directions. It has 16 beams. The angle of the bottom beam is -15&deg;. Thus, the parameters in "utility.h" are listed as below. When you implement new sensor, make sure that the ground_cloud has enough points for matching. Before you post any issues, please read this.

```
extern const int N_SCAN = 16;
extern const int Horizon_SCAN = 1800;
extern const float ang_res_x = 0.2;
extern const float ang_res_y = 2.0;
extern const float ang_bottom = 15.0;
extern const int groundScanInd = 7;
```

Another example for Velodyne HDL-32e range image projection:

```
extern const int N_SCAN = 32;
extern const int Horizon_SCAN = 1800;
extern const float ang_res_x = 360.0/Horizon_SCAN;
extern const float ang_res_y = 41.333/float(N_Scan-1);
extern const float ang_bottom = 30.666666;
extern const int groundScanInd = 20;
```

**New**: a new **useCloudRing** flag has been added to help with point cloud projection (i.e., VLP-32C, VLS-128). Velodyne point cloud has "ring" channel that directly gives the point row id in a range image. Other lidars may have a same type of channel, i.e., "r" in Ouster. If you are using a non-Velodyne lidar but it has a similar "ring" channel, you can change the PointXYZIR definition in utility.h and the corresponding code in imageProjection.cpp.

For **KITTI** users, if you want to use our algorithm with  **HDL-64e**, you need to write your own implementation for such projection. If the point cloud is not projected properly, you will lose many points and performance.

If you are using your lidar with an IMU, make sure your IMU is aligned properly with the lidar. The algorithm uses IMU data to correct the point cloud distortion that is cause by sensor motion. If the IMU is not aligned properly, the usage of IMU data will deteriorate the result. Ouster lidar IMU is not supported in the package as LeGO-LOAM needs a 9-DOF IMU.

## Run the package

1. Run the launch file:
```
roslaunch lego_loam run.launch
```
Notes: The parameter "/use_sim_time" is set to "true" for simulation, "false" to real robot usage.

2. Play existing bag files:
```
rosbag play *.bag --clock --topic /velodyne_points /imu/data
```
Notes: Though /imu/data is optinal, it can improve estimation accuracy greatly if provided. Some sample bags can be downloaded from [here](https://github.com/RobustFieldAutonomyLab/jackal_dataset_20170608). 

3. Save the map to PCD file:
```
rosbag record /laser_cloud_surround -O pcd_map
rosrun pcl_ros bag_to_pcd pcd_map.bag /laser_cloud_surround PATH_TO_PCD_FILE
```

## New data-set

This dataset, [Stevens data-set](https://github.com/TixiaoShan/Stevens-VLP16-Dataset), is captured using a Velodyne VLP-16, which is mounted on an UGV - Clearpath Jackal, on Stevens Institute of Technology campus. The VLP-16 rotation rate is set to 10Hz. This data-set features over 20K scans and many loop-closures. 

<p align='center'>
    <img src="/LeGO-LOAM/launch/dataset-demo.gif" alt="drawing" width="600"/>
</p>
<p align='center'>
    <img src="/LeGO-LOAM/launch/google-earth.png" alt="drawing" width="600"/>  
</p>

## Cite *LeGO-LOAM*

Thank you for citing [our *LeGO-LOAM* paper](./Shan_Englot_IROS_2018_Preprint.pdf) if you use any of this code: 
```
@inproceedings{legoloam2018,
  title={LeGO-LOAM: Lightweight and Ground-Optimized Lidar Odometry and Mapping on Variable Terrain},
  author={Shan, Tixiao and Englot, Brendan},
  booktitle={IEEE/RSJ International Conference on Intelligent Robots and Systems (IROS)},
  pages={4758-4765},
  year={2018},
  organization={IEEE}
}
```

## Loop Closure

The loop-closure method implemented in this package is a naive ICP-based method. It often fails when the odometry drift is too large. For more advanced loop-closure methods, there is a package called [SC-LeGO-LOAM](https://github.com/irapkaist/SC-LeGO-LOAM), which features utilizing point cloud descriptor.

## Speed Optimization

An optimized version of LeGO-LOAM can be found [here](https://github.com/facontidavide/LeGO-LOAM/tree/speed_optimization). All credits go to @facontidavide. Improvements in this directory include but not limited to:

    + To improve the quality of the code, making it more readable, consistent and easier to understand and modify.
    + To remove hard-coded values and use proper configuration files to describe the hardware.
    + To improve performance, in terms of amount of CPU used to calculate the same result.
    + To convert a multi-process application into a single-process / multi-threading one; this makes the algorithm more deterministic and slightly faster.
    + To make it easier and faster to work with rosbags: processing a rosbag should be done at maximum speed allowed by the CPU and in a deterministic way.
    + As a consequence of the previous point, creating unit and regression tests will be easier.


## LeGO-LOAM的不足之处
1、Lego-LOAM有一个显著的缺陷——依赖地面：如果用无人机，那么就难以确定地面了。 当然论文作者提到，对于无人机，则不提取地面点，直接就像LOAM中那样正常提取边缘点和平面点。但是我认为这样算法的核心优势就丢掉了 

2、保存的点云地图是稀疏的特征点地图，这也是此类算法的通病，地图有什么用？——可以用来辅助定位，却不能用来辅助路径规划

## 点云地图该怎么用
这个问题准确的来说，应该叫“点云地图应该怎么用来路径规划”。 由于在导航任务中，定位、决策往往分为了两个不同的模块。定位部分不用考虑“我该怎么走”，决策部分不用考虑“我走到了哪里”。 所以在三维重建（3D Rescontruction）中，“定位”用来辅助“建图”；在SLAM中，“建图”用来辅助“定位”。因此构建的点云地图，对于定位模块够用了，但对于决策模块则不能使用。

目前，业界基本是属于三种思路： 1.构建稠密点云地图，构建八叉树地图，滤除地面信息，取中间高度； 2.语义SLAM，构建物体级别的地图。（需要深度学习参与） 3.构建稀疏点云地图计算位姿，然后根据传感器数据，实时进行点云地图与栅格地图的更新，之后稀疏点云地图用来定位，栅格地图用来规划。
