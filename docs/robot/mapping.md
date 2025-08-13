# 3d slam

## PointLio建图

### 注意！！！！ 建图前需要先去把src/point_lio_unilidar/PCD下的scans.pcd 重命名，否则会被覆盖

`https://github.com/unitreerobotics/point_lio_unilidar` Point_lio 三维 SLAM

`https://github.com/unitreerobotics/unilidar_sdk2`宇树 L2 雷达

主要参考这两个官方的文档进行操作完成操作。

`https://github.com/66Lau/NEXTE_Sentry_Nav?tab=readme-ov-file`这个项目也作为主要的参考。


可以使用 `roslaunch nav lio_mapping.launch `打开建图的 launch 文件，再打开电机控制节点 `rosrun service motor_control.py`， 然后使用键盘控制节点 `rosrun service key_scans.py` 进行键盘遥控建图，在 rviz 中确定建图质量之后退出节点就会自动保存在默认路径的 PCD/ 下，默认名称是 scans.pcd。

需要确保运行建图节点之后，需要将保存路径下的 PCD 文件重命名防止被覆盖，如果发现建的图关闭节点之后没有保存下来，在 `src/point_lio_unilidar/config/unilidar_l2.yaml `的尾部看看 `pcd_save_en` 有没有设置为 true。

```
pcd_save:
    pcd_save_en: false       # save map to pcd file
    interval: -1            # how many LiDAR frames saved in each pcd file; 
                                 # -1 : all frames will be saved in ONE pcd file, may lead to memory crash when having too much frames.
```

在保存好 3D 的 PCD 地图之后，使用静态方法转换为二维栅格地图

`roslaunch pcd2pgm run.launch`

`roslaunch nav map_saver.launch`

可以分别去这两个文件下修改文件的路径和保存的配置。

```
<node pkg="pcl_ros" type="pcd_to_pointcloud" name="map_publisher" output="screen"
        args="/home/orangepi/max_ws/src/point_lio_unilidar/PCD/scans_0807.pcd 5 _frame_id:=map cloud_pcd:=map" />

```

快速运行指令中的 point_loc.launch 的这条指令就是将保存好的 PCD 3 维地图转换为 pointcloud 供 src/service/scripts/localisation.py 进行匹配定位的。

宇树雷达可能出现的问题，要是雷达上电初始化的时候不够平稳抖动激烈的话整个系统都会跑飞。


## gmapping建图

使用 gmapping 建图必须有 odom 到基坐标系的 tf 变换，这样才能建立起 map 到 odom 之间的变换。在src/service/scripts/motor_control.py 中已经实现了 odom 到 laser 的坐标系变换。

首先先确保雷达节点已经打开，roslaunch rplidar_ros rplidar_a1.launch,然后启动电机节点 rosrun service motor_control.py，启动建图节点 roslaunch rpliar_ros gmapping.launch ，通过键盘进行遥控建图，确保建图质量之后使用 roslaunch nav map_saver.launch 保存地图即可。
