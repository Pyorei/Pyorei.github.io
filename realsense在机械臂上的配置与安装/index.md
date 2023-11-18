# realsense在机械臂上的配置与安装


早在上半年导师就给了一个realsense d455做机器视觉相关的内容，此次将其配置到ros环境下的机械臂末端组成eye-in-hand机械臂平台，其中主要任务是realsense在ros环境下的配置和相机坐标系与机械臂末端的固定。

<!--more-->

## realsense相机的配置

[官网](https://github.com/IntelRealSense/librealsense/blob/master/doc/distribution_linux.md)很好的介绍了realsense d455相机的配置流程，且现在英特尔官方的安装包已经更新添加了d455相机相关的内容，便于我们调用其坐标以实现相机的配置。官网的安装方法如下：

```linux
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-key F6E65AC044F831AC80A06380C8B3A55A6F3EFCDE || sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-key F6E65AC044F831AC80A06380C8B3A55A6F3EFCDE In case the public key still cannot be retrieved, check and specify proxy settings: export http_proxy="http://<proxy>:<port>"

sudo add-apt-repository "deb https://librealsense.intel.com/Debian/apt-repo $(lsb_release -cs) main" -u

sudo apt-get install librealsense2-dkms
sudo apt-get install librealsense2-utils

sudo apt-get install librealsense2-dev
sudo apt-get install librealsense2-dbg
```

完成安装后可以在终端输入"realsense-viewer"打开自带的预览界面，完成SDK的安装后还需要在工作空间配置realsense-ros，[官网的方法](https://github.com/IntelRealSense/realsense-ros)也很简单，首先需要将realsense-ros的功能包解压或clone到工作空间的src下，随后退回到工作空间的根目录进行catkin_make即可完成配置，官网是新建了一个工作空间，指令如下：

```linux
mkdir -p ~/catkin_ws/src
cd ~/catkin_ws/src/

git clone https://github.com/IntelRealSense/realsense-ros.git
cd realsense-ros/
git checkout `git tag | sort -V | grep -P "^2.\d+\.\d+" | tail -1`
cd ..

catkin_init_workspace
cd ..
catkin_make clean
catkin_make -DCATKIN_ENABLE_TESTING=False -DCMAKE_BUILD_TYPE=Release
catkin_make install

echo "source ~/catkin_ws/devel/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

将realsense相机插上后运行"roslaunch realsense2_camera rs_camera.launch"即可打开相机进行图像和imu等内容的发布。在配置机械臂之前，我参考了[这篇文章](https://blog.csdn.net/qq_40186909/article/details/113104595)对realsense相机有了较为深入的了解，且试着跑了一下VINS-Fusion效果也还可以，下面就是将realsense与机械臂进行连接。

## 机械臂连接realsense

完成先前的moveit机械臂文件搭建后，只需要添加机械臂末端坐标与摄像头坐标的关系即可将摄像头固定到机械臂末端。可以在moveit的launch文件中添加静态坐标关系，或采用[这篇文章](https://blog.csdn.net/qq_38649880/article/details/89298996)的其他方法，以我的机械臂为例launch中的代码如下：

```xml
<node pkg="tf" name="static_transform_publisher" name="camera_link" 
      args="x y z yaw pitch roll frame_id child_frame_id 100"/>
```

其中(x, y, z, yaw, pitch, roll)为child_frame（摄像头坐标）和frame（机械臂末端坐标）的坐标变换关系，随后在终端中逐个开启各个节点，指令如下：

```linux
roslaunch realsense2_camera rs_camera.launch
roslaunch myrobot_moveit demo.launch
rosrun myserial get_data
rosrun myserial serial_port
```

将所有指令输入完成后需要在moveit的rviz中添加相机的pointcloud2节点，从而实现在rivz的3D图像中显示摄像头的点云数据，最后可以获得的效果如视频所示：

## 总结

至此，最简单的eye-in-hand的机械臂平台已经建立完成，接下来需要学习CV相关的知识了，这里推荐[北邮鲁老师的课程](https://www.bilibili.com/video/BV1nz4y197Qv?p=1)，相对全面且易懂一些。

