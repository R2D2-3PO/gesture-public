# 项目运行指南

本文档说明如何获取完整项目、配置 Basilisk、运行姿态控制场景、导出实验数据、使用仪表盘和 Vizard、进行 Jetson HIL 联调，以及执行测试和批量实验。

## 1. 项目结构

```text
gesture/
├── readme.md                       # 项目背景与动力学说明
├── RUNNING.md                      # 本运行指南
└── basilisk/                       # 固定版本的 Basilisk 子模块
    ├── examples/
    │   ├── scenarioCausalRLAttitude.py
    │   ├── scenerioWorldModelBasicAttitude.py
    │   ├── checkpoints/
    │   ├── dashboard/
    │   └── hil/
    ├── experiments/
    │   └── run_inertia_sweep_20.sh
    └── src/tests/
```

主要入口是：

```text
basilisk/examples/scenarioCausalRLAttitude.py
```

后续命令除非特别说明，都从 `gesture/basilisk` 目录执行。

## 2. 首次获取项目

本项目的完整版本位于 `gesture` 仓库的 `github-update-20260928` 分支。`basilisk` 是子模块，其中两个大型 Unity 记录文件使用 Git LFS 存储。

先安装 Git LFS，然后克隆：

```bash
git lfs install
git clone --branch github-update-20260928 --recurse-submodules \
  https://github.com/R2D2-3PO/gesture.git
cd gesture
git submodule update --init --recursive
git -C basilisk lfs pull
```

检查子模块和 LFS 文件：

```bash
git submodule status
git -C basilisk rev-parse --short HEAD
git -C basilisk lfs ls-files
```

预期的 Basilisk 提交是 `5189b4f8e`，来源分支是 `gesture-local-exact-20260928`。子模块初次检出后显示为 detached HEAD 是正常现象。不要为了运行本项目而将它合并或变基到上游最新 `develop`。

## 3. Python 环境和 Basilisk 构建

这是修改过的 Basilisk 源码，推荐使用项目源码构建，而不是用 `pip install bsk` 替代。后者会安装官方发布版，不能保证包含本项目所依赖的源码状态。

### 3.1 macOS/Linux

```bash
cd basilisk
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements_dev.txt
python conanfile.py
```

### 3.2 Windows PowerShell

```powershell
cd basilisk
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements_dev.txt
python conanfile.py
```

`conanfile.py` 会配置并编译 Basilisk，同时检查运行依赖。首次构建需要编译器、CMake、Conan 和网络访问，耗时通常明显长于普通 Python 包安装。

神经残差模型、检查点和 Jetson 控制服务需要 PyTorch。请在同一个虚拟环境中安装与当前操作系统或 CUDA 环境匹配的 PyTorch：

```bash
python -m pip install torch
```

如果没有安装 PyTorch，主场景请求 `--residual-mode neural` 时会自动退回 `analytic`；Jetson HIL 服务则会拒绝启动。

验证环境：

```bash
python -c "import Basilisk; print('Basilisk import OK')"
python -c "import numpy, matplotlib; print('scientific stack OK')"
python -c "import torch; print(torch.__version__, torch.cuda.is_available())"
```

最后一条只用于需要 PyTorch 的运行方式。如果不使用神经模式或 HIL，可以跳过。

每次打开新终端后，都需要重新激活环境：

```bash
cd gesture/basilisk
source .venv/bin/activate
```

## 4. 最小可运行检查

先执行一个短时间、无神经模型的冒烟测试：

```bash
python examples/scenarioCausalRLAttitude.py \
  --simulation-minutes 0.1 \
  --residual-mode none \
  --no-vizard \
  --no-dashboard
```

查看当前代码支持的全部参数：

```bash
python examples/scenarioCausalRLAttitude.py --help
```

## 5. 主场景的常用运行方法

### 5.1 默认运行

```bash
python examples/scenarioCausalRLAttitude.py
```

命令行默认仿真 90 分钟，使用神经残差模式、真值姿态导航、太阳翼部署和天底指向，不显示 Matplotlib 窗口。

显示结果图：

```bash
python examples/scenarioCausalRLAttitude.py \
  --simulation-minutes 5 \
  --show-plots
```

### 5.2 三种残差估计模式

```bash
# 不使用残差估计
python examples/scenarioCausalRLAttitude.py --residual-mode none

# 解析残差估计
python examples/scenarioCausalRLAttitude.py --residual-mode analytic

# 神经残差估计，需要 PyTorch
python examples/scenarioCausalRLAttitude.py --residual-mode neural
```

### 5.3 姿态任务

任务编号：

- `0`：天底指向；
- `1`：太阳指向；
- `2`：惯性姿态保持。

```bash
# 天底指向
python examples/scenarioCausalRLAttitude.py --attitude-task-code 0

# 太阳指向
python examples/scenarioCausalRLAttitude.py --attitude-task-code 1

# 惯性姿态保持，目标使用 MRP
python examples/scenarioCausalRLAttitude.py \
  --attitude-task-code 2 \
  --target-sigma-rn 0.1 -0.2 0.05
```

如需使用 `inertialUKF` 估计值而不是真值姿态导航：

```bash
python examples/scenarioCausalRLAttitude.py \
  --no-truth-attitude-navigation
```

### 5.4 B-dot 消旋和磁力矩器卸载

```bash
# 先进行 B-dot 消旋，再切换至姿态任务
python examples/scenarioCausalRLAttitude.py \
  --b-dot-detumble \
  --initial-body-rate 0.08 -0.05 0.04

# 开启飞轮动量卸载
python examples/scenarioCausalRLAttitude.py \
  --momentum-dumping
```

可调 B-dot 参数包括：

```text
--b-dot-gain
--b-dot-filter-time-constant
--b-dot-rate-threshold
--b-dot-dwell-time
--max-mtb-dipole
```

### 5.5 太阳翼部署

先部署太阳翼，再进入天底指向：

```bash
python examples/scenarioCausalRLAttitude.py \
  --solar-array-deployment \
  --solar-array-deployment-sequence deploy_then_point \
  --solar-array-deployment-duration 30
```

先进入天底指向，再在控制过程中展开太阳翼：

```bash
python examples/scenarioCausalRLAttitude.py \
  --solar-array-deployment-sequence point_then_deploy \
  --solar-array-pointing-lead-time 10
```

完全禁用部署：

```bash
python examples/scenarioCausalRLAttitude.py --no-solar-array-deployment
```

### 5.6 推进剂消耗和单次推力器点火

```bash
python examples/scenarioCausalRLAttitude.py \
  --simulation-minutes 2 \
  --propulsion \
  --propellant-mass 40 \
  --thruster-burn-start 1 \
  --thruster-burn-duration 5
```

`--thruster-burn-start` 和 `--thruster-burn-duration` 的单位是秒。点火必须在仿真结束前开始，持续时间不能小于推力器最小开机时间，也不能耗尽油箱。

### 5.7 自定义惯量矩阵

惯量按照 3×3 矩阵逐行输入，单位为 `kg*m^2`：

```bash
python examples/scenarioCausalRLAttitude.py \
  --spacecraft-inertia \
  520 18 -8 \
  18 710 12 \
  -8 12 640
```

程序会检查矩阵有限、对称且正定。

## 6. 神经检查点和实验数据

### 6.1 从仓库检查点启动

```bash
python examples/scenarioCausalRLAttitude.py \
  --residual-mode neural \
  --checkpoint examples/checkpoints/causal_rl_attitude.pt
```

默认会恢复检查点中的 Adam 优化器状态。只加载网络权重：

```bash
python examples/scenarioCausalRLAttitude.py \
  --checkpoint examples/checkpoints/causal_rl_attitude.pt \
  --no-load-optimizer-state
```

### 6.2 随机初始化并保存检查点

```bash
mkdir -p experiment_outputs/run01
python examples/scenarioCausalRLAttitude.py \
  --random-init \
  --save-checkpoint experiment_outputs/run01/final_model.pt
```

### 6.3 导出训练与诊断数据

```bash
python examples/scenarioCausalRLAttitude.py \
  --simulation-minutes 10 \
  --training-data-directory experiment_outputs/run01
```

输出目录通常包含：

```text
training_data.npz       # NumPy 压缩数组
dashboard_payload.json  # 仪表盘数据
manifest.json           # 文件、数组形状和元数据清单
```

如果同时传入 `--vizard`，还会生成：

```text
scenarioCausalRLAttitude_UnityViz.bin
```

## 7. 仪表盘、Vizard 和消息图

### 7.1 浏览器仪表盘

```bash
python examples/scenarioCausalRLAttitude.py \
  --simulation-minutes 5 \
  --dashboard \
  --dashboard-port 8765 \
  --dashboard-hold-open
```

仿真结束后，程序会启动本地 HTTP 服务并打开浏览器。默认地址是：

```text
http://127.0.0.1:8765/
```

按 `Ctrl+C` 停止服务。如果端口冲突，可以换一个端口。服务器只绑定回环地址，不会直接暴露到局域网。

### 7.2 Vizard

实时连接或启动 Vizard 数据接口：

```bash
python examples/scenarioCausalRLAttitude.py --vizard
```

保存可之后打开的 Vizard 记录：

```bash
python examples/scenarioCausalRLAttitude.py \
  --vizard \
  --training-data-directory experiment_outputs/vizard_run
```

Vizard 是可选的独立程序。没有安装 Vizard 时，先使用 `--no-vizard` 完成普通仿真验证。

### 7.3 Basilisk 消息连接图

```bash
python examples/scenarioCausalRLAttitude.py \
  --simulation-minutes 0.1 \
  --message-graph
```

该功能会在场景目录附近生成消息连接图文件，系统可能还需要 Graphviz。

## 8. 故障注入和构型变化事件

`--scenario-events` 接收一个 JSON 文件。目前支持：

- `rw_dropout`：指定反作用飞轮失效；
- `attach_mass`：在指定位置附加质量和惯量。

示例 `events.json`：

```json
[
  {
    "type": "rw_dropout",
    "name": "rw2_failure",
    "timeMinutes": 2.0,
    "wheelIndex": 1
  },
  {
    "type": "attach_mass",
    "name": "payload_capture",
    "timeMinutes": 4.0,
    "massKg": 12.0,
    "location_B": [0.4, 0.0, 0.1],
    "inertiaPntAdded_B": [
      [0.20, 0.0, 0.0],
      [0.0, 0.30, 0.0],
      [0.0, 0.0, 0.25]
    ],
    "updateFlightSoftwareModel": false
  }
]
```

运行：

```bash
python examples/scenarioCausalRLAttitude.py \
  --simulation-minutes 6 \
  --scenario-events events.json \
  --training-data-directory experiment_outputs/event_run
```

`wheelIndex` 从 `0` 到 `3`。事件时间单位为分钟，必须落在仿真区间内。

## 9. 从 Python 调用

如果需要在自己的实验脚本中直接取得数组和图对象：

```python
import sys
from pathlib import Path

examples_directory = Path("examples").resolve()
sys.path.insert(0, str(examples_directory))

import scenarioCausalRLAttitude as scenario

figures, data = scenario.run(
    show_plots=False,
    residualMode="analytic",
    simulationMinutes=1.0,
    enableVizard=False,
    returnData=True,
)

print(data["time_min"].shape)
print(data["sigma_BN"][-1])
print(sorted(figures))
```

比较三种残差模式：

```python
import sys
from pathlib import Path

sys.path.insert(0, str(Path("examples").resolve()))
import scenarioCausalRLAttitude as scenario

results = scenario.compare_residual_modes(
    residualModes=("none", "analytic", "neural"),
    show_plots=False,
    enableVizard=False,
    simulationMinutes=5.0,
)

for mode, result in results.items():
    print(mode, result["summary"])
```

## 10. Jetson HIL 联调

HIL 目录中的 Jetson 端不依赖 Basilisk，但需要 NumPy 和适合 JetPack/CUDA 环境的 PyTorch。

### 10.1 Jetson 端

将整个目录复制到 Jetson：

```text
basilisk/examples/hil/
```

在 Jetson 上启动：

```bash
cd attitude-hil
python jetson_controller_server.py --host 0.0.0.0 --port 5555
```

看到以下信息表示服务已监听：

```text
Attitude controller listening on 0.0.0.0:5555
```

### 10.2 Basilisk 电脑端

将示例 IP 替换为 Jetson 的实际局域网 IP：

```bash
python examples/scenarioCausalRLAttitude.py \
  --simulation-minutes 1 \
  --no-truth-attitude-navigation \
  --remote-controller-endpoint 192.168.1.12:5555 \
  --training-data-directory experiment_outputs/hil_run
```

运行前确认：

- 两台机器网络互通；
- TCP 端口 `5555` 未被防火墙拦截；
- Jetson 端先启动；
- Jetson 的 PyTorch 能正确加载；
- 两端系统时间不要求完全同步，协议使用仿真时间和序列号检查数据。

需要开机自启时，可以参考：

```text
examples/hil/attitude-controller.service
```

其中的用户、工作目录和 Python 路径必须改成 Jetson 上的真实路径。

## 11. 测试

只测试 HIL 协议和独立控制器：

```bash
python -m pytest src/tests/test_attitude_hil.py -q
```

测试主场景功能：

```bash
python -m pytest src/tests/test_scenarioCausalRLAttitude.py -q
```

运行这两个项目相关测试：

```bash
python -m pytest \
  src/tests/test_attitude_hil.py \
  src/tests/test_scenarioCausalRLAttitude.py \
  -q
```

运行 Basilisk 全部测试：

```bash
python run_all_test.py
```

完整测试耗时较长。日常修改优先运行项目相关的两个测试文件。

## 12. 批量惯量实验

批量入口：

```text
experiments/run_inertia_sweep_20.sh
```

该脚本会运行 20 组实验，每组包含随机初始化或预训练初始化，并检查 CUDA、检查点哈希、队列锁和输出目录。

脚本当前含有机器相关的绝对路径，不能在另一台机器上直接运行。至少先修改：

```bash
REPO_ROOT="/你的路径/gesture/basilisk"
PYTHON_BIN="/你的虚拟环境/bin/python"
SOURCE_CHECKPOINT="/你的检查点路径/source.pt"
SOURCE_CHECKPOINT_SHA256="检查点的真实SHA256"
OUTPUT_ROOT="/你的输出路径/inertia_sweep"
```

计算检查点哈希：

```bash
shasum -a 256 /你的检查点路径/source.pt     # macOS
sha256sum /你的检查点路径/source.pt         # Linux
```

Linux/NVIDIA 环境运行：

```bash
chmod +x experiments/run_inertia_sweep_20.sh
CUDA_VISIBLE_DEVICES=0 ./experiments/run_inertia_sweep_20.sh
```

该脚本依赖 Linux `flock` 和可用的 CUDA GPU。它会拒绝覆盖已有队列目录；需要重新实验时，请使用新的 `OUTPUT_ROOT`，不要直接删除仍在运行的队列。

## 13. 旧版参考场景

仓库还保留了一个早期硬物理世界模型场景：

```text
examples/scenerioWorldModelBasicAttitude.py
```

无图形、无 Vizard 运行：

```bash
python -c "import sys; sys.path.insert(0, 'examples'); import scenerioWorldModelBasicAttitude as s; s.run(False, False, False)"
```

这个文件名中的 `scenerio` 是仓库现有拼写。新实验应优先使用 `scenarioCausalRLAttitude.py`。

## 14. 常见问题

### `ModuleNotFoundError: No module named 'Basilisk'`

当前终端没有激活构建 Basilisk 时使用的虚拟环境，或者源码尚未成功编译。重新激活 `.venv`，然后执行环境验证命令。

### 神经模式自动变成 `analytic`

当前环境无法导入 PyTorch。安装与当前 Python、操作系统或 CUDA 环境匹配的 PyTorch，再重新运行。

### `--checkpoint` 报错

检查点只能与 `--residual-mode neural` 一起使用；同时确认文件路径存在，且模型中的反作用飞轮数量与当前场景一致。

### LFS 文件只有几行文本

Git LFS 对象没有拉取完整。回到项目根目录执行：

```bash
git -C basilisk lfs pull
```

### 仪表盘打不开

确认仿真已经结束、终端仍在运行，并检查端口是否被占用。可以改用：

```bash
python examples/scenarioCausalRLAttitude.py \
  --dashboard \
  --dashboard-port 8877 \
  --dashboard-hold-open
```

### HIL 无法连接

先在 Jetson 上确认监听信息，再从 Basilisk 电脑检查 Jetson IP、端口、防火墙和网络路由。连接中断时，桥接器会短暂保持最后一条指令，然后将控制力矩逐步降到零。

### 想确认命令是否使用了当前源码

```bash
git status --short --branch
git log -3 --oneline
python -c "import Basilisk; print(Basilisk.__path__)"
```

预期 Basilisk 提交为 `5189b4f8e`；如果不是 detached HEAD，则对应分支应为 `gesture-local-exact-20260928`。
