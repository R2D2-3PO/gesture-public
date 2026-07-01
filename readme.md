# 基于角动量守恒的卫星姿态动力学建模

## 1 引言
卫星姿态运动的根源并非姿态本身，而是系统角动量的产生、传递与重分布。因此，以角动量守恒定律为基础重新构建卫星姿态动力学模型，将卫星视为一个角动量存储、交换和调度系统。

## 2 基础物理规律
卫星在空间中的转动满足角动量定理：
$$
\frac{d\mathbf H}{dt} = \mathbf T_{ext}
$$
其中：
* $\mathbf H$：系统总角动量
* $\mathbf T_{ext}$：系统受到的外力矩

当外力矩为零时：
$$
\frac{d\mathbf H}{dt} = 0
$$
因此：
$$
\mathbf H = \text{Const}
$$
即系统总角动量保持守恒。这是卫星姿态动力学的根本约束。

## 3 系统角动量组成
对于配置飞轮的卫星，总角动量可表示为：
$$
\mathbf H = \mathbf H_b + \mathbf H_w
$$
其中：

### 3.1 卫星本体角动量
$$
\mathbf H_b = \mathbf I \boldsymbol\omega
$$
其中：
* $\mathbf I$：卫星惯量矩阵
* $\boldsymbol\omega$：卫星角速度

### 3.2 飞轮角动量
$$
\mathbf H_w = \sum_{i=1}^{n} J_i \Omega_i \mathbf e_i
$$
其中：
* $J_i$：第 $i$ 个飞轮转动惯量
* $\Omega_i$：第 $i$ 个飞轮转速
* $\mathbf e_i$：飞轮安装方向单位向量

因此，系统总角动量为：
$$
\mathbf H = \mathbf I \boldsymbol\omega + \sum_{i=1}^{n} J_i \Omega_i \mathbf e_i
$$
系统所有姿态变化均来源于该角动量分布的变化。

## 4 状态变量建模
从角动量视角出发，系统状态可分为内部状态与外部状态。

### 4.1 内部状态
**姿态状态：** 采用四元数表示 $\mathbf{q} = [q_0, q_1, q_2, q_3]^T$，用于描述卫星相对于惯性坐标系的姿态。

**角速度状态：** $\boldsymbol\omega$ 表示卫星当前旋转状态。

**飞轮角动量状态：**
$$
\mathbf h_w = \sum_{i=1}^{n} J_i \Omega_i \mathbf e_i
$$
表示系统内部储存的可控角动量。

因此系统内部状态向量可表示为：
$$
\mathbf x = \begin{bmatrix} \mathbf q \\ \boldsymbol\omega \\ \mathbf h_w \end{bmatrix}
$$

### 4.2 外部状态
外部状态主要表现为空间环境产生的扰动力矩。
* 重力梯度力矩：$\mathbf T_g$
* 气动力矩：$\mathbf T_a$
* 太阳光压力力矩：$\mathbf T_s$
* 地磁扰动力矩：$\mathbf T_m$

总扰动力矩为：
$$
\mathbf T_{ext} = \mathbf T_g + \mathbf T_a + \mathbf T_s + \mathbf T_m
$$

## 5 控制输入建模
飞轮通过改变自身转速实现角动量交换。定义控制输入：
$$
\mathbf u = \dot{\mathbf h}_w
$$
对应控制力矩：
$$
\mathbf T_c = -\dot{\mathbf h}_w
$$
负号来源于角动量守恒。即：飞轮角动量增加，本体角动量减少；飞轮角动量减少，本体角动量增加。控制过程本质上是系统内部角动量重分配过程。

## 6 姿态动力学方程
根据欧拉刚体方程：
$$
\mathbf I \dot{\boldsymbol\omega} + \boldsymbol\omega \times (\mathbf I \boldsymbol\omega) = \mathbf T_{ext} + \mathbf T_c
$$
代入飞轮控制力矩：
$$
\mathbf I \dot{\boldsymbol\omega} + \boldsymbol\omega \times (\mathbf I \boldsymbol\omega) = \mathbf T_{ext} - \dot{\mathbf h}_w
$$
该方程完整描述了飞轮控制卫星的动力学过程。

## 7 姿态运动学方程
动力学方程描述角速度变化。姿态变化还需满足四元数运动学方程：
$$
\dot{\mathbf q} = \frac{1}{2} \mathbf\Omega(\boldsymbol\omega) \mathbf q
$$
其中：
$$
\mathbf\Omega(\boldsymbol\omega) = \begin{bmatrix} 0 & -\omega_x & -\omega_y & -\omega_z \\ \omega_x & 0 & \omega_z & -\omega_y \\ \omega_y & -\omega_z & 0 & \omega_x \\ \omega_z & \omega_y & -\omega_x & 0 \end{bmatrix}
$$
因此：动力学决定角速度；运动学决定姿态变化。两者共同构成完整姿态模型。

## 8 角动量观测模型
传统姿态控制通常观测：$[\mathbf q, \boldsymbol\omega]$。而从角动量管理角度，更重要的是：$[\boldsymbol\omega, \mathbf h_w]$。

因为总角动量可直接计算：
$$
\mathbf H = \mathbf I \boldsymbol\omega + \mathbf h_w
$$
因此对于某些稳定控制任务，即使不获取完整姿态信息，也能够建立控制闭环。

## 9 角动量最小化目标
传统姿态控制目标：$\mathbf q \rightarrow \mathbf q_{target}$（实现姿态跟踪）。

从角动量管理角度出发，可以定义更本质的目标：$\mathbf H \rightarrow 0$ 或者是：
$$
\min ||\mathbf H||
$$
当总角动量趋近于零时：卫星自旋最小；转动动能最小；飞轮容量利用率最高；系统稳定性增强。因此角动量最小化可以作为卫星自主稳定的重要目标函数。

## 10 面向自主控制的状态抽象
进一步抽象：
* **状态：** $s = [\boldsymbol\omega, \mathbf h_w]$
* **动作：** $a = \dot{\mathbf h}_w$
* **环境扰动：** $d = \mathbf T_{ext}$
* **奖励函数：** $r = -||\mathbf H||^2$

其中：
$$
\mathbf H = \mathbf I \boldsymbol\omega + \mathbf h_w
$$
此时控制问题转化为：如何在卫星本体与飞轮之间进行角动量调度，使系统总角动量保持最小。
