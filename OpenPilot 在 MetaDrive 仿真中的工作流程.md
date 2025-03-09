# OpenPilot 在 MetaDrive 仿真中的工作流程

## 1. OpenPilot 运行
首先，需要启动 OpenPilot 以加载自动驾驶系统，并准备与 MetaDrive 进行交互。
```bash
./tools/sim/launch_openpilot.sh
```

## 2. 启动 OpenPilot 和 MetaDrive 之间的桥接
桥接进程 (`run_bridge.py`) 负责在 OpenPilot 和 MetaDrive 之间传输数据。
```bash
./run_bridge.py
```

### 2.1 运行参数
```bash
./run_bridge.py -h
```
可选参数：
- `--joystick`：允许使用游戏手柄控制。
- `--high_quality`：使用高质量渲染模式。
- `--dual_camera`：启用双摄像头模式。

### 2.2 桥接控制方式
| 按键  | 功能                     |
|------|-----------------------|
|  1   | 巡航恢复 / 加速          |
|  2   | 设置巡航 / 减速          |
|  3   | 取消巡航                |
|  S   | 模拟刹车（解除自动驾驶） |
|  r   | 重置仿真                |
|  i   | 切换点火状态            |
|  q   | 退出                   |
| wasd | 手动控制车辆（调试用）  |

## 3. MetaDrive 处理 OpenPilot 输入
在 MetaDrive 端，核心代码如下：
```python
import gym
import metadrive

# 创建 MetaDrive 环境
env = gym.make("MetaDrive-v0")
obs = env.reset()

while True:
    # 读取 OpenPilot 通过桥接进程传来的数据
    action = get_openpilot_action()  # 解析 OpenPilot 发来的加速/刹车/方向盘指令
    
    # 发送动作到 MetaDrive 进行仿真
    obs, reward, done, info = env.step(action)
    
    if done:
        obs = env.reset()

env.close()
```

### 3.1 数据传输流程
1. **OpenPilot 计算驾驶信号** → 加速/制动/转向数据发送到 `run_bridge.py`
2. **`run_bridge.py` 处理数据** → 转换成 MetaDrive 可识别的格式
3. **MetaDrive 执行车辆控制** → `env.step(action)` 处理输入，并返回仿真状态

### 3.2 MetaDrive 如何解析 OpenPilot 数据？
在 `run_bridge.py` 中，假设 OpenPilot 传输的信号格式如下：
```json
{
    "steering": 0.1,   # 方向盘角度
    "acceleration": 0.3,  # 加速
    "brake": 0.0   # 刹车
}
```
MetaDrive 解析并转换为 Gym 兼容格式：
```python
def get_openpilot_action():
    # 从 OpenPilot 读取控制指令
    data = receive_data_from_openpilot()  
    return [data["steering"], data["acceleration"], data["brake"]]
```
`env.step(action)` 执行相应的车辆操作，并更新仿真状态。

## 4. 总结
- **OpenPilot 通过 `launch_openpilot.sh` 运行，并连接 `run_bridge.py` 进行数据传输。**
- **`run_bridge.py` 充当数据桥梁，将 OpenPilot 计算出的控制信号发送给 MetaDrive，并回传仿真状态。**
- **MetaDrive 解析 OpenPilot 的数据，并在 `env.step(action)` 中执行对应的车辆操作，实现自动驾驶仿真。**
