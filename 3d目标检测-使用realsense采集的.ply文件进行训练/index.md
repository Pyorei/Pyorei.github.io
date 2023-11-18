# 3D目标检测-使用realsense采集的.ply文件进行训练


## OpenPCDet环境配置

首先需要安装好open3d，在OpenPCDet根目录下运行以下命令完成基本的环境配置

```sh
pip install open3d
pip install -r requirements.txt
python setup.py develop
```

在运行setup脚本时可能会出现权限不足的问题，可以改用 ``pip install -e .`` 对pcdet库进行安装。安装完成后可以下载部分kitti数据集并运行demo.py进行测试，或是在命令行尝试``import pcdet``

```shell
python tools/demo.py --cfg_file tools/cfgs/kitti_models/pointpillar.yaml --ckpt tools/pointpillar_7728.pth --data_path data/kitti/training/velodyne/000008.bin 
```

## 建立自定义数据集

数据集标定使用了kitti的格式，因此可以模仿kitti数据集结构分配文件，文件树结构如下：

```shell
OpenPCDet
├── data
│   ├── kitti
│   │   │── ImageSets
│   │   │   ├──train.txt & test.txt & val.txt
│   │   │── training
│   │   │   ├──calib & velodyne & label_2 & image_2
│   │   │── testing
│   │   │   ├──calib & velodyne & image_2 & label_2
```

ImageSets文件夹保存了用于训练、测试以及验证的数据集分类结果，将标定数据的文件名写入train.txt文件，并且需要将所对应的点云和label数据放到training文件夹的velodyne和label_2文件夹下。

### 点云数据转换

kitti数据集的点云.bin文件中包含了点云的x、y、z空间坐标以及激光反射强度intensity数据，而realsense采集到的点云.ply文件为xyzrgb格式，为了快速使用OpenPCDet进行训练，因此可以将.ply文件通过tools/ply2bin.py进行转换。（此处可优化custom_dataset.py中的数据集加载进行修改，从而省去点云数据转换一步和生成pkl文件步骤）

```python
import open3d as o3d
import numpy as np
import os

# load ply
def read_ply(filepath):
    lidar = []
    pcd = o3d.io.read_point_cloud(filepath, format='ply')
    points = np.array(pcd.points)
    for linestr in points:
        if len(linestr) == 3:  # only x,y,z
            linestr_convert = list(map(float, linestr))
            linestr_convert.append(0)
            lidar.append(linestr_convert)
        if len(linestr) == 4:  # x,y,z,i
            linestr_convert = list(map(float, linestr))
            lidar.append(linestr_convert)
    return np.array(lidar)


def pcd2bin(pcd_fullname, bin_fullname):
    pl = read_ply(pcd_fullname)
    
    np_x = (np.array(pl[:,0], dtype=np.float32)).astype(np.float32)
    np_y = (np.array(pl[:,1], dtype=np.float32)).astype(np.float32)
    np_z = (np.array(pl[:,2], dtype=np.float32)).astype(np.float32)
    np_i = (np.array(pl[:,3], dtype=np.float32)).astype(np.float32) / 256
    points_32 = np.transpose(np.vstack((np_x, np_y, np_z, np_i)))

    points_32.tofile(bin_fullname)
 

if __name__ == '__main__':
    root_folder = "data/kitti/training/velodyne"
    plylist = os.listdir(root_folder)
    for plyfile in plylist:
        plypath = os.path.join(root_folder, plyfile)
        binpath = plypath.split('.')[0]+".bin"
        # ply -> bin
        pcd2bin(plypath,binpath)
```

完成了数据集文件的建立，需要对数据集进行初始化，即编写相对应的custom_dataset.py和.yaml文件。

## custom_dataset.py修改

仿照kitti_dataset.py进行修改，参考博文：[https://blog.csdn.net/JulyLi2019/article/details/126351276](https://blog.csdn.net/JulyLi2019/article/details/126351276)

## 通过.yaml文件配置参数

创建tools/cfgs/dataset_configs/cpte_dataset.yaml定义数据集相关参数

创建tools/cfgs/custom_models/pointpillar.yaml定义模型和训练相关参数

```shell
# 生成训练所需的pkl文件
python -m pcdet.datasets.custom.custom_dataset create_custom_infos /data/pl/OpenPCDet/tools/cfgs/dataset_configs/cpte_dataset.yaml
# 训练
python tools/train.py --cfg_file tools/cfgs/custom_models/pointpillar.yaml --batch_size=2 --epochs=300
```

## 待解决

训练结束后的验证方法

