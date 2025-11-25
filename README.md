## 编译安装

```bash
# 进入工作空间
cd ~/ros2_ws

# 编译包
colcon build --packages-select ros_mqtt_bridge_node

# 加载环境
source install/setup.bash

# 直接运行节点
ros2 run ros_mqtt_bridge_node multi_bridge_manager
```

## 📄 许可证

本项目采用 [MIT License](LICENSE)

## 👥 作者

- 开发者：OrionWongg,sc,Yesord


## 📮 联系方式

如有问题或建议，请提交Issue或联系开发团队。

---

**最后更新：** 2025-11-25
