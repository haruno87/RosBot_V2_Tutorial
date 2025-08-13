## 1. 启动 ros 核心节点

打开终端

```
roscore
```

记得在 ros 的工作空间下，如果出现不知名报错，首先检查 roscore 有没有打开，其次检查运行指令的终端窗口有没有 source 环境变量

```
source devel/setup.bash
```

## 2. 进入工作空间并启动机器人

代码已在[https://github.com/haruno87/RosBot_V2_source]()中备份，接下来的指令都是基于这份代码进行运行的

打开终端

```
cd max_ws
```

```
code .
```

成功打开 vscode 之后，在下方终端

```
source devel/setup.bash
```

```
roslaunch nav point_loc.launch
```

运行指令前确保 3d 雷达已经在运行，在弹出的 rviz 界面中，根据机器人所在的地图位置，使用 rviz 上方工具栏中的 `2D Pose Estimate` 初始化机器人坐标，此时机器人就在地图中完成了定位，想要机器人运动到指定位置，使用 rviz 上方工具栏 `2D Goal Pose` 发布目标点信息，机器人就可以运行到指定地点。

## 3. 编译工作空间

当添加新的功能包或者编写新的 cpp 代码之后，记得更新对应的 cmakelist 和 package.xml ， 然后在工作空间（max_ws）路径下运行 `catkin_make `即可完成工作空间的编译。
