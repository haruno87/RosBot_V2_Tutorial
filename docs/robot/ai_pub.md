## 1.语音控制机器人运动

通过大模型的 function calling 方法，当检测到用户的特定意图时，在指定的回调函数中通过 ros_pub 发布给指定话题信息，其他节点（如 move_point.py）订阅给话题，并对内容解析执行特定的动作。

例如我们提前标定出饮水机的位置，然后在 function calling 中的 go_to_location 函数中发布需要去达的目标点，move_point.py 这个节点就会发布坐标点信息使机器人前往，效果类似与在 rviz 中打上目标点。

如何获得特殊点的坐标：

1）先使用 rviz 让机器人前往想要标定的位置

2）使用rosrun tf tf_echo map aft_mapped 获取多维坐标，将不必要的位姿设为 0 就好。


其他 AI 节点语音控制思路也类似，例如 1 代机中语音指令“请你跟着我”，ai 识别到用户的意图时调用函数，在函数中把视觉跟踪的节点打开就可以，当接收到“停下来，不要跟着了”再另一个函数中把该节点 kill 掉。


初代机的 ai 节点可以参考 `src/service/scripts/llm_qwen.py `中的实现。


## 2. 视觉跟随

初代机的实现是通过 realsense 对齐深度图和 rgb 图像，然后在 rgb 图像中使用 mediapipe 识别到人的髋关节和膝关节，在获取对应的深度信息，然后向`/cmd_vel`话题发布计算出来的运动指令即可完成视觉跟随。


## 3.摔倒检测

初代机使用的是山东 123 物联网家的 mqtt 毫米波雷达，我们需要将香橙派板卡作为 mqtt 的订阅端和服务端，在本地部署 mqtt 服务器，初代机上部署的是 mosqiutto 这个平台，需要确保毫米波雷达和板卡在同一局域网下，然后订阅这个话题，检测到摔倒信息之后在通过 ros 话题发布出来供其他节点接收。

mqtt 部署可以参考：
速通：

```
https://www.bilibili.com/video/BV1MG4y1k7mx/?spm_id_from=333.1387.favlist.content.click&vd_source=16653787726583a107f817924f9f09fe
```

```
https://www.bilibili.com/video/BV1JG411b7oa/?spm_id_from=333.1387.upload.video_card.click&vd_source=16653787726583a107f817924f9f09fe
```

```
https://blog.csdn.net/lclfans1983/article/details/105694696
```

系统性：

```
https://www.runoob.com/w3cnote/mqtt-intro.html
```

```
https://blog.csdn.net/weixin_44788542/article/details/129690265?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522ac83dfa9ac90158e1c58d2bd4147b970%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=ac83dfa9ac90158e1c58d2bd4147b970&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~top_positive~default-1-129690265-null-null.142^v102^pc_search_result_base7&utm_term=mqtt&spm=1018.2226.3001.4187
```

## 4. 智能寻物

```
{ # 添加的函数描述
                "name": "find_item",
                "description": "去桌子附近然后在桌面上寻找指定的物品",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "item_name": {
                            "type": "string",
                            "description": "要寻找的物品的名称"
                        }
                    },
                    "required": ["item_name"]
                }
            }
```

```

def find_item(self, item_name=""):
        """
        导航到指定位置，到达后发布目标物品名称。
        Args:
            item_name (str): 要寻找的物品的名称。
        """
        rospy.loginfo(f"开始导航到桌子并寻找物品: {item_name}")

        # 播放语音提示
        self.synthesize_and_play(f"好的，我即将运动到桌子附近，并寻找{item_name}。\n")
        #[ INFO] [1751780065.113742926]: Setting goal: Frame:map, Position(5.398, -1.921, 0.000), Orientation(0.000, 0.000, 0.012, 1.000) = Angle: 0.023

        # 定义目标位姿
        goal_pose = PoseStamped()
        goal_pose.header.frame_id = "map"  # 替换为你的地图坐标系
        goal_pose.pose.position.x = 5.412
        goal_pose.pose.position.y = -1.970
        goal_pose.pose.position.z = 0.033
        # 将欧拉角转换为四元数 (x, y, z, w)
        quaternion = [0.000, 0.000, -0.001, 1.000]
        goal_pose.pose.orientation.x = quaternion[0]
        goal_pose.pose.orientation.y = quaternion[1]
        goal_pose.pose.orientation.z = quaternion[2]
        goal_pose.pose.orientation.w = quaternion[3]

        # 创建一个 SimpleActionClient 连接到 move_base action server
        client = actionlib.SimpleActionClient('move_base', MoveBaseAction) # 请确认你的导航 Action Server 名称
        rospy.loginfo("等待 move_base action server 启动...")
        client.wait_for_server()
        rospy.loginfo("move_base action server 已启动。")

        # 创建一个导航目标
        goal = MoveBaseGoal()
        goal.target_pose = goal_pose
        goal.target_pose.header.stamp = rospy.Time.now()

        rospy.loginfo("发送导航目标...")
        client.send_goal(goal)

        # 等待导航完成
        client.wait_for_result()

        # 检查导航结果
        if client.get_state() == GoalStatus.SUCCEEDED:
            rospy.loginfo("成功到达目标位置！")
            # 到达目的地后发布目标物品消息
            target_item_msg = json.dumps({"target_item": item_name})
            self.keyword_pub.publish(target_item_msg)
            self.synthesize_and_play(f"我已到达，开始寻找。\n")
            rospy.loginfo(f"已发布目标物品消息: {target_item_msg}")
        else:
            rospy.logwarn("导航失败！")

        # 找完东西后回到初始状态
        self._transition_to_state('listening_for_wake_up')
```

实现的思路是，通过 function calling 识别用户的意图，检测到用户的意图之后，将需要寻找的物品提取出来送给另一个节点发给 vlm 节点做识别并得到返回信息。还需要提取出用户指定的地点如茶几，桌子等。通过提前标定坐标等方式发布指令让机器人运动到目标点，然后提取出 reach goal 的信息之后在发送给 vlm 节点。
