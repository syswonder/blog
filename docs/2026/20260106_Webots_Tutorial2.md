# webots 开发日志（二）webots 上手指南

时间: 2026/01/06

作者: 陈衍廷

联系方式：ytchen25@pku.edu.cn

## 摘要
  本文主要对 webots 一些经常使用的功能做介绍，更加详细且系统的教程可见网上资源，比如[该教程](https://www.bilibili.com/opus/678930782193975296)。
  本文将着重对 webots 里常用的一些核心概念，如世界文件、场景树（Scene Tree）、节点、铰链等进行介绍。

## 从节点到场景树再到世界文件
#### 一、核心概念概述

| 概念 | 说明 | 类比 |
|------|------|------|
| **节点 (Node)** | 场景中的基本构建单元 | HTML 中的元素/标签 |
| **场景树 (Scene Tree)** | 所有节点的层级组织结构 | HTML 的 DOM 树 |
| **世界文件 (.wbt)** | 存储整个场景的文本文件，在webots项目目录下的 `worlds/` 文件夹中生成，后缀为 .wbt | HTML 文件本身 |

---

#### 二、节点 (Node) 详解

###### 2.1 什么是节点

节点是 Webots 仿真世界的**基本构建块**。每个节点代表一个具体的对象或属性，如机器人、传感器、光源、地面等。

##### 2.2 节点的结构

每个节点通常包含：
- **类型名称**：如 `Robot`、`Solid`、`PointLight`
- **字段 (Fields)**：节点的属性/参数
- **子节点**：嵌套在内部的其他节点

##### 2.3 一些常见的节点

```
节点类型
├── 基础节点 (Base Nodes)
│   ├── Solid          # 刚体，有物理属性
│   ├── Transform      # 仅做坐标变换，无物理属性
│   ├── Group          # 节点容器
│   └── Shape          # 定义几何形状和外观
│
├── 设备节点 (Device Nodes)
│   ├── Camera         # 摄像头
│   ├── Lidar          # 激光雷达
│   ├── GPS            # 定位
│   ├── DistanceSensor # 距离传感器
│   ├── Motor          # 电机
│   └── ...
│
├── 环境节点 (Environment Nodes)
│   ├── WorldInfo      # 世界全局信息
│   ├── Viewpoint      # 视角
│   ├── Background     # 背景/天空
│   ├── PointLight     # 点光源
│   ├── DirectionalLight # 平行光
│   └── Floor          # 地面
│
└── 特殊节点
    ├── Robot          # 机器人（可运行控制器）
    ├── Supervisor     # 超级控制器（可修改仿真）
    └── PROTO          # 自定义节点模板
```

##### 2.4 节点示例

```
# 一个简单的立方体节点
Solid {
  translation 0 0.5 0           # 位置：x=0, y=0.5, z=0
  rotation 0 1 0 0.785          # 绕Y轴旋转45度
  children [
    Shape {                      # 子节点：形状
      appearance PBRAppearance { # 外观（材质）
        baseColor 1 0 0          # 红色
        metalness 0
        roughness 0.5
      }
      geometry Box {             # 几何体：立方体
        size 1 1 1               # 尺寸 1x1x1 米
      }
    }
  ]
  boundingObject Box {           # 碰撞边界
    size 1 1 1
  }
  physics Physics {              # 物理属性
    density 1000                 # 密度 kg/m³
  }
}
```

---

#### 三、场景树 (Scene Tree) 详解

##### 3.1 什么是场景树

场景树是 Webots 中所有节点的**层级组织结构**，以树状形式展示整个仿真世界的构成。

##### 3.2 场景树示例

```
WorldInfo                    # 根级：世界信息
Viewpoint                    # 根级：视角
Background                   # 根级：背景
DirectionalLight             # 根级：光源
│
Floor                        # 根级：地面
│
Robot "my_robot"             # 根级：机器人
├── DEF BODY Shape           # 子级：机器人本体
│   ├── appearance           # 孙级：外观
│   └── geometry             # 孙级：几何
├── HingeJoint               # 子级：关节
│   ├── jointParameters      # 孙级：关节参数
│   ├── device Motor         # 孙级：电机设备
│   └── endPoint Solid       # 孙级：轮子
│       └── Shape            # 曾孙级：轮子形状
└── Camera                   # 子级：摄像头传感器
```

##### 3.3 场景树的特点

| 特点 | 说明 |
|------|------|
| **父子关系** | 子节点的坐标相对于父节点 |
| **继承变换** | 移动父节点时，子节点跟随移动 |
| **DEF/USE** | 可定义节点别名并重用 |
| **可视化编辑** | 在 Webots GUI 左侧面板直接操作 |

##### 3.4 DEF/USE 机制

```
# DEF 定义一个可重用的节点
DEF RED_MATERIAL PBRAppearance {
  baseColor 1 0 0
  metalness 0
}

# USE 引用之前定义的节点
Shape {
  appearance USE RED_MATERIAL    # 重用上面定义的材质
  geometry Sphere { radius 0.5 }
}

Shape {
  appearance USE RED_MATERIAL    # 再次重用
  geometry Box { size 1 1 1 }
}
```

---

#### 四、世界文件 (.wbt) 详解

##### 4.1 什么是 .wbt 文件

`.wbt` 文件是 Webots 的**世界描述文件**，是一个**纯文本文件**，使用 VRML97 派生的语法来描述整个仿真场景。

##### 4.2 .wbt 文件示例

```vrml
#VRML_SIM R2025a utf8
# 文件头：版本标识

EXTERNPROTO "https://raw.githubusercontent.com/cyberbotics/webots/R2023b/projects/objects/floors/protos/Floor.proto"
# 外部 PROTO 引用

WorldInfo {
  title "My Simulation"
  basicTimeStep 16              # 仿真步长 16ms
  coordinateSystem "NUE"        # 坐标系
}

Viewpoint {
  orientation -0.2 0.9 0.3 1.2
  position 3 2 5
}

Background {
  skyColor [0.4 0.7 1.0]        # 天空颜色
}

DirectionalLight {
  direction -0.5 -1 -0.5
  intensity 1
}

Floor {
  size 10 10
}

# 机器人定义
Robot {
  name "e-puck"
  translation 0 0 0
  children [
    # ... 机器人组件
  ]
  controller "my_controller"    # 控制器程序名
}
```

##### 4.3 .wbt 文件的组成部分

```
.wbt 文件
│
├── 1. 文件头
│   └── #VRML_SIM R2023b utf8
│
├── 2. EXTERNPROTO 声明
│   └── 引用外部 PROTO 定义文件
│
├── 3. 全局节点（必需）
│   ├── WorldInfo     # 仿真参数
│   ├── Viewpoint     # 初始视角
│   └── Background    # 背景设置
│
├── 4. 环境节点
│   ├── 光源 (DirectionalLight, PointLight, SpotLight)
│   ├── 地面 (Floor)
│   └── 天空盒 (Background)
│
├── 5. 物体节点
│   ├── 静态物体 (Solid, Transform)
│   ├── 可移动物体 (带 Physics 的 Solid)
│   └── 机器人 (Robot, Supervisor)
│
└── 6. 注释
    └── # 这是注释
```

---

#### 五、三者的关系图

```
┌─────────────────────────────────────────────────────────────┐
│                    世界文件 (.wbt)                           │
│                   ─────────────────                          │
│   存储格式：纯文本 (VRML 语法)                                │
│   作用：持久化保存整个仿真世界                                 │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                  场景树 (Scene Tree)                 │   │
│   │                 ──────────────────                   │   │
│   │   作用：组织和管理所有节点的层级结构                    │   │
│   │                                                     │   │
│   │   WorldInfo ─────────────────────┐                  │   │
│   │   Viewpoint                      │                  │   │
│   │   Background                     │ 根级节点          │   │
│   │   DirectionalLight               │                  │   │
│   │   Floor ─────────────────────────┘                  │   │
│   │       │                                             │   │
│   │   Robot "my_robot" ◄── 节点 (Node)                  │   │
│   │       ├── Shape      ◄── 子节点                     │   │
│   │       │   ├── appearance                            │   │
│   │       │   └── geometry                              │   │
│   │       ├── HingeJoint ◄── 子节点                     │   │
│   │       │   └── Solid (wheel)                         │   │
│   │       └── Camera     ◄── 子节点                     │   │
│   │                                                     │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘

         │ 保存                              │ 加载
         ▼                                  ▼
┌──────────────────┐              ┌──────────────────┐
│  my_world.wbt    │   ◄─────►   │  Webots 仿真器    │
│  (磁盘文件)       │              │  (内存中运行)     │
└──────────────────┘              └──────────────────┘
```

---

#### 六、完整示例：一个简单的机器人世界

##### 6.1 文件：`simple_robot_world.wbt`

```vrml
#VRML_SIM R2023b utf8

# ========== 外部 PROTO 引用 ==========
EXTERNPROTO "https://raw.githubusercontent.com/cyberbotics/webots/R2023b/projects/objects/floors/protos/RectangleArena.proto"

# ========== 世界全局设置 ==========
WorldInfo {
  title "Simple Robot World"
  info ["A simple demonstration world"]
  basicTimeStep 32                    # 仿真步长 32ms
  gravity 9.81                        # 重力加速度
  coordinateSystem "NUE"              # 北-上-东坐标系
}

# ========== 视角设置 ==========
Viewpoint {
  orientation -0.3 0.9 0.2 1.0        # 视角旋转
  position 2 1.5 3                    # 相机位置
  follow "my_robot"                   # 跟随机器人
}

# ========== 背景 ==========
Background {
  skyColor [0.15 0.45 0.8]            # 天空蓝色
}

# ========== 光源 ==========
DirectionalLight {
  ambientIntensity 0.5
  direction -0.5 -1 -0.3
  intensity 0.8
  castShadows TRUE
}

# ========== 地面/竞技场 ==========
RectangleArena {
  floorSize 4 4                       # 4x4 米
  wallHeight 0.3
}

# ========== 障碍物 ==========
DEF OBSTACLE Solid {
  translation 1 0.25 0                # 位置
  children [
    Shape {
      appearance DEF RED_PLASTIC PBRAppearance {
        baseColor 0.8 0.1 0.1
        metalness 0
        roughness 0.5
      }
      geometry Box { size 0.5 0.5 0.5 }
    }
  ]
  name "obstacle"
  boundingObject Box { size 0.5 0.5 0.5 }
  physics Physics {
    density 500
  }
}

# ========== 机器人 ==========
Robot {
  translation 0 0.04 0                # 初始位置
  rotation 0 1 0 0                    # 初始朝向
  
  children [
    # --- 机器人本体 ---
    DEF BODY Shape {
      appearance PBRAppearance {
        baseColor 0.2 0.2 0.8         # 蓝色
        metalness 0.3
      }
      geometry Cylinder {
        height 0.08
        radius 0.1
      }
    }
    
    # --- 左轮关节 ---
    HingeJoint {
      jointParameters HingeJointParameters {
        axis 0 0 1                    # 旋转轴
        anchor -0.1 0 0               # 锚点
      }
      device [
        RotationalMotor {
          name "left_motor"
          maxVelocity 10
        }
      ]
      endPoint Solid {
        translation -0.1 0 0
        rotation 1 0 0 1.5708
        children [
          Shape {
            appearance PBRAppearance { baseColor 0.1 0.1 0.1 }
            geometry Cylinder {
              height 0.02
              radius 0.04
            }
          }
        ]
        name "left_wheel"
        boundingObject Cylinder {
          height 0.02
          radius 0.04
        }
        physics Physics { density 1000 }
      }
    }
    
    # --- 右轮关节 ---
    HingeJoint {
      jointParameters HingeJointParameters {
        axis 0 0 1
        anchor 0.1 0 0
      }
      device [
        RotationalMotor {
          name "right_motor"
          maxVelocity 10
        }
      ]
      endPoint Solid {
        translation 0.1 0 0
        rotation 1 0 0 1.5708
        children [
          Shape {
            appearance PBRAppearance { baseColor 0.1 0.1 0.1 }
            geometry Cylinder {
              height 0.02
              radius 0.04
            }
          }
        ]
        name "right_wheel"
        boundingObject Cylinder {
          height 0.02
          radius 0.04
        }
        physics Physics { density 1000 }
      }
    }
    
    # --- 距离传感器 ---
    DistanceSensor {
      translation 0 0.02 0.08
      rotation 0 1 0 0
      name "front_sensor"
      lookupTable [0 0 0, 1 1000 0]   # 量程 0-1米
    }
    
    # --- 摄像头 ---
    Camera {
      translation 0 0.05 0.1
      name "camera"
      width 128
      height 128
    }
  ]
  
  name "my_robot"
  boundingObject Cylinder {
    height 0.08
    radius 0.1
  }
  physics Physics {
    density 1000
  }
  controller "robot_controller"       # 控制器程序名
}
```

##### 6.2 对应的场景树结构

```
📁 simple_robot_world.wbt
│
├── WorldInfo
├── Viewpoint
├── Background
├── DirectionalLight
├── RectangleArena (PROTO)
│
├── DEF OBSTACLE Solid
│   └── Shape
│       ├── DEF RED_PLASTIC PBRAppearance
│       └── Box geometry
│
└── Robot "my_robot"
    ├── DEF BODY Shape (机器人本体)
    │   ├── PBRAppearance
    │   └── Cylinder geometry
    │
    ├── HingeJoint (左轮)
    │   ├── HingeJointParameters
    │   ├── RotationalMotor "left_motor"
    │   └── Solid "left_wheel"
    │       └── Shape (轮子外观)
    │
    ├── HingeJoint (右轮)
    │   ├── HingeJointParameters
    │   ├── RotationalMotor "right_motor"
    │   └── Solid "right_wheel"
    │       └── Shape (轮子外观)
    │
    ├── DistanceSensor "front_sensor"
    └── Camera "camera"
```

---

#### 七、总结

| 概念 | 定义 | 关键点 |
|------|------|--------|
| **节点** | 仿真世界的基本单元 | 有类型、字段、可嵌套 |
| **场景树** | 节点的层级组织 | 父子关系、坐标继承 |
| **世界文件** | 序列化的场景描述 | VRML 语法、纯文本、可版本控制 |

**关系**：世界文件 (.wbt) **存储**场景树，场景树**组织**节点，节点**构成**仿真世界。
  

## PROTO 文件详解

#### 什么是 PROTO

**PROTO (Prototype)** 是 Webots 中的**模板/原型机制**，类似于面向对象编程中的"类"概念。它允许你定义可重用的自定义节点，封装复杂结构。比如 webots 里的原生定义的机器人的结构通常不满足我们的需求，因此需要自行编写 PROTO 文件来进行自定义机器人。

#### PROTO vs 普通节点

| 特性 | 普通节点 | PROTO |
|------|----------|-------|
| 复用性 | 每次都要重写 | 定义一次，多处使用 |
| 参数化 | 固定值 | 可通过字段自定义 |
| 封装性 | 结构暴露 | 内部细节隐藏 |
| 维护性 | 修改麻烦 | 改一处，处处生效 |

#### PROTO 文件结构

```vrml
#VRML_SIM R2025a utf8
# 文件头：版本标识

# 文档注释（可选）
# license: Apache License 2.0
# documentation url: https://...
# tags: robot, mobile
# keywords: differential drive

# PROTO 定义开始
PROTO MyRobot [
  # ===== 字段定义（参数列表）=====
  field SFVec3f    translation    0 0 0       # 位置
  field SFRotation rotation       0 1 0 0     # 旋转
  field SFString   name           "my_robot"  # 名称
  field SFString   controller     "default"   # 控制器
  field SFFloat    wheelRadius    0.05        # 轮子半径（可自定义）
  field SFFloat    bodyLength     0.2         # 车身长度
  field SFColor    bodyColor      0.2 0.2 0.8 # 车身颜色
]
{
  # ===== PROTO 主体（节点结构）=====
  Robot {
    translation IS translation    # IS 关键字绑定外部字段
    rotation IS rotation
    name IS name
    controller IS controller
    
    children [
      # 使用参数化字段
      Shape {
        appearance PBRAppearance {
          baseColor IS bodyColor  # 绑定颜色参数
        }
        geometry Box {
          size IS bodyLength 0.1 0.15  # 使用长度参数
        }
      }
      # ... 更多结构
    ]
  }
}
```

#### PROTO 字段类型

```
字段类型前缀:
├── SF (Single Field) - 单值
│   ├── SFBool      # 布尔值: TRUE/FALSE
│   ├── SFInt32     # 整数: 42
│   ├── SFFloat     # 浮点数: 3.14
│   ├── SFVec2f     # 2D向量: 1.0 2.0
│   ├── SFVec3f     # 3D向量: 1.0 2.0 3.0
│   ├── SFRotation  # 旋转: 0 1 0 1.57 (轴+角度)
│   ├── SFColor     # RGB颜色: 1.0 0.5 0.0
│   ├── SFString    # 字符串: "hello"
│   └── SFNode      # 节点引用
│
└── MF (Multiple Field) - 多值/数组
    ├── MFBool      # [TRUE, FALSE, TRUE]
    ├── MFInt32     # [1, 2, 3, 4]
    ├── MFFloat     # [1.0, 2.0, 3.0]
    ├── MFVec3f     # [0 0 0, 1 1 1, 2 2 2]
    ├── MFString    # ["a", "b", "c"]
    └── MFNode      # 节点数组
```

#### PROTO 使用方式

**方式1：在 .wbt 文件中引用**
```vrml
#VRML_SIM R2023b utf8

# 引用外部 PROTO（URL 或相对路径）
EXTERNPROTO "../protos/MyRobot.proto"
EXTERNPROTO "https://raw.githubusercontent.com/.../MyRobot.proto"

# 使用 PROTO（像使用普通节点一样）
MyRobot {
  translation 1 0 0
  name "robot_1"
  wheelRadius 0.06      # 自定义轮子半径
  bodyColor 1 0 0       # 红色车身
}

MyRobot {
  translation -1 0 0
  name "robot_2"
  wheelRadius 0.04      # 不同的轮子半径
  bodyColor 0 1 0       # 绿色车身
}
```
**方式2：手动添加**
在 webots 的可视化界面中的场景树栏手动添加节点，保存后会自动添加到 .wbt 世界文件中。要求编写的PROTO文件必须放在项目目录下的 `proto/` 文件夹中。

---

## 铰链/关节 (Joint) 详解

#### 关节的作用

关节连接两个 `Solid`，定义它们之间的**运动关系**。

```
┌─────────┐     HingeJoint     ┌─────────┐
│  Solid  │◄──────────────────►│  Solid  │
│ (父节点) │     可相对旋转      │(endPoint)│
└─────────┘                    └─────────┘
  机器人本体                       轮子
```

#### 关节类型对比

| 关节类型 | 自由度 | 运动方式 | 典型应用 |
|----------|--------|----------|----------|
| **HingeJoint** | 1 DOF | 绕单轴旋转 | 轮子、舵机、门 |
| **Hinge2Joint** | 2 DOF | 绕两轴旋转 | 转向轮（转向+滚动） |
| **SliderJoint** | 1 DOF | 沿轴平移 | 升降机、伸缩臂 |
| **BallJoint** | 3 DOF | 任意方向旋转 | 人体关节、万向节 |

#### HingeJoint 详细结构

```vrml
HingeJoint {
  # ===== 关节参数 =====
  jointParameters HingeJointParameters {
    position 0                    # 当前位置(弧度)，仿真开始时的初始角度
    axis 0 0 1                    # 旋转轴方向 [x, y, z]
    anchor 0 0 0                  # 旋转锚点（相对于父节点）
    
    # 物理约束（可选）
    minStop -1.57                 # 最小角度限制（弧度）
    maxStop 1.57                  # 最大角度限制
    springConstant 0              # 弹簧刚度
    dampingConstant 0             # 阻尼系数
    staticFriction 0              # 静摩擦
  }
  
  # ===== 设备列表 =====
  device [
    RotationalMotor {             # 电机（驱动关节）
      name "motor1"
      maxVelocity 10              # 最大角速度 rad/s
      maxTorque 1.5               # 最大扭矩 N·m
      minPosition -3.14           # 位置下限
      maxPosition 3.14            # 位置上限
      acceleration 5              # 加速度限制
    }
    PositionSensor {              # 位置传感器（编码器）
      name "encoder1"
      resolution 0.001            # 分辨率
    }
    Brake {                       # 制动器
      name "brake1"
    }
  ]
  
  # ===== 子节点（被连接的物体）=====
  endPoint Solid {                # 关节连接的子刚体
    translation 0 0 0.1           # 相对于锚点的位置
    rotation 1 0 0 1.5708         # 初始旋转
    children [
      Shape { ... }               # 外观
    ]
    name "wheel"
    boundingObject Cylinder { ... }
    physics Physics { ... }
  }
  
  # 或者使用 DEF/USE 引用
  # endPoint DEF WHEEL Solid { ... }
}
```

#### SliderJoint (滑动关节)

```vrml
SliderJoint {
  jointParameters JointParameters {
    position 0                    # 当前位置(米)
    axis 0 1 0                    # 滑动方向（Y轴向上）
    minStop -0.5                  # 最小位置
    maxStop 0.5                   # 最大位置
  }
  device [
    LinearMotor {                 # 直线电机
      name "lift_motor"
      maxVelocity 0.5             # 最大速度 m/s
      maxForce 100                # 最大推力 N
    }
    PositionSensor {
      name "lift_sensor"
    }
  ]
  endPoint Solid {
    # 可上下移动的平台
    children [ Shape { ... } ]
    physics Physics { }
  }
}
```

#### 旋转轴 (axis) 详解

```
axis 字段定义旋转/滑动的方向:

      Y (up)
      │
      │    axis [0, 1, 0] = 绕Y轴旋转（如转盘）
      │
      │         Z (forward)
      │        /
      │       /
      │      /  axis [0, 0, 1] = 绕Z轴旋转（如轮子侧向转动）
      │     /
      └────────── X (right)
      
      axis [1, 0, 0] = 绕X轴旋转（如轮子前进方向转动）
```

---

## 机器人节点 (Robot Node) 详解

#### Robot 节点的特殊性

`Robot` 是 Webots 中最重要的节点类型，它：
- 可以运行**控制器程序**（Python/C++/Java等）
- 可以包含**传感器和执行器**设备
- 具有**物理仿真**能力

#### Robot 节点结构

```vrml
Robot {
  # ===== 基础字段（继承自 Solid）=====
  translation 0 0.1 0              # 位置 [x, y, z]
  rotation 0 1 0 0                 # 旋转 [轴x, 轴y, 轴z, 角度(弧度)]
  scale 1 1 1                      # 缩放
  
  children [
    # 子节点列表：形状、传感器、关节等
  ]
  
  name "my_robot"                  # 机器人名称（控制器中用于识别）
  model "MyRobotModel"             # 模型名称
  description "A simple robot"     # 描述
  
  boundingObject Group {           # 碰撞边界
    children [ ... ]
  }
  
  physics Physics {                # 物理属性
    density 1000                   # 密度 kg/m³
    mass 2.5                       # 或直接指定质量 kg
    centerOfMass [0 0 0]           # 质心位置
  }
  
  # ===== Robot 特有字段 =====
  controller "robot_controller"    # 控制器程序名
  controllerArgs ["--arg1"]        # 控制器参数
  customData "user_data"           # 自定义数据
  supervisor FALSE                 # 是否为超级控制器
  synchronization TRUE             # 是否同步仿真
  battery [1000, 100]              # 电池 [最大容量, 当前电量]
  cpuConsumption 10                # CPU 能耗
  selfCollision FALSE              # 是否检测自碰撞
  window "robot_window"            # 自定义窗口
}
```

#### 2.3 Robot 与 Solid 的区别

```
           Solid                              Robot
          ───────                            ───────
┌─────────────────────┐           ┌─────────────────────┐
│ • 刚体物理仿真       │           │ • 刚体物理仿真       │
│ • 可碰撞             │           │ • 可碰撞             │
│ • 可有质量           │           │ • 可有质量           │
│                     │           │                     │
│ ✗ 无控制器          │           │ ✓ 可运行控制器       │
│ ✗ 无设备            │           │ ✓ 可包含传感器/电机  │
│ ✗ 被动物体          │           │ ✓ 主动智能体         │
└─────────────────────┘           └─────────────────────┘
      (障碍物、地形)                    (机器人、车辆)
```

#### 2.4 Robot 子节点中可包含的设备

```
Robot children 可包含的设备节点:
│
├── 传感器 (Sensors)
│   ├── Camera              # 摄像头
│   ├── Lidar               # 激光雷达
│   ├── RangeFinder         # 深度相机
│   ├── DistanceSensor      # 距离/超声波传感器
│   ├── TouchSensor         # 触碰传感器
│   ├── LightSensor         # 光传感器
│   ├── GPS                 # 全球定位
│   ├── Compass             # 罗盘
│   ├── Gyro                # 陀螺仪
│   ├── Accelerometer       # 加速度计
│   ├── InertialUnit        # IMU（惯性测量单元）
│   ├── Receiver            # 无线接收器
│   └── PositionSensor      # 位置/编码器
│
├── 执行器 (Actuators)
│   ├── RotationalMotor     # 旋转电机
│   ├── LinearMotor         # 直线电机
│   ├── Brake               # 制动器
│   ├── LED                 # LED灯
│   ├── Display             # 显示屏
│   ├── Speaker             # 扬声器
│   ├── Emitter             # 无线发射器
│   ├── Pen                 # 画笔
│   └── Propeller           # 螺旋桨
│
└── 机械结构
    ├── HingeJoint          # 铰链关节（单自由度旋转）
    ├── Hinge2Joint         # 双铰链关节
    ├── SliderJoint         # 滑动关节（直线运动）
    ├── BallJoint           # 球关节（3自由度）
    └── Shape               # 外观形状
```

---

#### 完整示例：差速驱动机器人

##### PROTO 文件：`DiffDriveRobot.proto`

```vrml
#VRML_SIM R2023b utf8
# 差速驱动机器人 PROTO
# 文档: 一个简单的两轮差速驱动机器人

PROTO DiffDriveRobot [
  # ===== 外部可配置参数 =====
  field SFVec3f    translation      0 0 0
  field SFRotation rotation         0 1 0 0
  field SFString   name             "diff_robot"
  field SFString   controller       "diff_drive_controller"
  field SFFloat    bodyRadius       0.1           # 车身半径
  field SFFloat    bodyHeight       0.05          # 车身高度
  field SFFloat    wheelRadius      0.04          # 轮子半径
  field SFFloat    wheelWidth       0.02          # 轮子宽度
  field SFFloat    axleLength       0.15          # 轴距（两轮间距）
  field SFColor    bodyColor        0.2 0.4 0.8   # 车身颜色
  field SFColor    wheelColor       0.1 0.1 0.1   # 轮子颜色
  field SFFloat    maxSpeed         10            # 最大轮速 rad/s
]
{
  Robot {
    translation IS translation
    rotation IS rotation
    name IS name
    controller IS controller
    
    children [
      # ========== 机器人本体 ==========
      DEF BODY_SHAPE Shape {
        appearance PBRAppearance {
          baseColor IS bodyColor
          metalness 0.3
          roughness 0.5
        }
        geometry Cylinder {
          height IS bodyHeight
          radius IS bodyRadius
        }
      }
      
      # ========== 前方指示器（方向标记）==========
      Transform {
        translation 0.08 0.03 0
        children [
          Shape {
            appearance PBRAppearance {
              baseColor 1 0 0
              emissiveColor 0.5 0 0
            }
            geometry Sphere { radius 0.015 }
          }
        ]
      }
      
      # ========== 左轮 HingeJoint ==========
      HingeJoint {
        jointParameters HingeJointParameters {
          axis 0 0 1                          # 绕Z轴旋转
          anchor 0 0 0.075                    # 锚点在车身左侧
        }
        device [
          RotationalMotor {
            name "left_wheel_motor"
            maxVelocity IS maxSpeed
            maxTorque 10
          }
          PositionSensor {
            name "left_wheel_sensor"
          }
        ]
        endPoint Solid {
          translation 0 0 0.075               # 轮子位置
          rotation 1 0 0 1.5708               # 旋转90度使轮子立起来
          children [
            DEF WHEEL_SHAPE Shape {
              appearance PBRAppearance {
                baseColor IS wheelColor
                metalness 0.5
                roughness 0.3
              }
              geometry Cylinder {
                height IS wheelWidth
                radius IS wheelRadius
              }
            }
          ]
          name "left_wheel"
          boundingObject USE WHEEL_SHAPE
          physics Physics {
            density 1000
          }
        }
      }
      
      # ========== 右轮 HingeJoint ==========
      HingeJoint {
        jointParameters HingeJointParameters {
          axis 0 0 1                          # 绕Z轴旋转
          anchor 0 0 -0.075                   # 锚点在车身右侧
        }
        device [
          RotationalMotor {
            name "right_wheel_motor"
            maxVelocity IS maxSpeed
            maxTorque 10
          }
          PositionSensor {
            name "right_wheel_sensor"
          }
        ]
        endPoint Solid {
          translation 0 0 -0.075              # 轮子位置
          rotation 1 0 0 1.5708
          children [
            USE WHEEL_SHAPE                   # 重用左轮形状定义
          ]
          name "right_wheel"
          boundingObject Cylinder {
            height 0.02
            radius 0.04
          }
          physics Physics {
            density 1000
          }
        }
      }
      
      # ========== 前支撑轮（万向轮/球轮）==========
      BallJoint {
        jointParameters BallJointParameters {
          anchor 0.07 -0.02 0                 # 车身前方底部
        }
        endPoint Solid {
          translation 0.07 -0.02 0
          children [
            Shape {
              appearance PBRAppearance {
                baseColor 0.5 0.5 0.5
              }
              geometry Sphere { radius 0.02 }
            }
          ]
          name "caster_wheel"
          boundingObject Sphere { radius 0.02 }
          physics Physics {
            density 500
          }
        }
      }
      
      # ========== 传感器：距离传感器阵列 ==========
      # 前方传感器
      DistanceSensor {
        translation 0.1 0.02 0
        rotation 0 1 0 0                      # 朝前
        name "ds_front"
        lookupTable [0 0 0, 1 1000 0]
        type "sonar"
      }
      
      # 左前传感器
      DistanceSensor {
        translation 0.07 0.02 0.07
        rotation 0 1 0 0.785                  # 左前45度
        name "ds_front_left"
        lookupTable [0 0 0, 0.5 500 0]
        type "sonar"
      }
      
      # 右前传感器
      DistanceSensor {
        translation 0.07 0.02 -0.07
        rotation 0 1 0 -0.785                 # 右前45度
        name "ds_front_right"
        lookupTable [0 0 0, 0.5 500 0]
        type "sonar"
      }
      
      # ========== 摄像头 ==========
      Camera {
        translation 0.09 0.04 0
        rotation 0 1 0 0
        name "camera"
        width 320
        height 240
        fieldOfView 1.0
      }
      
      # ========== IMU ==========
      InertialUnit {
        name "imu"
      }
      
      # ========== GPS ==========
      GPS {
        name "gps"
      }
    ]
    
    # ===== 碰撞边界 =====
    boundingObject Transform {
      translation 0 0 0
      children [
        Cylinder {
          height 0.05
          radius 0.1
        }
      ]
    }
    
    # ===== 物理属性 =====
    physics Physics {
      density -1              # 使用 mass 而非 density
      mass 1.5                # 总质量 1.5 kg
      centerOfMass [0 0 0]
    }
  }
}
```
##### 世界文件：robot_world.wbt
```vrml
#VRML_SIM R2023b utf8

# 引用 PROTO
EXTERNPROTO "protos/DiffDriveRobot.proto"
EXTERNPROTO "https://raw.githubusercontent.com/cyberbotics/webots/R2023b/projects/objects/floors/protos/RectangleArena.proto"
EXTERNPROTO "https://raw.githubusercontent.com/cyberbotics/webots/R2023b/projects/objects/obstacles/protos/OilBarrel.proto"

# 世界设置
WorldInfo {
  title "Differential Drive Robot Demo"
  basicTimeStep 16
  gravity 9.81
  contactProperties [
    ContactProperties {
      material1 "wheel"
      material2 "floor"
      coulombFriction [1.0]
    }
  ]
}

Viewpoint {
  orientation -0.3 0.9 0.2 0.8
  position 1 0.8 1.5
  follow "robot_1"
}

Background {
  skyColor [0.1 0.3 0.6]
}

DirectionalLight {
  direction -0.5 -1 -0.3
  intensity 0.8
  castShadows TRUE
}

# 地面
RectangleArena {
  floorSize 5 5
  floorAppearance PBRAppearance {
    baseColor 0.8 0.8 0.8
    roughness 1
  }
  wallHeight 0.3
}

# 障碍物
OilBarrel {
  translation 1.5 0.44 0
  name "barrel_1"
}

OilBarrel {
  translation -1 0.44 1
  name "barrel_2"
}

# ===== 使用 PROTO 创建机器人 =====
DiffDriveRobot {
  translation 0 0.05 0
  rotation 0 1 0 0
  name "robot_1"
  controller "diff_drive_controller"
  bodyColor 0.2 0.4 0.8         # 蓝色
  wheelRadius 0.04
  maxSpeed 15
}

# 可以轻松创建多个机器人
DiffDriveRobot {
  translation -2 0.05 -1
  rotation 0 1 0 1.57
  name "robot_2"
  controller "diff_drive_controller"
  bodyColor 0.8 0.2 0.2         # 红色
  wheelRadius 0.05              # 不同的轮子大小
  maxSpeed 10
}
```

#### 控制器：diff_drive_controller.py
``` vrml
"""差速驱动机器人控制器"""
from controller import Robot

# 创建机器人实例
robot = Robot()

# 获取仿真时间步长
timestep = int(robot.getBasicTimeStep())

# ===== 获取电机设备 =====
left_motor = robot.getDevice('left_wheel_motor')
right_motor = robot.getDevice('right_wheel_motor')

# 设置电机为速度控制模式（position = infinity）
left_motor.setPosition(float('inf'))
right_motor.setPosition(float('inf'))

# 初始速度设为0
left_motor.setVelocity(0)
right_motor.setVelocity(0)

# ===== 获取传感器 =====
ds_front = robot.getDevice('ds_front')
ds_front_left = robot.getDevice('ds_front_left')
ds_front_right = robot.getDevice('ds_front_right')

# 启用传感器
ds_front.enable(timestep)
ds_front_left.enable(timestep)
ds_front_right.enable(timestep)

# 获取编码器
left_encoder = robot.getDevice('left_wheel_sensor')
right_encoder = robot.getDevice('right_wheel_sensor')
left_encoder.enable(timestep)
right_encoder.enable(timestep)

# ===== 主循环 =====
MAX_SPEED = 10.0
OBSTACLE_THRESHOLD = 300  # 障碍物距离阈值

while robot.step(timestep) != -1:
    # 读取传感器值
    front_dist = ds_front.getValue()
    left_dist = ds_front_left.getValue()
    right_dist = ds_front_right.getValue()
    
    # 简单避障逻辑
    left_speed = MAX_SPEED
    right_speed = MAX_SPEED
    
    if front_dist > OBSTACLE_THRESHOLD:
        # 前方有障碍，后退转弯
        left_speed = -MAX_SPEED * 0.5
        right_speed = MAX_SPEED * 0.5
    elif left_dist > OBSTACLE_THRESHOLD:
        # 左前有障碍，右转
        left_speed = MAX_SPEED
        right_speed = MAX_SPEED * 0.3
    elif right_dist > OBSTACLE_THRESHOLD:
        # 右前有障碍，左转
        left_speed = MAX_SPEED * 0.3
        right_speed = MAX_SPEED
    
    # 设置电机速度
    left_motor.setVelocity(left_speed)
    right_motor.setVelocity(right_speed)
    
    # 调试输出
    # print(f"Sensors: F={front_dist:.0f}, L={left_dist:.0f}, R={right_dist:.0f}")
```  